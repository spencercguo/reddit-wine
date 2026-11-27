# Project Research Summary

**Project:** r/wine Review Analysis Pipeline
**Domain:** Community wine review NLP extraction and aggregation pipeline
**Researched:** 2026-11-27
**Confidence:** HIGH

## Executive Summary

This project is a **local, single-machine NLP pipeline** that scrapes semi-structured wine reviews from r/wine, extracts structured entities (region, varietal, vintage, price, score) from natural-language posts, links extracted entities to canonical forms via fuzzy matching, and produces analytical datasets explored in interactive marimo notebooks. It is explicitly **not** a consumer app, web service, or recommendation engine — it occupies the niche of community-driven review aggregation, which is underserved in the existing ecosystem of consumer tracking apps (Vivino, CellarTracker) and academic Kaggle notebooks.

The recommended approach follows a **four-layer ETL architecture** with Pydantic-typed contracts between stages: (1) Data Acquisition via old.reddit.com JSON API with robust error handling for Reddit's undocumented HTML responses; (2) NLP Extraction using a spaCy pipeline with EntityRuler + NER + custom regex components, driven by a YAML-based wine gazetteer; (3) Entity Linking & Normalization using RapidFuzz with length-adaptive thresholds, synonym-aware canonical resolution, and hierarchical region propagation; (4) Analysis & Output using Polars DataFrames for analytical queries and marimo + Plotly for interactive exploration. The stack is aggressively simple: Python 3.13, no databases, no Docker, no GPU.

**Key risks and mitigations:** (a) Reddit serves HTML instead of JSON at 200 OK — mitigated by Content-Type validation and circuit breakers; (b) review/non-review boundary is a spectrum, not binary — mitigated by a review-quality scoring pass rather than a single keyword gate; (c) synonym fragmentation ("Syrah"/"Shiraz") and region hierarchy collapse corrupts analysis — mitigated by a tree-structured gazetteer with alias maps and ancestor propagation; (d) fuzzy matching tuned to a single threshold produces false positives on short terms — mitigated by length-adaptive thresholds and exact-match-first strategy; (e) starting the gazetteer from nothing creates a chicken-and-egg problem — mitigated by seeding from high-signal public sources (Wine-Searcher top lists, OIV grape data) and iterating.

## Key Findings

### Recommended Stack

A lean Python 3.13 stack optimized for single-machine research, not production services. **spaCy 3.8.18** with `en_core_web_lg` provides EntityRuler (deterministic wine entity matching) + NER without GPU. **Polars 2.2.0** replaces pandas for 10-50x faster analytical queries with native Parquet I/O. **RapidFuzz 3.14.6** provides 10-100x faster fuzzy matching than deprecated FuzzyWuzzy. **marimo 0.26.0** replaces Jupyter with Git-friendly `.py` notebooks and deterministic execution. All versions are pinned with verified compatibility — no version conflicts exist in the recommended combination.

**Core technologies:**
- **spaCy 3.8.18 + en_core_web_lg:** NLP engine — EntityRuler for deterministic wine entity extraction, NER for fallback, custom regex components for structured values (price/score/vintage). Disabled tagger/parser/lemmatizer recover ~60% processing time.
- **Polars 2.2.0:** Analytical engine — LazyFrame API for composable queries, groupby/aggregation for cross-tabs, native Parquet I/O, zero-copy Arrow interop. Chosen over pandas for speed, memory efficiency, and explicit project decisions.
- **RapidFuzz 3.14.6:** Entity linking — `process.extract()` and `cdist()` for batch fuzzy matching against gazetteer. `token_sort_ratio` handles word-reordered multi-word terms. Chosen over FuzzyWuzzy (deprecated, GPL, 10-100x slower).
- **marimo 0.26.0 + Plotly 7.2.0:** Interactive exploration — reactive notebooks stored as `.py` files. Plotly express for declarative charts. Replaces Jupyter entirely.
- **Pydantic 2.x:** Data contracts — typed models (`WineReview`, `CanonicalReview`) validate at layer boundaries. `Optional` fields with `None` sentinels handle missing data explicitly.

### Expected Features

