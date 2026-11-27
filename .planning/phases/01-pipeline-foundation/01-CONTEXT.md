# Phase 1: Pipeline Foundation - Context

**Gathered:** 2026-11-27
**Status:** Ready for planning

## Phase Boundary

This phase delivers the data acquisition foundation: a command-line scraper that fetches posts from r/wine via old.reddit.com's JSON API, handles errors and rate limits robustly, filters non-review posts to reduce downstream noise, and persists everything with provenance tracking. No NLP, no entity extraction — just clean, auditable raw data.

## Implementation Decisions

### Gazetteer Approach
- **D-01:** Use Path B — lightweight starter gazetteer (~40 regions, ~60 varietals, common aliases). Run the sample pipeline first, let the candidate queue (unmapped_spans.csv from entity linking) tell us what to add. Iteratively grow. This is the faster-to-ship path and doesn't block Phase 1 scaffolding.

### Project Scaffold & Packaging
- **D-02:** Use uv for package management with pyproject.toml. Project structure follows the layout in `.planning/specs/PLAN.md` (src/scrape.py, data/raw/, etc.). Python 3.13 is the runtime.

### Data Storage
- **D-03:** Store raw scraped posts as JSONL (one JSON object per line). Append-friendly for incremental scraping runs. Each run gets a timestamped file in data/raw/. Include scrape metadata (timestamp, post count, errors encountered) in a sidecar manifest.

### Scraping Resilience
- **D-04:** Use tenacity's exponential backoff (2 req/sec base rate with jitter). Validate Content-Type on every response — if a non-JSON response is received, retry up to 3 times. After 5 consecutive non-JSON responses across different requests, trigger a circuit breaker and halt. Log every retry attempt.

### Post Filter
- **D-05:** 30-token minimum gate + multi-signal keyword gate. Keywords grouped into: sensory (tannin, oak, acidity, finish, palate, nose, aroma, bouquet, body), structure cues (price, paid, bought, $, retail), score cues (points, pts, /100, score, rating), and varietal/region names. Pass threshold: ≥1 keyword match. Rejected posts logged with rejection reason (too_short / no_keywords / image_only_no_comment).

### Provenance Tracking
- **D-06:** Manifest system tracks which post IDs were scraped in each run, what processing stages they've passed through, and any errors. Enables incremental re-scrapes (skip already-fetched posts) and re-extraction after rule changes. Simple JSON-based manifest, not a database.

## Canonical References

Downstream agents MUST read these before planning or implementing.

### Project Definition
- `.planning/PROJECT.md` — Project context, constraints, scope
- `.planning/REQUIREMENTS.md` — SCRP-01..06, ORCH-03 requirements for this phase
- `.planning/ROADMAP.md` § Phase 1 — Phase goal and success criteria

### Technical Specs
- `.planning/specs/PLAN.md` § Stage 0 — Scraping approach, endpoints, rate limits
- `.planning/specs/PLAN.md` § Stage 1 — Pre-filtering heuristics
- `.planning/research/STACK.md` — Recommended libraries and versions
- `.planning/research/PITFALLS.md` — Reddit HTML response pitfall (P1), manifest provenance (P9), Content-Type validation

### Architecture
- `.planning/research/ARCHITECTURE.md` § Data Acquisition Layer — Component boundaries, scraper responsibility

## Existing Code Insights

### Reusable Assets
- No existing scrapers or utilities in the codebase yet. This is the first phase.

### Established Patterns
- No established patterns — this phase establishes them:
  - All Python files go under src/
  - Data goes under data/raw/ and data/processed/
  - Pipelines use scripts/ for orchestration
  - CLI uses click for argument parsing

### Integration Points
- The JSONL files written by this phase feed Phase 2's extraction pipeline
- The manifest system must be forward-compatible with Phase 2-5 stage tracking

## Specific Ideas

From `.planning/specs/PLAN.md`:
- Scrape sample: ~100 posts. Full run: paginate to 2000+
- OP comment fetch for image posts: GET old.reddit.com/r/wine/comments/{post_id}.json?limit=1, match author == OP
- review_text = title + "\n" + selftext + "\n" + op_comment

## Deferred Ideas

None — discussion stayed within phase scope.

---

*Phase: 1-Pipeline Foundation*
*Context gathered: 2026-11-27*
