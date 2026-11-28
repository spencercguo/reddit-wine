# Roadmap: r/wine Review Analysis

## Overview

A five-phase journey from raw Reddit posts to interactive wine analytics. Phase 1 establishes the data acquisition foundation — scraping r/wine posts with robust error handling and filtering out non-reviews. Phase 2 builds the wine-domain knowledge base (gazetteer) and extracts structured entities from unstructured review text using spaCy. Phase 3 links extracted entities to canonical forms via fuzzy matching and normalizes scores/prices. Phase 4 generates analytical datasets answering community questions about regional frequency, price distributions, and rating patterns. Phase 5 ties everything together with an interactive marimo notebook and end-to-end pipeline scripts.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Pipeline Foundation** - Scrape r/wine posts with robust error handling, filter non-reviews, and establish provenance tracking
- [ ] **Phase 2: Gazetteer + Entity Extraction** - Build wine knowledge base and extract regions, varietals, vintages, prices, and scores from review text
- [ ] **Phase 3: Entity Linking + Normalization** - Resolve extracted entities to canonical forms with fuzzy matching and normalize scores and prices
- [ ] **Phase 4: Analysis** - Generate analytical datasets answering wine community questions
- [ ] **Phase 5: Exploration & Pipeline** - End-to-end pipeline scripts and interactive marimo notebook with Plotly visualizations

## Phase Details

### Phase 1: Pipeline Foundation
**Goal**: Reliably scrape and filter r/wine posts into a clean, auditable dataset
**Mode**: mvp
**Depends on**: Nothing (first phase)
**Requirements**: SCRP-01, SCRP-02, SCRP-03, SCRP-04, SCRP-05, SCRP-06, ORCH-03
**Success Criteria** (what must be TRUE):
  1. User can initiate a scrape of r/wine from the command line and receive paginated posts with rate limiting applied (2 req/sec with jitter)
  2. Image posts with empty selftext automatically include OP's first top-level comment merged into the review text
  3. The scraper survives non-JSON responses (HTML) by detecting Content-Type mismatches, retrying with backoff, and halting after consecutive failures via circuit breaker
  4. Scraped posts are saved to data/raw/ as JSON files with scrape run metadata (timestamp, post count, errors encountered)
  5. Non-review posts are filtered out and logged with rejection reasons, while genuine reviews pass through a multi-signal scoring gate
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md -- Walking Skeleton: scaffold, storage, single-page scraper, complete filter, integration test
- [ ] 01-02-PLAN.md -- Scraper resilience: pagination, circuit breaker, OP comment fetch, resume/--fresh logic

### Phase 2: Gazetteer + Entity Extraction
**Goal**: Extract structured wine entities from unstructured review text using a seed knowledge base
**Mode**: mvp
**Depends on**: Phase 1
**Requirements**: GAZE-01, GAZE-02, GAZE-03, GAZE-04, GAZE-05, EXTR-01, EXTR-02, EXTR-03, EXTR-04, EXTR-05, EXTR-06, EXTR-07
**Success Criteria** (what must be TRUE):
  1. A seed gazetteer exists covering 200+ wine regions, 100+ varietals with synonyms/abbreviations, and 50+ AVAs/AOCs/DOCGs, including non-European regions
  2. Vintage years (19xx–20xx) are extracted from review text with false positives filtered by wine-context proximity rules
  3. Prices are detected across multiple currencies ($/€/£) and shorthand formats, with multi-bottle purchase amounts correctly divided
  4. Explicit scores on multiple scales (100-point, 20-point, 5-point) are extracted with the original scale preserved for audit
  5. Wine regions, subregions, and grape varietals are extracted via spaCy EntityRuler, with NER fallback discovery on posts where EntityRuler coverage is low
**Plans**: TBD

### Phase 3: Entity Linking + Normalization
**Goal**: Resolve extracted entities to canonical forms and normalize scores and prices for analysis
**Mode**: mvp
**Depends on**: Phase 2
**Requirements**: LINK-01, LINK-02, LINK-03, LINK-04, LINK-05, LINK-06, NORM-01, NORM-02, NORM-03
**Success Criteria** (what must be TRUE):
  1. Extracted region and varietal spans are matched to canonical gazetteer entries via exact lookup, alias resolution, and length-adaptive fuzzy matching with confidence tiers (HIGH/MEDIUM/UNMATCHED)
  2. Domain inference rules fill in missing entities — for example, "Barolo" implies Nebbiolo varietal, Italy country, and Piedmont region
  3. Subregion mentions automatically propagate counts upward through the region hierarchy (Pauillac → Médoc → Bordeaux → France)
  4. All scores are normalized to a 0–100 scale regardless of original scoring system, with original scale preserved and ambiguous scores flagged
  5. All prices are normalized to approximate USD with currency conversion, multi-bottle division, and qualifier handling applied
**Plans**: TBD

### Phase 4: Analysis
**Goal**: Generate analytical datasets answering wine community questions about regions, prices, ratings, and descriptors
**Mode**: mvp
**Depends on**: Phase 3
**Requirements**: ANAL-01, ANAL-02, ANAL-03, ANAL-04, ANAL-05, ANAL-06, ANAL-07, ANAL-08
**Success Criteria** (what must be TRUE):
  1. Region frequency table is produced showing which regions are most discussed, with over/under-representation analysis
  2. Price statistics by region (histogram bins, mean, median, quartiles) are computed and exportable as CSV
  3. Rating statistics are broken down by region, price bracket, and vintage, with vintage-by-score trend analysis
  4. Cross-tab analysis datasets (region × price brackets, region × rating buckets) are available for export
  5. Tasting note word frequency analysis surfaces the top descriptors across the community, and value analysis (QPR) flags high-quality/low-price standouts
**Plans**: TBD

### Phase 5: Exploration & Pipeline
**Goal**: Run the full pipeline end-to-end and explore results in an interactive marimo notebook
**Mode**: mvp
**Depends on**: Phase 4
**Requirements**: NBOOK-01, NBOOK-02, NBOOK-03, NBOOK-04, NBOOK-05, NBOOK-06, NBOOK-07, NBOOK-08, ORCH-01, ORCH-02
**Success Criteria** (what must be TRUE):
  1. User can run the complete pipeline on sample data (~100 posts) with a single command and see summary output
  2. User can run the complete pipeline on the full dataset (2000+ posts) with progress bars showing stage-by-stage progress
  3. An interactive marimo notebook loads the processed data and displays a region frequency bar chart (top 15), price histogram, and rating histogram with rating-by-region boxplots
  4. The notebook includes a scatter plot of price vs. rating colored by region, a vintage timeline bar chart, and a tasting note word frequency horizontal bar chart (top 20)
  5. The notebook presents a data quality overview dashboard showing what percentage of posts have extracted scores, prices, and inferred vs. explicit fields
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Pipeline Foundation | 0/2 | Planned | - |
| 2. Gazetteer + Entity Extraction | 0/0 | Not started | - |
| 3. Entity Linking + Normalization | 0/0 | Not started | - |
| 4. Analysis | 0/0 | Not started | - |
| 5. Exploration & Pipeline | 0/0 | Not started | - |