**Must have (table stakes — v1 launch):**
- Scrape 2000+ r/wine posts with rate limiting and robust error handling
- Pre-filter non-review posts via review-quality scoring (token + keyword gates)
- Extract core entities: region, varietal, vintage, price, score — each with extraction strategy appropriate to entity type (EntityRuler for regions/varietals, regex for structured values)
- Entity linking via fuzzy matching with tiered confidence (HIGH/MEDIUM/UNMATCHED)
- Score normalization to 0–100 scale with original-scale preservation
- Price normalization to USD with multi-bottle purchase handling
- Analytical CSV/Parquet export with statistical summaries (count, mean, median, std per dimension)
- marimo notebook with Plotly interactive visualizations (region frequency bar, price histogram, score distribution, vintage histogram, price vs. score scatter)

**Should have (differentiators — add after validation):**
- Cross-tabulated analysis (region × varietal, price × score)
- Tasting note frequency analysis with wine-domain stopword list
- Value analysis (QPR — Quality-Price Ratio, a score/price derived metric)
- Vintage histogram analysis with score cross-reference

**Defer (v2+):**
- Sentiment analysis — domain-specific wine language breaks generic tools (VADER/TextBlob misclassify "aggressive tannins"). Evaluate fine-tuned approach only if justified.
- Multi-subreddit support (r/bourbon, r/scotch) — parameterize after r/wine proves the concept
- Automated report generation — PDF/HTML summaries for non-technical audiences

### Architecture Approach

Four-layer ETL pipeline with Pydantic-typed contracts at every boundary. Each layer is independently testable and follows a strict dependency order: Data Acquisition (scraper, filter, raw JSONL storage) → NLP Extraction (spaCy pipeline with EntityRuler + regex components, YAML-driven patterns) → Entity Linking & Normalization (RapidFuzz fuzzy matching, tree-structured gazetteer, hierarchy propagation, score/price normalization) → Analysis & Output (Polars analytical datasets, Parquet export, marimo notebooks). The architecture treats raw data as immutable — stored once in JSONL, replayed through extraction and linking without re-scraping. Analysis depends only on the canonical record schema, enabling parallel development with hand-crafted test data.

**Major components:**
1. **Scraper + Filter** — Fetch posts from old.reddit.com/r/wine/.json with pagination; enforce 2 req/sec with jitter; validate Content-Type before parsing; filter non-review posts via review-quality scoring pass; persist raw JSONL with manifest tracking for provenance and incremental re-processing.
2. **spaCy Extraction Pipeline** — Tokenize text, apply EntityRuler (wine patterns loaded from YAML), run NER (with `overwrite_ents=True`), then custom regex extractors for price/score/vintage/tasting notes. Process via `nlp.pipe(batch_size=50)` for 20-50x throughput. Output Pydantic-validated entity dicts per post with raw + canonical forms preserved.
3. **Entity Linking Engine** — Exact-match hash lookup (O(1)) for speed, fall back to length-adaptive RapidFuzz fuzzy matching against tree-structured gazetteer. Resolve synonyms ("Syrah" → "Shiraz" → canonical). Propagate hierarchy (mention of "Pauillac" increments counts for Pauillac, Médoc, Bordeaux, France). Normalize scores/prices with scale detection heuristics and multi-bottle division.
4. **Polars Analysis + marimo Exploration** — Load canonical records into LazyFrame, chain analytical queries (region frequency, price stats, cross-tabs, vintage histograms, tasting note frequency), collect once. Export as Parquet. Interactive marimo notebook loads Parquet files for reactive Plotly exploration. Pure consumer of pipeline output — no backward dependencies.

### Critical Pitfalls

1. **Silent non-JSON responses from old.reddit.com** — Reddit serves HTML (login pages, blocks, GDPR consent) at 200 OK with no error indication. Mitigation: Always validate `Content-Type: application/json` before parsing; detect HTML in response body; add circuit breaker after 5+ consecutive non-JSON responses; use `raw_json=1` query parameter.

2. **Non-review posts contaminating the dataset** — A binary keyword gate is too brittle; r/wine contains questions, news, memes, and bottle pics without notes. Mitigation: Implement a review-quality scoring pass (multiple signals: vintage mention, varietal mention, tasting descriptors, price, score); filter on subreddit flair; log all filtered posts with reason codes; calibrate against manually reviewed sample of 200 posts.

