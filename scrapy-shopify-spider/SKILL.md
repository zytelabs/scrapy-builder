---
name: scrapy-shopify-spider
description: Build a complete Scrapy spider for any Shopify store using the /products.json API endpoint. Paginates automatically, yields plain dict items, no HTML scraping or page objects needed.
license: MIT
compatibility:
  - opencode
  - claude-code
---

# scrapy-shopify-spider

## Purpose

Build a complete Scrapy spider for a Shopify store that reads product data directly from the `/products.json` API endpoint — no HTML scraping, no CSS selectors, no page objects. Paginates through all products by incrementing the `?page=` parameter until the response returns an empty list.

## Trigger

Load this skill when:
- The user wants to scrape a Shopify store
- `scrapy-builder` detects a Shopify platform in an audit report or probe result and redirects here

## Instructions

### 1. Collect input

Ask the user for the following if not already provided:

- **Store URL** (required) — the Shopify store domain, e.g. `https://direct4x4.co.uk`. Strip any trailing slash.
- **Working directory** — where to create the project (default: current directory).

Validate that the store URL is a well-formed HTTPS URL. Stop and prompt the user if it is missing or malformed.

### 2. Check environment

Verify:
- `ZYTE_API_KEY` is set in the environment:
  ```
  test -n "$ZYTE_API_KEY" && echo "set" || echo "ZYTE_API_KEY is not set"
  ```
- A `.venv/` directory exists in the working directory.

If `ZYTE_API_KEY` is unset, stop and tell the user to export it before proceeding.

If `.venv/` is missing, stop and tell the user to run the `scrapy-env-setup` skill first.

### 2a. Verify the /products.json endpoint is accessible and derive names

Run these two operations as **concurrent tool calls** — they are independent and can be issued in a single parallel batch.

**Endpoint check:**

```
echo '{"url": "<store_url>/products.json?limit=1&page=1", "httpResponseBody": true}' \
    | uv run zyte-api - -o /dev/stdout
```

Base64-decode `httpResponseBody` from the result. Check that:
- The status code is 200
- The decoded body starts with `{"products":`

If the endpoint returns a 404, redirects to a login page, or the body does not contain `"products":`, stop and tell the user:
> This Shopify store has disabled the `/products.json` endpoint. Use the standard `scrapy-builder` pipeline with HTML scraping instead.

**Name derivation** (run concurrently with the endpoint check):

```python
import re
from urllib.parse import urlparse

parsed = urlparse(store_url)
domain = parsed.netloc  # e.g. "direct4x4.co.uk"
domain_slug = re.sub(r"[^a-z0-9]+", "_", domain.lower()).strip("_")
# e.g. "direct4x4_co_uk"
project_name = domain_slug
spider_name = domain_slug + "_spider"  # avoids name == project collision
spider_class = "".join(w.capitalize() for w in domain_slug.split("_")) + "Spider"
# e.g. "Direct4x4CoUkSpider"
```

Wait for both to complete. If the endpoint check fails, stop (name derivation result is discarded). Otherwise proceed to Step 4 using the derived names.

### 4. Scaffold the project

If `scrapy.cfg` does not already exist in the working directory:

```
uv run scrapy startproject <project_name> .
```

Note the trailing `.` — this scaffolds the project in the current directory rather than creating a subdirectory.

### 5. Write settings.py

Replace the contents of `<project_name>/settings.py` with:

```python
BOT_NAME = "<project_name>"

SPIDER_MODULES = ["<project_name>.spiders"]
NEWSPIDER_MODULE = "<project_name>.spiders"

ROBOTSTXT_OBEY = False

CONCURRENT_REQUESTS = 32
CONCURRENT_REQUESTS_PER_DOMAIN = 32

ADDONS = {
    "scrapy_zyte_api.Addon": 500,
}

ZYTE_API_KEY = ""  # set via environment variable ZYTE_API_KEY

REQUEST_FINGERPRINTER_IMPLEMENTATION = "2.7"
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
FEED_EXPORT_ENCODING = "utf-8"

AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1.0
AUTOTHROTTLE_MAX_DELAY = 10.0
AUTOTHROTTLE_TARGET_CONCURRENCY = 32.0
```

### 6. Write the spider

Create `<project_name>/spiders/<spider_name>.py`:

