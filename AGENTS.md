# AGENTS.md

## Project purpose

This repository contains skill definition files for building Scrapy spiders with Zyte API. Each skill is a standalone `SKILL.md` that instructs an agent how to perform a specific task.

## Skill locations

Skills are stored in two places that must stay in sync:

| Location | Purpose |
|---|---|
| `<skill-name>/SKILL.md` | Source of truth — edit skills here |
| `.claude/skills/<skill-name>/SKILL.md` | Picked up automatically by Claude Code and OpenCode when working in this repo |

When you edit a skill, update both copies. To install skills globally (available in any project):
- **OpenCode / Claude Code**: copy to `~/.claude/skills/<skill-name>/SKILL.md`

## Critical constraint

Skills in this project are **fully independent** from any skills at `~/.claude/skills/`. Do not reference, load, reuse, or adapt anything from that location.

## Skill structure

Each skill is a single file:

```
<skill-name>/SKILL.md
```

A `SKILL.md` must contain:
- YAML frontmatter with `name`, `description`, `license`, and `compatibility` fields
- **Purpose** — one sentence describing what the skill does
- **Trigger** — when an agent should load this skill
- **Instructions** — step-by-step workflow the agent must follow
- **Output** — what the skill is expected to produce

Keep instructions executable and specific. Avoid generic advice.

## Conventions

- One skill per directory; directory name matches the skill name (kebab-case)
- The `name` field in frontmatter must exactly match the directory name
- Skill names must be unique within this project
- Instructions must be self-contained — no references to external skills or files outside the skill's own directory
- Prefer concrete commands and file patterns over prose descriptions

## Agent ramp-up

- Only skills defined under this project directory are in scope
- When building a new skill, read existing skills in this repo first to match conventions

## Skills in this project

Use `scrapy-builder` as the single entry point for building a new spider end-to-end. It orchestrates the 7 pipeline skills below in sequence, and automatically routes to `scrapy-shopify-spider` when a Shopify site is detected.

| Skill | What it does |
|---|---|
| `scrapy-builder` | **Orchestrator** — runs the full pipeline from URLs to working spider in one shot |
| `scrapy-shopify-spider` | **Shopify shortcut** — builds a JSON API spider for any Shopify store; auto-triggered by `scrapy-builder` when Shopify is detected |
| `scrapy-zyte-site-audit` | **Standalone diagnostic** — audits a URL for fetch method, platform, anti-bot, JS dependency, and AI extraction viability |

The 7 pipeline skills (run in this order by `scrapy-builder`, or individually if resuming):

| Order | Skill | What it does |
|---|---|---|
| 1 | `scrapy-env-setup` | Creates a `uv` virtualenv and installs Scrapy + Zyte dependencies |
| 2 | `scrapy-zyte-probe` | Fetches one or more URLs via the Zyte API CLI and saves HTML + result JSON per URL |
| 3 | `scrapy-schema-infer` | Reads the saved HTML files and classifies each page as a `zyte-common-items` type |
| 4 | `scrapy-project-setup` | Scaffolds the Scrapy project, `settings.py`, and `items.py` from probe results |
| 5 | `scrapy-page-objects` | Creates `web_poet` page object classes for each classified page type |
| 6 | `scrapy-selectors` | Fills in `@field` method bodies with CSS/microdata selectors from probe HTML |
| 7 | `scrapy-spider-update` | Rewrites the scaffolded spider into a fully async, scrapy-poet-injected spider |

## Known gotchas

### Tooling
- Always use `uv` — never `pip`, `poetry`, or `conda`
- Python version: **3.12**

### Zyte API CLI
- `httpResponseStatus` in JSONL output may be `None` even on a 200 — use the CLI summary line or check that the HTML body is non-empty to confirm success

### Scrapy
- Scrapy 2.14 generates `ADDONS = {}` in `settings.py` — replace the entire dict, do not append to it
- If the spider name equals the project name, `scrapy genspider` will error — append `_spider` to the spider name (e.g. project `scan` → spider `scan_spider`)

### scrapy-poet / web_poet
- `handle_urls` is imported from `web_poet`, **not** `scrapy_poet`
- To verify registered rules after importing pages, use `web_poet.default_registry.get_rules()` — `PageObjectRegistry` is not directly importable from `web_poet.rules` in current versions
- Page objects subclass the `zyte_common_items` page base class directly (e.g. `ProductPage`, `ProductNavigationPage`) — do not use `WebPage + Returns[ItemClass]`
- Items are provided by the base class automatically — never create custom item subclasses in `items.py`
- The spider imports item types directly from `zyte_common_items` (e.g. `from zyte_common_items import Product, ProductNavigation`) — never from `<project_name>.items`
- `pages/__init__.py` must import every domain module so that `handle_urls` rules register when the package is imported
- The spider must `import <project_name>.pages` to trigger `handle_urls` registration at startup — without this scrapy-poet has no rules and injection silently fails with `missing 1 required positional argument`
- `WebPage` / `ResponseShortcutsMixin` already expose a `url` property — do not declare `url` as a `@field`

### Schema inference
- `ProductNavigation` is **always preferred** over `ProductList` when a page has multiple product links, a filter sidebar, category CSS classes, or a listing body attribute
- `ProductList` is reserved for flat, non-navigational item feeds (e.g. pure API/search result lists with no filters or pagination context)
- `multiple_priced_items` alone is not a reliable signal for `ProductList` — category pages with hundreds of priced items are still `ProductNavigation`

### Items
- Never create custom item subclasses in `items.py` — items are provided directly by the `zyte_common_items` page base classes
- `zyte_common_items` LSP type annotation errors in the venv are a known upstream issue — safe to ignore

### Shopify JSON API (`scrapy-shopify-spider`)
- `/products.json` prices are **strings** (`"29.99"`), not numbers — always parse before arithmetic
- `compare_at_price: "0.00"` means no sale price — treat identically to `null`
- No currency field in `/products.json` — fetch `/cart.js` if currency is required
- No image `alt` text in `/products.json` — available only via Admin/Storefront GraphQL API
- Some Shopify stores disable `/products.json` — check for 404 or login redirect before proceeding
- Legacy `?page=N` pagination is capped at 1,000 pages (250,000 products); use `page_info` cursor for larger stores