3. **Synonym fragmentation + region hierarchy collapse** — "Syrah"/"Shiraz" appear as separate varietals; "Pauillac" doesn't roll up to "Bordeaux." Mitigation: Tree-structured gazetteer with alias maps; every mention of a subregion increments all ancestor counts; length-adaptive fuzzy thresholds prevent false positives on short terms; log every resolution for audit.

4. **Fuzzy matching threshold tuning — the false positive cliff** — A single threshold (e.g., 85) produces false positives on short terms (4-char "Napa" matches "Napa" at 100% but also many 4-char noise terms) and false negatives on long terms. Mitigation: Length-adaptive thresholds (95+ for 4-char, 90-95 for 5-7 char, 85-90 for 8+ char); exact-match first via hash lookup; pre-filter gazetteer by string length (±30%); build test suite of 50+ known difficult examples.

5. **Gazetteer bootstrap — starting from nothing** — The pipeline needs a knowledge base before it can extract, but building it manually is hours of work and misses terms people actually use. Mitigation: Seed from high-signal public data (Wine-Searcher "most popular" lists, OIV top 50 grape varieties, Wikipedia wine regions hierarchy); prioritize top 200 high-frequency entries over completeness; run extraction, analyze unmatched terms, iterate; plan for two extraction passes with gazetteer refinement between them.

6. **Score scale ambiguity** — A reviewer writes "Score: 17" (17/20 → 85/100) and the pipeline normalizes to 17/100. Mitigation: Multi-heuristic scale detection (fraction detection "17/20", >50 → 100-point, ≤20 → check for European context clues, ≤5 → star rating, letter grades → range mapping); store `score_scale` column and `score_confidence` flag; exclude ambiguous scores from aggregates.

7. **Price format proliferation** — "Bought a case for $240" extracts as $240/bottle instead of $20/bottle. Mitigation: Multi-pass extraction (extract amounts, detect per-unit modifiers, detect multi-bottle signals like "case"/"6-pack"/"dozen," divide total by quantity); store `original_price_string`, `price_unit`, and `original_currency` alongside `normalized_usd`; use approximate FX lookup table, not live API.

## Implications for Roadmap

Based on combined research, the pipeline's dependency chain and the architecture's build order dictate a four-phase structure. Each phase maps to one architecture layer, produces independently testable output, and can be verified before the next phase begins.

### Phase 1: Pipeline Foundation

**Rationale:** Everything depends on having data. The scraper is the sole data source, and its output format is the contract that all downstream stages depend on. Scraper bugs propagate to extraction, linking, and analysis — so robust error handling (Content-Type validation, HTML detection, circuit breakers, retry with jitter) must be built first. The manifest tracking system (post-level provenance) is foundational infrastructure needed by every subsequent phase for incremental re-processing and audit. The post filter (review-quality scoring) must be calibrated against real r/wine data before extraction runs — building it here prevents wasted NLP cycles later.

**Delivers:** Project scaffold (`pyproject.toml`, `uv` venv, ruff/mypy/pre-commit), Click CLI skeleton, Reddit scraper with robust error handling (Content-Type validation, HTML detection, circuit breaker), post filter with review-quality scoring pass and subreddit-flair gating, raw JSONL storage with run-ID-named files, manifest tracking system (SQLite or JSON manifest recording `post_id`, `scraped_at`, stage status per run).

**Features from FEATURES.md:** Scrape r/wine posts (P1), Pre-filter non-review posts (P1).

**Pitfalls avoided:** P1 (silent non-JSON responses), P9 (data provenance — manifest system prevents "can't tell what's processed").

**Standard patterns.** Skip research-phase. The scraper follows well-documented Reddit JSON API patterns. No domain-specific uncertainty.

### Phase 2: Gazetteer + Entity Extraction

**Rationale:** This is the core value of the pipeline — transforming unstructured Reddit prose into structured wine entities. The EntityRuler approach depends on a gazetteer seed (curated list of known regions, varietals, AVAs with surface-form patterns). Building this seed from high-signal public sources (Wine-Searcher top lists, OIV grape data, Wikipedia hierarchy) must happen before any extraction code. The spaCy pipeline architecture (EntityRuler ordering, custom components, `nlp.pipe()` batching) follows established patterns but requires domain-specific pattern design and overlap resolution. The extraction extracts raw entities; normalization and linking come later.

