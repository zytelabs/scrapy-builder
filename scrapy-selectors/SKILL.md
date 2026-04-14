---
name: scrapy-selectors
description: Fill in all @field stub bodies in existing page object files with CSS and microdata selectors derived from saved probe HTML, leaving no pass stubs remaining.
license: MIT
compatibility: opencode
---

# scrapy-selectors

## Purpose

Fill in the `@field` method bodies of existing page object stubs with CSS and microdata selectors derived from saved probe HTML, producing a fully runnable page object file with no `pass` stubs remaining.

## Trigger

Load this skill after `scrapy-page-objects` has run and `zyte-probe-sample-*.html` files exist in the cwd alongside their corresponding `zyte-probe-result-*.json` files.

## Instructions

### 1. Read inputs

- Find all `zyte-probe-result-*.json` files in the cwd.
- For each file read: `url`, `page_type`, `item_type`, `html_file`.
- Read the HTML file named in `html_file` for each probe result.
- Derive the project name and domain using the same slugify logic as `scrapy-project-setup`:
  - Extract the registered domain from any probe URL (e.g. `books.toscrape.com` → `books.toscrape.com`)
  - Strip subdomains and TLD → slugify → project name (e.g. `books_toscrape`)
- Identify the page object file: `<project_name>/pages/<domain_underscored>.py`
  - Domain filename: replace `.` and `-` with `_`, lowercase (e.g. `books.toscrape.com` → `books_toscrape_com.py`)
- Read the page object file to get the current list of classes and `@field` methods.

### 2. For each `@field` method, determine the implementation

Collect the full list of `@field` methods to implement from the page object file.

**Concurrent field analysis:** Issue all field implementations as concurrent tool calls — one per `@field` method (or per class if there are multiple classes in the file). Each tool call reads the probe HTML, applies the decision tree below, and returns the implementation string for that field. Do not wait for one field to complete before starting the next — launch all field analyses in a single parallel batch. Collect all results before proceeding to Step 6.

Work through every `@field` method in the page object file. For each field, apply the decision tree below in order — use the first strategy that succeeds.

#### Strategy 1 — Microdata (`itemprop`)

Check the probe HTML for an element with a matching `itemprop` attribute using the canonical mapping below. If found, use the microdata selector.

**Canonical `itemprop` → field mapping:**

| `itemprop` value | Field | Selector | Extract |
|---|---|---|---|
| `name` | `name` | `[itemprop="name"]` | `::text` |
| `price` | `price` | `[itemprop="price"]` | `::attr(content)` if `content` attr present, else `::text` |
| `priceCurrency` | `currency` | `[itemprop="priceCurrency"]` | `::attr(content)` if `content` attr present, else `::text` |
| `availability` | `availability` | `link[itemprop="availability"]` | `::attr(href)` |
| `sku` | `sku` | `[itemprop="sku"]` | `::text` |
| `mpn` | `mpn` | `[itemprop="mpn"]` | `::text` |
| `brand` | `brand` | `[itemprop="brand"]` | `::attr(content)` if `content` attr present, else `::text` |
| `description` | `description` | `[itemprop="description"]` | `::text` |
| `gtin13` | `gtin` | `[itemprop="gtin13"]` | `::text` — wrap: `return [Gtin(type="gtin13", value=v)]` |
| `gtin8` | `gtin` | `[itemprop="gtin8"]` | `::text` — wrap: `return [Gtin(type="gtin8", value=v)]` |

For microdata scalars where `.get()` is used, strip leading/trailing whitespace by appending `.strip()` if the field return type is `Optional[str]`.

#### Strategy 2 — Structural CSS

If no microdata match is found, look for the field's content using structural CSS patterns in the probe HTML. Use the structural pattern tables in the **Field Pattern Reference** section at the end of this skill.

#### Strategy 3 — Not found

If neither strategy finds a matching element in the probe HTML, emit `return None`.

### 3. Map fields to implementation tiers

Once a selector is identified, assign the correct implementation tier:

#### Tier A — Scalar string

Return type `Optional[str]`. Selector returns a single text or attribute value.

```python
return self.css("selector::text").get("").strip() or None
```

Use `or None` so that empty strings are treated as missing. Use `::attr(x)` instead of `::text` when the value is in an attribute.

#### Tier B — List of strings

Return type `Optional[List[str]]`. Selector returns multiple text values.

```python
return self.css("selector::text").getall() or None
```

#### Tier C — Single Image

Return type `Optional[Image]`.

```python
url = self.css("selector::attr(src)").get()
return Image(url=url) if url else None
```

#### Tier D — List of Images

Return type `Optional[List[Image]]`.

```python
urls = self.css("selector::attr(src)").getall()
return [Image(url=u) for u in urls] if urls else None
```

#### Tier E — Brand

Return type `Optional[Brand]`.

