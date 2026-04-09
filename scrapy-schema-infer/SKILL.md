---
name: scrapy-schema-infer
description: Read saved probe HTML files and classify each page as a zyte-common-items type (Product, ProductNavigation, Article, etc.), then write the result back into each probe result JSON.
license: MIT
compatibility: opencode
---

# scrapy-schema-infer

## Purpose

Read the saved HTML from each probe result and infer the page type and item type using `zyte-common-items` types, then update each `zyte-probe-result-<slug>.json` file with the results.

## Trigger

Load this skill after `scrapy-zyte-probe` has run and one or more `zyte-probe-result-<slug>.json` files exist in the cwd.

## Instructions

### 1. Read inputs

- Find all files in the cwd matching the pattern `zyte-probe-result-*.json`.
- For each file, read it to get the `url` and `html_file` path.
- Read the HTML file named in `html_file`.
- Run steps 2–5 for each file independently.

### 2. Extract signals from the HTML

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

### 3. Map signals to zyte-common-items types

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

### 4. Assign confidence

- **high** — at least one Tier 1 or Tier 2 signal fired, and all signals agree on the same type.
- **medium** — only Tier 3/4 signals fired, but they all agree.
- **low** — signals are ambiguous or point to more than one type. Record all candidate types in `candidate_types`.

When confidence is **low**:
- Pick the type supported by the most and highest-tier signals as the primary result.
- List all other candidate types in `candidate_types`.
- Do not stop — always produce an output.

### 5. Update `zyte-probe-result-<slug>.json`

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

## Output

Each `zyte-probe-result-<slug>.json` updated in place with `page_type`, `item_type`, `confidence`, `signals`, and `candidate_types` fields added.
