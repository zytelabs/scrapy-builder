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

### 2. Check the environment and platform signal

Run these two checks as **concurrent tool calls** — issue both in a single parallel batch, then use the results to decide the next step.

**Environment check:** Inspect the working directory for `.venv/` and `pyproject.toml`.

**Shopify signal check:** Check the working directory for a Shopify signal in any of these locations:
1. A `zyte-audit-report-*.md` file containing the line `**Detected platform:** Shopify`
2. A `zyte-probe-result-*.json` file containing `"platform": "Shopify"`

Once both checks complete:

- **If `.venv/` is missing** — load and execute the `scrapy-env-setup` skill before continuing.
- **If a Shopify signal is found** — extract the store domain from the matching file's `url` field, stop the standard pipeline, load and execute the `scrapy-shopify-spider` skill instead, and do not continue to Step 3.
- **Otherwise** — proceed to Step 3 with the environment confirmed ready.

### 3. Run the pipeline

The pipeline runs in three phases. Load each skill via the `skill` tool and execute it fully (including its own verify step) before moving on. Do not skip a step unless the "Skip if" condition is met.

#### Phase A — Sequential setup

Run steps 1 and 2 in order. Each must complete before the next starts.

| Step | Skill | Skip if |
|---|---|---|
| 1 | `scrapy-env-setup` | `.venv/` already exists (handled in step 2 above) |
| 2 | `scrapy-zyte-probe` | `zyte-probe-result-*.json` files already exist in the cwd for all provided URLs |

When loading `scrapy-zyte-probe` in step 2, pass the URLs collected in step 1 — do not ask the user for URLs again.

#### Phase B — Concurrent classification and scaffolding

Once `scrapy-zyte-probe` completes (or is skipped), launch steps 3 and 4 as **concurrent Task tool calls** — issue both in a single parallel batch and wait for both to finish before continuing.

| Step | Skill | Skip if |
|---|---|---|
| 3 | `scrapy-schema-infer` | All `zyte-probe-result-*.json` files already contain a `page_type` field **and** at least two probe result files exist (one for the listing page, one for the detail page) |
| 4 | `scrapy-project-setup` | A `scrapy.cfg` already exists in the cwd |

These two skills are independent: `scrapy-schema-infer` reads probe HTML to classify pages, while `scrapy-project-setup` reads probe JSON only for the domain name to scaffold the project. Neither depends on the other's output. Both must complete before Phase C begins because `scrapy-page-objects` requires both the `page_type` fields (from schema-infer) and the `pages/` package skeleton (from project-setup).

If only one of the two needs to run (the other meets its skip condition), run that single skill normally — no concurrency needed.

#### Phase C — Sequential finalisation

Run steps 5, 6, and 7 in order. Each must complete before the next starts.

| Step | Skill | Skip if |
|---|---|---|
| 5 | `scrapy-page-objects` | All page object files already exist with no `pass` stubs remaining |
| 6 | `scrapy-selectors` | All `@field` methods in the page object files already have implementations (no `pass` stubs) |
| 7 | `scrapy-spider-update` | The spider file already contains `async def start(` |

> **Audit report integration:** If `scrapy-zyte-site-audit` has already been run for any of the URLs, the resulting `zyte-audit-report-*.md` files will be present in the working directory. `scrapy-zyte-probe` automatically detects and consumes these files — extracting the recommended fetch method — instead of making redundant Zyte API calls. No user action is required; this is the integration point between the audit skill and the builder pipeline.

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
