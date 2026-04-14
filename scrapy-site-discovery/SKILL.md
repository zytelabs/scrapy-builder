---
name: scrapy-site-discovery
description: >
  Given a product topic, discover the top ecommerce sites selling that product,
  verify them with Zyte AI extraction, find an all-products entry URL for each,
  and optionally hand off to scrapy-builder.
license: MIT
compatibility:
  - opencode
  - claude-code
---

# scrapy-site-discovery

## Purpose

Given a product topic (e.g. "running shoes", "vintage watches"), discover the top ecommerce sites
selling that product, verify each site with Zyte AI product list extraction, identify the broadest
all-products entry URL, and produce a ranked report. Optionally hands off to `scrapy-builder` for
any confirmed sites the user wants to scrape.

## Trigger

Load this skill when the user asks to:
- "Find and scrape the top X product sites"
- "Discover sites selling Y"
- "Find ecommerce stores for Z"

This skill is **standalone** — it is not called by any other skill in this project.

## Instructions

### 1. Collect inputs

Extract from the user's prompt:

- **`topic`** (required) — the product category, e.g. `running shoes`, `vintage watches`. If not
  present, ask the user before continuing.
- **`count`** (optional) — number of confirmed sites to return. Default: 10 if not specified.
- **`pages`** (optional) — number of Google SERP pages to fetch during discovery. Default: 2. User
  may specify more (e.g. "search 4 pages" or "use 5 pages").
- **`working_directory`** (optional) — where to write output files. Default: current directory.

Derive `<topic-slug>`: lowercase, spaces → hyphens, strip non-alphanumeric characters except hyphens
(e.g. `running shoes` → `running-shoes`, `men's watches` → `mens-watches`).

---

### 2. Discover candidate sites

Call `zyte_extract_serp` with:
- `url`: `https://www.google.com/search?q=top+<topic>+online+stores+buy`
- `pages`: `<pages>` (default: 2; honour any value the user specified in step 1)

From the returned `organic_results` list, collect candidate domains:
- For each result, extract the registered domain from `result.url`
  (strip `www.` and any subdomain — keep only the registered domain, e.g. `example.com`)

**Exclusion list** — always drop these domains regardless of source:

- Marketplaces: `amazon.*`, `ebay.*`, `etsy.*`, `aliexpress.*`, `walmart.*`, `target.*`
- Social media: `facebook.*`, `instagram.*`, `pinterest.*`, `tiktok.*`, `youtube.*`, `twitter.*`, `x.com`
- Search engines: `google.*`, `bing.*`, `yahoo.*`, `duckduckgo.*`
- Reference/news: `wikipedia.*`, `bbc.*`, `cnn.*`, `theguardian.*`, `reddit.*`
- Any domain the user has explicitly excluded in this session

Deduplicate by registered domain (strip `www.` and subdomains before comparing).

**Listicle fetching (conditional):** Count the organic candidates remaining after exclusions.
If the pool is **below 1.5× count** (e.g. fewer than 15 for count=10), then and only then:
- Identify listicle results — any result where `result.name` or `result.url` contains the words
  `best`, `top`, `review`, or `recommended` — take up to 3, in rank order.
- For each listicle URL (up to 3), call `zyte_extract_article` **concurrently**.
- Extract all domain mentions from the returned article body text and add new registered domains to
  the candidate pool (applying the same exclusion list).
- **Listicle fallback:** If `zyte_extract_article` returns a body shorter than 500 characters,
  re-fetch the same URL using `zyte_render_page` and extract domain mentions from the rendered HTML.

**Target candidate pool size:** 1.5× count (e.g. 15 candidates for count=10). If the pool after
all fetches and fallbacks is still below count, warn the user and continue with what's available.

---

### 3. Verify candidates in parallel

For each candidate domain, probe all entry URLs **concurrently** by calling
`zyte_extract_product_list` for all four patterns at the same time — do not stop at the first hit:

1. `https://<domain>/`
2. `https://<domain>/shop`
3. `https://<domain>/products`
4. `https://<domain>/collections/all`

Launch all candidate verifications as **concurrent tool calls** — one batch per candidate domain,
all candidates running simultaneously. Do not verify candidates serially.

Once all probes for a candidate return, pick the result with the **most items** as the `entry_url`.
If two results tie, prefer the one with the shorter URL path.

**Classification rules:**

| Classification | Condition |
|---|---|
| **Confirmed** | Best probe returns ≥ 3 items AND at least one item name or the page title/URL contains a word from the topic |
| **Partial** | Best probe returns ≥ 3 items but no clear topic match in item names, title, or URL |
| **Failed** | All 4 probe URLs return 0 items or errors |

