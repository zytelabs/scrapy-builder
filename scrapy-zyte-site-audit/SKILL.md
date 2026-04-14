---
name: scrapy-zyte-site-audit
description: >
  Given a single URL, run a comprehensive Zyte API diagnostic audit and produce
  a structured report recommending the optimal scraping strategy for that site,
  including platform detection (Shopify, WordPress, Magento, Next.js, and more).
license: MIT
compatibility:
  - opencode
  - claude-code
---

## Purpose

Run a comprehensive Zyte API capability audit against a single URL and produce a
human-readable report that recommends the best scraping approach: fetch method,
platform detection (Shopify, WordPress, Magento, Next.js, Wix, and 13 more),
AI extraction viability, anti-bot signals, JavaScript dependency, network capture
findings, and geolocation sensitivity.

## Trigger

Load this skill when a user asks to audit, analyse, or assess how best to scrape
a specific URL using Zyte API.

This skill is **standalone** — it is not called by any other skill in this project.

## Instructions

### 1. Verify prerequisites

- Confirm `ZYTE_API_KEY` is set in the environment. If not, stop and ask the user to set it.
- Confirm the user has provided a URL. If not, ask for one.
- Confirm `uv` is available and a `.venv` exists in the current directory. If not, tell the
  user to run the `scrapy-env-setup` skill first to create the virtualenv.

### 2. Derive a slug

Derive a filesystem-safe `<slug>` from the URL:

```python
import re
from urllib.parse import urlparse

parsed = urlparse(url)
raw = (parsed.netloc + parsed.path).strip("/")
slug = re.sub(r"[^a-z0-9]+", "-", raw.lower()).strip("-") or "root"
```

All output files use `<slug>` in their name.

### 3. Write the 8 input JSONL files

Write one `.jsonl` file per Zyte API call. Each file contains exactly one JSON line.

**audit-http.jsonl** — raw HTTP fetch with headers:
```json
{"url": "<URL>", "httpResponseBody": true, "httpResponseHeaders": true}
```

**audit-browser.jsonl** — browser render, screenshot, response headers, and catch-all
network capture. Use `"http"` as the value — the minimum valid string (≥3 chars) that
matches all HTTP and HTTPS sub-requests:
```json
{"url": "<URL>", "browserHtml": true, "screenshot": true, "httpResponseHeaders": true, "networkCapture": [{"filterType": "url", "matchType": "contains", "value": "http", "httpResponseBody": true}]}
```

**audit-geo.jsonl** — HTTP fetch from a US geolocation to test geo sensitivity:
```json
{"url": "<URL>", "httpResponseBody": true, "httpResponseHeaders": true, "geolocation": "US"}
```

**audit-product.jsonl** — AI product extraction using browser HTML as source:
```json
{"url": "<URL>", "product": true, "productOptions": {"extractFrom": "browserHtml"}}
```

**audit-productnavigation.jsonl** — AI product navigation extraction:
```json
{"url": "<URL>", "productNavigation": true}
```

**audit-article.jsonl** — AI article extraction:
```json
{"url": "<URL>", "article": true}
```

**audit-jobposting.jsonl** — AI job posting extraction:
```json
{"url": "<URL>", "jobPosting": true}
```

**audit-pagecontent.jsonl** — AI generic page content extraction:
```json
{"url": "<URL>", "pageContent": true}
```

### 4. Run all 8 calls in parallel

Run all 8 calls concurrently using background processes. Always use `uv run zyte-api`,
never the bare `zyte-api` command. Save each result to the corresponding JSON file.
Delete all `.jsonl` input files immediately after all calls complete.

```bash
uv run zyte-api audit-http.jsonl            > zyte-audit-http-<slug>.json &
uv run zyte-api audit-browser.jsonl         > zyte-audit-browser-<slug>.json &
uv run zyte-api audit-geo.jsonl             > zyte-audit-geo-<slug>.json &
uv run zyte-api audit-product.jsonl         > zyte-audit-product-<slug>.json &
uv run zyte-api audit-productnavigation.jsonl > zyte-audit-productnavigation-<slug>.json &
uv run zyte-api audit-article.jsonl         > zyte-audit-article-<slug>.json &
uv run zyte-api audit-jobposting.jsonl      > zyte-audit-jobposting-<slug>.json &
uv run zyte-api audit-pagecontent.jsonl     > zyte-audit-pagecontent-<slug>.json &
wait

rm -f audit-http.jsonl audit-browser.jsonl audit-geo.jsonl \
      audit-product.jsonl audit-productnavigation.jsonl \
      audit-article.jsonl audit-jobposting.jsonl audit-pagecontent.jsonl
```