**Delivers:** Seed gazetteer in YAML (200+ regions, 100+ varietals with aliases, 50+ AVAs), spaCy pipeline construction with `@Language.factory` custom components (price extractor, score extractor, vintage extractor, tasting note extractor), EntityRuler patterns loaded from YAML, Pydantic `ExtractedEntities` and `WineReview` models, `nlp.pipe(batch_size=50)` batch processing, extraction output as JSONL with raw + extracted entities per post.

**Features from FEATURES.md:** Extract regions/subregions (P1), Extract varietals (P1), Extract vintage year (P1), Extract price (P1), Extract score/rating (P1).

**Pitfalls avoided:** P8 (EntityRuler pattern ordering — priority-ordered patterns, test suite with hand-annotated reviews), P10 (gazetteer bootstrap — seed from public sources, iterate). Partially addresses P6 (score extraction — parses raw scores, normalization deferred to Phase 3) and P7 (price extraction — extracts amounts + context, normalization deferred to Phase 3).

**Needs research.** The gazetteer seed requires sourcing from public wine data (Wine-Searcher scraping, Wikipedia parsing, OIV databases). The EntityRuler test suite needs 30-50 hand-annotated r/wine review snippets to catch pattern collisions. Use `/gsd-plan-phase --research-phase 2` during planning.

### Phase 3: Entity Linking + Normalization

**Rationale:** Raw extracted entities are noisy and fragmented. Without linking, "Napa" and "Napa Valley" are separate counts; without hierarchy, "Pauillac" doesn't contribute to "Bordeaux." Without normalization, scores on different scales (100-point vs. 20-point) are incomparable and prices from multi-bottle purchases distort means. This phase transforms extraction output into analysis-ready canonical records. It depends on Phase 2's Pydantic models as input contract and the gazetteer YAML files as the linking target. The tree-structured gazetteer (with parent pointers for hierarchy propagation) is the architectural cornerstone.

**Delivers:** Tree-structured gazetteer with alias maps and parent pointers (YAML, validated by Pydantic at load time), RapidFuzz entity linking engine with length-adaptive thresholds and exact-match-first strategy, tiered confidence output (HIGH/MEDIUM/UNMATCHED) per linked entity, synonym resolution rules and hierarchy propagation logic, score normalizer with multi-heuristic scale detection (fraction parsing, range detection, letter grades, scale context clues), price normalizer with multi-pass extraction (unit detection, multi-bottle division, currency conversion via hardcoded FX table), `CanonicalReview` Pydantic model with `raw_*` and `canonical_*` fields, fuzzy match test suite (50+ known difficult examples).

**Features from FEATURES.md:** Entity linking via fuzzy matching (P1), Score normalization to 0–100 (P1), Price normalization to USD (P1).

**Pitfalls avoided:** P3 (synonym fragmentation — alias map resolves variants to canonical), P4 (region hierarchy collapse — tree structure with ancestor propagation), P5 (fuzzy threshold false positives — length-adaptive thresholds, pre-filter by length), P6 (score scale ambiguity — scale detection heuristics, `score_scale` column), P7 (price format proliferation — multi-pass detection, `price_unit` column).

**Needs research.** Fuzzy threshold tuning is inherently empirical — the length-adaptive thresholds (95/90/85) are starting points that need calibration against real r/wine extraction output. The scale detection heuristics need testing against a diverse sample of scored reviews. Use `/gsd-plan-phase --research-phase 3` during planning.

### Phase 4: Analysis + Interactive Exploration

**Rationale:** Clean canonical records from Phase 3 are analysis-ready. This phase is pure data processing — Polars queries that can be developed with hand-crafted test data in parallel with earlier phases. It produces the pipeline's deliverable: analytical datasets (Parquet) and interactive marimo notebooks. Every analysis feature reads from the same canonical Polars DataFrame — this is both a strength (consistent data) and a risk (errors in upstream phases propagate). The marimo notebook is the user-facing output; its chart selection and interactivity design should be driven by the analytical questions the project aims to answer.

