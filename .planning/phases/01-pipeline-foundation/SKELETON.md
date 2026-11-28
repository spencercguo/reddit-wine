# Walking Skeleton — r/wine Review Analysis

**Phase:** 1
**Generated:** 2026-11-27

## Capability Proven End-to-End

> A user runs `python -m src.scrape --subreddit wine --limit 5` from the command line and receives 5 scraped Reddit posts written to a timestamped JSONL file with a sidecar manifest. A subsequent `python -m src.filter --input data/raw/rwine_*.jsonl` pass classifies each post as a wine review or non-review, adding `review_flag` and `reject_reason` to every record. The entire pipeline — CLI → HTTP → JSONL storage → filter → manifest — works in one end-to-end integration test.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Runtime | Python 3.13 | Latest stable with GIL improvements; required by Polars 2.x in later phases; macOS default (CONTEXT.md D-01) |
| Package management | uv + pyproject.toml | Fast pip alternative; single-source dependency spec (CONTEXT.md D-01, RESEARCH.md) |
| HTTP client | requests 2.34.2 | Synchronous HTTP sufficient for single-machine pipeline; simpler than httpx async (RESEARCH.md) |
| Retry/resilience | tenacity 9.2.1 | Composable decorator API (stop_after_attempt, wait_exponential + wait_random); prevents hand-rolled retry bugs (RESEARCH.md Pattern 2) |
| CLI framework | click 8.5.1 | Clean decorator API; auto-generated help; CliRunner for testing (CONTEXT.md D-02, RESEARCH.md) |
| Storage format | JSONL (one JSON object per line) | Append-friendly; human-readable; no database needed; line-delimited for streaming readers (CONTEXT.md D-05) |
| Provenance tracking | JSON manifest sidecar | Tracks run metadata (timestamp, counts, last_post_id); enables resume and audit (CONTEXT.md D-06/D-14) |
| Filter architecture | Separate pass (scrape-then-filter) | Enables re-filtering with updated rules without re-scraping; decouples acquisition from classification (CONTEXT.md D-04) |
| Filter configuration | JSON config file (data/filters/keywords.json) | Enables keyword tuning without code changes; CLI-overridable (CONTEXT.md D-09) |
| Directory layout | Flat src/ modules + data/raw/ + data/filters/ | Simple structure for single-machine research tool; no nested packages (CONTEXT.md code_context, RESEARCH.md) |
| Test framework | pytest 9.2.0 | Standard Python test runner; fixtures for mock Reddit responses; parametrized tests for filter edge cases (RESEARCH.md) |
| Deployment target | Local macOS CLI | Single-machine research pipeline; no server, no Docker, no deployment (CONTEXT.md) |

## Stack Touched in Phase 1

- [x] Project scaffold (pyproject.toml with dependencies, .venv, .gitignore)
- [x] CLI entry point — `python -m src.scrape` with click options
- [x] HTTP layer — requests with Content-Type validation before JSON parsing
- [x] Storage — JSONL append-write and read, manifest JSON sidecar
- [x] Filter pipeline — token gate, keyword gate, exclusion gate, post type gate
- [x] Test runner — pytest with mock Reddit fixtures, integration tests
- [x] Resilience — tenacity retry with exponential backoff + jitter, circuit breaker

## Out of Scope (Deferred to Later Slices)

- NLP entity extraction (spaCy, en_core_web_lg) — Phase 2
- Wine gazetteer construction — Phase 2
- Fuzzy entity linking (RapidFuzz) — Phase 3
- Score/price normalization — Phase 3
- Polars analytical datasets — Phase 4
- marimo interactive notebook — Phase 5
- Multi-subreddit support — v2
- Automated periodic scraping — v2
- Database (SQLite/Postgres) — never (local JSONL only)
- Docker/containerization — never (local macOS only)
- Web UI or API server — never (CLI + marimo notebook only)

## Subsequent Slice Plan

Each later phase adds one vertical slice on top of this skeleton without altering its architectural decisions:

- Phase 2: Gazetteer construction + spaCy entity extraction (regions, varietals, vintages, prices, scores)
- Phase 3: Entity linking via fuzzy matching + score/price normalization
- Phase 4: Polars analytical datasets (region frequency, price distributions, rating analysis)
- Phase 5: End-to-end pipeline scripts + interactive marimo notebook with Plotly visualizations
