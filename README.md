# Tool Researcher

An agentic research pipeline that takes a tool name and produces senior-engineer-level documentation, ready to serve on GitHub Pages via MkDocs.

## What It Produces

For each tool, it generates:

```
docs/
  <tool>/
    index.md        ← Technical overview, use cases, senior/staff highlights
    sources.md      ← Top 5-10 annotated sources
    examples/
      01_*.md       ← Walkthrough + runnable code
      02_*.md
      ...
mkdocs.yml          ← Auto-updated with nav for all tools
```

Run it multiple times for different tools — each adds a new folder side-by-side. One MkDocs site serves everything.

## Setup

```bash
# 1. Clone and install
pip install -r requirements.txt

# 2. Set API keys
export ANTHROPIC_API_KEY=sk-ant-...
export TAVILY_API_KEY=tvly-...     # or BRAVE_API_KEY / SERPER_API_KEY

# 3. Using virtual env
cd /Users/elinora/my_git_repos/researcher
python3 -m venv .venv
source .venv/bin/activate
pip install anthropic
python main.py
```

## Usage

```bash
# Research a tool (interactive override prompt if it already exists)
python main.py dbt

# Force overwrite without prompting
python main.py dbt --force

# Use a different search provider
python main.py airflow --provider brave
python main.py polars --provider serper

# Preview locally
mkdocs serve

# Deploy to GitHub Pages
mkdocs gh-deploy
```

## Search Providers

| Provider | Env Var | Notes |
|----------|---------|-------|
| `tavily` (default) | `TAVILY_API_KEY` | Best for LLM agents, returns full page content |
| `brave` | `BRAVE_API_KEY` | Free tier available, good coverage |
| `serper` | `SERPER_API_KEY` | Google results, snippet-only |

Tavily is recommended — it returns full page content which produces richer summaries.

## GitHub Pages Setup

1. Create a GitHub repo
2. Push this project to it
3. Run `mkdocs gh-deploy` — this builds the site and pushes to the `gh-pages` branch
4. In repo Settings → Pages, set source to `gh-pages` branch

Your site will be live at `https://<username>.github.io/<repo-name>/`

## Pipeline Stages

```
Input: "dbt"
  ↓
[Researcher]      Searches 7 queries, deduplicates, returns top 10 sources
  ↓
[Synthesizer]     Claude reads all sources → overview + senior highlights MD
  ↓
[ExampleGen]      Claude plans 5 examples → writes each as full walkthrough
  ↓
[Writer]          Assembles folder structure, writes all MD files
  ↓
[MkDocs Config]   Regenerates mkdocs.yml nav to include new tool
```

## Extending

- **Add a search provider:** subclass `search_providers/base.py`, add to `PROVIDERS` dict in `__init__.py`
- **Change Claude model:** edit `model=` in `synthesizer.py` / `example_generator.py`
- **Adjust example count:** edit `PLANNER_PROMPT` in `example_generator.py`
- **Add new doc sections:** extend prompts in `synthesizer.py`


## Keys
brew install direnv
* add `eval "$(direnv hook zsh)"` to .zshrc
* then put your keys in .envrc inside the project folder
* researcher

