---
title: Database Migration Generation with Approval Gates
---

# Database Migration Generation with Approval Gates

## Overview

Database migrations are among the highest-risk automated operations in any system. A malformed migration can drop columns, truncate tables, or corrupt referential integrity—often irreversibly in production. This walkthrough demonstrates how to configure Claude Code to generate Prisma migrations from natural language schema change requests while enforcing strict approval workflows that prevent destructive operations from auto-executing.

The pattern establishes three safety layers:
1. **Dry-run validation** - Claude generates and analyzes migrations without applying them
2. **Destructive operation detection** - Automatic flagging of DROP, TRUNCATE, and data-loss operations
3. **Human approval gates** - Mandatory review checkpoints before any database modification

This approach lets teams move fast on additive schema changes (new tables, nullable columns) while forcing deliberate review on anything that could lose data.

## Prerequisites

**Required installations:**
- Node.js 18+ 
- PostgreSQL 14+ (local instance or Docker)
- Claude Code CLI (`npm install -g @anthropic-ai/claude-code`)
- Prisma CLI (`npm install -g prisma`)

**Required knowledge:**
- Familiarity with Prisma schema syntax
- Understanding of database migration concepts
- Basic shell scripting

**Environment setup:**
```bash
# Verify installations
node --version    # Should be 18+
psql --version    # Should be 14+
claude --version  # Claude Code CLI
npx prisma --version
```

**Database connection:**
```bash
# Set up a local PostgreSQL database for this walkthrough
createdb migration_demo

# Export connection string
export DATABASE_URL="postgresql://localhost:5432/migration_demo"
```

## Implementation

### Step 1: Initialize the Project Structure

We start by creating a project structure that separates concerns: schema definitions, generated migrations, and approval workflow state.

```bash
mkdir -p migration-gates/{prisma,scripts,approvals,.claude}
cd migration-gates
npm init -y
npm install prisma @prisma/client
npm install --save-dev typescript @types/node ts-node
```

Create the initial Prisma schema representing a SaaS application's core tables:

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Core user management - established table structure
model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String    @map("password_hash")
  displayName   String?   @map("display_name")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  // Relations
  organizationMemberships OrganizationMember[]
  auditLogs               AuditLog[]
  
  @@map("users")
}

// Multi-tenant organization support
model Organization {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  plan      String   @default("free") // free, pro, enterprise
  createdAt DateTime @default(now()) @map("created_at")
  
  members   OrganizationMember[]
  projects  Project[]
  
  @@map("organizations")
}

model OrganizationMember {
  id             String       @id @default(uuid())
  userId         String       @map("user_id")
  organizationId String       @map("organization_id")
  role           String       @default("member") // owner, admin, member
  joinedAt       DateTime     @default(now()) @map("joined_at")
  
  user           User         @relation(fields: [userId], references: [id])
  organization   Organization @relation(fields: [organizationId], references: [id])
  
  @@unique([userId, organizationId])
  @@map("organization_members")
}

model Project {
  id             String       @id @default(uuid())
  organizationId String       @map("organization_id")
  name           String
  description    String?
  isArchived     Boolean      @default(false) @map("is_archived")
  createdAt      DateTime     @default(now()) @map("created_at")
  
  organization   Organization @relation(fields: [organizationId], references: [id])
  
  @@map("projects")
}

