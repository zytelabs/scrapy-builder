---
name: scrapy-zyte-probe
description: Probe one or more URLs via the Zyte API CLI to determine the fetch method needed, then save one HTML sample and one result JSON per URL for downstream skills.
license: MIT
compatibility: opencode
---

# scrapy-zyte-probe

## Purpose

Probe one or more URLs using the Zyte API to determine whether `httpResponseBody` or `browserHtml` is needed, then save one result file per URL for the next skill to read.

## Trigger

Load this skill when the user asks to scrape or interrogate one or more URLs before building a spider.

## Instructions

### 1. Get the URLs

Collect all URLs from the user. There may be one or many. If no URLs were provided, ask.

### 2. Check prerequisites

Verify `ZYTE_API_KEY` is set in the environment:

```
test -n "$ZYTE_API_KEY" && echo "ZYTE_API_KEY is set" || echo "ZYTE_API_KEY is not set"
```

If it is empty or unset, stop and tell the user:
> `ZYTE_API_KEY` is not set. Export it in your shell before running this skill.

Do not proceed.

### 3. Derive a slug for each URL

For each URL, derive a short filesystem-safe slug used in all output filenames:

- Take the URL path, strip leading/trailing slashes
- Replace `/`, `.`, `-`, `?`, `=`, `&` with `_`
- Truncate to 60 characters
- If the slug is empty (e.g. the URL is just a domain), use `index`

Examples:
- `https://www.scan.co.uk/shop/gpu-amd/all` → `shop_gpu_amd_all`
- `https://www.scan.co.uk/products/some-product` → `products_some_product`
- `https://www.scan.co.uk/` → `index`

### 4. Try `httpResponseBody` for all URLs in one batch

Write a single temporary input file `probe-http.jsonl` containing one JSON line per URL:

```
{"url": "<url1>", "httpResponseBody": true, "httpResponseHeaders": true}
{"url": "<url2>", "httpResponseBody": true, "httpResponseHeaders": true}
```

Run:

```
uv run zyte-api probe-http.jsonl -o probe-http-result.jsonl --store-errors
```

Delete `probe-http.jsonl` immediately after the command completes.

Parse `probe-http-result.jsonl` — each line is a result keyed by `url`:
- Check `httpResponseStatus`. A status of `200` is required.
- Base64-decode `httpResponseBody`.
- Check the decoded content contains a non-empty HTML document (presence of `<html` or `<body` tag).

**Success condition per URL:** status is `200` AND decoded body contains HTML.

Separate URLs into two groups:
- **succeeded** — record `fetch_method: "httpResponseBody"`, `http_status`, decoded body
- **failed** — carry forward to step 5

Delete `probe-http-result.jsonl`.

### 5. Fall back to `browserHtml` for failed URLs

If any URLs failed step 4, write `probe-browser.jsonl` containing one line per failed URL:

```
{"url": "<failed_url1>", "browserHtml": true}
{"url": "<failed_url2>", "browserHtml": true}
```

Run:

```
uv run zyte-api probe-browser.jsonl -o probe-browser-result.jsonl --store-errors
```

Delete `probe-browser.jsonl` immediately after the command completes.

Parse `probe-browser-result.jsonl`:
- Check `browserHtml` field is present and non-empty.

For each URL:
- **succeeded** — record `fetch_method: "browserHtml"`, `http_status: null`, `browserHtml` string
- **failed** — add to the total failure list

Delete `probe-browser-result.jsonl`.

### 6. Handle total failures

If any URL failed both steps 4 and 5, report its error response to the user. Do not write output files for that URL. Continue processing URLs that succeeded.

If **all** URLs failed, stop entirely.

### 7. Write output files — one set per successful URL

For each successfully probed URL, write two files using the URL's slug:

**`zyte-probe-result-<slug>.json`:**

```json
{
  "url": "<the tested url>",
  "fetch_method": "httpResponseBody",
  "http_status": 200,
  "html_file": "zyte-probe-sample-<slug>.html"
}
```

**`zyte-probe-sample-<slug>.html`:** the decoded HTML body.

### 8. Clean up

Ensure all temp files are deleted:
- `probe-http.jsonl`
- `probe-http-result.jsonl`
- `probe-browser.jsonl`
- `probe-browser-result.jsonl`

These should already be deleted in steps 4 and 5, but remove any that still exist.

## Rules

- Always try `httpResponseBody` first — it is cheaper than `browserHtml`.
- Batch all HTTP probes into a single `zyte-api` call. Do not call the CLI once per URL.
- Only fall back to `browserHtml` for URLs that failed the HTTP probe.
- Batch all browser fallback probes into a single `zyte-api` call.
- Never hardcode the API key. It is read from `ZYTE_API_KEY` automatically by the CLI.
- Always use `uv run zyte-api`, not `zyte-api` directly.
- Always delete temp `.jsonl` files after use.
- Output filenames always include the URL slug — never use bare `zyte-probe-result.json`.

## Output

One pair of files per successfully probed URL:

- `zyte-probe-result-<slug>.json` — machine-readable probe result for the next skill
- `zyte-probe-sample-<slug>.html` — the fetched HTML for inspection