```python
name = self.css("selector::attr(content)").get("").strip() or self.css("selector::text").get("").strip() or None
return Brand(name=name) if name else None
```

#### Tier F — List of Breadcrumbs

Return type `Optional[List[Breadcrumb]]`.

```python
crumbs = []
for el in self.css("selector-for-each-item"):
    name = el.css("span[itemprop='name']::text").get("").strip()
    url = el.css("a[itemprop='item']::attr(href)").get()
    if name:
        crumbs.append(Breadcrumb(name=name, url=url))
return crumbs or None
```

#### Tier G — List of AdditionalProperty

Return type `Optional[List[AdditionalProperty]]`.

```python
props = []
for row in self.css("selector-for-each-row"):
    name = row.css("td:first-child::text").get("").strip()
    value = row.css("td:last-child::text").get("").strip()
    if name and value:
        props.append(AdditionalProperty(name=name, value=value))
return props or None
```

Skip rows that are section headers (i.e. rows containing a `<th>` element instead of `<td>`).

#### Tier H — List of Gtin

Return type `Optional[List[Gtin]]`.

When `itemprop="gtin13"` / `gtin8"` is present — use microdata directly:

```python
value = self.css("[itemprop='gtin13']::text").get("").strip()
return [Gtin(type="gtin13", value=value)] if value else None
```

When gtin is only available in a spec/properties table (no itemprop), scan `additionalProperties`-style rows for a key matching a known GTIN type name:

```python
gtin_keys = {"GTIN", "GTIN-13", "EAN", "EAN-13", "UPC", "ISBN", "GTIN-8"}
for row in self.css("selector-for-each-row"):
    name = row.css("td:first-child::text").get("").strip()
    if name.upper() in {k.upper() for k in gtin_keys}:
        value = row.css("td:last-child::text").get("").strip()
        if value:
            return [Gtin(type="gtin13", value=value)]
return None
```

#### Tier I — AggregateRating

Return type `Optional[AggregateRating]`.

```python
rating = self.css("selector::attr(content)").get() or self.css("selector::text").get()
count = self.css("selector::attr(content)").get() or self.css("selector::text").get()
if not rating:
    return None
return AggregateRating(
    ratingValue=float(rating),
    reviewCount=int(count) if count else None,
)
```

If the probe HTML has no rating elements (dynamically loaded widget, or "no reviews" state), emit `return None`.

#### Tier J — List of ProbabilityRequest (navigation items / subCategories)

Return type `Optional[List[ProbabilityRequest]]`. Used for `items` and `subCategories` fields on navigation page objects.

```python
from urllib.parse import urljoin
urls = self.css("selector::attr(href)").getall()
return [ProbabilityRequest(url=urljoin(self.url, u)) for u in urls] if urls else None
```

`urljoin` ensures relative hrefs become absolute URLs. Import `urljoin` from `urllib.parse` at the top of the page object file if not already present.

#### Tier K — Request (nextPage)

Return type `Optional[Request]`.

```python
from urllib.parse import urljoin
url = self.css("selector::attr(href)").get()
return Request(url=urljoin(self.url, url)) if url else None
```

### 4. Handle features field

`features` is a `Optional[List[str]]` field on `Product`. It is typically stored as a single block of text with bullet separators rather than as individual elements. Apply this pattern when the field content is inside a single element delimited by `•`:

```python
raw = self.css("selector::text").get("").strip()
items = [f.strip() for f in raw.split("•") if f.strip()]
return items or None
```

If feature items are in individual `<li>` elements, use Tier B instead:

```python
return self.css("selector li::text").getall() or None
```

### 5. Handle descriptionHtml field

`descriptionHtml` is `Optional[str]`. Use `.get()` on the container element to return the outer HTML of the block:

```python
return self.css("selector").get()
```

### 6. Update imports in the page object file

After generating all field implementations, scan the generated code for `zyte_common_items` types used (e.g. `Image`, `Brand`, `Breadcrumb`, `AggregateRating`, `AdditionalProperty`, `ProbabilityRequest`, `Request`, `Gtin`, `ProductVariant`). Ensure every type used is present in the `from zyte_common_items import (...)` block at the top of the file.

Also add `from urllib.parse import urljoin` if any `urljoin(...)` calls were emitted.

### 7. Rewrite the page object file

**This step is a single atomic write.** All concurrent field analyses from Step 2 must be complete and all implementations collected before this step runs. Do not write the file incrementally or in multiple passes.

Replace each `pass` stub with the generated implementation. Preserve all existing structure:
- Module-level imports
- `@handle_urls(...)` decorators
- Class definitions and inheritance
- `@field` decorators
- Return type annotations

Do not change any method signature, decorator, or class definition — only replace the body of each `@field` method.

### 8. Verify

Run:

