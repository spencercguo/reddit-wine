# Requirements: r/wine Review Analysis

**Defined:** 2026-11-27
**Core Value:** Turn thousands of unstructured community wine reviews into queryable, analyzable data — surfacing the hidden patterns in r/wine's tasting notes, price distributions, and regional preferences that no single post reveals alone.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Scraping & Pre-Filtering

- [ ] **SCRP-01**: Fetch posts from r/wine via old.reddit.com JSON API with pagination (up to 2000+ posts), respecting 2 req/sec rate limit
- [ ] **SCRP-02**: For image posts (empty selftext), fetch OP's first top-level comment and merge into review_text
- [ ] **SCRP-03**: Validate Content-Type of all API responses; handle non-JSON (HTML) responses with retry and circuit breaker
- [ ] **SCRP-04**: Save raw JSON posts to data/raw/ with scrape run metadata (timestamp, count, errors)
- [ ] **SCRP-05**: Filter non-review posts using token gate (≥30 tokens) and wine-specific keyword gate (sensory terms, structure cues, score cues, varietal/region names)
- [ ] **SCRP-06**: Log skipped posts with rejection reason for pipeline auditability

### Gazetteer & Knowledge Base

- [ ] **GAZE-01**: Create seed gazetteer with canonical wine regions, subregions, AVAs, AOCs, DOCGs from public sources
- [ ] **GAZE-02**: Create seed varietal list with canonical grape names and common synonyms/abbreviations
- [ ] **GAZE-03**: Build alias table supporting common shorthand ("CDR" → Côtes du Rhône, "Napa" → Napa Valley)
- [ ] **GAZE-04**: Encode hierarchical relationships (France → Burgundy → Côte de Nuits → Vosne-Romanée)
- [ ] **GAZE-05**: Include non-European regions (USA, Australia, NZ, South Africa, Chile, Argentina) from start

### Entity Extraction

- [ ] **EXTR-01**: Extract vintage years (19xx–20xx) with wine-context proximity guardrails (avoid non-wine year mentions)
- [ ] **EXTR-02**: Extract prices with currency detection ($/€/£ + shorthand like "~40 bucks", "sub-$30") and multi-bottle price division
- [ ] **EXTR-03**: Extract explicit scores on 100-point and 5-point scales via regex with scale detection
- [ ] **EXTR-04**: Extract wine regions, subregions, AVAs, AOCs, DOCGs via spaCy EntityRuler loaded from gazetteer
- [ ] **EXTR-05**: Extract grape varietals via spaCy EntityRuler with homonym guardrails (context proximity + capitalization check)
- [ ] **EXTR-06**: Run spaCy NER discovery pass (ORG/PERSON → candidate producers, GPE/LOC → candidate regions) on low-coverage posts
- [ ] **EXTR-07**: Batch process posts via nlp.pipe() for performance; disable unused pipeline components (tagger, parser, lemmatizer)

### Entity Linking

- [ ] **LINK-01**: Exact + alias lookup against canonical gazetteer for all extracted entity spans
- [ ] **LINK-02**: Fuzzy fallback with RapidFuzz (length-adaptive Jaro-Winkler threshold: higher for short terms, relaxed for long terms) against all canonical labels
- [ ] **LINK-03**: Domain inference rules (if appellation == "Barolo" and varietal missing → varietal: Nebbiolo, country: Italy, region: Piedmont; extend for Bordeaux blends, Rioja, Chianti, etc.)
- [ ] **LINK-04**: Flag unmapped spans appearing ≥3 times to data/processed/unmapped_spans.csv for periodic gazetteer updates
- [ ] **LINK-05**: Preserve both raw extracted text and canonical match for audit trails
- [ ] **LINK-06**: Hierarchical region propagation — link subregions to parent regions for aggregation

### Score & Price Normalization

- [ ] **NORM-01**: Normalize all scores to 0–100 scale (detect and convert 5pt, 10pt, 20pt scales)
- [ ] **NORM-02**: Normalize all prices to approximate USD (convert €/£/CAD/AUD; strip qualifiers like ~ and sub-$)
- [ ] **NORM-03**: Handle range scores ("88-92 points"), multi-bottle prices ("$15 each"), and qualitative-only reviews

### Analysis

- [ ] **ANAL-01**: Generate region frequency table with count, percentage, over/under-representation analysis
- [ ] **ANAL-02**: Generate price statistics by region (histogram bins, mean, median, q1, q3)
- [ ] **ANAL-03**: Generate rating statistics by region, price bracket, and vintage
- [ ] **ANAL-04**: Generate cross-tab analysis: region × price brackets, region × rating buckets
- [ ] **ANAL-05**: Generate vintage histogram (year distribution) and vintage × score analysis
- [ ] **ANAL-06**: Generate tasting note word frequency (wine-domain filtered) — top-N descriptors
- [ ] **ANAL-07**: Generate value analysis (QPR) flagging high-quality/low-price standouts
- [ ] **ANAL-08**: All analysis uses Polars LazyFrame chaining; export datasets to data/processed/ as CSV