If any call fails (empty output or error JSON), record the failure and continue — do not
abort the entire audit.

### 5. Decode and save the screenshot

Extract and decode the Base64-encoded screenshot from the browser response:

```bash
jq --raw-output .screenshot zyte-audit-browser-<slug>.json \
    | base64 --decode > zyte-audit-screenshot-<slug>.jpg
```

If the browser call failed or `screenshot` is null, skip this step and note the failure in
the report.

### 6. Analyse results

Issue all analysis sub-sections as concurrent tool calls — each sub-section reads its own independent JSON file(s). Launch all of the following analyses in a single parallel batch, then collect all results before writing the report in Step 7.

Read each saved JSON file and extract the following signals:

#### HTTP check (`zyte-audit-http-<slug>.json`)
- `statusCode` — was the page reachable via plain HTTP?
- `httpResponseHeaders` — scan for anti-bot indicators:
  - `cf-ray`, `cf-cache-status` → Cloudflare
  - `x-datadome-*`, `datadome` → DataDome
  - `x-px-*` → PerimeterX
  - `x-akamai-*`, `akamai-ghost-ip` → Akamai
  - `server` header value → nginx, Apache, Varnish, custom CDN
  - `set-cookie` names → session tokens, consent cookies, fingerprinting cookies
  - `cache-control`, `vary` → caching behaviour
- `httpResponseBody` (decoded from Base64) — approximate content length; check whether
  meaningful HTML content is present (look for `<html`, `<body`, `<main`, structured data
  markers `application/ld+json`, `itemprop`).

#### Platform detection (`zyte-audit-http-<slug>.json`)

Scan `httpResponseHeaders` and the decoded `httpResponseBody` for platform fingerprints.
Check both sources — some platforms only expose signals in one. Assign confidence:

- **Definite** — 2 or more signals across both header and body
- **Likely** — 1 strong unambiguous signal (e.g. `powered-by: Shopify`)
- **Uncertain** — only weak body signals with no header confirmation

| Platform | Header signals | Body signals |
|---|---|---|
| **Shopify** | `powered-by: Shopify` | `cdn.shopify.com`, `window.Shopify`, `Shopify.shop`, `_shopify_` in cookies |
| **WooCommerce** | `x-powered-by: WooCommerce` | `wp-content/plugins/woocommerce`, `woocommerce_` in cookies |
| **WordPress** | `x-powered-by: WordPress` | `/wp-content/`, `/wp-json/`, `<meta name="generator" content="WordPress` |
| **Magento 2** | `x-magento-*` headers | `/pub/static/`, `requirejs-config.js`, `window.require.config`, `/graphql` |
| **Magento 1** | `x-magento-*` headers | `/skin/frontend/`, `/js/mage/`, `Mage.Cookies` |
| **BigCommerce** | `x-bc-*` headers | `cdn.bigcommerce.com`, `bc_` in cookies |
| **PrestaShop** | `x-powered-by: PrestaShop` | `PrestaShop-` in cookies, `prestashop` in body JS |
| **Drupal** | `x-generator: Drupal` | `/sites/default/files/`, `<meta name="generator" content="Drupal` |
| **Joomla** | — | `<meta name="generator" content="Joomla`, `/components/com_`, `joomla_user_state` in cookies |
| **Wix** | `x-wix-*` headers | `static.wixstatic.com`, `wixsite.com` |
| **Squarespace** | — | `static1.squarespace.com`, `window.Squarespace` |
| **Webflow** | `x-powered-by: Webflow` | `assets.website-files.com`, `webflow.js` |
| **Next.js** | `x-powered-by: Next.js` | `/_next/static/`, `__NEXT_DATA__` |
| **Nuxt / Vue** | `x-powered-by: Nuxt` | `/_nuxt/`, `window.__NUXT__` |
| **Laravel / PHP** | `x-powered-by: PHP` | `laravel_session` or `XSRF-TOKEN` in cookies |
| **Ruby on Rails** | `x-runtime` header | `_session_id` in cookies, `data-turbo` in body |
| **Django** | — | `csrfmiddlewaretoken` in body, `csrftoken` + `sessionid` in cookies |

Check Magento version by presence: if `/pub/static/` or `requirejs-config.js` → Magento 2;
if `/skin/frontend/` or `Mage.Cookies` → Magento 1. Magento 2 takes priority if both appear.

Record the matched platform, confidence level, and every matched signal (header name/value or
body pattern) for inclusion in the report.

#### Browser check (`zyte-audit-browser-<slug>.json`)
- `statusCode` — was the browser render successful?
- `browserHtml` length — compare to HTTP body length. A significant increase (>50%)
  indicates the site is JavaScript-heavy.