```
uv run python3 -c "from <project_name>.pages.<domain_module> import <ClassName1>, <ClassName2>"
```

If the import fails, fix the error and re-verify before finishing.

---

## Field Pattern Reference

This section contains the structural CSS patterns to use for each field when no microdata match exists. These are encoding of common e-commerce page structures — apply the first pattern whose selector matches an element in the probe HTML.

### Product detail fields

| Field | Try selectors in order | Extract | Tier |
|---|---|---|---|
| `name` | `h1[itemprop="name"]`, `h1.product-name`, `h1` | `::text` | A |
| `price` | `[itemprop="price"]`, `[data-price]`, `.price` | `::attr(content)` then `::text` | A |
| `regularPrice` | `.was-price`, `.wasPrice`, `.original-price`, `[data-was-price]` | `::text` | A |
| `currency` | `[itemprop="priceCurrency"]` | `::attr(content)` | A |
| `currencyRaw` | `.price small:first-child`, `.price .currency` | `::text` | A |
| `availability` | `link[itemprop="availability"]`, `[itemprop="availability"]` | `::attr(href)` then `::text` | A |
| `sku` | `[itemprop="sku"]`, `[data-sku]`, `.sku` | `::text` | A |
| `mpn` | `[itemprop="mpn"]`, `[data-mpn]` | `::text` | A |
| `productId` | `[data-product-id]`, `[data-wpid]`, `[data-id]` | `::attr(data-product-id)` etc. | A |
| `color` | `[itemprop="color"]`, `.color-value`, `[data-color]` | `::text` | A |
| `size` | `[itemprop="size"]`, `.size-value`, `[data-size]` | `::text` | A |
| `style` | `[itemprop="style"]`, `.style-value` | `::text` | A |
| `description` | `[itemprop="description"]`, `.product-description p`, `.overview p:first-of-type` | `::text` | A |
| `descriptionHtml` | `[itemprop="description"]`, `.product-description`, `.overview` | `.get()` | descriptionHtml |
| `features` | `.features li`, `.product-features li`, `.overview p:last-of-type` | `::text` / split on `•` | features |
| `brand` | `[itemprop="brand"]`, `meta[itemprop="brand"]`, `.brand` | `::attr(content)` then `::text` | E |
| `canonicalUrl` | `link[rel="canonical"]` | `::attr(href)` | A |
| `mainImage` | `.product-image img`, `.productImages img:first-of-type`, `[itemprop="image"]` | `::attr(src)` | C |
| `images` | `.product-images img`, `.productImages img`, `.gallery img` | `::attr(src)` | D |
| `breadcrumbs` | `[itemprop="itemListElement"]`, `.breadcrumb li`, `.breadcrumbs li` | iterate → Breadcrumb | F |
| `additionalProperties` | `.specifications tr`, `.spec-table tr`, `.product-specs tr` | iterate → AdditionalProperty | G |
| `gtin` | `[itemprop="gtin13"]`, `[itemprop="gtin8"]`, spec table row with GTIN key | `::text` | H |
| `aggregateRating` | `[itemprop="aggregateRating"]`, `.rating`, `.review-score` | nested selectors | I |
| `variants` | `.product-variants`, `.variant-selector` | return None if absent | — |

### Navigation / listing fields

| Field | Try selectors in order | Extract | Tier |
|---|---|---|---|
| `categoryName` | `h1`, `.category-title`, `.page-title` | `::text` | A |
| `subCategories` | `.sub-categories a`, `.categories a`, `.category-nav a` | `::attr(href)` + urljoin | J |
| `items` | `.product-list a`, `.products a.image`, `li.product a[href]` | `::attr(href)` + urljoin | J |
| `nextPage` | `a[rel="next"]`, `link[rel="next"]`, `.pagination .next a` | `::attr(href)` + urljoin | K |
| `pageNumber` | `.pagination .current`, `.page-number`, `[data-page]` | `::text` cast to int | A (cast) |

For `pageNumber`, wrap in `int()` and return `None` if no match:

```python
raw = self.css("selector::text").get("").strip()
return int(raw) if raw.isdigit() else None
```

---

## Rules

- Never import page object classes — only import from `zyte_common_items` and stdlib.
- Always use `self.css(...)` — never `response.css(...)` (page objects do not have `response`).
- Always use `urljoin(self.url, href)` for relative URLs — never concatenate strings.
- Use `or None` at the end of scalar returns so empty strings are treated as missing.
- Use `.get("").strip()` not `.get().strip()` — the latter raises `AttributeError` when the selector misses.
- Skip table rows that contain a `<th>` element (section headers, not data rows).
- Do not emit extraction logic for `url` or `metadata` — these are handled by the base class.
- All verification must use `uv run python3`, never bare `python3`.

## Output

```
<project_name>/pages/<domain_underscored>.py  ← all @field stubs replaced with implementations
```
