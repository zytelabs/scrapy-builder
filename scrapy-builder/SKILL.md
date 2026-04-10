---
name: scrapy-builder
description: Build a complete Scrapy spider from scratch given a navigation URL and an optional detail URL. Orchestrates scrapy-env-setup through scrapy-spider-update in sequence. Use this as the single entry point when starting a new spider project.
license: MIT
compatibility: opencode
---

# scrapy-builder

## Purpose

Orchestrate the full 7-skill pipeline to build a production-ready Scrapy spider from a set of URLs in one end-to-end run.

## Trigger

Load this skill when the user wants to build a new Scrapy spider. Collect a navigation URL and a detail URL from the same domain before starting.

## Instructions

### 1. Collect inputs

Ask the user for the following if not already provided:

- **Navigation/listing URL** (required) — e.g. a category or search results page. Drives the spider's crawl pattern and generates the navigation page object.

- **Detail URL** (optional) — e.g. a single product page. If provided, it is passed to `scrapy-zyte-probe` along with the navigation URL so both are probed upfront. If not provided, `scrapy-schema-infer` will discover and probe the detail page automatically from the navigation page's links.

- **Working directory** — where to create the project (default: current directory).

Do not ask for anything else. All other decisions (project name, spider name, page types, selectors) are derived automatically by the downstream skills.

### 2. Check the environment

Inspect the working directory for `.venv/` and `pyproject.toml`.

- **If `.venv/` already exists** — skip to step 3. The environment is ready.
- **If `.venv/` is missing** — load and execute the `scrapy-env-setup` skill before continuing.

### 3. Run the pipeline

Execute each skill below in order. Load each skill via the `skill` tool, execute it fully (including its own verify step), then move to the next. Do not skip a step unless the condition in the "Skip if" column is met.

| Step | Skill | Skip if |
|---|---|---|
| 1 | `scrapy-env-setup` | `.venv/` already exists (handled in step 2 above) |
| 2 | `scrapy-zyte-probe` | `zyte-probe-result-*.json` files already exist in the cwd for all provided URLs |
| 3 | `scrapy-schema-infer` | All `zyte-probe-result-*.json` files already contain a `page_type` field **and** at least two probe result files exist (one for the listing page, one for the detail page) |
| 4 | `scrapy-project-setup` | A `scrapy.cfg` already exists in the cwd |
| 5 | `scrapy-page-objects` | All page object files already exist with no `pass` stubs remaining |
| 6 | `scrapy-selectors` | All `@field` methods in the page object files already have implementations (no `pass` stubs) |
| 7 | `scrapy-spider-update` | The spider file already contains `async def start(` |

When loading `scrapy-zyte-probe` in step 2, pass the URLs collected in step 1 — do not ask the user for URLs again.

### 4. Confirm completion

After all steps have run successfully, report:

- The spider name (from `scrapy list` output)
- The project directory structure (top-level only)
- The start URLs configured in the spider
- How to run the spider:
  ```
  uv run scrapy crawl <spider_name> -o items.jsonl
  ```

## Rules

- Never ask the user for URLs more than once — collect them in step 1 and carry them forward.
- The navigation URL is required. The detail URL is optional — if not provided, `scrapy-schema-infer` discovers it automatically.
- Never skip the verify step within each sub-skill — each skill's own verification must pass before moving on.
- If any skill fails its verify step, stop and report the error clearly. Do not continue to the next skill.
- Always use `uv run scrapy` — never bare `scrapy`.
- The skip conditions in step 3 are resumability guards — they allow re-running `scrapy-builder` in a partially-complete project without redoing finished work.

## Output

A fully working Scrapy project in the working directory:

```
.venv/
pyproject.toml
scrapy.cfg
<project_name>/
  settings.py               ← ADDONS + ZYTE_API_KEY configured
  items.py                  ← untouched
  pages/
    __init__.py             ← imports domain module
    <domain_underscored>.py ← page objects with full selector implementations
  spiders/
    <spider_name>.py        ← async scrapy-poet spider
```
