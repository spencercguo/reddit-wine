<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SCRP-01 | Fetch posts from r/wine via old.reddit.com JSON API with pagination (up to 2000+ posts), respecting 2 req/sec rate limit | requests 2.34.2 for HTTP, tenacity 9.2.1 for rate limiting with exponential backoff + jitter, old.reddit.com/.json pagination via `after` cursor parameter |
| SCRP-02 | For image posts (empty selftext), fetch OP's first top-level comment and merge into review_text | requests GET to comments endpoint, author matching, post_hint detection |
| SCRP-03 | Validate Content-Type of all API responses; handle non-JSON (HTML) responses with retry and circuit breaker | Content-Type header check before .json(), tenacity retry with retry_if_exception_type, circuit breaker via consecutive failure counter in retry_state |
| SCRP-04 | Save raw JSON posts to data/raw/ with scrape run metadata (timestamp, count, errors) | json.dump per-line JSONL, manifest JSON sidecar |
| SCRP-05 | Filter non-review posts using token gate (>=30 tokens) and multi-signal keyword gate | Pure Python functions, keyword config from JSON file, exclusion lists take priority |
| SCRP-06 | Log skipped posts with rejection reason for pipeline auditability | Structured reject_reason strings, summary stats at end, printed via click.echo to stderr |
| ORCH-03 | Manifest/provenance system tracking which posts were processed by each pipeline stage | JSON manifest sidecar with run metadata, last_post_id for resume, separate filter pass adds review_flag and reject_reason |

**Coverage:** All 7 phase requirements are anchored to specific libraries, patterns, and verified versions.
</phase_requirements>

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Use uv for package management with pyproject.toml. Project structure follows the layout in `.planning/specs/PLAN.md` (src/scrape.py, data/raw/, etc.). Python 3.13 is the runtime.
- **D-02:** Single `scrape` command using click. All parameters are configurable flags with sensible defaults: `--subreddit` (default: wine), `--limit` (number of posts to fetch), `--output-dir` (default: data/raw/), `--rate-limit` (default: 2.0 req/sec), `--min-tokens` (default: 30), `--keyword-file` (default: data/filters/keywords.json), `--fresh` (skip resume, start new scrape).
- **D-03:** Output shows current page number, posts scraped count, and a final summary with counts (total scraped, filtered in, rejected, errors). Warnings printed to stderr. Continue on recoverable errors; circuit breaker still halts on consecutive failures.
- **D-04:** Filtering is a separate pass — scraping saves all raw posts first, then a filter pass reads the JSONL and adds `review_flag` (boolean) and `reject_reason` (string) fields. Enables re-filtering with updated rules without re-scraping.
- **D-05:** Single JSONL file per scrape session with timestamped naming (`rwine_2026-11-27T16-03.jsonl`). One JSON object per line. All posts (raw) go into one file with `review_flag` (true/false) and `reject_reason` (null or string) fields added by the filter pass. Sidecar manifest at same path (`manifest_2026-11-27T16-03.json`).
- **D-06:** Manifest is run metadata only: timestamp, subreddit, posts_requested, posts_returned, posts_filtered_in, posts_rejected, errors_count, duration_seconds, last_post_id (for resume). No full post ID list — IDs live in the JSONL.
- **D-07:** Use tenacity's exponential backoff (2 req/sec base rate with jitter). Validate Content-Type on every response — if a non-JSON response is received, retry up to 3 times. After 5 consecutive non-JSON responses across different requests, trigger a circuit breaker and halt. Log every retry attempt to stderr.
- **D-08:** 30-token minimum gate + multi-signal keyword gate. Keywords grouped into: sensory (tannin, oak, acidity, finish, palate, nose, aroma, bouquet, body), structure cues (price, paid, bought, $, retail), score cues (points, pts, /100, score, rating), and varietal/region names. Binary gate: pass if >=1 keyword match across any category.
- **D-09:** Keywords and token threshold read from `data/filters/keywords.json` config file. CLI overridable via `--keyword-file` and `--min-tokens`. Enables filter tuning without code changes.
- **D-10:** Exclusion keyword list in the same config — posts matching any exclusion term (e.g., "looking for recommendations", "where can I buy", "wedding wine suggestion") are rejected regardless of positive keyword matches. Rejection reason records which exclusion term triggered.
- **D-11:** Skip non-text posts before keyword gate: link posts (selftext = empty + URL only), AutoModerator posts (author == "AutoModerator"), and stickied posts. Process only self-posts and image posts (with OP comment fetch). Rejection reason logged as `post_type_link`, `post_type_auto`, or `post_type_sticky`.
- **D-12:** Resume by default — scraper reads last_post_id from most recent manifest for the same subreddit and sends `?after={last_post_id}` to continue pagination. `--fresh` flag starts a fresh scrape (creates new timestamped file).
- **D-13:** On resume, append to the existing timestamped JSONL file, updating the same manifest. Before resuming forward pagination, scan existing posts for image posts missing OP comments and refetch those comments.
- **D-14:** Manifest system as described in D-06. Enables incremental scrapes and re-extraction after rule changes. Simple JSON-based, no database.
- **D-15:** Use Path B — lightweight starter gazetteer (~40 regions, ~60 varietals, common aliases). Run the sample pipeline first, let the candidate queue (unmapped_spans.csv from entity linking) tell us what to add. Iteratively grow. Carried forward to Phase 2.

