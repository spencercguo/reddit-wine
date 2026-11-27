# r/wine Review Analysis

## What This Is

A data pipeline that scrapes wine review posts from Reddit's r/wine community, extracts structured entities (wine regions, varietals, vintages, prices, scores) using regex and NLP, and produces analytical datasets to uncover patterns in what people drink, what they pay, and what they think. Built as a local Python tool — one person's pipeline, not a hosted product.

## Core Value

Turn thousands of unstructured community wine reviews into queryable, analyzable data — surfacing the hidden patterns in r/wine's tasting notes, price distributions, and regional preferences that no single post reveals alone.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Scrape 2000+ review posts from r/wine via old.reddit.com JSON API
- [ ] Pre-filter non-review posts (token gate, keyword gate)
- [ ] Extract structured entities: wine region, subregion, AVA/AOC/DOCG, grape varietal, vintage, price, score
- [ ] Perform entity linking with fuzzy matching and domain inference rules
- [ ] Normalize scores to 0–100 scale and prices to USD
- [ ] Generate analytical datasets: region frequency, price stats, rating stats, cross-tabs, vintage histogram, tasting note frequency
- [ ] Explore results in a marimo notebook with Plotly visualizations

### Out of Scope

- Real-time or automated periodic scraping — manual pipeline runs only
- Web UI or hosted dashboard — analysis is local/notebook-based
- Sentiment analysis (method left open — TBD, not committed)
- Mobile app or API
- Multi-subreddit support (r/wine only)

## Context

The r/wine subreddit is a rich source of community wine reviews — people post tasting notes, ask for value recommendations, and share finds. These posts contain semi-structured information (regions, varietals, prices, scores) written in natural language prose. Extracting and aggregating this data surfaces community-level trends: which regions are over/under-represented, what people pay, how they rate what they drink, and how vintage distributions shift.

**Technical approach:** Python pipeline using regex + spaCy EntityRuler for deterministic extraction, spaCy NER for discovery, RapidFuzz for fuzzy entity linking, Polars for data analysis, marimo + Plotly for exploration. Scraping via old.reddit.com's JSON API (no auth needed for public data). Runs locally on a single machine — CPU-friendly, no GPU requirements.

**Data sources:** Public Reddit posts from r/wine. No private data, no user profiles beyond author name attribution.

## Constraints

- **Rate limit**: 2 requests/second to old.reddit.com
- **Compute**: CPU-only, no GPU required (spaCy `en_core_web_lg` is adequate)
- **Language**: Python only, single codebase
- **Scope**: Single subreddit (r/wine), English-language posts only
- **Gazetteer**: Wine region/varietal knowledge base needed — either built from public sources or iteratively grown
- **Environment**: macOS (darwin), local development

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Use old.reddit.com JSON API | No auth required, structured JSON output, simpler than Pushshift | — Pending |
| spaCy EntityRuler for determinism | High precision for known entities vs. pure NER which hallucinates | — Pending |
| Polars over pandas | Faster for the analytical queries needed, better memory profile | — Pending |
| marimo over Jupyter | Reactive notebook model, better Python integration | — Pending |
| Sentiment analysis deferred | Method selection requires evaluation (VADER vs TextBlob vs transformer); not blocking the pipeline | — Pending |
| Pipeline model (not interactive) | Script-based execution is simpler and sufficient for research-grade analysis | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-11-27 after initialization*