**Delivers:** Polars analysis module loading canonical records into LazyFrame, analytical dataset generation (region frequency, varietal frequency, price statistics by region, score statistics by varietal, region × varietal cross-tab, vintage histogram, tasting note frequency, value analysis/QPR, reviewer stats), Parquet export with explicit schemas, basic statistical summaries (count, mean, median, std, min/max), marimo notebook with Plotly visualizations (region frequency bar chart, price distribution histogram, score distribution, vintage histogram, price vs. score scatter with region color-coding, interactive filters, reactive cell dependencies).

**Features from FEATURES.md:** Analytical CSV/Parquet export (P1), Basic statistical summaries (P1), marimo + Plotly interactive exploration (P1), Cross-tabulated analysis (P2), Tasting note frequency analysis (P2), Value analysis / QPR (P2), Vintage histogram analysis (P2), Reviewer consistency stats (P3 — optional, low value/cost ratio).

**Pitfalls avoided:** Performance traps (LazyFrame chaining avoids premature `.collect()`, categorical dtypes on repeated string columns, `scan_parquet()` for lazy loading), integration gotchas (explicit Parquet schemas prevent type inference drift, `mo.cache()` for large dataset loading in marimo).

**Standard patterns.** Skip research-phase. Polars analysis queries and Plotly visualizations follow well-documented APIs. The analytical questions (region frequency, price stats, etc.) are standard wine data analysis dimensions confirmed by competitor research.

### Phase Ordering Rationale

- **Phase 1 first** — The scraper is the sole data source. Without real r/wine data, extraction patterns cannot be designed or tested against actual post formats. The manifest system must exist before any stage that produces output that might need re-processing.
- **Phase 2 before Phase 3** — Extraction produces raw entities; linking normalizes them. Score and price extraction (Phase 2) captures the raw values; normalization (Phase 3) resolves scales and currencies. The Pydantic models from Phase 2 are Phase 3's input contract.
- **Phase 3 before Phase 4** — Analysis needs canonical records. Cross-tabulated analysis and value analysis (QPR) specifically require normalized scores and prices from Phase 3.
- **Phase 4 can overlap** — Analysis can be developed with hand-crafted test records while Phase 2 and 3 are being built, since it depends only on the `CanonicalReview` schema, not the extraction implementation.
- **Groupings reflect architecture layers** — Each phase maps to exactly one architecture layer (except Phase 1, which bundles the Data Acquisition layer with the project scaffold). This keeps dependencies clean and enables independent testing of each layer.

### Research Flags

**Phases needing deeper research during planning:**
- **Phase 2 (Gazetteer + Entity Extraction):** The gazetteer seed construction requires sourcing and curating wine-domain data from public sources (Wine-Searcher most-popular lists, OIV grape variety data, Wikipedia wine regions). The EntityRuler pattern design requires a test suite of 30-50 hand-annotated r/wine review snippets to validate pattern ordering and overlap resolution. Domain-specific uncertainty in pattern design.
- **Phase 3 (Entity Linking + Normalization):** Fuzzy matching threshold calibration is inherently empirical — the length-adaptive starting points (95/90/85) must be validated against real extraction output. Score scale detection heuristics need testing against diverse review samples. Domain-specific uncertainty in threshold tuning and scale detection.

**Phases with standard patterns (skip research-phase):**
- **Phase 1 (Pipeline Foundation):** Well-documented Reddit JSON API with established scraping patterns (pagination via `after` cursors, rate limiting with jitter, Content-Type validation). The manifest tracking approach follows standard data engineering provenance patterns. No domain-specific uncertainty.
- **Phase 4 (Analysis + Exploration):** Polars groupby/aggregation queries and Plotly express charts follow well-documented APIs. The analytical dimensions (region frequency, price stats, cross-tabs) are standard wine data analysis questions confirmed by competitor research (Wine-Review-EDA, Vivino feature set).

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies are version-pinned with verified Python compatibility. spaCy, Polars, and RapidFuzz are sourced from official PyPI listings and official documentation. No version conflicts exist in the recommended combination. |
| Features | MEDIUM | Feature landscape is well-understood from competitor analysis (Vivino, CellarTracker, open-source EDA projects) and direct r/wine observation. However, open-source ecosystem is thin — no reusable Reddit wine scraping pipelines exist. Feature prioritization is reasoned but not validated against real extraction output. |
| Architecture | HIGH | Four-layer ETL architecture follows established data engineering patterns with official documentation for every component (spaCy pipeline architecture, RapidFuzz API, Polars LazyFrame, Reddit JSON API). Pydantic contracts, tiered confidence linking, and gazetteer-driven extraction are patterns validated by library documentation and industry practice. |
| Pitfalls | HIGH | Pitfall 1 (non-JSON responses) was directly confirmed by live testing (old.reddit.com returned HTML on 2026-11-27). All other pitfalls are derived from official documentation (spaCy pattern resolution, RapidFuzz threshold behavior, Reddit API pagination limits) or well-documented wine domain conventions (scoring scales, regional hierarchy, grape synonyms). Prevention strategies are concrete and implementable. |