### the agent's Discretion

- How to structure the filter module (separate file vs. inline in scrape.py)
- Exact keyword.json schema and initial seed keywords (within the categories locked by D-08)
- Exact circuit breaker implementation strategy (retry_state callback vs. decorator composition)
- How to handle edge cases: deleted posts `[removed]`, rate-limited posts, empty listing responses
- Test fixture design and coverage decisions
- Error message formatting and logging verbosity

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

# Phase 01: Pipeline Foundation - Research

**Researched:** 2026-11-27
**Domain:** Web scraping, HTTP resilience, data filtering, ETL provenance tracking
**Confidence:** HIGH

## Summary

Phase 1 establishes the data acquisition foundation of the r/wine Review Analysis pipeline. The primary work is building a command-line scraper that fetches posts from `old.reddit.com/r/wine/.json`, handles the fragile unofficial Reddit JSON API with robust Content-Type validation and circuit breaker patterns, filters non-review posts through a two-pass architecture (scrape-then-filter), and persists everything with auditable provenance tracking via JSONL + manifest sidecar files.

The key technical risks are (1) Reddit's unpredictable response behavior (HTML instead of JSON with 200 OK), addressed by Content-Type validation with a 5-consecutive-failure circuit breaker, and (2) review/non-review classification accuracy, addressed by a configurable multi-signal keyword gate with exclusion lists that can be tuned without code changes. The scraping, filtering, and provenance layers are decoupled — re-filtering with updated rules never requires re-scraping.

**Primary recommendation:** Build the scraper and filter as separate `src/` modules sharing a common data contract (JSONL with `review_flag` + `reject_reason`). Keep the filter's keyword configuration in a JSON file, not in code. Test Content-Type validation and HTML detection agressively before any real scraping.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| HTTP request/response handling | API/Backend | — | requests performs direct HTTP calls from the pipeline script; no browser or SSR involved |
| Rate limiting & retry | API/Backend | — | tenacity decorators wrap network calls server-side; no client-side throttling |
| Content-Type validation & circuit breaker | API/Backend | — | Response inspection happens post-fetch in Python; decision to halt is programmatic |
| JSONL file I/O | Database/Storage | — | All persistence is local filesystem; no external database |
| Manifest/provenance | Database/Storage | — | JSON sidecar files on disk; no workflow engine |
| CLI command parsing | API/Backend | — | click processes argv and dispatches to Python functions |
| Post filtering (token + keyword gates) | API/Backend | — | Pure Python logic operating on in-memory dicts or JSONL streams |
| OP comment fetch (image posts) | API/Backend | — | Additional requests.get() call within the scraping loop |

## Standard Stack

