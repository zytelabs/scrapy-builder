---
name: scrapy-schema-infer
description: Given a listing page URL, fetch and classify both the listing page and an auto-discovered detail page using zyte-common-items types, then write results into probe JSON files.
license: MIT
compatibility: opencode
---

# scrapy-schema-infer

## Purpose

Given a listing page URL provided by the user, fetch and classify the listing page, automatically discover detail page links by URL pattern frequency, probe one representative detail page, and write `page_type`, `item_type`, `confidence`, `signals`, and `candidate_types` into each probe result JSON.

## Trigger

Load this skill when the user provides a listing page URL and wants to classify both the listing page and the detail pages it links to.

## Instructions

### 0. Accept input URL

Ask the user for the listing page URL if not already provided. Validate that it is non-empty and is a well-formed HTTP or HTTPS URL. Stop and prompt the user if it is missing or malformed.

### 1. Fetch the listing page

Run the Zyte API CLI to fetch the listing page:

```
zyte-api fetch --output zyte-probe-result-<slug>.json <url>
```

Where `<slug>` is derived from the URL by taking the hostname + path, replacing non-alphanumeric characters with hyphens, and lowercasing (e.g. `example-com-shop-women`).

Save the returned HTML to `zyte-probe-<slug>.html`.

Update the probe JSON to include:
```json
{
  "url": "<url>",
  "html_file": "zyte-probe-<slug>.html",
  "fetch_method": "<browserHtml or httpResponseBody>"
}
```

### 2. Classify the listing page

Run signal analysis (see below) on the listing page HTML. Write the classification fields (`page_type`, `item_type`, `confidence`, `signals`, `candidate_types`) into `zyte-probe-result-<slug>.json`.

### 2a. Check URL structure samples

Before scanning links, check whether the listing page matches a known platform in the **URL structure samples** table (see below). Use two signals:

1. **HTML platform signals** — check the listing page HTML for:
   - `<meta name="generator" content="...">` value
   - Script `src` URLs containing platform-specific CDN paths (e.g. `cdn.shopify.com`, `wp-content`)
   - `<body class="...">` containing platform keywords (e.g. `woocommerce`, `mage`)
   - CSS `href` paths containing platform keywords (e.g. `oscar`)

2. **Listing URL path pattern** — match the listing page URL path against the `Listing path pattern` column.

If a platform match is found:
- Construct a candidate detail URL by replacing the listing path with the known detail path pattern, substituting a plausible slug (take the first product-like segment visible in the listing page HTML if available, otherwise use `example-product`).
- Probe that candidate URL immediately via Zyte API CLI.
- If it classifies as a detail page → write results, skip Steps 3–4.
- If the probe fails or returns a navigation page → fall through to Step 3.

If no platform match is found, proceed directly to Step 3.

### 3. Scan links and probe eagerly

Parse the listing page HTML. Walk every `<a href="...">` element in document order. For each link:

1. Resolve the href to an absolute URL.
2. Skip if: different domain, fragment-only (`#...`), mailto, javascript, or already visited.
3. Score the link against the **detail page heuristic** (see below).
4. If the link scores as a **likely detail page**, immediately fire the Zyte API CLI probe for that URL — do not wait to finish scanning all links.
5. Continue scanning remaining links in parallel with the probe running.
6. As soon as the probe result is available, run signal analysis on the returned HTML.
   - If the page classifies as a detail type (`Product`, `Article`, `JobPosting`, etc.) → stop scanning, use this page as the confirmed detail page.
   - If the page classifies as a navigation/listing type → discard this result, continue scanning for the next candidate.
7. If no link scores above the threshold after scanning all links, fall back to the **pattern-frequency fallback** (Step 3a below).

#### Detail page heuristic — fire probe immediately if ALL of these are true:

- The link element is **not** inside `<nav>`, `<header>`, `<footer>`
- The path does **not** contain any of: `/category/`, `/browse/`, `/search`, `/tag/`, `/page/`, `/blog/`, `/news/`, `/jobs/`, `/careers/`
- The path has **2 or more segments** after the hostname
- The final path segment looks like a slug: contains at least one hyphen OR at least one digit OR ends with a file extension like `.html`/`.htm`

#### Step 3a. Pattern-frequency fallback (only if eager probing found no detail page)

Group all collected same-domain links by URL path pattern:
- Split the path by `/`
- Replace segments that contain digits, hyphens, or are longer than 20 chars with `*`
- Keep static segments as-is

Count distinct URLs per pattern. Exclude patterns whose static segments include navigation keywords: `category`, `browse`, `search`, `tag`, `page`, `blog`, `news`, `jobs`, `careers`.

Pick the pattern with the highest count. In a tie, prefer patterns whose static segments contain: `product`, `item`, `detail`, `pd`, `article`, `job`, `listing`.

Probe the first URL from the winning pattern. If it classifies as a navigation page, probe the next URL in that group.