**Overall confidence: HIGH**

The stack and architecture are well-supported by official documentation and version-verified. The feature landscape is clear even though the open-source wine NLP ecosystem is thin. The pitfalls research is the strongest area — direct observation confirmed the most critical pitfall, and every pitfall has a detailed, actionable prevention strategy. The main uncertainty is in the empirical calibration steps (fuzzy thresholds, review-quality scoring heuristics, scale detection) which are inherently data-dependent and can only be resolved during implementation with real r/wine data.

### Gaps to Address

- **Gazetteer seed sources:** The research identifies that the seed should come from Wine-Searcher "most popular" lists, OIV grape data, and Wikipedia wine regions — but the specific URLs/APIs, data formats, and extraction approach for these sources need to be determined during Phase 2 planning. Handle during planning with a gazetteer sourcing spike.
- **Review-quality scoring calibration:** The multi-signal scoring pass (vintage mention + varietal mention + tasting descriptors + price + score) needs empirical calibration against real r/wine posts. The specific weights and threshold need to be tuned. Handle during Phase 1 planning: plan a calibration step that processes 200 manually-reviewed posts to set scoring parameters.
- **marimo notebook design:** The specific charts, interactivity patterns, and layout for the marimo notebook are described at a high level but not designed in detail. A UI-phase pass during Phase 4 planning can solidify chart selection, filter design, and reactive dependency structure.
- **Tasting note stopword list:** The wine-domain stopword list needed for tasting note frequency analysis (removing generic words like "wine," "bottle," "drink" while preserving descriptors like "oaky," "mineral") is not sourced. Handle during Phase 4 planning: build an initial list from wine-tasting glossaries and refine based on frequency analysis output.

## Sources

### Primary (HIGH confidence)
- spaCy 3.8.18 official documentation — EntityRuler API, pipeline architecture, custom components, `nlp.pipe()` batching, model availability
- Polars 2.2.0 official documentation — LazyFrame API, query optimizer, Parquet I/O, categorical optimization
- RapidFuzz 3.14.6 official documentation — `process.extract()`, `cdist()`, `token_sort_ratio`, scorers, thresholds
- marimo 0.26.0 official documentation — Plotly integration, reactive cells, `mo.cache()`, `.py` file format
- Reddit JSON API documentation — listing endpoint, pagination with `after`/`before`/`limit`, `raw_json` parameter, post object structure
- Old Reddit JSON API rendering verified working (2026-11-27) — used for STACK scraping approach validation
- OIV (International Organisation of Vine and Wine) — grape variety data for gazetteer seed
- Wine-Searcher.com — region/varietal popularity lists for gazetteer seed prioritization

### Secondary (MEDIUM confidence)
- PyPI listings for all recommended packages — version numbers, Python compatibility, release dates
- Competitor analysis: Vivino (78M users), CellarTracker, Wine-Review-EDA (GitHub) — feature landscape and analytical dimensions
- r/wine community rules and post patterns (direct observation) — review post characteristics, data quality signals
- Wine domain conventions: Robert Parker 100-point scale, Jancis Robinson 20-point scale, AOC/AVA/DOCG hierarchy systems, grape synonym documentation

### Tertiary (LOW confidence)
- Reddit rate limiting for unauthenticated old.reddit.com — community-observed ~30 req/min before HTML blocks; not officially documented
- Pipeline provenance patterns from data engineering ETL practice — manifest files, incremental processing, content-addressed storage
- Web search results for wine regex extraction patterns, Reddit scraping patterns, ETL architecture — used for pattern guidance, not authoritative

---

*Research completed: 2026-11-27*
*Ready for roadmap: yes*
