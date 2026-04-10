---
name: scrapy-page-objects
description: Generate scrapy-poet page object classes for each probed URL, subclassing the correct zyte-common-items base, with all item fields as blank @field stubs and handle_urls registered.
license: MIT
compatibility: opencode
---

# scrapy-page-objects

## Purpose

Generate scrapy-poet page object classes for each probed URL, one file per domain in the project's `pages/` directory, with all item fields as blank `@field` methods.

## Trigger

Load this skill after `scrapy-project-setup` has run and one or more `zyte-probe-result-<slug>.json` files exist in the cwd.

## Instructions

### 1. Read inputs

- Find all `zyte-probe-result-*.json` files in the cwd.
- For each file read: `url`, `page_type`, `item_type`.
- Determine the project name using the same derivation as `scrapy-project-setup`:
  - Extract the registered domain from any probe URL (e.g. `books.toscrape.com`)
  - Strip the TLD, slugify → project name (e.g. `books_toscrape`)

### 2. Group probe results by domain

- For each probe result, extract the registered domain (e.g. `books.toscrape.com`).
- Group all probe results that share the same domain — they go into one file.

### 3. Derive names

For each domain:

**Domain filename** — replace `.` and `-` with `_`, lowercase:
- `books.toscrape.com` → `books_toscrape_com.py`
- `books-to-scrape.com` → `books_to_scrape_com.py`

**Class name prefix** — title-case each segment of the domain, joined:
- `books.toscrape.com` → `BooksToscrapeCom`
- `books-to-scrape.com` → `BooksToScrape` (hyphens removed, not split)

**Page object class name** — prefix + page type + `Page`:
- `page_type: Product` → `BooksToscrapeComProductPage`
- `page_type: ProductNavigation` → `BooksToscrapeComProductNavigationPage`
- `page_type: Article` → `BooksToscrapeComArticlePage`

**Base class** — use the `zyte_common_items` page base class for the item type:

| `page_type` | Base class | Import |
|---|---|---|
| `Product` | `ProductPage` | `from zyte_common_items import ProductPage` |
| `ProductNavigation` | `ProductNavigationPage` | `from zyte_common_items import ProductNavigationPage` |
| `Article` | `ArticlePage` | `from zyte_common_items import ArticlePage` |
| `ArticleNavigation` | `ArticleNavigationPage` | `from zyte_common_items import ArticleNavigationPage` |
| `JobPosting` | `JobPostingPage` | `from zyte_common_items import JobPostingPage` |
| `JobPostingNavigation` | `JobPostingNavigationPage` | `from zyte_common_items import JobPostingNavigationPage` |
| `RealEstate` | `RealEstatePage` | `from zyte_common_items import RealEstatePage` |
| `BusinessPlace` | `BusinessPlacePage` | `from zyte_common_items import BusinessPlacePage` |

### 4. Look up fields for each item type

Use this canonical field list per item type. Exclude `url` and `metadata` — these are handled by the base class.

**Product fields:**
```
name: Optional[str]
price: Optional[str]
regularPrice: Optional[str]
currency: Optional[str]
currencyRaw: Optional[str]
availability: Optional[str]
sku: Optional[str]
mpn: Optional[str]
productId: Optional[str]
color: Optional[str]
size: Optional[str]
style: Optional[str]
description: Optional[str]
descriptionHtml: Optional[str]
features: Optional[List[str]]
breadcrumbs: Optional[List[Breadcrumb]]
mainImage: Optional[Image]
images: Optional[List[Image]]
additionalProperties: Optional[List[AdditionalProperty]]
aggregateRating: Optional[AggregateRating]
brand: Optional[Brand]
gtin: Optional[List[Gtin]]
canonicalUrl: Optional[str]
variants: Optional[List[ProductVariant]]
```

**ProductNavigation fields:**
```
categoryName: Optional[str]
subCategories: Optional[List[ProbabilityRequest]]
items: Optional[List[ProbabilityRequest]]
nextPage: Optional[Request]
pageNumber: Optional[int]
```

**Article fields:**
```
headline: Optional[str]
datePublished: Optional[str]
datePublishedRaw: Optional[str]
dateModified: Optional[str]
dateModifiedRaw: Optional[str]
authors: Optional[List[Author]]
inLanguage: Optional[str]
breadcrumbs: Optional[List[Breadcrumb]]
mainImage: Optional[Image]
images: Optional[List[Image]]
description: Optional[str]
articleBody: Optional[str]
articleBodyHtml: Optional[str]
videos: Optional[List[Video]]
audios: Optional[List[Audio]]
canonicalUrl: Optional[str]
```