// Security audit trail
model AuditLog {
  id        String   @id @default(uuid())
  userId    String?  @map("user_id")
  action    String   // user.login, project.create, etc.
  resource  String   // Table/entity affected
  resourceId String? @map("resource_id")
  metadata  Json?    // Additional context
  ipAddress String?  @map("ip_address")
  createdAt DateTime @default(now()) @map("created_at")
  
  user      User?    @relation(fields: [userId], references: [id])
  
  @@index([userId, createdAt])
  @@index([resource, resourceId])
  @@map("audit_logs")
}
```

Initialize the database with this baseline:

```bash
npx prisma migrate dev --name initial_schema
```

### Step 2: Configure Claude Code with Migration Permissions

The Claude Code settings file defines what operations Claude can perform autonomously versus what requires human approval. This is the critical safety configuration.

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      // Claude can read any project files to understand context
      "Read(**)",
      
      // Claude can modify Prisma schema files - this is the input to migration generation
      "Edit(prisma/schema.prisma)",
      
      // Claude can create new migration analysis files
      "Write(approvals/**)",
      
      // Claude can run Prisma commands in dry-run/diff mode only
      "Shell(npx prisma migrate diff *)",
      "Shell(npx prisma format)",
      "Shell(npx prisma validate)",
      
      // Claude can run our analysis scripts
      "Shell(node scripts/analyze-migration.js *)",
      "Shell(./scripts/check-destructive.sh *)"
    ],
    "deny": [
      // NEVER allow direct migration execution - this is the core safety gate
      "Shell(npx prisma migrate dev *)",
      "Shell(npx prisma migrate deploy *)",
      "Shell(npx prisma migrate reset *)",
      "Shell(npx prisma db push *)",
      
      // Prevent direct database access that could bypass migrations
      "Shell(psql *)",
      "Shell(pg_dump *)",
      
      // Prevent modification of safety infrastructure
      "Edit(.claude/**)",
      "Edit(scripts/**)"
    ]
  },
  "settings": {
    "model": "claude-sonnet-4-20250514",
    "maxTokens": 8192
  }
}
```

The key insight here: Claude has full permission to *generate* and *analyze* migrations, but zero permission to *apply* them. This creates an inherent approval gate—the human must run the final migration command.

### Step 3: Create the Destructive Operation Detector

This script parses Prisma's migration diff output and categorizes operations by risk level. It's intentionally conservative—false positives (flagging safe operations) are acceptable; false negatives (missing dangerous ones) are not.

