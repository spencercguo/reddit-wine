# Phase 1 Discussion Log

**Phase:** 1 - Pipeline Foundation
**Date:** 2026-11-27
**Mode:** --auto (autonomous, no interactive questioning)

## Areas Discussed

### All gray areas auto-selected (--auto mode)

| Area | Question | Auto-selected | Rationale |
|------|----------|---------------|-----------|
| Gazetteer Approach | Path A (build from public sources) vs Path B (lightweight starter)? | Path B — Lightweight starter | Faster to ship. Candidate queue (unmapped_spans.csv) tells us what to add. Doesn't block Phase 1 scaffolding. |
| Project Scaffold | Package manager and structure? | uv + pyproject.toml with `src/` layout | Matches STACK.md recommendations. Already established in project specs. |
| Data Storage | Raw scraped posts format? | JSONL (one JSON per line) | Append-friendly, human-readable, compatible with incremental scraping. |
| Scraping Resilience | Retry/circuit breaker parameters? | tenacity with 5-retry backoff + circuit breaker after 5 non-JSON responses | Strong enough to survive intermittent Reddit issues, conservative enough to stop on persistent failures. |
| Post Filter | Token/keyword thresholds? | 30-token minimum + multi-signal keyword gate | From PLAN.md specs. Balances filtering noise vs. capturing genuine reviews. |
| Provenance Tracking | How to track what was processed? | JSON-based manifest system | Simple, file-based, no database. Tracks post IDs through processing stages. |

## the agent's Discretion

All decisions were auto-selected using recommended defaults from PLAN.md and research findings. No areas were left to agent discretion.

## Deferred Ideas

None — discussion stayed within phase scope.
