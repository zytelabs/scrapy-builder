# scrapy-builder-skills

Agent skills for building production-ready Scrapy spiders with [Zyte API](https://www.zyte.com/zyte-api/). Each skill is a self-contained `SKILL.md` that instructs an AI agent how to perform one step of the pipeline.

## Quick start

Give an agent two URLs from the target site (a listing/navigation URL and a detail URL) and ask it to build a spider. The `scrapy-builder` skill orchestrates the full 7-step pipeline automatically.

**Prerequisites:** `uv`, a Zyte API key in `ZYTE_API_KEY`.

## Pipeline overview

`scrapy-builder` is the single entry point. It runs 7 skills in sequence:

| Step | Skill | What it does |
|---|---|---|
| 1 | `scrapy-env-setup` | Creates a Python 3.12 `uv` virtualenv and installs Scrapy, scrapy-zyte-api, zyte-common-items, scrapy-poet, and pytest |
| 2 | `scrapy-zyte-probe` | Fetches each URL via the Zyte API CLI; saves one HTML sample and one result JSON per URL |
| 3 | `scrapy-schema-infer` | Reads the saved HTML and classifies each page as a `zyte-common-items` type (`Product`, `ProductNavigation`, `Article`, etc.) |
| 4 | `scrapy-project-setup` | Scaffolds the Scrapy project with `scrapy-poet` and `scrapy-zyte-api` wired in via the addons system |
| 5 | `scrapy-page-objects` | Generates `web_poet` page object classes with all item fields as blank `@field` stubs |
| 6 | `scrapy-selectors` | Fills every `@field` stub with CSS or microdata selectors derived from the saved HTML |
| 7 | `scrapy-spider-update` | Rewrites the scaffolded spider into a fully async, scrapy-poet-injected spider |

Each pipeline skill can also be run individually to resume a partial build.

## Output

A complete, runnable Scrapy project:

```
<project>/
├── .venv/
├── pyproject.toml
├── scrapy.cfg
├── <project_name>/
│   ├── settings.py          # scrapy-poet + scrapy-zyte-api addons wired in
│   ├── pages/
│   │   ├── __init__.py
│   │   └── <domain>.py      # page objects with selectors
│   └── spiders/
│       └── <spider>.py      # async scrapy-poet spider
```

Run the spider with:

```
uv run scrapy crawl <spider_name> -o items.jsonl
```

## Skill reference

| Skill | Description |
|---|---|
| `scrapy-builder` | **Orchestrator** — full pipeline from URLs to working spider |
| `scrapy-env-setup` | Set up the Python 3.12 `uv` virtualenv |
| `scrapy-zyte-probe` | Probe URLs via Zyte API; save HTML + result JSON |
| `scrapy-schema-infer` | Classify saved HTML pages as `zyte-common-items` types |
| `scrapy-project-setup` | Scaffold the Scrapy project with addons configured |
| `scrapy-page-objects` | Generate page objects with `@field` stubs |
| `scrapy-selectors` | Fill `@field` stubs with CSS/microdata selectors |
| `scrapy-spider-update` | Rewrite the spider as a fully async scrapy-poet spider |

## Installing skills

Skills live in two locations that must stay in sync:

| Location | Purpose |
|---|---|
| `<skill-name>/SKILL.md` | Source of truth — edit skills here |
| `.claude/skills/<skill-name>/SKILL.md` | Auto-loaded by Claude Code and OpenCode in this repo |

To make skills available in any project, copy them to `~/.claude/skills/<skill-name>/SKILL.md`.

## Adding a skill

1. Create `<skill-name>/SKILL.md` following the structure of existing skills.
2. Copy to `.claude/skills/<skill-name>/SKILL.md`.
3. Add a row to the tables in this README and in `AGENTS.md`.

Each `SKILL.md` must include YAML frontmatter (`name`, `description`, `license`, `compatibility`) and sections for **Purpose**, **Trigger**, **Instructions**, and **Output**.

## Known gotchas

- Always use `uv` — never `pip`, `poetry`, or `conda`. Python version: **3.12**.
- `handle_urls` is imported from `web_poet`, not `scrapy_poet`.
- The spider must `import <project_name>.pages` at startup to register `handle_urls` rules — without this, scrapy-poet injection silently fails.
- Scrapy 2.14 generates `ADDONS = {}` in `settings.py` — replace the entire dict, do not append to it.
- `ProductNavigation` is always preferred over `ProductList` for category/listing pages with filters, pagination, or navigation context.
- Never create custom item subclasses in `items.py` — items are provided by the `zyte_common_items` page base classes.