If all patterns have count = 1, pick the first link not inside `<nav>`, `<header>`, or `<footer>`.

### 4. Probe the detail page candidate

Run the Zyte API CLI to fetch the candidate detail page URL. Use the same fetch method determined for the listing page.

Save HTML to `zyte-probe-<detail-slug>.html` and the result to `zyte-probe-result-<detail-slug>.json`.

### 5. Classify the detail page

Run the signal analysis (see below) on the detail page HTML. Write the classification fields into `zyte-probe-result-<detail-slug>.json`.

---

## Signal analysis (run for each page)

### Extract signals from the HTML

Check the following signals in order of confidence. Record every signal that fires.

#### Tier 1 — Structured data (highest confidence)

**JSON-LD blocks** — find all `<script type="application/ld+json">` tags and inspect `@type`:

| `@type` value | Signal name |
|---|---|
| `ItemList`, `OfferCatalog` | `jsonld_ItemList` |
| `Product` | `jsonld_Product` |
| `Article`, `NewsArticle`, `BlogPosting` | `jsonld_Article` |
| `JobPosting` | `jsonld_JobPosting` |
| `BreadcrumbList` | `jsonld_BreadcrumbList` |
| `SearchResultsPage` | `jsonld_SearchResultsPage` |
| `RealEstateListing` | `jsonld_RealEstate` |

**Microdata** — find all `itemtype="http://schema.org/..."` attributes and apply the same mapping as JSON-LD.

#### Tier 2 — Explicit page-type attributes (high confidence)

- `<body data-page="category">` or `data-page="product-list"` → `body_data_page_listing`
- `<body data-page="product">` or `data-page="pdp"` → `body_data_page_detail`
- `<body data-page="search">` → `body_data_page_search`
- `<body data-page="article">` or `data-page="blog"` → `body_data_page_article`

#### Tier 3 — Structural HTML signals (medium confidence)

**Listing signals:**
- 3 or more elements that each contain a price, a name, and a link to another page → `multiple_priced_items`
- Presence of `rel="next"` link, `.pagination`, `.pager`, or `paginationNext` element → `pagination_present`
- Presence of a product filter/facet sidebar → `filter_sidebar_present`
- CSS bundle or class names containing `productlist`, `product-list`, `category` → `productlist_css_class`

**Detail signals:**
- A single dominant price block with an add-to-basket/add-to-cart button → `single_product_with_atb`
- A single `<h1>` with a product name alongside a price → `single_product_h1_price`

**Navigation/category signals:**
- A grid or list of links to product pages (with or without prices) → `multiple_product_links`
- Presence of subcategory links or filter widgets → `category_nav_links`

**Article signals:**
- `<article>` tag or `.article-body` class present → `article_tag_present`
- `datePublished` or `<time>` element near a headline → `article_date_present`

**Job signals:**
- Keywords: "apply now", "job title", "salary", "employment type" in visible text → `job_posting_signals`

#### Tier 4 — URL pattern (supporting evidence only)

| URL pattern | Signal name |
|---|---|
| `/shop/`, `/category/`, `/c/`, `/browse/`, `/search` | `url_listing_pattern` |
| `/products/`, `/item/`, `/p/`, `/pd/` | `url_detail_pattern` |
| `/articles/`, `/blog/`, `/news/` | `url_article_pattern` |
| `/jobs/`, `/careers/` | `url_job_pattern` |

### Map signals to zyte-common-items types

Use this decision table, applying the highest-tier signals first. **`ProductNavigation` is always preferred over `ProductList`** when a page contains multiple product links — `ProductList` is only used when the page is clearly a flat list of items with no navigation role (e.g. a search results feed or API-style item list).

| Signals present | `page_type` | `item_type` |
|---|---|---|
| `jsonld_Product` OR `single_product_with_atb` OR `single_product_h1_price` | `Product` | `Product` |
| `multiple_product_links` OR `multiple_priced_items` OR `body_data_page_listing` OR `filter_sidebar_present` OR `productlist_css_class` | `ProductNavigation` | `ProductNavigation` |
| `jsonld_ItemList` AND no `multiple_product_links` AND no listing signals | `ProductList` | `ProductFromList` |
| `jsonld_Article` OR (`article_tag_present` AND `article_date_present`) | `Article` | `Article` |
| `article_tag_present` AND `multiple_article_items` | `ArticleList` | `ArticleFromList` |
| `jsonld_JobPosting` OR `job_posting_signals` | `JobPosting` | `JobPosting` |
| `jsonld_SearchResultsPage` OR `body_data_page_search` | `Serp` | `Serp` |
| `jsonld_RealEstate` | `RealEstate` | `RealEstate` |

**`ProductList` vs `ProductNavigation` rule:** If the page has multiple product links, multiple priced items, a filter sidebar, category CSS classes, or a listing body attribute — always classify as `ProductNavigation`. Only use `ProductList` if the page is a flat itemised list with no navigational structure (e.g. a pure API/feed output with no filters, pagination, or category context).