- `httpResponseHeaders` — same anti-bot header scan as above.
- `networkCapture` array:
  - Count total captured sub-requests.
  - List unique URL patterns (deduplicate by hostname + path prefix).
  - Flag requests that look like internal APIs: URLs containing `/api/`, `/graphql`,
    `.json`, `/v1/`, `/v2/`, `/xhr/`, `_ajax`, `fetch`, `rest` — **but exclude URLs
    ending in static asset extensions** (`.svg`, `.css`, `.js`, `.png`, `.jpg`, `.woff`,
    `.woff2`, `.ico`, `.gif`, `.webp`, `.ttf`, `.eot`).
    A URL like `/static/v2/logo.svg` is a versioned asset, not an API.
  - For each API-looking URL, note whether `httpResponseBody` was captured and its
    approximate decoded size.

#### Geo check (`zyte-audit-geo-<slug>.json`)
- Compare `httpResponseBody` length and `httpResponseHeaders` values to the baseline HTTP
  check. Flag if:
  - Body length differs by >10%
  - `content-language` header differs
  - `set-cookie` headers differ (geo-triggered consent flows)

#### AI extraction checks
For each of the 8 extraction result files, parse the relevant top-level key (`product`,
`productNavigation`, `article`, `jobPosting`, `pageContent`).

- If the key is missing or the call returned an error, record as **not applicable**.
- If the key is present, count populated (non-null, non-empty) fields vs total possible
  fields.
- Record the field completeness percentage.
- For `product`: check `name`, `price`, `currency`, `description`, `sku`, `brand`,
  `images`, `availability`.
- For `productNavigation`: check `items` array length, `nextPage` URL.
- For `article`: check `headline`, `datePublished`, `author`, `articleBody`.
- For `jobPosting`: check `title`, `datePosted`, `hiringOrganization`, `jobLocation`.
- For `pageContent`: check `title`, `body` content length.

### 7. Produce the report

**Wait for all concurrent analyses from Step 6 to complete before writing.** Then write `zyte-audit-report-<slug>.md` with the following structure.

---

