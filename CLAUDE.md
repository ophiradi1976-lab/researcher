# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

Agentic research pipeline that generates senior-engineer-level documentation for developer tools. Given a tool name (e.g., `dbt`), it runs web searches, synthesizes findings with Claude, generates 5 example walkthroughs, and writes a complete MkDocs documentation section.

## Commands

**Setup:**
```bash
pip install -r requirements.txt
# Set API keys via direnv (.envrc) or export manually
```

**Run the pipeline:**
```bash
python main.py <tool>                        # research a tool (prompts if docs already exist)
python main.py <tool> --force                # overwrite existing docs
python main.py <tool> --provider brave       # use a specific search provider
```

**Preview and deploy docs:**
```bash
mkdocs serve        # local preview at http://127.0.0.1:8000
mkdocs gh-deploy    # deploy to GitHub Pages
```

## Architecture

The pipeline runs 5 sequential stages, each implemented as an agent class:

1. **`agents/researcher.py`** — Runs 7 hardcoded search queries per tool (getting started, advanced usage, production tips, GitHub, docs, alternatives, performance), deduplicates by URL, returns top 10 sources.

2. **`agents/synthesizer.py`** — Sends all source content to Claude; produces a structured Markdown overview with Core Concepts, Use Cases, Senior/Staff Highlights, Tradeoffs, and Quick Reference sections.

3. **`agents/example_generator.py`** — Two-stage: first asks Claude to plan 5 examples as JSON (foundational → advanced/production), then writes each as a full walkthrough (Overview, Prerequisites, Implementation, Running It, Gotchas, Next Steps).

4. **`agents/writer.py`** — Creates `docs/<tool>/` with `index.md`, `sources.md`, and `examples/0N_*.md`. Also regenerates the top-level `docs/index.md`.

5. **`mkdocs_config.py`** — Scans `docs/` and rewrites `mkdocs.yml` navigation to include the new tool.

## Search Providers

Pluggable via `search_providers/`. All implement `BaseSearchProvider.search(query, max_results)` returning `[{title, url, description, content}]`.

- **Tavily** (recommended) — returns full page content via `include_raw_content: True`
- **Brave** — free tier, returns metadata only (no page content)
- **Serper** — Google results via third-party proxy, returns snippets only

Provider is selected by `--provider` flag; defaults to whichever key is set. Add a new provider by subclassing `BaseSearchProvider` and registering it in `search_providers/__init__.py`.

## Environment Variables

Required (set in `.envrc` or shell):
- One search key: `TAVILY_API_KEY`, `BRAVE_API_KEY`, or `SERPER_API_KEY`
- Claude API key — **do not use `ANTHROPIC_API_KEY`**; use a different key name as configured for this project

## Output Structure

```
docs/<tool>/
  index.md          # synthesized overview
  sources.md        # annotated research sources
  examples/
    01_*.md … 05_*.md
```
