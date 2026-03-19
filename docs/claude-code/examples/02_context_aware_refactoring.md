---
title: Large-Scale Refactoring with Context Window Management
---

# Large-Scale Refactoring with Context Window Management

## Overview

This walkthrough demonstrates how to execute a cross-cutting refactor—replacing a deprecated ActiveRecord-style ORM pattern with a Repository pattern—across 50+ files in a Python codebase using Claude Code. The key challenge isn't the refactor itself; it's managing Claude's context window effectively when the total scope exceeds what can fit in a single session.

We'll use three techniques that make large refactors tractable:
1. **Strategic file batching**: Grouping files by dependency order and functional similarity
2. **Plan mode checkpoints**: Using Claude's plan mode to establish contracts before touching code
3. **Incremental verification**: Running targeted tests after each batch to catch regressions early

The scenario: A Django-based e-commerce platform with 67 model files still using the deprecated `Model.objects.filter()` pattern directly in service layers. We're migrating to explicit repository classes to improve testability and prepare for a potential database migration.

## Prerequisites

**Environment:**
- Python 3.11+
- Claude Code CLI installed and authenticated
- Git repository with clean working tree
- pytest configured with existing test suite

**Codebase familiarity:**
- Understanding of the current ORM usage patterns
- Knowledge of which services are most critical (we'll prioritize those)

**Project structure assumed:**
```
ecommerce/
├── models/
│   ├── __init__.py
│   ├── customer.py
│   ├── order.py
│   ├── product.py
│   └── ... (67 files)
├── services/
│   ├── checkout_service.py
│   ├── inventory_service.py
│   └── ... (43 files)
├── repositories/    # We'll create this
└── tests/
```

## Implementation

### Step 1: Generate the Refactor Inventory

Before writing any code, we need Claude to understand the scope. We'll use plan mode to analyze the codebase without making changes.

```bash
# Start Claude Code in plan mode to analyze without modifying
claude --plan "Analyze this codebase and identify all files that use direct ORM 
queries (Model.objects.*) outside of model classes themselves. Group them by:
1. Core business logic (checkout, payments, inventory)
2. Supporting services (notifications, analytics, reporting)
3. Admin/backoffice operations

For each file, note the specific query patterns used and any complex queries 
that will need special attention during refactoring."
```

Claude will produce an analysis. Save this to a tracking file:

```bash
# Claude's output gets saved as our refactor manifest
claude "Save your analysis to docs/refactor-inventory.md with a checklist 
format that we can update as we complete each file"
```

The resulting inventory might look like:

```markdown
<!-- docs/refactor-inventory.md -->
# ORM Refactor Inventory

## Batch 1: Core Business Logic (Priority: Critical)
- [ ] services/checkout_service.py - 12 query sites, includes transaction handling
- [ ] services/payment_service.py - 8 query sites, external API integration
- [ ] services/inventory_service.py - 15 query sites, complex aggregations

## Batch 2: Order Management
- [ ] services/order_service.py - 9 query sites
- [ ] services/fulfillment_service.py - 7 query sites
- [ ] services/returns_service.py - 6 query sites

## Batch 3: Customer Operations  
- [ ] services/customer_service.py - 11 query sites
- [ ] services/loyalty_service.py - 5 query sites
...
```

### Step 2: Establish the Repository Contract

Before touching any service files, we define what the repository layer looks like. This gives Claude a consistent target pattern across all batches.

```bash
claude "Create the base repository infrastructure in repositories/base.py. 
Requirements:
- Abstract base class with common CRUD operations
- Type hints using generics for the model type
- Support for both single-item and bulk operations
- Include a unit_of_work context manager for transaction handling
- Add docstrings explaining the pattern for future maintainers"
```

```python
# repositories/base.py
from abc import ABC, abstractmethod
from contextlib import contextmanager
from typing import Generic, TypeVar, List, Optional, Iterator
from django.db import models, transaction

T = TypeVar('T', bound=models.Model)


class BaseRepository(ABC, Generic[T]):
    """
    Abstract repository providing data access patterns.
    
    Repositories encapsulate query logic, making services testable
    via dependency injection and preparing for potential ORM migrations.
    
    Usage:
        class OrderRepository(BaseRepository[Order]):
            model_class = Order
            
            def find_pending_for_customer(self, customer_id: int) -> List[Order]:
                return self._filter(customer_id=customer_id, status='pending')
    """
    
    model_class: type[T]  # Subclasses must define this
    
    def get_by_id(self, entity_id: int) -> Optional[T]:
        """Retrieve single entity by primary key, or None if not found."""
        try:
            return self.model_class.objects.get(pk=entity_id)
        except self.model_class.DoesNotExist:
            return None
    
    def get_by_id_or_raise(self, entity_id: int) -> T:
        """Retrieve single entity by primary key, raising if not found."""
        return self.model_class.objects.get(pk=entity_id)
    
    def find_all(self) -> List[T]:
        """Retrieve all entities. Use sparingly on large tables."""
        return list(self.model_class.objects.all())
    
    def save(self, entity: T) -> T:
        """Persist entity changes to database."""
        entity.save()
        return entity
    
    def save_batch(self, entities: List[T]) -> List[T]:
        """Bulk save with single database round-trip where possible."""
        # bulk_update for existing, bulk_create for new
        existing = [e for e in entities if e.pk is not None]
        new = [e for e in entities if e.pk is None]
        
        if existing:
            # Determine which fields to update by inspecting the model
            field_names = [f.name for f in self.model_class._meta.fields 
                          if not f.primary_key]
            self.model_class.objects.bulk_update(existing, field_names)
        
        if new:
            self.model_class.objects.bulk_create(new)
        
        return entities
    
    def delete(self, entity: T) -> None:
        """Remove entity from database."""
        entity.delete()
    
    def _filter(self, **kwargs) -> List[T]:
        """
        Protected method for subclass query building.
        Subclasses should wrap this with domain-specific method names.
        """
        return list(self.model_class.objects.filter(**kwargs))
    
    def _filter_single(self, **kwargs) -> Optional[T]:
        """Protected method returning first match or None."""
        return self.model_class.objects.filter(**kwargs).first()
    
    @contextmanager
    def unit_of_work(self) -> Iterator[None]:
        """
        Transaction context manager for atomic operations.
        
        Usage:
            with repo.unit_of_work():
                repo.save(order)
                repo.save(payment)
                # Both committed together, or both rolled back
        """
        with transaction.atomic():
            yield
```

### Step 3: Create the First Domain Repository as a Template

We create one complete repository that Claude can reference for consistency across all future batches:

```bash
claude "Create repositories/order_repository.py based on the patterns used in 
services/order_service.py. This will serve as our reference implementation.
Include methods for:
- Finding orders by customer
- Finding orders by status with date ranges
- The complex aggregation query for order totals
Add comprehensive docstrings since this is our template."
```

```python
# repositories/order_repository.py
from datetime import datetime
from decimal import Decimal
from typing import List, Optional, Dict, Any
from django.db.models import Sum, Count, Q, F

from models.order import Order, OrderStatus
from repositories.base import BaseRepository


class OrderRepository(BaseRepository[Order]):
    """
    Repository for Order entity access.
    
    This serves as the reference implementation for the repository pattern
    migration. When creating new repositories, follow the conventions here:
    
    1. Method names should be domain-focused (find_pending_for_customer)
       not ORM-focused (filter_by_customer_id_and_status)
    2. Complex queries get their own methods with clear documentation
    3. Return types are always explicit (List[Order], not QuerySet)
    """
    
    model_class = Order
    
    def find_by_customer(
        self, 
        customer_id: int,
        include_cancelled: bool = False
    ) -> List[Order]:
        """
        Retrieve all orders for a customer.
        
        Args:
            customer_id: The customer's primary key
            include_cancelled: If False, excludes cancelled orders (default behavior
                             since cancelled orders are rarely needed in customer-facing contexts)
        """
        filters = {'customer_id': customer_id}
        if not include_cancelled:
            filters['status__ne'] = OrderStatus.CANCELLED
        
        return self._filter(**filters)
    
    def find_pending_for_customer(self, customer_id: int) -> List[Order]:
        """Get orders awaiting payment or processing for a customer."""
        return self._filter(
            customer_id=customer_id,
            status__in=[OrderStatus.PENDING_PAYMENT, OrderStatus.PROCESSING]
        )
    
    def find_by_status_in_date_range(
        self,
        status: OrderStatus,
        start_date: datetime,
        end_date: datetime
    ) -> List[Order]:
        """
        Find orders with given status created within date range.
        
        Used primarily by fulfillment batch processing and reporting.
        Dates are inclusive on both ends.
        """
        return self._filter(
            status=status,
            created_at__gte=start_date,
            created_at__lte=end_date
        )
    
    def find_requiring_fulfillment(self, warehouse_id: int) -> List[Order]:
        """
        Find orders ready for picking at a specific warehouse.
        
        Returns orders that are:
        - Payment confirmed
        - Not yet shipped
        - Have at least one item stocked at the given warehouse
        """
        # This complex query was previously duplicated in three services
        return list(
            Order.objects.filter(
                status=OrderStatus.PAYMENT_CONFIRMED,
                line_items__product__warehouse_stock__warehouse_id=warehouse_id,
                line_items__product__warehouse_stock__quantity__gt=0
            ).distinct().select_related(
                'customer'  # Prefetch to avoid N+1 in fulfillment processing
            ).prefetch_related(
                'line_items__product'
            )
        )
    
    def calculate_customer_totals(
        self,
        customer_id: int,
        since: Optional[datetime] = None
    ) -> Dict[str, Any]:
        """
        Aggregate order statistics for a customer.
        
        Returns:
            Dict with keys:
            - total_spent: Decimal sum of all completed order amounts
            - order_count: Number of completed orders
            - average_order_value: Mean order amount, or None if no orders
        """
        filters = Q(customer_id=customer_id, status=OrderStatus.COMPLETED)
        if since:
            filters &= Q(created_at__gte=since)
        
        result = Order.objects.filter(filters).aggregate(
            total_spent=Sum('total_amount'),
            order_count=Count('id')
        )
        
        total = result['total_spent'] or Decimal('0')
        count = result['order_count'] or 0
        
        return {
            'total_spent': total,
            'order_count': count,
            'average_order_value': total / count if count > 0 else None
        }
    
    def find_with_payment_issues(self) -> List[Order]:
        """
        Find orders where payment status is inconsistent.
        
        Used by the payment reconciliation job to identify orders needing
        manual review. This catches cases like:
        - Order marked paid but no payment record
        - Payment exists but order still pending
        """
        return list(
            Order.objects.filter(
                Q(status=OrderStatus.PAYMENT_CONFIRMED, payments__isnull=True) |
                Q(status=OrderStatus.PENDING_PAYMENT, payments__status='completed')
            ).select_related('customer')
        )
```

### Step 4: Refactor Services in Dependency-Ordered Batches

Now we process files in batches, using checkpoints to manage context and verify correctness incrementally.

```bash
# Create a refactor session script that we'll run for each batch
cat > scripts/refactor_batch.sh << 'EOF'
#!/bin/bash
set -e

BATCH_NAME=$1
FILES=$2  # Comma-separated list of files

echo "=== Starting refactor batch: $BATCH_NAME ==="
echo "Files: $FILES"

# Step 1: Create repositories for this batch (if not existing)
claude "For the following service files, create corresponding repository 
classes if they don't exist yet. Follow the pattern in 
repositories/order_repository.py exactly.

Files: $FILES

After creating each repository, list the methods you added."

# Step 2: Refactor the services to use repositories
claude "Refactor these service files to use dependency-injected repositories 
instead of direct ORM calls:

Files: $FILES

Requirements:
1. Add repository parameters to __init__ with defaults for production
2. Replace all Model.objects.* calls with repository method calls
3. If a needed repository method doesn't exist, add it to the repository
4. Preserve all existing behavior—this is a pure refactor