### Assign confidence

- **high** — at least one Tier 1 or Tier 2 signal fired, and all signals agree on the same type.
- **medium** — only Tier 3/4 signals fired, but they all agree.
- **low** — signals are ambiguous or point to more than one type. Record all candidate types in `candidate_types`.

When confidence is **low**:
- Pick the type supported by the most and highest-tier signals as the primary result.
- List all other candidate types in `candidate_types`.
- Do not stop — always produce an output.

### Write classification fields into probe JSON

Add the following fields to the existing JSON object (preserve all existing fields):

```json
{
  "page_type": "ProductNavigation",
  "item_type": "ProductNavigation",
  "confidence": "high",
  "signals": [
    "body_data_page_listing",
    "multiple_product_links",
    "filter_sidebar_present",
    "productlist_css_class"
  ],
  "candidate_types": []
}
```

`candidate_types` is an empty list when confidence is high or medium. When confidence is low, it contains objects like:
```json
[
  {"page_type": "ProductNavigation", "item_type": "ProductNavigation"}
]
```

---

## zyte-common-items type reference

Only use types from this list. Do not invent new types.

**Listing pages:**
- `page_type: ProductList` → `item_type: ProductFromList`
- `page_type: ProductNavigation` → `item_type: ProductNavigation`
- `page_type: ArticleList` → `item_type: ArticleFromList`
- `page_type: ArticleNavigation` → `item_type: ArticleNavigation`
- `page_type: JobPostingNavigation` → `item_type: JobPostingNavigation`

**Detail pages:**
- `page_type: Product` → `item_type: Product`
- `page_type: Article` → `item_type: Article`
- `page_type: JobPosting` → `item_type: JobPosting`
- `page_type: RealEstate` → `item_type: RealEstate`
- `page_type: BusinessPlace` → `item_type: BusinessPlace`
- `page_type: SocialMediaPost` → `item_type: SocialMediaPost`

**Other:**
- `page_type: Serp` → `item_type: Serp`

---

## URL structure samples

Use this table in Step 2a to detect the platform and derive a candidate detail URL before scanning links.

| Platform | Detection signals | Listing path pattern | Detail path pattern | Example detail URL |
|---|---|---|---|---|
| **Shopify** | `cdn.shopify.com` in script `src`; or `Shopify` in meta generator | `/collections/*` | `/products/<slug>` | `/products/example-product` |
| **WooCommerce** | `woocommerce` in `<body class>`; or `WooCommerce` in meta generator; or `wp-content/plugins/woocommerce` in scripts | `/product-category/*` | `/product/<slug>/` | `/product/example-product/` |
| **WordPress (blog)** | `wp-content` in script/link `src`; meta generator contains `WordPress` | `/category/*` or `/?cat=` | `/<year>/<month>/<slug>/` or `/?p=<id>` | `/2024/01/example-post/` |
| **Magento 2** | `Magento_Ui` or `requirejs/require.js` in scripts; or `mage` in `<body class>` | `/<category>.html` or `/*/<category>.html` | `/<slug>.html` (single path segment + `.html`) | `/example-product.html` |
| **BigCommerce** | `bigcommerce` in script `src` or meta generator | `/category/*` or `*/categories/*` | `/<slug>/` (top-level single segment) | `/example-product/` |
| **Oscar (Django)** | `oscar` in CSS/JS `href` paths | `/catalogue/category/*/*/` | `/catalogue/<slug>_<id>/` | `/catalogue/a-light-in-the-attic_1000/` |
| **PrestaShop** | `prestashop` in body class or meta generator; or `/modules/` in script `src` | `/<n>-<category-name>` or `/<category>/` | `/<n>-<product-slug>.html` | `/12-example-product.html` |
| **Generic ecommerce** | — | `/c/*`, `/browse/*`, or `/shop/*` | Derived from first product-card link found on listing page | — |

### How to construct the candidate detail URL

For platforms with a fixed detail path pattern (all except Generic ecommerce):

1. Take the listing page's scheme + hostname (e.g. `https://example.com`).
2. Find the first product title or slug visible inside a product card element on the listing page HTML (look inside `article`, `[class*="product"]`, `[class*="item"]` elements for an `<a>` href or a heading text).
3. Slugify it: lowercase, replace spaces and special chars with hyphens.
4. Substitute into the detail path pattern.

If no product slug is identifiable from the listing HTML, use `example-product` as a placeholder slug and expect the probe to return a 404 — treat 404 as a miss and fall through to Step 3.

---

## Output

- `zyte-probe-result-<listing-slug>.json` — listing page probe result with classification fields
- `zyte-probe-<listing-slug>.html` — listing page HTML
- `zyte-probe-result-<detail-slug>.json` — detail page probe result with classification fields
- `zyte-probe-<detail-slug>.html` — detail page HTML