**JobPosting fields:**
```
jobPostingId: Optional[str]
jobTitle: Optional[str]
headline: Optional[str]
datePublished: Optional[str]
datePublishedRaw: Optional[str]
dateModified: Optional[str]
dateModifiedRaw: Optional[str]
validThrough: Optional[str]
validThroughRaw: Optional[str]
description: Optional[str]
descriptionHtml: Optional[str]
employmentType: Optional[str]
jobLocation: Optional[JobLocation]
baseSalary: Optional[BaseSalary]
requirements: Optional[List[str]]
hiringOrganization: Optional[HiringOrganization]
jobStartDate: Optional[str]
jobStartDateRaw: Optional[str]
remoteStatus: Optional[str]
```

**BusinessPlace fields:**
```
placeId: Optional[str]
name: Optional[str]
address: Optional[Address]
telephone: Optional[str]
website: Optional[str]
categories: Optional[List[str]]
description: Optional[str]
priceRange: Optional[str]
mainImage: Optional[Image]
images: Optional[List[Image]]
aggregateRating: Optional[AggregateRating]
```

**RealEstate fields:**
```
realEstateId: Optional[str]
name: Optional[str]
datePublished: Optional[str]
datePublishedRaw: Optional[str]
description: Optional[str]
mainImage: Optional[Image]
images: Optional[List[Image]]
address: Optional[Address]
price: Optional[str]
currency: Optional[str]
currencyRaw: Optional[str]
tradeType: Optional[str]
propertyType: Optional[str]
breadcrumbs: Optional[List[Breadcrumb]]
```

### 5. Collect all component imports needed

Scan the field lists for the page objects being generated and collect every component type referenced (e.g. `Image`, `Brand`, `Breadcrumb`, `AggregateRating`, `ProbabilityRequest`, `Request`, etc.). Only import what is actually used.

All component types and page base classes are imported from `zyte_common_items`.

### 6. Write the domain file

Write `<project_name>/pages/<domain_filename>.py`.

If the file already exists, append new classes to it — do not overwrite existing ones. If a class with the same name already exists, skip it.

**File structure:**

```python
from typing import List, Optional

from web_poet import field, handle_urls
from zyte_common_items import (
    <PageBaseClass>,
    <ComponentType1>,
    <ComponentType2>,
    # only types actually used in this file
)


@handle_urls("<domain>")
class <Prefix><PageType>Page(<PageBaseClass>):

    @field
    def <field_name>(self) -> <ReturnType>:
        pass

    @field
    def <field_name>(self) -> <ReturnType>:
        pass
    # ... one @field method per item field (excluding url and metadata)


# one class per page type probed for this domain, separated by a blank line
```

### 7. Update `<project_name>/pages/__init__.py`

After writing the domain file, add an import of the new module to `pages/__init__.py` so that `handle_urls` rules register automatically when the package is imported:

```python
from <project_name>.pages import <domain_module>  # noqa: F401 — registers handle_urls rules with scrapy-poet
```

If the import is already present, skip this step.

### 8. Verify

Confirm the file was written and is importable:

```
uv run python3 -c "from <project_name>.pages.<domain_module> import <ClassName>"
```

If the import fails, fix the syntax error before finishing.

## Rules

- `handle_urls` is imported from `web_poet`.
- Page object classes subclass the `zyte_common_items` page base class directly (e.g. `ProductPage`, `ProductNavigationPage`) — do not use `WebPage + Returns[ItemClass]`.
- Never import from `<project_name>.items` — items are provided by the base class automatically.
- Always apply `@handle_urls("<domain>")` to every page object class.
- Exclude `url` and `metadata` fields — do not generate `@field` methods for them.
- All `@field` methods return `pass` — no extraction logic.
- Return type annotations must match the zyte-common-items field type exactly.
- One file per domain, all page objects for that domain in the same file.
- If a domain file already exists, append — do not overwrite.
- Always update `pages/__init__.py` to import the domain module after writing it.
- Always use `uv run python3` to verify imports, never bare `python3`.

## Output

```
<project_name>/pages/
  __init__.py              ← updated to import the domain module
  <domain_underscored>.py  ← one class per page type for this domain
```
