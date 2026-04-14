---
name: scrapy-deploy
description: Deploy a Scrapy project to Scrapy Cloud using shub. Detects the correct stack version from pyproject.toml and available branches, generates requirements.txt from project dependencies, writes scrapinghub.yml, and reports the deployed version and project URL.
license: MIT
compatibility:
  - opencode
  - claude-code
---

# scrapy-deploy

## Purpose

Deploy an existing Scrapy project to Scrapy Cloud using `shub deploy`, automatically configuring the correct stack version and generating a `requirements.txt` for any dependencies not already bundled in the stack.

## Trigger

Load this skill when the user wants to deploy a Scrapy project to Scrapy Cloud.

This skill is **standalone** — it is not called by any other skill in this project.

## Instructions

### 1. Verify prerequisites

Run all three checks as **concurrent tool calls** — issue all in a single parallel batch.

**API key check:**

```bash
test -n "$SHUB_APIKEY" && echo "set" || echo "not set"
```

If `SHUB_APIKEY` is unset, also check whether `~/.scrapinghub.yml` exists (created by `shub login`). If neither is present, stop and tell the user:
> Set your Scrapy Cloud API key before deploying:
> ```
> export SHUB_APIKEY=<your_api_key>
> ```
> Your API key is available at https://app.zyte.com/account/apikey

**Project check:** Confirm a `scrapy.cfg` exists in the working directory. If not, stop and tell the user this skill must be run from a Scrapy project directory (where `scrapy.cfg` lives).

**shub availability check:** Check whether `shub` is installed in the `.venv`:

```bash
uv run shub --version 2>/dev/null && echo "ok" || echo "missing"
```

If `shub` is missing, install it before continuing:

```bash
uv add shub
```

Wait for all three checks to complete. Only continue to Step 2 if the API key is present (env var or `~/.scrapinghub.yml`) and `scrapy.cfg` exists.

### 2. Resolve project ID, stack version, and dependencies

Run all three sub-tasks as **concurrent tool calls** — they are independent and can be issued in a single parallel batch.

#### Sub-task A — Resolve project ID

Read `scrapinghub.yml` in the working directory if it exists.

**Case A — `scrapinghub.yml` exists and has a `project` or `projects.default` field:**

Parse the project ID from the file. No user prompt needed.

A simple single-project `scrapinghub.yml` looks like:
```yaml
project: 12345
stack: scrapy:2.14
requirements:
  file: requirements.txt
```

A multi-target file uses:
```yaml
projects:
  default: 12345
  prod: 33333
```

Read the value of `project` (simple) or `projects.default` (multi-target). If both keys are present, `projects.default` takes precedence.

**Case B — `scrapinghub.yml` is absent, or present but has no project ID:**