```markdown
# Zyte API Site Audit: <URL>

**Audited:** <ISO 8601 datetime>

---

## 1. Fetch Method Recommendation

**Recommended:** `httpResponseBody` | `browserHtml`  *(pick one)*

| Method | Status | Body size | Anti-bot signals |
|---|---|---|---|
| httpResponseBody | <statusCode> | <N bytes> | <list or "none"> |
| browserHtml | <statusCode> | <N chars> | <list or "none"> |

**Rationale:** <1–3 sentences explaining the recommendation. Key factors: did HTTP return
meaningful HTML? Is the browser HTML significantly larger? Were anti-bot headers present on
only one method?>

---

## 2. Anti-Bot Signals

**Detected protection:** <None | Cloudflare | DataDome | PerimeterX | Akamai | Other>

| Header | Value | Inference |
|---|---|---|
| <header-name> | <value> | <what it means> |

**Recommendation:** <Does Zyte API handle this automatically? Is `browserHtml` required?
Should sessions be used? Is residential IP (`ipType: residential`) likely needed?>

---

## 3. Platform Detection

**Detected platform:** <Shopify | WooCommerce | WordPress | Magento 2 | Magento 1 |
                        BigCommerce | PrestaShop | Drupal | Joomla | Wix | Squarespace |
                        Webflow | Next.js | Nuxt | Laravel/PHP | Ruby on Rails | Django |
                        Unknown>
**Confidence:** Definite | Likely | Uncertain

| Signal | Value | Source |
|---|---|---|
| <header name or body pattern> | <matched value> | <HTTP headers / HTML body> |

### Platform-specific scraping notes

*(Write only the block for the detected platform. If Unknown, skip this sub-section.)*

| Platform | Notes |
|---|---|
| **Shopify** | `httpResponseBody` sufficient for most themes. Some themes lazy-load prices via JS — check Section 4 (JavaScript Dependency). JSON product API at `/collections/<handle>/products.json?limit=250&page=N` is public and zero-cost. Sitemap at `/sitemap.xml`. |
| **WooCommerce** | `httpResponseBody` usually sufficient; some themes JS-render prices. REST API at `/wp-json/wc/v2/products?per_page=100` is public read-only by default. Sitemap at `/wp-sitemap.xml`. |
| **WordPress** | `httpResponseBody` correct for content. REST API at `/wp-json/wp/v2/posts?per_page=100`. Sitemap at `/wp-sitemap.xml`. |
| **Magento 2** | Some stores render prices via Ajax — check browser delta in Section 4. GraphQL API at `/graphql` (no auth for public catalogue). REST API at `/rest/V1/products?searchCriteria[pageSize]=100`. |
| **Magento 1** | Mostly server-rendered; `httpResponseBody` sufficient. REST API at `/api/rest/products?limit=100` if enabled. |
| **BigCommerce** | `httpResponseBody` usually sufficient. Storefront GraphQL API at `/graphql` requires `Authorization: Bearer <token>` — token is embedded in page source. Sitemap at `/xmlsitemap.php`. |
| **PrestaShop** | Server-rendered; `httpResponseBody` correct. REST API at `/api/products?output_format=JSON` requires an API key. |
| **Drupal** | Server-rendered; `httpResponseBody` correct. JSON:API at `/jsonapi/node/<type>` if the JSON:API module is enabled. |
| **Joomla** | Server-rendered; `httpResponseBody` correct. API at `/api/index.php/v1/<resource>` available on Joomla 4+ if API application is enabled. |
| **Wix** | Heavily JS-rendered — `browserHtml` required. No public API. Rely on AI extraction or custom selectors from browser HTML. |
| **Squarespace** | Check browser delta in Section 4; some templates are server-rendered. No public API. |
| **Webflow** | Mostly server-rendered CMS; `httpResponseBody` usually correct. No public API. |
| **Next.js** | `__NEXT_DATA__` JSON blob embedded in `<script id="__NEXT_DATA__">` often contains full structured page data. Parse directly from HTTP body — AI extraction may be unnecessary. |
| **Nuxt** | `window.__NUXT__` JSON blob in HTTP body. Parse directly — often richer than AI extraction output. |
| **Laravel/PHP** | Server-rendered; `httpResponseBody` correct. Watch for CSRF tokens if scraping forms. No standard API. |
| **Ruby on Rails** | Server-rendered; `httpResponseBody` correct. `X-Runtime` header confirms Rails. No standard API. |
| **Django** | Server-rendered; `httpResponseBody` correct. API paths vary by project — check network capture for `/api/` endpoints. |

### Known platform APIs

*(Write only rows that apply to the detected platform.)*

| Endpoint | Description | Auth required |
|---|---|---|
| <url pattern> | <what it returns> | <Yes / No / Token in page source> |

---

## 4. JavaScript Dependency

**Assessment:** <Static | Mildly dynamic | Heavily JS-rendered>

- HTTP body size: <N bytes>
- Browser HTML size: <N chars>
- Size delta: <+N% / -N%>
- Script tags in browser HTML: <count>
- Inline JSON-LD blocks: <count>

**Recommendation:** <Use `httpResponseBody` / `browserHtml` / `browserHtmlOnly` for
`extractFrom`. Explain the reasoning.>

---

## 5. Network Capture Findings

**Sub-requests captured:** <N total>

**API / XHR endpoints detected:**

| URL pattern | Captured body size | Likely purpose |
|---|---|---|
| <url> | <N bytes> | <products API / pagination / auth / analytics / other> |

*(If no API endpoints were detected, state that and note whether direct HTTP scraping
is therefore sufficient.)*

**Direct API opportunity:** <Yes – describe the endpoint and how to use it directly>
| <No – content appears server-rendered>

---

## 6. Geolocation Sensitivity

**Sensitive:** Yes | No

| | Default geo | geolocation: US |
|---|---|---|
| Body size | <N bytes> | <N bytes> |
| content-language | <value> | <value> |
| Notable header differences | <list> | <list> |

**Recommendation:** <Is an explicit `geolocation` needed? Which country code? Or is the
default Zyte API geolocation selection sufficient?>

---

## 7. AI Auto-Extraction Viability

| Type | Status | Populated fields | Total fields | Completeness |
|---|---|---|---|---|
| product | <OK / Error / N/A> | <N> | <N> | <N%> |
| productNavigation | <OK / Error / N/A> | <N> | <N> | <N%> |
| article | <OK / Error / N/A> | <N> | <N> | <N%> |
| jobPosting | <OK / Error / N/A> | <N> | <N> | <N%> |
| pageContent | <OK / Error / N/A> | <N> | <N> | <N%> |

**Best-matching type:** <type name> — <completeness>% populated

**Recommendation:** <Use AI extraction if completeness ≥ 70%. Use custom selectors if
< 50%. Suggest `extractFrom` option: `httpResponseBody` (fast/cheap) vs `browserHtml`
(higher quality on JS-heavy pages) vs `browserHtmlOnly` (most robust on render issues).>

---

## 8. Recommended Scrapy Strategy

### Primary recommendation

> **<One sentence summary of the recommended approach>**

### Configuration

```python
# scrapy-zyte-api meta for this site
# Platform: <detected platform> — <one-line platform-specific tip, e.g.
#   Shopify: consider /products.json API instead of HTML scraping
#   Next.js: parse __NEXT_DATA__ JSON blob directly from HTTP body
#   Wix: browserHtml required — heavily JS-rendered
#   Unknown: no platform detected>
meta = {
    "zyte_api_automap": {
        # [If geolocation is needed]: "geolocation": "<CC>",
        # [If IP type matters]: "ipType": "residential",
        "<httpResponseBody|browserHtml>": True,  # Fetch method
        "<product|productNavigation|article|...>": True,  # [If AI extraction is viable]
    }
}
```

### Decision rationale

1. **Fetch method:** <why httpResponseBody or browserHtml>
2. **Extraction:** <why AI extraction or custom selectors>
3. **Anti-bot:** <what Zyte API handles automatically vs what needs explicit config>
4. **Geolocation:** <whether to set explicitly>
5. **Sessions:** <whether sessions are likely needed for stateful scraping>

### Cost and efficiency notes

- <Preferred method cost tier: httpResponseBody is cheapest; browserHtml costs more>
- <AI extraction adds cost on top of fetch; custom selectors do not>
- <Residential IPs significantly increase cost — only use if datacenter IPs are blocked>
- <Any other site-specific efficiency advice>

---

## 9. Output Files

| File | Contents |
|---|---|
| `zyte-audit-http-<slug>.json` | Raw HTTP response JSON |
| `zyte-audit-browser-<slug>.json` | Raw browser response JSON (includes networkCapture) |
| `zyte-audit-geo-<slug>.json` | Raw geo-baseline response JSON |
| `zyte-audit-product-<slug>.json` | AI product extraction result |
| `zyte-audit-productnavigation-<slug>.json` | AI productNavigation extraction result |
| `zyte-audit-article-<slug>.json` | AI article extraction result |
| `zyte-audit-jobposting-<slug>.json` | AI jobPosting extraction result |
| `zyte-audit-pagecontent-<slug>.json` | AI pageContent extraction result |
| `zyte-audit-screenshot-<slug>.jpg` | Decoded browser screenshot |
| `zyte-audit-report-<slug>.md` | **This report** |
```