```javascript
// scripts/analyze-migration.js

const fs = require('fs');
const path = require('path');

// Risk classification for different SQL operations
const RISK_LEVELS = {
  CRITICAL: {
    patterns: [
      /DROP\s+TABLE/i,
      /TRUNCATE/i,
      /DELETE\s+FROM\s+\w+\s*;/i,  // DELETE without WHERE
      /DROP\s+DATABASE/i,
    ],
    description: 'Data loss - requires explicit approval and backup verification',
    autoApprove: false,
    requiresBackup: true,
  },
  HIGH: {
    patterns: [
      /DROP\s+COLUMN/i,
      /ALTER\s+COLUMN.*TYPE/i,      // Type changes can truncate data
      /ALTER\s+COLUMN.*NOT\s+NULL/i, // Adding NOT NULL can fail on existing data
      /DROP\s+INDEX/i,
      /DROP\s+CONSTRAINT/i,
    ],
    description: 'Potential data loss or breaking change - requires review',
    autoApprove: false,
    requiresBackup: true,
  },
  MEDIUM: {
    patterns: [
      /RENAME\s+COLUMN/i,
      /RENAME\s+TABLE/i,
      /ALTER\s+COLUMN.*DEFAULT/i,
    ],
    description: 'Schema change may affect application code',
    autoApprove: false,
    requiresBackup: false,
  },
  LOW: {
    patterns: [
      /CREATE\s+TABLE/i,
      /ADD\s+COLUMN/i,
      /CREATE\s+INDEX/i,
      /CREATE\s+UNIQUE\s+INDEX/i,
    ],
    description: 'Additive change - generally safe',
    autoApprove: true,
    requiresBackup: false,
  },
};

function analyzeMigrationSQL(sql) {
  const findings = [];
  const lines = sql.split('\n');
  
  for (let lineNum = 0; lineNum < lines.length; lineNum++) {
    const line = lines[lineNum].trim();
    if (!line || line.startsWith('--')) continue;
    
    for (const [level, config] of Object.entries(RISK_LEVELS)) {
      for (const pattern of config.patterns) {
        if (pattern.test(line)) {
          findings.push({
            level,
            line: lineNum + 1,
            statement: line.substring(0, 100) + (line.length > 100 ? '...' : ''),
            description: config.description,
            autoApprove: config.autoApprove,
            requiresBackup: config.requiresBackup,
          });
          break; // Only report first matching pattern per line
        }
      }
    }
  }
  
  return findings;
}

function generateApprovalManifest(findings, migrationName) {
  const criticalCount = findings.filter(f => f.level === 'CRITICAL').length;
  const highCount = findings.filter(f => f.level === 'HIGH').length;
  const requiresApproval = findings.some(f => !f.autoApprove);
  const requiresBackup = findings.some(f => f.requiresBackup);
  
  const manifest = {
    migrationName,
    analyzedAt: new Date().toISOString(),
    summary: {
      totalOperations: findings.length,
      criticalCount,
      highCount,
      requiresApproval,
      requiresBackup,
    },
    findings,
    approvalStatus: requiresApproval ? 'PENDING' : 'AUTO_APPROVED',
    approvedBy: requiresApproval ? null : 'system',
    approvedAt: requiresApproval ? null : new Date().toISOString(),
  };
  
  return manifest;
}

// Main execution
const migrationSQL = fs.readFileSync(process.argv[2], 'utf-8');
const migrationName = path.basename(process.argv[2], '.sql');

const findings = analyzeMigrationSQL(migrationSQL);
const manifest = generateApprovalManifest(findings, migrationName);

// Write approval manifest
const outputPath = `approvals/${migrationName}.json`;
fs.mkdirSync('approvals', { recursive: true });
fs.writeFileSync(outputPath, JSON.stringify(manifest, null, 2));

// Output summary for Claude to read
console.log('\n=== MIGRATION ANALYSIS REPORT ===\n');
console.log(`Migration: ${migrationName}`);
console.log(`Total operations: ${findings.length}`);
console.log(`Critical: ${manifest.summary.criticalCount}`);
console.log(`High risk: ${manifest.summary.highCount}`);
console.log(`Requires approval: ${manifest.summary.requiresApproval}`);
console.log(`Requires backup: ${manifest.summary.requiresBackup}`);

if (findings.length > 0) {
  console.log('\nDetailed findings:');
  findings.forEach((f, i) => {
    console.log(`\n${i + 1}. [${f.level}] Line ${f.line}`);
    console.log(`   Statement: ${f.statement}`);
    console.log(`   ${f.description}`);
  });
}

if (manifest.summary.requiresApproval) {
  console.log('\n⚠️  THIS MIGRATION REQUIRES HUMAN APPROVAL');
  console.log(`Approval manifest written to: ${outputPath}`);
  console.log('Review the manifest and run: ./scripts/approve-migration.sh ' + migrationName);
  process.exit(1); // Non-zero exit signals Claude that approval is needed
}

console.log('\n✅ Migration auto-approved (additive changes only)');
process.exit(0);
```

### Step 4: Create the Approval Workflow Scripts

These scripts manage the human approval process and ensure migrations can only be applied after explicit sign-off.

```bash
#!/bin/bash
# scripts/approve-migration.sh

set -e

MIGRATION_NAME=$1
MANIFEST_PATH="approvals/${MIGRATION_NAME}.json"

if [ -z "$MIGRATION_NAME" ]; then
    echo "Usage: ./scripts/approve-migration.sh <migration-name>"
    exit 1
fi

if [ ! -f "$MANIFEST_PATH" ]; then
    echo "Error: Approval manifest not found at $MANIFEST_PATH"
    echo "Run migration analysis first."
    exit 1
fi

# Display the manifest for review
echo "=== MIGRATION APPROVAL REQUEST ==="
echo ""
cat "$MANIFEST_PATH" | j