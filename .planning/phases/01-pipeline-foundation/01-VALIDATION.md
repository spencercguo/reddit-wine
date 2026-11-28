---
phase: "01"
slug: "pipeline-foundation"
status: draft
nyquist_compliant: false
wave_0_complete: false
created: "2026-11-27"
---

# Phase 01 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest 9.2.0 |
| **Config file** | `pyproject.toml` `[tool.pytest.ini_options]` |
| **Quick run command** | `pytest tests/ -x --tb=short` |
| **Full suite command** | `pytest tests/ -v --tb=long` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pytest tests/ -x --tb=short`
- **After every plan wave:** Run `pytest tests/ -v --tb=long`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 01-01-01 | 01 | 1 | SCRP-01 | T-01-01 | Pagination uses after cursor, not page numbers | unit | `pytest tests/test_scrape.py::test_paginate_after_cursor -x` | ❌ W0 | ⬜ pending |
| 01-01-02 | 01 | 1 | SCRP-01 | T-01-02 | Rate limit enforced at 2 req/sec with jitter | unit | `pytest tests/test_scrape.py::test_rate_limit_wait_time -x` | ❌ W0 | ⬜ pending |
| 01-01-03 | 01 | 1 | SCRP-02 | T-01-03 | Image posts fetch OP comment; text posts skip | unit | `pytest tests/test_scrape.py::test_fetch_op_comment_for_image_post -x` | ❌ W0 | ⬜ pending |
| 01-01-04 | 01 | 1 | SCRP-02 | T-01-04 | Text posts do not trigger OP comment fetch | unit | `pytest tests/test_scrape.py::test_skip_op_comment_for_text_post -x` | ❌ W0 | ⬜ pending |
| 01-01-05 | 01 | 1 | SCRP-03 | T-01-05 | Content-Type header validated before .json() call | unit | `pytest tests/test_scrape.py::test_rejects_html_content_type -x` | ❌ W0 | ⬜ pending |
| 01-01-06 | 01 | 1 | SCRP-03 | T-01-06 | Circuit breaker trips at 5 consecutive non-JSON responses | unit | `pytest tests/test_scrape.py::test_circuit_breaker_trips -x` | ❌ W0 | ⬜ pending |
| 01-01-07 | 01 | 1 | SCRP-03 | T-01-07 | Retry recovers from transient non-JSON response | unit | `pytest tests/test_scrape.py::test_retry_recovers -x` | ❌ W0 | ⬜ pending |
| 01-01-08 | 01 | 1 | SCRP-04 | T-01-08 | JSONL file created with one JSON object per line | unit | `pytest tests/test_scrape.py::test_jsonl_output_format -x` | ❌ W0 | ⬜ pending |
| 01-01-09 | 01 | 1 | SCRP-04 | T-01-09 | Manifest contains timestamp, subreddit, counts, last_post_id | unit | `pytest tests/test_scrape.py::test_manifest_fields -x` | ❌ W0 | ⬜ pending |
| 01-01-10 | 01 | 1 | SCRP-05 | T-01-10 | Posts below token threshold rejected | unit | `pytest tests/test_filter.py::test_token_gate_rejects_short_posts -x` | ❌ W0 | ⬜ pending |
| 01-01-11 | 01 | 1 | SCRP-05 | T-01-11 | Posts with no keyword matches rejected | unit | `pytest tests/test_filter.py::test_keyword_gate_rejects_no_matches -x` | ❌ W0 | ⬜ pending |
| 01-01-12 | 01 | 1 | SCRP-05 | T-01-12 | Sensory keyword match passes filter | unit | `pytest tests/test_filter.py::test_sensory_keyword_passes -x` | ❌ W0 | ⬜ pending |
| 01-01-13 | 01 | 1 | SCRP-05 | T-01-13 | Exclusion terms override positive keywords | unit | `pytest tests/test_filter.py::test_exclusion_overrides_positive -x` | ❌ W0 | ⬜ pending |
| 01-01-14 | 01 | 1 | SCRP-06 | T-01-14 | Rejection reason string recorded per post | unit | `pytest tests/test_filter.py::test_reject_reason_recorded -x` | ❌ W0 | ⬜ pending |
| 01-01-15 | 01 | 1 | SCRP-06 | T-01-15 | Skipped post count in summary output | unit | `pytest tests/test_filter.py::test_skip_count_in_summary -x` | ❌ W0 | ⬜ pending |
| 01-01-16 | 01 | 1 | ORCH-03 | T-01-16 | Manifest tracks posts_requested vs posts_returned | unit | `pytest tests/test_scrape.py::test_manifest_tracks_counts -x` | ❌ W0 | ⬜ pending |
| 01-01-17 | 01 | 1 | ORCH-03 | T-01-17 | Resume reads last_post_id from manifest | unit | `pytest tests/test_scrape.py::test_resume_from_last_post_id -x` | ❌ W0 | ⬜ pending |
| 01-01-18 | 01 | 1 | ORCH-03 | T-01-18 | --fresh creates new timestamped file | unit | `pytest tests/test_scrape.py::test_fresh_flag_creates_new_file -x` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/conftest.py` — shared fixtures: mock Reddit JSON responses, temporary JSONL files, keyword config
- [ ] `tests/test_scrape.py` — all SCRP-01 through SCRP-04 tests (with ORCH-03 manifest tests)
- [ ] `tests/test_filter.py` — all SCRP-05 and SCRP-06 tests
- [ ] Framework install: `uv pip install pytest==9.2.0`
- [ ] `.venv` creation: `uv venv .venv && source .venv/bin/activate`
- [ ] `data/filters/keywords.json` — filter configuration file

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `python -m src.scrape --subreddit wine --limit 5` succeeds end-to-end | SCRP-01 | Requires live Reddit connection | Run scrape against real Reddit API; verify JSONL file created |
| `python -m src.filter --input data/raw/rwine_*.jsonl` processes output | SCRP-05 | Integration test across scrape+filter | Run scrape first, then filter; verify review_flag values |

*Two behaviors require live Reddit API connectivity and cannot be fully tested with mocks.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