---

## Output

After this skill completes, the following files are present in the working directory:

```
zyte-audit-http-<slug>.json
zyte-audit-browser-<slug>.json
zyte-audit-geo-<slug>.json
zyte-audit-product-<slug>.json
zyte-audit-productnavigation-<slug>.json
zyte-audit-article-<slug>.json
zyte-audit-jobposting-<slug>.json
zyte-audit-pagecontent-<slug>.json
zyte-audit-screenshot-<slug>.jpg
zyte-audit-report-<slug>.md
```

The report (`zyte-audit-report-<slug>.md`) is the primary deliverable. Present it to the
user with a brief summary of the top recommendation.

## Known gotchas

- `networkCapture.value` must be between 3 and 8192 characters — an empty string is
  rejected with a 400 error. Use `"http"` as the minimum catch-all value to match all
  HTTP and HTTPS sub-requests.
- `statusCode` is always present in the top-level Zyte API response; it is not a field
  you need to request explicitly.
- Each extraction type (`product`, `article`, etc.) requires a **separate** Zyte API
  request — you cannot combine two extraction types in one call.
- `networkCapture` with an empty `value` and `matchType: "contains"` captures every
  sub-request. On pages with many resources this can produce a large response; the
  browser call will be slower and more expensive than a plain `browserHtml` call.
- The `screenshot` field in the browser response is Base64-encoded. Decode it before
  saving.
- If `geolocation: "US"` uses an extended geolocation (not in the standard set), it will
  incur additional cost. US is in the standard set; this is safe to use.
- Always use `uv run zyte-api`, never the bare `zyte-api` binary, to ensure the correct
  virtualenv is active.
- **Platform detection false positives:** Generic path segments like `/modules/` appear in
  many non-PrestaShop asset CDN paths. For PrestaShop, require `PrestaShop-` in cookies or
  `prestashop` in body JS — never match on `/modules/` alone. Similarly, `/v1/` and `/v2/`
  appear in versioned static asset URLs; always exclude URLs ending in `.svg`, `.css`, `.js`,
  `.png`, `.jpg`, `.woff`, `.woff2`, `.ico`, `.gif`, `.webp`, `.ttf`, `.eot` before flagging
  a URL as an API endpoint.
- **`theguardian.com` is blocked** (Zyte API returns HTTP 451 — domain forbidden). Do not
  use it as a test site. Replace with an unblocked domain.