### Core (Phase 1 Only)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| requests | 2.34.2 | HTTP client for Reddit JSON API | De facto Python HTTP library. Supports session reuse, timeout control, custom headers, and response.json(). Reddit's old.reddit.com/.json endpoint returns structured JSON parseable natively. [VERIFIED: https://requests.readthedocs.io/en/latest/] |
| tenacity | 9.2.1 | Retry logic with exponential backoff | Apache 2.0 licensed, actively maintained. `@retry(wait=wait_exponential(multiplier=1, min=0.5, max=10) + wait_random(0, 1))` enforces 2 req/sec with jitter. `before_sleep` callback logs retry attempts. [VERIFIED: https://tenacity.readthedocs.io/en/latest/] |
| click | 8.5.1 | CLI interface for scrape/filter commands | Pallets project (Flask maintainers). Clean decorator-based API. `@click.command()`, `@click.option()`, `click.echo()` to stderr. Auto-generated help pages. Testing via `click.testing.CliRunner`. [VERIFIED: https://click.palletsprojects.com/en/stable/] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| pytest | 9.2.0 | Testing framework | All test files. Fixtures for mock Reddit responses. Parametrized tests for filter edge cases. |
| json (stdlib) | built-in | JSONL serialization/deserialization | `json.dumps(post) + '\n'` per line; `json.load()` for manifest reading |
| logging (stdlib) | built-in | Structured logging to stderr | Retry warnings, rejection reasons, scrape summary |

### Alternatives Considered (Phase 1)
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| requests | httpx | httpx adds async complexity with no benefit — pipeline is CPU-bound, not I/O-bound. Synchronous requests is simpler and sufficient. Already decided. |
| click | argparse | argparse is stdlib but requires more boilerplate for subcommands, help text, and type validation. click's decorator API is cleaner. Already decided. |
| tenacity | stamina | stamina is newer and less battle-tested. tenacity has 9+ years of production use, better docs, and more flexible retry condition combinators (| operator). |

**Installation:**
```bash
uv venv .venv
source .venv/bin/activate
uv pip install requests==2.34.2 tenacity==9.2.1 click==8.5.1
uv pip install pytest==9.2.0
```

**Version verification:**
- `requests`: 2.34.2 (latest as of 2026-11-27) [VERIFIED: pip index]
- `tenacity`: 9.2.1 (latest as of 2026-11-27) [VERIFIED: pip index]
- `click`: 8.5.1 (latest as of 2026-11-27) [VERIFIED: pip index]
- `pytest`: 9.2.0 (latest as of 2026-11-27) [VERIFIED: pip index]

## Package Legitimacy Audit

| Package | Registry | Age | Downloads | Source Repo | Verdict | Disposition |
|---------|----------|-----|-----------|-------------|---------|-------------|
| requests | PyPI | 14+ yrs | ~500M+/mo | https://github.com/psf/requests | OK | Approved |
| tenacity | PyPI | 9+ yrs | ~30M+/mo | https://github.com/jd/tenacity | OK | Approved |
| click | PyPI | 12+ yrs | ~100M+/mo | https://github.com/pallets/click/ | OK | Approved |
| pytest | PyPI | 14+ yrs | ~100M+/mo | https://github.com/pytest-dev/pytest | OK | Approved |

**Packages removed due to [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

*Note: The automated legitimacy checker returned `SUS` for all four packages due to `unknown-downloads` (download counts unavailable via the npm-centric registry check on the PyPI ecosystem). All four are among the most downloaded Python packages in existence — requests (psf), click (pallets), tenacity (jd), and pytest (pytest-dev) — each with 10+ years of history and maintained by well-known organizations. These are safe to install.*

## Architecture Patterns

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CLI LAYER (click)                          │
│                                                               │
│  $ python -m src.scrape --subreddit wine --limit 100          │
│  $ python -m src.filter --input data/raw/rwine_*.jsonl        │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│                    SCRAPE LAYER                               │
│                                                               │
│  POST FETCH LOOP                                  RESILIENCE │
│  ┌──────────────┐       ┌──────────────┐      ┌──────────┐  │
│  │ old.reddit   │──GET──▶│ Content-Type │─JSON─▶│ Parse   │  │
│  │ .json?after= │       │ validation   │      │ JSON    │  │
│  └───────┬──────┘       └──────┬───────┘      └────┬─────┘  │
│          │                     │                    │        │
│          │              ┌──────▼───────┐            │        │
│          │              │ HTML detected│            │        │
│          │              │ → retry up   │            │        │
│          │              │   to 3x      │            │        │
│          │              │ → circuit    │            │        │
│          │              │   breaker    │            │        │
│          │              │   at 5 consec│            │        │
│          │              └──────────────┘            │        │
│          │                                          │        │
│    ┌─────▼──────┐     ┌──────────────┐     ┌───────▼──────┐ │
│    │ Image post │     │ Deleted post │     │ Write to     │ │
│    │ detection  │     │ detection    │     │ JSONL file    │ │
│    │ → fetch OP │     │ → skip with  │     │ (append mode) │ │
│    │   comment  │     │   reason     │     └───────┬──────┘ │
│    └─────┬──────┘     └──────┬───────┘             │        │
│          │                   │                     │        │
│    ┌─────▼───────────────────▼─────────────────────▼──────┐ │
│    │              WRITE TO JSONL FILE                      │ │
│    │              data/raw/rwine_2026-11-27T16-03.jsonl    │ │
│    │              WRITE MANIFEST                           │ │
│    │              data/raw/manifest_2026-11-27T16-03.json  │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
├───────────────────────────────┼──────────────────────────────┤
│                    FILTER LAYER                              │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │              READ JSONL FILE                          │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │  POST TYPE GATE (D-11)                                │ │
│    │  ├─ AutoModerator? → reject: post_type_auto           │ │
│    │  ├─ Stickied post? → reject: post_type_sticky          │ │
│    │  ├─ Link post (selftext=empty, no image)?             │ │
│    │  │  → reject: post_type_link                          │ │
│    │  └─ Pass → continue to token gate                     │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │  TOKEN GATE (D-08)                                    │ │
│    │  review_text tokens < 30? → reject: token_gate_fail   │ │
│    │  Pass → continue to keyword gates                     │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │  EXCLUSION GATE (D-10)                                │ │
│    │  Match against exclusion keywords?                    │ │
│    │  → reject with specific exclusion reason              │ │
│    │  Pass → continue to positive keyword gate             │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │  POSITIVE KEYWORD GATE (D-08)                         │ │
│    │  >= 1 match across sensory/price/score/region groups? │ │
│    │  → review_flag: true, reject_reason: null             │ │
│    │  No matches? → review_flag: false,                    │ │
│    │    reject_reason: keyword_gate_fail                   │ │
│    └──────────────────────────┬───────────────────────────┘ │
│                               │                              │
│    ┌──────────────────────────▼───────────────────────────┐ │
│    │  WRITE BACK TO JSONL (adds review_flag +             │ │
│    │  reject_reason to each post object)                    │ │
│    │  UPDATE MANIFEST (posts_filtered_in, posts_rejected)  │ │
│    └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Recommended Project Structure

```
reddit_wine/
├── pyproject.toml
├── src/
│   ├── __init__.py           # (exists, empty)
│   ├── scrape.py             # CLI entry: scrape command, fetch loop, resilience
│   ├── filter.py             # CLI entry: filter command, gate logic
│   └── storage.py            # JSONL read/write, manifest helpers
├── data/
│   ├── raw/                  # Scraped JSONL + manifest files (.gitignored)
│   └── filters/
│       └── keywords.json     # Filter configuration (committed to git)
└── tests/
    ├── conftest.py           # Shared fixtures: mock Reddit responses
    ├── test_scrape.py        # Scraper unit tests (fetch, retry, circuit breaker)
    └── test_filter.py        # Filter unit tests (token gate, keyword gate, exclusion)
```

**Rationale:** Flat `src/` modules (not nested packages) per CONTEXT.md code_context. `storage.py` extracted to avoid circular dependencies between scrape and filter (both import JSONL helpers). `data/raw/` is .gitignored; `data/filters/keywords.json` is tracked. Tests mirror `src/` 1:1.

### Pattern 1: Content-Type Validation Before JSON Parsing

**What:** Every Reddit API response is checked for `Content-Type: application/json` before calling `response.json()`. Non-JSON responses trigger retry or circuit breaker.

**When to use:** Every request to old.reddit.com.

**Example:**
```python
# Source: requests docs [VERIFIED: https://requests.readthedocs.io/en/latest/]
import requests
from requests.exceptions import JSONDecodeError

def safe_fetch_json(url: str, headers: dict, timeout: int = 30) -> dict:
    """Fetch JSON from Reddit, validating Content-Type before parsing."""
    r = requests.get(url, headers=headers, timeout=timeout)

    content_type = r.headers.get("content-type", "")
    if "application/json" not in content_type:
        body_preview = r.text[:500]
        raise ValueError(
            f"Non-JSON response: Content-Type={content_type}, "
            f"status={r.status_code}, body={body_preview}"
        )

    return r.json()
```

### Pattern 2: Tenacity Retry with Jitter and Circuit Breaker

**What:** Exposeential backoff with random jitter for rate limiting, combined with a consecutive-failure counter that halts after 5 sequential non-JSON responses.

**When to use:** Wrapping every network call to Reddit.

**Example:**
```python
# Source: tenacity docs [VERIFIED: https://tenacity.readthedocs.io/en/latest/]
from tenacity import (
    retry, stop_after_attempt, wait_exponential, wait_random,
    retry_if_exception_type, before_sleep, RetryError
)
import sys

CIRCUIT_BREAKER_LIMIT = 5
consecutive_failures = 0

def log_retry(retry_state):
    print(
        f"[RETRY] Attempt {retry_state.attempt_number} "
        f"after {retry_state.outcome.exception()}",
        file=sys.stderr
    )

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=0.5, min=0.5, max=10) + wait_random(0, 1),
    retry=retry_if_exception_type((ValueError, JSONDecodeError)),
    before_sleep=log_retry,
    reraise=True,
)
def fetch_with_retry(url: str, headers: dict) -> dict:
    global consecutive_failures
    try:
        result = safe_fetch_json(url, headers)
        consecutive_failures = 0  # reset on success
        return result
    except (ValueError, JSONDecodeError):
        consecutive_failures += 1
        if consecutive_failures >= CIRCUIT_BREAKER_LIMIT:
            raise CircuitBreakerError(
                f"Circuit breaker tripped after {CIRCUIT_BREAKER_LIMIT} "
                f"consecutive non-JSON responses"
            )
        raise
```

### Pattern 3: JSONL Append-Write with Line Delimiters

**What:** Each Reddit post is serialized as a single JSON object per line (`json.dumps(post) + '\n'`). The file is opened in append mode for incremental writes. No in-memory accumulation of all posts.

**When to use:** Writing scraped posts to `data/raw/`.

**Example:**
```python
# Source: json stdlib docs [CITED: https://docs.python.org/3/library/json.html]
import json
from pathlib import Path

def append_post_to_jsonl(post: dict, filepath: Path) -> None:
    """Append a single post object as a JSON line to a JSONL file."""
    filepath.parent.mkdir(parents=True, exist_ok=True)
    with open(filepath, "a", encoding="utf-8") as f:
        f.write(json.dumps(post, ensure_ascii=False) + "\n")

def read_jsonl(filepath: Path) -> list[dict]:
    """Read all JSON objects from a JSONL file."""
    posts = []
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                posts.append(json.loads(line))
    return posts
```

### Pattern 4: Two-Pass Scrape-Then-Filter Architecture

**What:** Scraping writes all raw posts (no classification). A separate filter pass reads the JSONL, applies gates, and writes back `review_flag` and `reject_reason` to each record. This decoupling enables re-filtering with updated rules without re-scraping.

**When to use:** The entire pipeline — locked by D-04.

**Example:**
```python
# Scrape pass: save raw posts without classification
for post in fetch_posts():
    append_post_to_jsonl(post, jsonl_path)

# Filter pass: add review_flag and reject_reason
posts = read_jsonl(jsonl_path)
for post in posts:
    result = classify_post(post, keywords_config)
    post["review_flag"] = result.is_review
    post["reject_reason"] = result.reject_reason

# Rewrite JSONL with classification fields
rewrite_jsonl(posts, jsonl_path)  # overwrites same file, now includes flags
```

### Anti-Patterns to Avoid

- **Anti-pattern: `response.json()` without Content-Type check.** If Reddit returns HTML (login page, block, GDPR consent), `json()` raises cryptic `JSONDecodeError` that looks like corrupted data, not a policy block. Always check Content-Type first. [CITED: PITFALLS.md P1]
- **Anti-pattern: Filtering inline during scraping.** Mixing classification with fetching means changing filter rules requires re-scraping. Separate passes enable rule iteration without touching the network.
- **Anti-pattern: Hardcoding keywords in Python.** Every filter tuning cycle requires a code change and tests. JSON config file enables tuning without touching code (D-09).
- **Anti-pattern: Not logging rejection reasons.** If 40% of posts are filtered, you cannot audit whether real reviews were lost. Every `reject_reason` must be specific and auditable.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP retry with backoff | Custom `while` loop with `time.sleep()` | tenacity `@retry` decorator | tenacity handles edge cases: max retries, jitter, chained wait strategies, callback hooks, exception filtering. A hand-rolled loop inevitably misses `RetryError` propagation, logging, and composability. |
| Exponential backoff math | Custom formula with `2**attempt * base` | `tenacity.wait_exponential(multiplier=0.5, min=0.5, max=10)` | tenacity's `wait_exponential` has built-in `min`/`max` clamping, supports `+ wait_random(0, 1)` for jitter chaining, and is tested across Python versions. |
| CLI argument parsing | `sys.argv` slicing or manual argparse setup | `click.option()` and `click.command()` | click auto-generates `--help`, validates types (int, float, Path), handles defaults, and provides `CliRunner` for testing. argparse requires ~3x the code for equivalent functionality. |
| Circuit breaker state | Global boolean flags checked before each request | tenacity retry_state tracking + custom Exception | State machine with state transitions (closed → open → half-open) is deceptively complex. A counter + custom exception pattern (as shown in Pattern 2) is simpler and sufficient for this single-threaded pipeline. |

**Key insight:** retry logic is the #1 place where hand-rolled solutions silently miss edge cases. tenacity's composable decorator API (`wait=...`, `stop=...`, `retry=...`, `before_sleep=...`) handles every retry concern in 5 lines that would take 60+ lines of hand-rolled code.

## Common Pitfalls

### Pitfall 1: Silent Non-JSON Responses (HTML Instead of JSON)

**What goes wrong:** old.reddit.com returns HTML (login pages, rate-limit interstitials, "you've been blocked") with HTTP 200 OK. Calling `response.json()` raises `JSONDecodeError` with a cryptic message about the first character of `<html>` not being valid JSON.

**Why it happens:** The `.json` suffix is an unofficial API endpoint. Reddit's bot detection, geolocation-based blocking (EU GDPR), and aggressive rate limiting all serve HTML at 200 status instead of proper error codes.

**How to avoid:**
1. Always validate `Content-Type: application/json` before parsing
2. Check for HTML signatures (`<html`, `<!DOCTYPE`, `<body`) in response text on parse failure
3. Circuit breaker: halt after 5 consecutive non-JSON responses across different URLs
4. Log the full URL, status code, headers, and first 500 chars on each failure

**Warning signs:** `JSONDecodeError` at predictable times of day (peak Reddit traffic = more blocking), intermittent failures without clear pattern.

### Pitfall 2: Review Filter False Positives

**What goes wrong:** Non-review posts (recommendation requests, bottle porn, news articles, memes) pass the keyword gate because they coincidentally contain wine terms. "What should I pair with steak?" contains no review, but might match "tannin" or "acidity" in answers.

**Why it happens:** The keyword gate is a simple binary classifier with no semantic understanding. It cannot distinguish between someone describing a wine they drank vs. someone asking about wine pairings.

**How to avoid:**
1. Exclusion list takes priority — check exclusion terms first (D-10)
2. Multi-signal scoring (sensory + structure + score + varietal/region) rather than single keyword match
3. `reject_reason` is always populated — enables post-hoc audit
4. Manually review first 200 posts to calibrate gates before full scrape

**Warning signs:** Many posts with `review_flag=true` but 0-1 extracted entities in Phase 2. Exclusion terms that are too broad catch real reviews.

### Pitfall 3: Reddit Pagination Exhaustion

**What goes wrong:** Reddit listings max out at ~1000 items per sort order. Scraping `new.json` with `?after=...` repeatedly will eventually return an empty listing (no `after` cursor) before reaching `--limit 2000`.

**Why it happens:** Reddit's listing indexes are capped at 1000 items by the backend. Old posts fall out of the sort index.

**How to avoid:**
1. Detect empty `data.children` list or missing `data.after` in response — this is the end of available posts
2. For Phase 1 MVP, accept the ~1000-post ceiling from a single sort order. Phase 5 can add multi-sort strategies if needed.
3. Log actual vs. requested count in manifest (D-06's `posts_requested` vs `posts_returned`)

### Pitfall 4: Inconsistent `created_utc` Handling

**What goes wrong:** Reddit's `created_utc` is a Unix timestamp (float). If treated as an integer, fractional seconds are lost. If treated as local time, timezone confusion arises.

**How to avoid:** Store `created_utc` as a float. Convert to `datetime` only for display. Never modify the raw value.

## Code Examples

### Complete Scrape Command Skeleton

```python
# Source: click docs + CONTEXT.md D-02/D-03
# File: src/scrape.py
import click
import sys
from pathlib import Path

@click.command()
@click.option("--subreddit", default="wine", help="Subreddit to scrape")
@click.option("--limit", default=100, type=int, help="Max posts to fetch")
@click.option("--output-dir", default="data/raw", type=click.Path(), help="Output directory")
@click.option("--rate-limit", default=2.0, type=float, help="Requests per second")
@click.option("--min-tokens", default=30, type=int, help="Minimum tokens for review filter")
@click.option("--keyword-file", default="data/filters/keywords.json", type=click.Path(), help="Filter keyword config")
@click.option("--fresh", is_flag=True, default=False, help="Skip resume, start fresh scrape")
def scrape(subreddit, limit, output_dir, rate_limit, min_tokens, keyword_file, fresh):
    """Scrape posts from r/wine via old.reddit.com JSON API."""
    click.echo(f"Scraping r/{subreddit}, target: {limit} posts", err=True)
    # ... implementation
```

### Reddit JSON Post Shape (Key Fields)

```python
# Source: old.reddit.com/.json response structure [CITED: Reddit API documentation]
# Each post in response["data"]["children"][i]["data"] has:
{
    "id": "1abc23",               # base-36 post ID
    "name": "t3_1abc23",          # fullname (prefix + id) — use for ?after=
    "title": "2019 Domaine X...",
    "selftext": "Tasting notes: ...",
    "author": "wine_lover_42",
    "created_utc": 1732745600.0,
    "score": 42,
    "num_comments": 15,
    "permalink": "/r/wine/comments/1abc23/...",
    "link_flair_text": "Tasting Notes",
    "post_hint": "image",          # present only for image/link posts
    "stickied": false,
    "url": "https://i.redd.it/..." # for link/image posts
}
```

### OP Comment Fetch for Image Posts

```python
# Source: CONTEXT.md D-02 + specs/PLAN.md Stage 0
def fetch_op_comment(post_id: str, op_author: str, headers: dict) -> str | None:
    """Fetch the OP's first top-level comment for an image post."""
    url = f"https://old.reddit.com/r/wine/comments/{post_id}.json?limit=1"
    data = safe_fetch_json(url, headers)

    # Reddit comment listing: first element is the post, second is comments
    if len(data) < 2:
        return None
    comments = data[1]["data"]["children"]
    for child in comments:
        if child["kind"] != "t1":  # t1 = comment
            continue
        comment = child["data"]
        if comment.get("author") == op_author:
            return comment.get("body", "")
    return None
```

### keywords.json Schema

```json
{
  "min_tokens": 30,
  "positive_keywords": {
    "sensory": ["tannin", "tannins", "oak", "acidity", "acidic", "finish", "palate", "nose", "aroma", "bouquet", "body", "full-bodied", "medium-bodied", "light-bodied"],
    "structure_cues": ["price", "paid", "bought", "$", "retail", "cost", "worth", "value"],
    "score_cues": ["points", "pts", "/100", "/5", "score", "rating", "rated"],
    "varietal_region": ["cabernet", "merlot", "pinot noir", "chardonnay", "sauvignon blanc", "syrah", "shiraz", "zinfandel", "malbec", "tempranillo", "sangiovese", "nebbiolo", "barolo", "bordeaux", "burgundy", "napa", "rioja", "chianti", "beaujolais"]
  },
  "exclusion_keywords": [
    "looking for recommendations",
    "where can I buy",
    "wedding wine suggestion",
    "what should I pair",
    "gift for",
    "new to wine",
    "just getting into wine",
    "beginner here"
  ]
}
```

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python 3.13 | Runtime | ✓ | 3.13.0 | — |
| uv | Package management | ✓ | 0.12.7 | pip3 |
| git | Version control | ✓ | 2.54.0 | — |
| requests (PyPI) | HTTP client | ✓* | 2.34.2 | — |
| tenacity (PyPI) | Retry logic | ✓* | 9.2.1 | — |
| click (PyPI) | CLI | ✓* | 8.5.1 | — |
| pytest (PyPI) | Testing | ✓* | 9.2.0 | — |
| .venv | Virtual environment | ✗ | — | Create with `uv venv .venv` |
| data/ directory | Storage | ✗ | — | Create during scrape |

*Available on PyPI; not yet installed in project (no .venv exists). Install during Wave 0.

**Missing dependencies with no fallback:**
- `.venv` (virtual environment): Must be created before any code runs. Install step: `uv venv .venv && source .venv/bin/activate`

**Missing dependencies with fallback:**
- `requests`, `tenacity`, `click`, `pytest`: Available from PyPI. Install during Wave 0 setup.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 9.2.0 |
| Config file | none — use `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `pytest tests/ -x --tb=short` |
| Full suite command | `pytest tests/ -v --tb=long` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SCRP-01 | Fetch posts with pagination, rate limit applied | unit | `pytest tests/test_scrape.py::test_paginate_after_cursor -x` | ❌ Wave 0 |
| SCRP-01 | Rate limit enforced (2 req/sec with jitter) | unit | `pytest tests/test_scrape.py::test_rate_limit_wait_time -x` | ❌ Wave 0 |
| SCRP-02 | Image posts fetch OP comment | unit | `pytest tests/test_scrape.py::test_fetch_op_comment_for_image_post -x` | ❌ Wave 0 |
| SCRP-02 | Text posts skip OP comment fetch | unit | `pytest tests/test_scrape.py::test_skip_op_comment_for_text_post -x` | ❌ Wave 0 |
| SCRP-03 | Content-Type validation rejects HTML | unit | `pytest tests/test_scrape.py::test_rejects_html_content_type -x` | ❌ Wave 0 |
| SCRP-03 | Circuit breaker trips after 5 consecutive failures | unit | `pytest tests/test_scrape.py::test_circuit_breaker_trips -x` | ❌ Wave 0 |
| SCRP-03 | Retry recovers after transient failure | unit | `pytest tests/test_scrape.py::test_retry_recovers -x` | ❌ Wave 0 |
| SCRP-04 | JSONL file created with correct format | unit | `pytest tests/test_scrape.py::test_jsonl_output_format -x` | ❌ Wave 0 |
| SCRP-04 | Manifest contains required metadata fields | unit | `pytest tests/test_scrape.py::test_manifest_fields -x` | ❌ Wave 0 |
| SCRP-05 | Posts below 30 tokens rejected | unit | `pytest tests/test_filter.py::test_token_gate_rejects_short_posts -x` | ❌ Wave 0 |
| SCRP-05 | Posts with no keyword matches rejected | unit | `pytest tests/test_filter.py::test_keyword_gate_rejects_no_matches -x` | ❌ Wave 0 |
| SCRP-05 | Posts with sensory match pass | unit | `pytest tests/test_filter.py::test_sensory_keyword_passes -x` | ❌ Wave 0 |
| SCRP-05 | Exclusion terms override positive keywords | unit | `pytest tests/test_filter.py::test_exclusion_overrides_positive -x` | ❌ Wave 0 |
| SCRP-06 | Rejection reason recorded per post | unit | `pytest tests/test_filter.py::test_reject_reason_recorded -x` | ❌ Wave 0 |
| SCRP-06 | Skipped posts counted in summary | unit | `pytest tests/test_filter.py::test_skip_count_in_summary -x` | ❌ Wave 0 |
| ORCH-03 | Manifest tracks posts_requested vs posts_returned | unit | `pytest tests/test_scrape.py::test_manifest_tracks_counts -x` | ❌ Wave 0 |
| ORCH-03 | Resume reads last_post_id from manifest | unit | `pytest tests/test_scrape.py::test_resume_from_last_post_id -x` | ❌ Wave 0 |
| ORCH-03 | --fresh starts new file | unit | `pytest tests/test_scrape.py::test_fresh_flag_creates_new_file -x` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `pytest tests/ -x --tb=short`
- **Per wave merge:** `pytest tests/ -v --tb=long`
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `tests/conftest.py` — shared fixtures: mock Reddit JSON responses, temporary JSONL files, keyword config
- [ ] `tests/test_scrape.py` — all SCRP-01 through SCRP-04 tests
- [ ] `tests/test_filter.py` — all SCRP-05 and SCRP-06 tests
- [ ] Framework install: `uv pip install pytest==9.2.0` — none detected
- [ ] `.venv` creation: `uv venv .venv && source .venv/bin/activate`
- [ ] `data/filters/keywords.json` — filter configuration file

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | No auth required — Reddit old.reddit.com JSON API is public and unauthenticated |
| V3 Session Management | No | No user sessions — stateless HTTP requests |
| V4 Access Control | No | No access control — single-user CLI tool |
| V5 Input Validation | Yes | `Content-Type` header validation before JSON parsing; HTML detection in response bodies; reject `[removed]`/`[deleted]` posts; sanitize selftext for downstream processing |
| V6 Cryptography | No | No cryptographic operations — plain HTTP to public API |
| V7 Error Handling | Yes | Circuit breaker prevents infinite retry loops; all errors logged to stderr, not swallowed; `RetryError` surfaced to user with context |

### Known Threat Patterns for requests + tenacity

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Reddit returns HTML with 200 OK (content spoofing) | Spoofing | Content-Type validation before parsing. HTML body detection. Never trust status code alone. |
| Infinite retry loop on persistent errors | Denial of Service | `stop_after_attempt(3)` + circuit breaker at 5 consecutive failures. `stop_after_delay` as safety net. |
| PII exposure in log output | Information Disclosure | Never log full selftext in error messages. Log post IDs, not usernames, in retry logs. Author names stripped before analysis output (Phase 4 concern, but scraper should avoid logging them). |
| Request smuggling via URL params | Tampering | Sanitize `after` parameter (Reddit fullname format: `t3_[a-z0-9]+`). Validate subreddit name against `^[a-zA-Z0-9_]+$`. |
| Excessive requests overwhelm Reddit (rate limit bypass) | Denial of Service | tenacity `wait_exponential + wait_random` enforces minimum inter-request delay. Circuit breaker adds backpressure. |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Reddit's old.reddit.com JSON API remains accessible and returns the same response shape as documented | Standard Stack, Architecture Patterns | Scraping is impossible — the entire Phase 1 data source disappears. Mitigation: PITFALLS.md recovery strategy notes Pushshift or official OAuth API as fallbacks. |
| A2 | Reddit treats 2 req/sec with jitter as acceptable unauthenticated traffic (no IP block) | Common Pitfalls | IP may still be temporarily blocked. Mitigation: circuit breaker halts early; human can adjust rate limit afterward. |
| A3 | `en_core_web_lg` is not needed in Phase 1 (filtering is pure regex/keyword, no spaCy) | Standard Stack | If Phase 1 needs NLP for filtering after all, the dependency must be added. Unlikely given D-08's explicit keyword-based filter. |
| A4 | The `post_hint` field reliably distinguishes image posts from text posts | Architecture Patterns | Some Reddit clients don't set `post_hint`. Mitigation: also check `selftext == ""` and `url` ends with image extension as secondary signals. |
| A5 | The keyword seed values in the example `keywords.json` are sufficient for >80% recall on r/wine reviews | Code Examples | Filter may be too strict (false negatives) or too loose (false positives). Mitigation: config file enables tuning; two-pass architecture means re-filtering is cheap. |

## Open Questions

1. **Should `data/raw/` be .gitignored?**
   - What we know: Raw posts contain Reddit usernames + full text — potential PII/privacy concern and ToS gray area. CONTEXT.md code_context says "Data goes under data/raw/."
   - What's unclear: Whether the raw data should be committed for reproducibility vs. excluded for privacy.
   - Recommendation: Gitignore `data/raw/` but commit `data/filters/keywords.json`. Document in README that raw data is local-only.

2. **How many posts can we actually scrape before Reddit's listing index exhausts?**
   - What we know: Reddit caps listings at ~1000 items per sort order. `--limit 2000` may not be reachable from a single `/new.json` sort.
   - What's unclear: Whether `--limit` should cap at 1000 for MVP, or whether multi-sort strategies (hot, new, top) should be implemented now.
   - Recommendation: Cap Phase 1 scrape at the natural listing limit (~1000). Phase 5 orchestration (ORCH-01/02) can add multi-sort if needed. The manifest tracks `posts_requested` vs `posts_returned` so the discrepancy is always visible.

3. **Should `review_flag` be added inline during scraping or only by the filter pass?**
   - What we know: D-04 says "filtering is a separate pass" but the JSONL contains raw posts. The question is whether `review_flag` defaults to `null` (unclassified) or `false` (pending) until the filter runs.
   - Recommendation: Set `review_flag: null` and `reject_reason: null` during scraping. The filter pass is the only writer of these fields. This makes it unambiguous whether filtering has been applied.

## Sources

### Primary (HIGH confidence)
- [requests 2.34.2 Official Documentation](https://requests.readthedocs.io/en/latest/) — HTTP methods, response objects, Content-Type access, JSON parsing, custom headers, timeouts [VERIFIED: official docs]
- [tenacity Official Documentation](https://tenacity.readthedocs.io/en/latest/) — `@retry` decorator, `stop_after_attempt`, `wait_exponential`, `wait_random`, jitter, `before_sleep` logging, `retry_if_exception_type`, RetryCallState [VERIFIED: official docs]
- [click 8.x Official Documentation](https://click.palletsprojects.com/en/stable/) — `@click.command()`, `@click.option()`, `click.echo()`, `click.Path`, `click.Choice`, `CliRunner` testing [VERIFIED: official docs]
- [Python json stdlib](https://docs.python.org/3/library/json.html) — `json.dumps()`, `json.loads()`, `json.dump()`, `json.load()`, JSONL patterns [CITED: stdlib docs]
- `.planning/research/PITFALLS.md` — Pitfalls P1 (silent non-JSON), P2 (review contamination), P9 (provenance) [VERIFIED: project research docs]
- `.planning/research/ARCHITECTURE.md` — Data Acquisition Layer boundaries, data flow [VERIFIED: project research docs]
- `.planning/research/STACK.md` — Library versions, compatibility, installation [VERIFIED: project research docs]

### Secondary (MEDIUM confidence)
- `.planning/specs/PLAN.md § Stage 0-1` — Scraping approach, endpoint URLs, post fields, OP comment fetch pattern [CITED: project spec]
- `.planning/CONTEXT.md` (phase 1) — Locked decisions D-01 through D-15 [CITED: phase context]
- Reddit API dev documentation — Listing pagination (`after`/`before`), response shape (`data.children[]`, `data.after`), `raw_json` parameter [CITED: reddit.com/dev/api]

### Tertiary (LOW confidence)
- Real Python JSON tutorial — JSONL writing patterns (append mode, newline delimiters) [ASSUMED: tutorial content, not authoritative]
- WebSearch results on circuit breaker patterns — Implementation strategy validated against tenacity docs [ASSUMED]

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries verified on PyPI with official docs
- Architecture: HIGH — patterns anchored to locked decisions (D-01 through D-15) and verified project research
- Pitfalls: HIGH — confirmed by live test of old.reddit.com returning HTML (documented in PITFALLS.md)
- Filter logic: MEDIUM — keyword seed values are initial estimates; calibration requires real data (D-08 says `>=1` match, exact keywords TBD)

**Research date:** 2026-11-27
**Valid until:** 2026-12-27 (30 days — stable HTTP libraries, unlikely to change)