Build the confirmed list in order of confidence (Confirmed first, then Partial if more are needed
to reach `count`). Stop when the list reaches `count`. If fewer than `count` confirmed sites are
found, include all available and note the shortfall in the report.

For each confirmed site, record:
- The `entry_url` — the probe URL that returned the most items
- Platform signal if detectable from response headers or page HTML: `Shopify`, `WooCommerce`,
  `Magento`, `BigCommerce`, or `Unknown`
  - Shopify: `powered-by: Shopify` header or `cdn.shopify.com` in HTML
  - WooCommerce: `x-powered-by: WooCommerce` or `woocommerce` in body class
  - Magento: `x-magento-*` headers or `/pub/static/` in HTML
  - BigCommerce: `x-bc-*` headers or `cdn.bigcommerce.com` in HTML
- Up to 3 sample product names from the extraction result
- Estimated product count (use `len(items)` from the extraction — this is a lower bound, not total)

---

### 4. Write output files

Write both files to the working directory. Use `<topic-slug>` consistently across filenames.

**`zyte-discovery-<topic-slug>.json`:**

```json
[
  {
    "rank": 1,
    "domain": "example.com",
    "name": "Example Store",
    "entry_url": "https://example.com/collections/all",
    "platform": "Shopify",
    "confidence": "confirmed",
    "product_count_estimate": 42,
    "product_sample": ["Product Name A", "Product Name B", "Product Name C"]
  }
]
```

Fields:
- `rank` — 1-based integer, Confirmed sites ranked before Partial
- `domain` — registered domain only (no `www.`)
- `name` — store name as found in page title or search result; fall back to domain if unknown
- `entry_url` — the probe URL that returned the most items
- `platform` — detected platform string or `"Unknown"`
- `confidence` — `"confirmed"` or `"partial"`
- `product_count_estimate` — number of items returned by `zyte_extract_product_list` (lower bound)
- `product_sample` — list of up to 3 product name strings

**`zyte-discovery-<topic-slug>.md`:**

```markdown
# Site Discovery: <topic>

**Date:** <ISO 8601 datetime>
**Topic:** <topic>
**Target count:** <count>
**Results:** <N confirmed> confirmed / <N partial> partial / <N failed> failed

## Confirmed Sites

| Rank | Site | Entry URL | Platform | Products (est.) |
|---|---|---|---|---|
| 1 | Example Store (example.com) | https://example.com/collections/all | Shopify | 42 |

## Partial Matches

*(Sites where products were found but topic match was weak)*

| Rank | Site | Entry URL | Platform | Notes |
|---|---|---|---|---|

## Notes

- <List any sites that failed verification and the reason (bot-blocked, no products, wrong topic)>
- <Shortfall warning if fewer than `count` sites were confirmed>
- <Any Zyte browser render fallbacks that were triggered during discovery>
```

---

### 5. Offer scrapy-builder handoff

Print the markdown results table to the user, then prompt:

> Found {N} confirmed sites for "{topic}". Which would you like to scrape?
> Enter a number (e.g. `1`), a comma-separated list (e.g. `1,3,5`), `all`, or `none` to stop here.

For each chosen site, load and execute `scrapy-builder`, passing:
- The site's `entry_url` as the navigation URL
- The working directory

Run the scrapy-builder handoff for each selected site **sequentially** — complete one spider build
before starting the next, so the user can see progress.

---

## Rules

- Always use `zyte_extract_product_list` for verification — never fetch raw HTML manually for
  this step.
- Never verify candidates serially in step 3 — always launch all verifications concurrently.
- Probe all 4 entry URL patterns concurrently per candidate — do not stop at the first hit.
- Only fetch listicles when the organic candidate pool is below 1.5× count after exclusions.
- The `<topic-slug>` must be consistent across all output filenames and derived at step 1.
- Never include marketplaces, social media, search engines, or reference sites in the candidate pool.
- If `scrapy-builder` is invoked, always pass the `entry_url`, not the homepage.
- Always write both output files (JSON + markdown) before offering the scrapy-builder handoff.
- If `count` is not specified by the user, default to 10.

## Output

```
zyte-discovery-<topic-slug>.json   ← machine-readable ranked list with entry URLs
zyte-discovery-<topic-slug>.md     ← human-readable ranked report
```

After the skill completes, the user may optionally trigger `scrapy-builder` for any confirmed site.
