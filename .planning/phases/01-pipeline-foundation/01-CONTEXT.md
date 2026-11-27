# Phase 1: Pipeline Foundation - Context

**Gathered:** 2026-11-27
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the data acquisition foundation: a command-line scraper that fetches posts from r/wine via old.reddit.com's JSON API, handles errors and rate limits robustly, filters non-review posts to reduce downstream noise, and persists everything with provenance tracking. No NLP, no entity extraction — just clean, auditable raw data.
</domain>

<decisions>
## Implementation Decisions

### Project Scaffold & Packaging
- **D-01:** Use uv for package management with pyproject.toml. Project structure follows the layout in `.planning/specs/PLAN.md` (src/scrape.py, data/raw/, etc.). Python 3.13 is the runtime.

### CLI Design
- **D-02:** Single `scrape` command using click (matching stack choice). All parameters are configurable flags with sensible defaults: `--subreddit` (default: wine), `--limit` (number of posts to fetch), `--output-dir` (default: data/raw/), `--rate-limit` (default: 2.0 req/sec), `--min-tokens` (default: 30), `--keyword-file` (default: data/filters/keywords.json), `--fresh` (skip resume, start new scrape).
- **D-03:** Output shows current page number, posts scraped count, and a final summary with counts (total scraped, filtered in, rejected, errors). Warnings printed to stderr. Continue on recoverable errors; circuit breaker still halts on consecutive failures.
- **D-04:** Filtering is a separate pass — scraping saves all raw posts first, then a filter pass reads the JSONL and adds `review_flag` (boolean) and `reject_reason` (string) fields. Enables re-filtering with updated rules without re-scraping.

### Data Storage
- **D-05:** Single JSONL file per scrape session with timestamped naming (`rwine_2026-11-27T16-03.jsonl`). One JSON object per line. All posts (raw) go into one file with `review_flag` (true/false) and `reject_reason` (null or string) fields added by the filter pass. Sidecar manifest at same path (`manifest_2026-11-27T16-03.json`).
- **D-06:** Manifest is run metadata only: timestamp, subreddit, posts_requested, posts_returned, posts_filtered_in, posts_rejected, errors_count, duration_seconds, last_post_id (for resume). No full post ID list — IDs live in the JSONL.

### Scraping Resilience
- **D-07:** Use tenacity's exponential backoff (2 req/sec base rate with jitter). Validate Content-Type on every response — if a non-JSON response is received, retry up to 3 times. After 5 consecutive non-JSON responses across different requests, trigger a circuit breaker and halt. Log every retry attempt to stderr.

### Post Filter
- **D-08:** 30-token minimum gate + multi-signal keyword gate. Keywords grouped into: sensory (tannin, oak, acidity, finish, palate, nose, aroma, bouquet, body), structure cues (price, paid, bought, $, retail), score cues (points, pts, /100, score, rating), and varietal/region names. Binary gate: pass if >=1 keyword match across any category.
- **D-09:** Keywords and token threshold read from `data/filters/keywords.json` config file. CLI overridable via `--keyword-file` and `--min-tokens`. Enables filter tuning without code changes.
- **D-10:** Exclusion keyword list in the same config — posts matching any exclusion term (e.g., "looking for recommendations", "where can I buy", "wedding wine suggestion") are rejected regardless of positive keyword matches. Rejection reason records which exclusion term triggered.
- **D-11:** Skip non-text posts before keyword gate: link posts (selftext = empty + URL only), AutoModerator posts (author == "AutoModerator"), and stickied posts. Process only self-posts and image posts (with OP comment fetch). Rejection reason logged as `post_type_link`, `post_type_auto`, or `post_type_sticky`.

### Incremental Scraping
- **D-12:** Resume by default — scraper reads last_post_id from most recent manifest for the same subreddit and sends `?after={last_post_id}` to continue pagination. `--fresh` flag starts a fresh scrape (creates new timestamped file).
- **D-13:** On resume, append to the existing timestamped JSONL file, updating the same manifest. Before resuming forward pagination, scan existing posts for image posts missing OP comments and refetch those comments.

### Provenance Tracking
- **D-14:** Manifest system as described in D-06. Enables incremental scrapes and re-extraction after rule changes. Simple JSON-based, no database.

### Gazetteer Approach
- **D-15:** Use Path B — lightweight starter gazetteer (~40 regions, ~60 varietals, common aliases). Run the sample pipeline first, let the candidate queue (unmapped_spans.csv from entity linking) tell us what to add. Iteratively grow. Carried forward to Phase 2.
</decisions>

<canonical_refs>
## Canonical References

Downstream agents MUST read these before planning or implementing.

### Project Definition
- `.planning/PROJECT.md` — Project context, constraints, scope
- `.planning/REQUIREMENTS.md` — SCRP-01..06, ORCH-03 requirements for this phase
- `.planning/ROADMAP.md` § Phase 1 — Phase goal and success criteria

### Technical Specs
- `.planning/specs/PLAN.md` § Stage 0 — Scraping approach, endpoints, rate limits, project structure
- `.planning/specs/PLAN.md` § Stage 1 — Pre-filtering heuristics
- `.planning/research/STACK.md` — Recommended libraries and versions (click 8.x, requests 2.x, tenacity 9.x)
- `.planning/research/PITFALLS.md` — Reddit HTML response pitfall (P1), manifest provenance (P9), Content-Type validation

### Architecture
- `.planning/research/ARCHITECTURE.md` § Data Acquisition Layer — Component boundaries, scraper responsibility
</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- No existing scrapers or utilities. This is the first phase.
- `src/__init__.py` exists (empty). pyproject.toml has minimal scaffolding (dependencies empty).

### Established Patterns
- No established patterns — this phase establishes them:
  - All Python files go under `src/`
  - Data goes under `data/raw/` and `data/processed/`
  - Configuration files go under `data/filters/`
  - CLI uses `click` for argument parsing
  - `__init__.py` at package root only — modules are flat scripts under `src/`

### Integration Points
- The JSONL files written by this phase feed Phase 2's extraction pipeline
- The `review_flag` and `reject_reason` fields are the contract with downstream stages
- The manifest `last_post_id` enables Phase 5 orchestration (ORCH-01/02 resume)
</code_context>

<specifics>
## Specific Ideas

From `.planning/specs/PLAN.md` and discussion:
- Scrape entry point: `python -m src.scrape --subreddit wine --limit 100`
- OP comment fetch for image posts: GET `old.reddit.com/r/wine/comments/{post_id}.json?limit=1`, match author == OP
- review_text = title + "\n" + selftext + "\n" + op_comment (for image posts)
- Filter config: `data/filters/keywords.json` with positive and exclusion keyword groups, token threshold
- Filter pass is separate: `python -m src.filter --input data/raw/rwine_*.jsonl` adds review_flag + reject_reason
- Manifest tracks last_post_id for resume; --fresh starts clean
</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.
</deferred>

---

*Phase: 1-Pipeline Foundation*
*Context gathered: 2026-11-27*
