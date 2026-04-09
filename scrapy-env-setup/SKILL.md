---
name: scrapy-env-setup
description: Set up a Python 3.12 uv virtualenv and install Scrapy, scrapy-zyte-api, zyte-common-items, scrapy-poet, and pytest. Run once before scaffolding a new spider project.
license: MIT
compatibility: opencode
---

# scrapy-env-setup

## Purpose

Set up a Python 3.12 virtual environment with the required Scrapy packages using `uv`.

## Trigger

Load this skill when the user wants to set up or verify a Scrapy project environment.

## Instructions

### 1. Inspect the current working directory

Check for the presence of two things in the cwd:
- `.venv/` directory
- `pyproject.toml` file

### 2. Branch on what exists

**Case A — `.venv` exists, `pyproject.toml` missing:**
Stop. Tell the user:
> A `.venv` was found but no `pyproject.toml` exists. Cannot manage this environment. Set up the project with `uv` manually before using this skill.

Do not proceed further.

---

**Case B — `.venv` exists, `pyproject.toml` exists:**

1. Read `pyproject.toml` and identify which of the required packages are missing from `[project.dependencies]`:
   ```
   scrapy
   scrapy-zyte-api
   zyte-api
   zyte-common-items
   scrapy-poet
   pytest
   ```
2. For any missing packages, run:
   ```
   uv add <missing-package> [<missing-package> ...]
   ```
3. Run:
   ```
   uv sync
   ```

---

**Case C — `.venv` missing, `pyproject.toml` exists:**

Run:
```
uv add scrapy scrapy-zyte-api zyte-api zyte-common-items scrapy-poet pytest
```

---

**Case D — `.venv` missing, `pyproject.toml` missing:**

Run:
```
uv init --python 3.12
uv add scrapy scrapy-zyte-api zyte-api zyte-common-items scrapy-poet pytest
```

### 3. Verify

Confirm `.venv/` exists in the cwd. If it does, the environment is ready.

## Rules

- Only use `uv`. Never use `pip`, `pip-tools`, `poetry`, `conda`, or any other tool.
- Always use `--python 3.12` when running `uv init`.
- Only run `uv sync` in Case B (`.venv` already existed before this skill ran).

## Output

- A `.venv` directory in the cwd
- A `pyproject.toml` with all required packages listed under `[project.dependencies]`
- All packages installed and available in the virtual environment