```python
import re
import scrapy


def _strip_tags(html: str) -> str:
    return re.sub(r"<[^>]+>", "", html).strip()


class <SpiderClass>(scrapy.Spider):
    name = "<spider_name>"

    # Set to the store domain, e.g. "direct4x4.co.uk"
    domain = "<domain>"
    page = 1

    def start_requests(self):
        yield scrapy.Request(
            f"https://{self.domain}/products.json?limit=250&page={self.page}",
            callback=self.parse,
            meta={"zyte_api_automap": {"httpResponseBody": True}},
        )

    def parse(self, response):
        data = response.json()
        products = data.get("products") or []

        if not products:
            self.logger.info("No more products at page %d — crawl complete.", self.page)
            return

        for p in products:
            variants = p.get("variants") or []
            # prefer first in-stock variant; fall back to first variant
            v = next((v for v in variants if v.get("available")), variants[0] if variants else None)

            body_html = p.get("body_html") or ""
            description = _strip_tags(body_html) or None

            compare_at = v.get("compare_at_price") if v else None
            # "0.00" and None both mean no original/sale price
            original_price = compare_at if (compare_at and compare_at != "0.00") else None

            yield {
                "url": f"https://{self.domain}/products/{p['handle']}",
                "name": p.get("title"),
                "price": v["price"] if v else None,
                "original_price": original_price,
                "currency": None,  # not available from /products.json
                "sku": v.get("sku") if v else None,
                "availability": "InStock" if (v and v.get("available")) else "OutOfStock",
                "images": [img["src"] for img in (p.get("images") or [])],
                "description": description,
                "brand": p.get("vendor") or None,
                "product_type": p.get("product_type") or None,
                "tags": p.get("tags") or [],
                "shopify_id": p.get("id"),
            }

        # advance to next page
        self.page += 1
        yield scrapy.Request(
            f"https://{self.domain}/products.json?limit=250&page={self.page}",
            callback=self.parse,
            meta={"zyte_api_automap": {"httpResponseBody": True}},
        )
```

Substitute `<SpiderClass>`, `<spider_name>`, and `<domain>` with the values derived in Step 3.

### 7. Verify

Run both checks in order:

**Check 1 — spider is discoverable:**
```
uv run scrapy list
```
The output must contain `<spider_name>`. If not, check for syntax errors in the spider file.

**Check 2 — smoke test with 5 items:**
```
uv run scrapy crawl <spider_name> -o smoke.jsonl -s CLOSESPIDER_ITEMCOUNT=5
```
Inspect `smoke.jsonl`. Each item must contain `url`, `name`, and `price` with non-null values. Delete `smoke.jsonl` after verification.

If either check fails, fix the error before reporting success.

### 8. Report completion

Tell the user:

- Spider name: `<spider_name>`
- Full crawl command:
  ```
  uv run scrapy crawl <spider_name> -o items.jsonl
  ```
- Note: `currency` is always `null` — the `/products.json` endpoint does not expose currency. If currency is required, fetch `https://<domain>/cart.js` and read the `currency` field (one extra request).
- Note: If the store has more than ~2,000 products, the legacy `?page=N` pagination may miss items added between requests. For full fidelity on large stores, switch to the `page_info` cursor pagination (requires a Shopify Storefront API token).

---

## Output

A working Scrapy project in the working directory:

```
.venv/                          (pre-existing)
pyproject.toml                  (pre-existing)
scrapy.cfg
<project_name>/
  settings.py                   ← ADDONS + scrapy-zyte-api configured, CONCURRENT_REQUESTS=32, CONCURRENT_REQUESTS_PER_DOMAIN=32, AutoThrottle enabled
  spiders/
    <spider_name>.py            ← async JSON paginating spider
```

Each item yielded:

| Field | Source | Notes |
|---|---|---|
| `url` | `handle` | Constructed as `https://<domain>/products/<handle>` |
| `name` | `title` | Product-level |
| `price` | `variants[*].price` | String, e.g. `"29.99"` — first in-stock variant |
| `original_price` | `variants[*].compare_at_price` | `null` when `"0.00"` or absent |
| `currency` | — | Always `null` — not in this endpoint |
| `sku` | `variants[*].sku` | First in-stock variant |
| `availability` | `variants[*].available` | `"InStock"` or `"OutOfStock"` |
| `images` | `images[*].src` | List of CDN URLs; no alt text available |
| `description` | `body_html` | HTML tags stripped; `null` if empty |
| `brand` | `vendor` | Store-level vendor string |
| `product_type` | `product_type` | May be empty string on some products |
| `tags` | `tags` | List of strings |
| `shopify_id` | `id` | Shopify internal integer ID |

---

## Known gotchas

- **`compare_at_price: "0.00"`** — this means no original/sale price, same as `null`. Always treat `"0.00"` as absent.
- **No currency in response** — `/products.json` does not include currency. If needed, fetch `/cart.js` (one request) and read `response.json()["currency"]`.
- **No image alt text** — `images[*]` in `/products.json` has no `alt` field. Alt text requires the Admin or Storefront GraphQL API.
- **`body_html` can be `null`** — guard with `p.get("body_html") or ""` before stripping tags.
- **`?page=N` pagination limit** — Shopify's legacy page-based pagination is capped at 1,000 pages (250,000 products). For stores larger than this, the `page_info` cursor API is required.
- **Endpoint may be disabled** — some Shopify stores block `/products.json`. Check in Step 2a and fall back to HTML scraping if unavailable.
- **Rate limiting** — unauthenticated requests are subject to Shopify's standard rate limits. Zyte API handles this transparently.
