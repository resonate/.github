# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**resonate-utils** is a shared repository for reusable engineering utilities, diagnostics, and automation scripts. It is NOT a production service — it contains standalone tools used by engineers for automation, data checks, and operational tasks.

## Repository Structure

```
src/
├── python/
│   └── liveramp-automation/     # Playwright-based LiveRamp resend automation
│       ├── liveramp_resend.py   # Main automation class
│       └── tests/               # pytest test suite
├── scala/                       # Ad-hoc Spark/validation tools
└── shell/                       # Shell scripts for ops automation
.github/
└── workflows/
    ├── build-scala.yml          # Builds Scala fat JARs
    └── lint-python.yml          # Lints Python scripts
```

## Python Development

### LiveRamp Automation (`src/python/liveramp-automation/`)

Playwright-based browser automation for the LiveRamp segment resend workflow. Runs on ECS via scheduled execution.

**Key design pattern — always use Locators, never ElementHandles:**

```python
# CORRECT: Locators re-query the DOM on each interaction
more_btn = page.locator('[aria-label="More options"]')
more_btn.click()

# WRONG: ElementHandle can go stale after React/Pendo re-renders
more_btn = page.query_selector('[aria-label="More options"]')
more_btn.click()  # May raise "Element is not attached to the DOM"
```

The `_click_more_menu` method implements retry logic with:
1. Re-query of the element before each retry attempt (not reusing a stale handle)
2. 1000ms stabilization wait between retries
3. `force=True` fallback if Pendo still blocks after retries
4. Persistent CSS block (`display:none; pointer-events:none`) on all Pendo elements + `pendo.stopGuides()`

**Running tests:**
```bash
cd src/python/liveramp-automation
pip install -r requirements.txt
python -m pytest tests/ -v
```

**Monitoring:** CloudWatch logs at `/ecs/liveramp-automation`

### General Python Guidelines

- Use `pytest` for all tests
- Keep each utility self-contained with its own `requirements.txt`
- Add a `README.md` in each utility directory

## Scala Development

```bash
cd <tool-directory>
sbt assembly
# Submit via spark-submit
spark-submit --class com.resonate.SomeApp target/scala-2.12/your-tool.jar
```

## Key Constraints

- This repo contains **one-off tools and utilities** — not production services
- Do not add production-critical logic here; it belongs in the appropriate service repo
- The LiveRamp automation must run on ECS; keep Playwright dependencies compatible with the ECS container image
- **Never reuse stale `ElementHandle` objects** — always use Playwright `Locator` API