Ask the user:
> What is your Scrapy Cloud project ID? (Find it in the URL: https://app.zyte.com/p/**12345**/dashboard)

Store the supplied ID for use in Step 3.

#### Sub-task B — Determine stack version

1. Read `pyproject.toml` and find the `scrapy` entry under `[project.dependencies]`. The entry will look like one of:
   - `"scrapy>=2.14.0"` → extract `2.14`
   - `"scrapy==2.14.1"` → extract `2.14`
   - `"scrapy~=2.14"` → extract `2.14`
   - `"scrapy"` (no version) → treat as no version constraint

   Strip all version operators (`>=`, `==`, `~=`, `!=`, `^`, `<=`, `<`, `>`) and take only the first two version segments (`major.minor`).

2. Fetch the available stack branches from the GitHub API:

   ```bash
   curl -s "https://api.github.com/repos/scrapinghub/scrapinghub-stack-scrapy/branches?per_page=100" \
     | python3 -c "
   import json, sys, re
   branches = json.load(sys.stdin)
   versions = []
   for b in branches:
       m = re.match(r'^branch-(\d+\.\d+)$', b['name'])
       if m:
           parts = tuple(int(x) for x in m.group(1).split('.'))
           versions.append(parts)
   for v in sorted(versions):
       print(f'{v[0]}.{v[1]}')
   "
   ```

   This prints all available versions in ascending order (e.g. `2.13`, `2.14`, `2.15`).

3. Match the Scrapy version from `pyproject.toml` against the available branches:

   | Condition | Stack to use |
   |---|---|
   | Exact `major.minor` match found | Use that branch (e.g. `2.14` → `scrapy:2.14`) |
   | No exact match — use highest available branch ≤ project version | e.g. project `>=2.12`, branches `2.13`/`2.14`/`2.15` → use `scrapy:2.14` (highest ≤ constraint) |
   | No branch ≤ project version (all branches are higher) | Use the lowest available branch; warn the user that the stack version is higher than the pinned Scrapy version |
   | No Scrapy entry in `pyproject.toml` | Use the highest available branch (latest) |

   For the "highest available ≤ project version" case, extract the numeric minimum from the constraint (e.g. `>=2.12` → minimum `2.12`), then select the highest branch whose version ≤ that minimum is false — i.e. the highest branch `<= minimum` is the largest branch version that does not exceed the project's minimum. In practice: filter branches to those where `branch_version <= project_major_minor`, then take the maximum.

#### Sub-task C — Collect extra dependencies

1. Read `[project.dependencies]` from `pyproject.toml`.
2. Extract the bare package name from each entry (strip version specifiers).
3. Normalise each name: lowercase, replace `_` with `-`.
4. Exclude every package whose normalised name appears in the **stack exclusion list** below.
5. Also exclude `pytest` and any package whose normalised name starts with `pytest-`.
6. The remaining packages are the extra dependencies to deploy.

**Stack exclusion list** — packages already bundled in every Scrapy Cloud stack and must NOT appear in `requirements.txt`:

```
aiohappyeyeballs, aiohttp, aiosignal, arrow, attrs, automat, awscli,
boto, boto3, botocore, cachetools, certifi, cffi, charset-normalizer,
colorama, constantly, cryptography, cssselect, cssutils, defusedxml,
docutils, filelock, fqdn, frozenlist, h2, hpack, hyperframe, hyperlink,
idna, incremental, isoduration, itemadapter, itemloaders, jinja2,
jmespath, jsonpointer, jsonschema, jsonschema-specifications, lxml,
markupsafe, monkeylearn, more-itertools, multidict, packaging, parsel,
pillow, premailer, priority, propcache, protego, pyasn1, pyasn1-modules,
pycparser, pydispatcher, pyopenssl, python-dateutil, python-slugify,
pyyaml, queuelib, referencing, requests, requests-file, retrying,
rfc3339-validator, rfc3987, rpds-py, rsa, s3transfer, scrapinghub,
scrapinghub-entrypoint-scrapy, scrapy, scrapy-crawlera, scrapy-deltafetch,
scrapy-dotpersistence, scrapy-magicfields, scrapy-querycleaner,
scrapy-splash, scrapy-splitvariants, scrapy-zyte-smartproxy, sentry-sdk,
service-identity, setuptools, six, slack-sdk, spidermon, text-unidecode,
tldextract, twisted, types-python-dateutil, typing-extensions,
uri-template, urllib3, w3lib, webcolors, yarl, zope-interface
```

Wait for all three sub-tasks to complete before proceeding to Step 3.

### 3. Write `requirements.txt`

If the extra dependencies list from Sub-task C is non-empty, write `requirements.txt` in the working directory:

```
# Extra dependencies not bundled in the Scrapy Cloud stack
<package-1>
<package-2>
...
```

List packages alphabetically, one per line, with no version pins. Do not include a version pin — the stack provides a compatible environment and version conflicts are resolved at deploy time.

If the extra dependencies list is empty, do not create `requirements.txt` and skip the `requirements` key in `scrapinghub.yml` (Step 4).

**Example** — project with `scrapy-zyte-api`, `scrapy-poet`, `zyte-common-items`, `zyte-api`, `pytest` in `pyproject.toml`:

```
# Extra dependencies not bundled in the Scrapy Cloud stack
scrapy-poet
scrapy-zyte-api
zyte-api
zyte-common-items
```

(`pytest` is excluded as a dev-only package; all four others are absent from the stack exclusion list.)

### 4. Write `scrapinghub.yml`

Write (or overwrite) `scrapinghub.yml` in the working directory with the project ID, stack, and requirements reference:

**With extra dependencies (`requirements.txt` was written):**

```yaml
project: <project_id>
stack: scrapy:<stack_version>
requirements:
  file: requirements.txt
```

**Without extra dependencies:**

```yaml
project: <project_id>
stack: scrapy:<stack_version>
```

If `scrapinghub.yml` already existed with a multi-target `projects:` block, preserve the existing structure and only update/add the `stack` and `requirements` keys. Do not collapse a multi-target config into a single `project:` key.

**Example** for a project using Scrapy 2.14 with `scrapy-zyte-api` and `scrapy-poet`:

```yaml
project: 12345
stack: scrapy:2.14
requirements:
  file: requirements.txt
```

### 5. Deploy

Run:

```bash
uv run shub deploy -v
```

No project ID argument is needed — `scrapinghub.yml` now contains the `project` field and `shub` reads it automatically. The `-v` flag streams deploy progress to the console.

A successful deploy produces output like:

```
Packing version 3af023e-master
Deploying to Scrapy Cloud project "12345"
{"status": "ok", "project": 12345, "version": "3af023e-master", "spiders": 1}
Run your spiders at: https://app.zyte.com/p/12345/
```

If the command exits non-zero or the output contains `"status": "error"`, the deploy failed — report the full error output and stop. Do not proceed to Step 6.

### 6. Verify and report

Parse the JSON line from the deploy output and confirm `"status": "ok"`.

Extract:
- `version` — the deployed version string
- `spiders` — number of spiders registered
- `project` — the project ID

Report to the user:

```
Deployed successfully.

  Version:  <version>
  Spiders:  <N>
  Stack:    scrapy:<stack_version>
  Project:  https://app.zyte.com/p/<project_id>/

To schedule a spider run:
  uv run shub schedule <spider_name>

Or use the Scrapy Cloud dashboard:
  https://app.zyte.com/p/<project_id>/
```

If the spider name is known (from `scrapy list` output or the project files), use it directly in the `shub schedule` example. Otherwise use `<spider_name>` as a placeholder.

## Rules

- Always use `uv run shub`, never call `shub` directly — the virtualenv must be active.
- Never hardcode `SHUB_APIKEY` in any file.
- Always fetch the branch list live from the GitHub API — never hardcode a stack version.
- Use `?per_page=100` on the GitHub API call to ensure all branches are returned.
- Normalise package names before exclusion list comparison: lowercase and treat `-` and `_` as equivalent (PEP 503).
- Never include version pins in `requirements.txt` — bare package names only.
- Never include `scrapy` itself in `requirements.txt` — it is always provided by the stack.
- Never include `pytest` or `pytest-*` packages in `requirements.txt` — dev-only.
- If `scrapinghub.yml` already has a multi-target `projects:` block, preserve it — do not flatten to `project:`.
- The deploy command must be run from the project root (where `scrapy.cfg` lives).

## Known gotchas

- **`SHUB_APIKEY` vs `SH_APIKEY`** — `shub` accepts both env var names. `SHUB_APIKEY` is the documented name; `SH_APIKEY` is an older alias. Either works. `shub login` writes to `~/.scrapinghub.yml` — check that file before requiring the env var.
- **Version auto-detection** — `shub` uses the git commit hash as the version tag by default (`AUTO` mode). If the project is not a git repo, the version will be a timestamp. This is expected.
- **`uv add shub` modifies `pyproject.toml`** — if `shub` was not in the original project dependencies, `uv add shub` adds it. This causes `shub` to appear in `[project.dependencies]`. The exclusion list does not include `shub`, so it would be written to `requirements.txt`. However, `shub` is a deploy tool and not needed at runtime — exclude `shub` from `requirements.txt` as well.
- **GitHub API rate limit** — unauthenticated requests are limited to 60/hour. The branch fetch is a single call and will not hit this limit in normal use.
- **`scrapy.cfg` must be present** — `shub deploy` reads `scrapy.cfg` to find the project package. Running from a directory without `scrapy.cfg` will fail with a confusing error.
- **Egg size limit** — Scrapy Cloud has a default egg size limit. If the deploy fails with a size error, run `uv run shub deploy --ignore-size` to bypass the check (not recommended for production).
- **`jsonschema[format]` and `twisted[http2]`** — the stack exclusion list uses the bare name (`jsonschema`, `twisted`). When normalising project dependencies, strip any extras in brackets before comparing (e.g. `twisted[http2]` → `twisted`).
- **Stack branch naming** — only branches matching `branch-X.XX` (exactly two numeric segments) are considered. Branches like `main`, `master`, or `branch-2.15.1` are ignored.

## Output

```
requirements.txt     ← written if extra dependencies exist (omitted if empty)
scrapinghub.yml      ← written with project ID, stack version, and requirements reference
```

The deploy itself produces no additional local files. The deployed egg is stored on Scrapy Cloud.
