# Phase 1: Pipeline Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-11-27
**Phase:** 1-Pipeline Foundation
**Areas discussed:** CLI Design, Output file organization, Incremental scraping behavior, Filter scoring calibration

---

## CLI Design

| Option | Description | Selected |
|--------|-------------|----------|
| Single 'scrape' command | `python -m src.scrape --limit 200 --subreddit wine`. Filtering inline. Less ceremony. | ✓ |
| Subcommands: scrape + filter | Separate commands for pipeline stages. More modular but overkill for Phase 1. | |

**User's choice:** Single 'scrape' command (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| All configurable | --subreddit, --limit, --output-dir, --rate-limit, --min-tokens as flags. Sensible defaults. | ✓ |
| Minimal flags | Only --limit and --output-dir. Everything else hardcoded. | |

**User's choice:** All configurable (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Progress + summary | Show page #, posts scraped, final summary with counts. No per-post spam. | ✓ |
| Summary only | Quiet while scraping, print summary at end. | |
| Verbose per-post | Print each post title and filter result. Good for debug, noisy for 2000+. | |

**User's choice:** Progress + summary (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Stderr + continue | Print warnings to stderr, log to file, keep going on recoverable errors. | ✓ |
| Log file only | All errors to scrape_errors.log, silent on stdout/stderr. | |
| Crash on first error | Stop immediately on any error. | |

**User's choice:** Stderr + continue (Recommended)

---

## Output File Organization

| Option | Description | Selected |
|--------|-------------|----------|
| 1 file + manifest | Single JSONL with review_flag and reject_reason fields. Sidecar manifest. | ✓ |
| 3 files per run | Separate posts.jsonl, reviews.jsonl, rejected.jsonl + manifest. | |
| 4 files | Same as 3 but raw posts before OP comment enrichment. | |

**User's choice:** 1 file + manifest

---

| Option | Description | Selected |
|--------|-------------|----------|
| Separate filter pass | Scrape saves raw, filter pass adds review_flag + reject_reason. Reprocessable. | ✓ |
| Inline during scrape | Tag posts as they come in. Single pass, faster but can't re-filter without re-scraping. | |

**User's choice:** Separate filter pass (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Timestamped | rwine_2026-11-27T16-03.jsonl + manifest_2026-11-27T16-03.json. No overwrites. | ✓ |
| Fixed name | rwine_posts.jsonl + manifest.json. Simpler but overwritten each run. | |
| ISO timestamp + limit | rwine_200posts_2026-11-27.jsonl. Informative but verbose. | |

**User's choice:** Timestamped (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Run metadata only | Timestamp, subreddit, counts, errors, duration, last_post_id. Post IDs in JSONL. | ✓ |
| Run metadata + post ID list | Same + full post_ids array. Duplicates JSONL data. | |

**User's choice:** Run metadata only (Recommended)

---

## Incremental Scraping Behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Resume by default | Default picks up where previous run left off. --fresh to start clean. | ✓ |
| Fresh scrape default | Default from scratch each run. --resume to continue. | |
| Always fresh, no resume | Every run is new. Simplest but wasteful. | |

**User's choice:** Resume by default

---

| Option | Description | Selected |
|--------|-------------|----------|
| last_post_id in manifest | Manifest records last fetched post ID. Resume sends ?after={last_id}. | ✓ |
| JSONL scan | Scan existing file for last post ID. No manifest dependency but O(n). | |

**User's choice:** last_post_id in manifest (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Append to existing | Resume appends to same timestamped JSONL, updates manifest. Coherent file. | ✓ |
| New file per run | Every run creates a new file. Safer but scatters sessions. | |

**User's choice:** Append to existing file (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Refetch missing comments | On resume, scan for image posts without OP comments and refetch before continuing. | ✓ |
| Skip them | Only fetch comments for new posts during resume. Leaves gaps. | |

**User's choice:** Refetch missing comments (Recommended)

---

## Filter Scoring Calibration

| Option | Description | Selected |
|--------|-------------|----------|
| Keep binary gate | >=30 tokens AND >=1 keyword match = passes. Simple, predictable, clear audit trail. | ✓ |
| Add scoring function | Score 0-100 based on keyword density + category diversity. Harder to tune without labeled data. | |

**User's choice:** Keep binary gate (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Config file + CLI override | Keywords in data/filters/keywords.json. CLI overridable via --keyword-file, --min-tokens. | ✓ |
| Hardcoded in Python | Keywords and threshold as constants. Requires code changes to tune. | |

**User's choice:** Config file + CLI override (Recommended)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Exclusion keywords | Posts matching exclusion terms reject regardless of positive keywords. Catches rec requests, meta, buying advice. | ✓ |
| No exclusion keywords | Let positive keyword gate handle it. Recommendation requests naturally score low. | |

**User's choice:** Exclusion keywords

---

| Option | Description | Selected |
|--------|-------------|----------|
| Skip non-text posts | Skip link posts, AutoModerator, stickied. Only process self-posts and image posts. | ✓ |
| Process everything | Let token + keyword gate handle it. No special post-type handling. | |

**User's choice:** Skip non-text posts

---

## the agent's Discretion

None — all decisions were made by the user.

## Deferred Ideas

None — discussion stayed within phase scope.