### Exploration Notebook

- [ ] **NBOOK-01**: Region bar chart (top 15 by count)
- [ ] **NBOOK-02**: Price histogram (USD, log or trimmed at 99th percentile)
- [ ] **NBOOK-03**: Rating histogram + rating-by-region boxplots (top N regions)
- [ ] **NBOOK-04**: Scatter plot: price vs. rating, colored by region
- [ ] **NBOOK-05**: Vintage timeline bar chart by year
- [ ] **NBOOK-06**: Tasting note word frequency horizontal bar chart (top 20)
- [ ] **NBOOK-07**: Data quality overview dashboard (% posts with scores, prices, inferred vs. explicit fields)
- [ ] **NBOOK-08**: marimo notebook with reactive Plotly charts

### Pipeline Orchestration

- [ ] **ORCH-01**: End-to-end pipeline script for sample data (~100 posts) with summary output
- [ ] **ORCH-02**: End-to-end pipeline script for full dataset (2000+ posts) with tqdm progress bars
- [ ] **ORCH-03**: Manifest/provenance system tracking which posts were processed by each pipeline stage (enables resume and incremental updates)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Analysis Extensions

- **ANAL-10**: Reviewer/author posting frequency and score consistency analysis
- **ANAL-11**: Wine blend detection heuristics and multi-varietal handling

### Quality

- **QUAL-01**: Extraction accuracy evaluation suite (precision/recall against hand-labeled gold set of 100 posts)
- **QUAL-02**: Fuzzy threshold calibration from empirical test data

### Extra Features

- **EXT-01**: Multi-community support (r/bourbon, r/scotch) — parameterized pipeline
- **EXT-02**: Automated periodic scraping (if warranted after manual validation)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Web UI or hosted dashboard | Local pipeline only; export CSV for BI tools |
| Real-time/automated scraping | Manual pipeline runs only; ethical concerns with unattended scraping |
| Wine recommendation engine | ML undertaking crosses into consumer product territory |
| Sentiment analysis | Method not selected; wine language confuses generic tools ("aggressive tannins" reads negative but is neutral/positive) |
| Label/image scanning | Data source is text, not images; Vivino already dominates this space |
| Multi-subreddit support | Validate on r/wine first before diversifying |
| Personal user profiles/auth | This is what Vivino/CellarTracker do; pipeline is a data tool |
| Social features | Reddit is the discussion platform; the pipeline analyzes it |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| SCRP-01 | — | Pending |
| SCRP-02 | — | Pending |
| SCRP-03 | — | Pending |
| SCRP-04 | — | Pending |
| SCRP-05 | — | Pending |
| SCRP-06 | — | Pending |
| GAZE-01 | — | Pending |
| GAZE-02 | — | Pending |
| GAZE-03 | — | Pending |
| GAZE-04 | — | Pending |
| GAZE-05 | — | Pending |
| EXTR-01 | — | Pending |
| EXTR-02 | — | Pending |
| EXTR-03 | — | Pending |
| EXTR-04 | — | Pending |
| EXTR-05 | — | Pending |
| EXTR-06 | — | Pending |
| EXTR-07 | — | Pending |
| LINK-01 | — | Pending |
| LINK-02 | — | Pending |
| LINK-03 | — | Pending |
| LINK-04 | — | Pending |
| LINK-05 | — | Pending |
| LINK-06 | — | Pending |
| NORM-01 | — | Pending |
| NORM-02 | — | Pending |
| NORM-03 | — | Pending |
| ANAL-01 | — | Pending |
| ANAL-02 | — | Pending |
| ANAL-03 | — | Pending |
| ANAL-04 | — | Pending |
| ANAL-05 | — | Pending |
| ANAL-06 | — | Pending |
| ANAL-07 | — | Pending |
| ANAL-08 | — | Pending |
| NBOOK-01 | — | Pending |
| NBOOK-02 | — | Pending |
| NBOOK-03 | — | Pending |
| NBOOK-04 | — | Pending |
| NBOOK-05 | — | Pending |
| NBOOK-06 | — | Pending |
| NBOOK-07 | — | Pending |
| NBOOK-08 | — | Pending |
| ORCH-01 | — | Pending |
| ORCH-02 | — | Pending |
| ORCH-03 | — | Pending |

**Coverage:**
- v1 requirements: 45 total
- Mapped to phases: 0
- Unmapped: 45

---
*Requirements defined: 2026-11-27*
*Last updated: 2026-11-27 after initial definition*
