---
name: scrapy-spider-update
description: Rewrite a scaffolded Scrapy spider into a fully async, scrapy-poet-injected spider that follows navigation links and yields items directly from injected page objects.
license: MIT
compatibility: opencode
---

# scrapy-spider-update

## Purpose

Rewrite a previously scaffolded spider into a fully async, scrapy-poet-injected spider that follows navigation → product links using `response.follow` and yields items directly from injected page objects.

## Trigger

Load this skill after `scrapy-page-objects` has run and a spider file exists at `<project_name>/spiders/<spider_name>.py`.

## Instructions

### 1. Read inputs

- Find all `zyte-probe-result-*.json` files in the cwd.
- For each file read: `url`, `page_type`, `item_type`.
- Derive the project name using the same logic as `scrapy-project-setup`:
  - Extract the registered domain from any probe URL (e.g. `books.toscrape.com` → `books.toscrape.com`)
  - Strip subdomains and TLD → slugify → project name (e.g. `books_toscrape`)
- Spider name = project name. If they would collide, append `_spider` (e.g. project `books_toscrape` → spider `books_toscrape_spider`). Apply the same rule used during `scrapy-project-setup`.

### 2. Determine the start URLs

- Collect the `url` field from every probe result whose `page_type` is a navigation type:
  - `ProductNavigation`, `ArticleNavigation`, `JobPostingNavigation`
- If no navigation probe results exist, collect the `url` from every probe result regardless of type.
- This list becomes `urls` in the spider — always a list, even if it contains only one entry.

### 3. Determine the callback structure

Inspect the unique `page_type` values across all probe results and map them to callbacks using this table:

| page_type values present | start callback | navigation callback | leaf callback |
|---|---|---|---|
| `ProductNavigation` + `Product` | `parse_navigation` | `parse_navigation` → follows items + nextPage | `parse_product` |
| `ProductNavigation` only | `parse_navigation` | `parse_navigation` (yield item + follow nextPage) | — |
| `Product` only | `parse_product` | — | `parse_product` |
| `ArticleNavigation` + `Article` | `parse_navigation` | `parse_navigation` → follows items + nextPage | `parse_article` |
| `ArticleNavigation` only | `parse_navigation` | `parse_navigation` (yield item + follow nextPage) | — |
| `Article` only | `parse_article` | — | `parse_article` |
| `JobPostingNavigation` + `JobPosting` | `parse_navigation` | `parse_navigation` → follows items + nextPage | `parse_job_posting` |

For any combination not in the table, apply the same pattern: navigation types get `parse_navigation`, detail types get a `parse_<snake_case_type>` leaf callback.

### 4. Look up item class names

Import item types directly from `zyte_common_items` — do not use `<project_name>.items`. Use this map:

| `item_type` | Import |
|---|---|
| `Product` | `from zyte_common_items import Product` |
| `ProductNavigation` | `from zyte_common_items import ProductNavigation` |
| `Article` | `from zyte_common_items import Article` |
| `ArticleNavigation` | `from zyte_common_items import ArticleNavigation` |
| `JobPosting` | `from zyte_common_items import JobPosting` |
| `JobPostingNavigation` | `from zyte_common_items import JobPostingNavigation` |

### 5. Rewrite the spider file

Replace the full contents of `<project_name>/spiders/<spider_name>.py` with the pattern below.

**Template:**

```python
import scrapy
import <project_name>.pages  # noqa: F401 — triggers handle_urls registration via pages/__init__.py
from zyte_common_items import <ItemClass1>, <ItemClass2>


class <PrefixSpider>(scrapy.Spider):
    name = "<spider_name>"
    allowed_domains = ["<domain>"]

    urls: list[str] = [
        "<navigation_url_1>",
        "<navigation_url_2>",
        # one entry per navigation probe URL
    ]

    async def start(self):
        for url in self.urls:
            yield scrapy.Request(url, callback=self.parse_navigation)

    async def parse_navigation(self, response, page: <NavigationItemClass>):
        if page.items:
            for product in page.items:
                yield response.follow(product.url, callback=self.parse_product)
        if page.nextPage:
            yield response.follow(page.nextPage.url, callback=self.parse_navigation)

    async def parse_product(self, _, item: <ProductItemClass>):
        yield item
```

**Filling in the template:**

- `<project_name>` — the slugified project name (e.g. `books_toscrape`)
- `<ItemClass1>`, `<ItemClass2>` — item classes imported directly from `zyte_common_items` (e.g. `Product`, `ProductNavigation`)
- `<PrefixSpider>` — title-case of the project name + `Spider` (e.g. `BooksToscrapeSpider`); never `BooksToscrapeSpiderSpider`
- `<spider_name>` — the spider name derived in step 1
- `<domain>` — the registered domain (e.g. `books.toscrape.com`)
- `urls` list — one string per navigation URL collected in step 2
- `<NavigationItemClass>` — the navigation item class (e.g. `ProductNavigation`)
- `<ProductItemClass>` — the detail item class (e.g. `Product`)

**Omit callbacks that are not needed:**

- If there is no navigation type in the probe results, omit `parse_navigation` entirely; `start()` should point directly to the leaf callback.
- If there is no detail type in the probe results, omit the leaf callback and the `if page.items:` block in `parse_navigation`.

**Null guards in navigation callback:**

- `if page.items:` — guards against `None` before iterating
- `if page.nextPage:` — guards against `None` before following pagination

**Leaf callback response parameter:**

- When the leaf callback has no outbound links to follow, name the response parameter `_` (it is unused).
- When the leaf callback follows links (unusual), name it `response`.

### 6. Verify

Run:

```
uv run scrapy list
```

The spider name must appear in the output with no errors or tracebacks. If it fails, fix the error and re-verify before finishing.

## Rules

- Always import item classes from `zyte_common_items` directly — never from `<project_name>.items`.
- Never import page object classes in the spider file — scrapy-poet resolves them automatically via `handle_urls`.
- Always add `import <project_name>.pages` to trigger `handle_urls` registration via `pages/__init__.py`. Without this scrapy-poet has no rules and injection silently fails with `missing 1 required positional argument`.
- `urls` is always a list — never use a single `url` string attribute even when there is only one entry.
- `start()` is always an `async def` generator that loops over `self.urls`.
- Use `scrapy.Request` only in `start()` — all subsequent requests use `response.follow`.
- Never construct `scrapy.Request` manually to follow navigation or product links.
- All callbacks are `async def`.
- No parsing logic in the spider — it only yields items and follows links.
- Class name is `<TitleCasePrefix>Spider` — never duplicate the word `Spider`.
- Always verify with `uv run scrapy list` before finishing.

## Output

```
<project_name>/spiders/<spider_name>.py  ← fully rewritten async spider
```
