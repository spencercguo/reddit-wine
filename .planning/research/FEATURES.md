# Feature Research: r/wine Review Analysis

**Domain:** Wine review aggregation and analysis pipeline (community-driven, not commercial)
**Researched:** 2026-11-27
**Confidence:** MEDIUM

## Executive Summary

The wine data analysis ecosystem spans three tiers: consumer apps (Vivino, 78M+ users; CellarTracker), commercial data platforms (Wine-Searcher), and open-source/educational projects (Kaggle notebooks, GitHub repos). Consumer apps focus on personal tracking and discovery — label scanning, cellar management, personalized recommendations. The open-source ecosystem is thinner, dominated by one-off EDA notebooks on Kaggle datasets rather than reusable pipelines. This project occupies a distinct niche: a **community review aggregation pipeline** that extracts structure from natural-language forum posts, not from pre-structured review databases.

The feature landscape for this pipeline maps cleanly onto the standard ETL → analysis → visualization arc. Table stakes are entity extraction fidelity and basic aggregations. Differentiation comes from entity linking quality (handling community naming ambiguity), cross-tabulated analysis, and interactive exploration. The biggest risk is scope creep into app features (recommendations, personalization, web UI) that belong in consumer products, not an analytical pipeline.

## Feature Landscape

### Table Stakes (Users Expect These)

These are features that anyone consuming wine review data expects. Missing these = the pipeline feels broken or useless.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Scrape posts from r/wine | No data, no analysis. The entire pipeline starts here. | MEDIUM | old.reddit.com JSON API; rate-limited at 2 req/sec; ~2000 posts target. Must handle pagination, Reddit's 1000-post listing cap. |
| Pre-filter review vs. non-review posts | r/wine has questions, news, memes, bottle pics without notes. Without filtering, entity extraction wastes cycles on noise. | LOW-MEDIUM | Token gate (minimum length) + keyword gate (score/presence of tasting vocabulary). Subreddit rule requires tasting notes with bottle pictures — strong signal exists. |
| Extract region/subregion/AVA/AOC | The single most important analytical dimension. "Where are these wines from?" is the first question anyone asks. | HIGH | Requires spaCy EntityRuler with curated gazetteer. Wine regions are hierarchical (France → Bordeaux → Margaux). Community naming is inconsistent ("Napa" vs. "Napa Valley"). |
| Extract grape varietal | Second-most important dimension. "What are people drinking?" | MEDIUM | Known varietals map well to gazetteer rules. Edge cases: blends (often unlabeled), synonyms ("Syrah"/"Shiraz"). |
| Extract vintage year | Essential for time-series analysis. "What vintages are people opening now?" | LOW | Regex for 4-digit year patterns (19xx–20xx). Ambiguity: "the '15" vs. full year. |
| Extract price | Core economic dimension. "What are people paying?" | MEDIUM | Currency symbols, formats vary ($XX, XX USD, $XX.XX). Must normalize all to USD. Many posts lack price entirely. |
| Extract score/rating | Core quality dimension. "How good is it?" | MEDIUM | Multiple scales: 100-point, 20-point, 5-star, letter grades. Non-standard phrasing ("92pts", "92/100", "4/5"). Some posts have no explicit score. |
| Normalize scores to 0–100 scale | Comparable analysis requires a unified scale. | LOW | Pure arithmetic transform once scale is identified. Edge cases: qualitative-only reviews with no explicit score. |
| Normalize prices to USD | Comparable analysis across currencies. | LOW | Currency detection + conversion. But: most r/wine prices are already in USD given the subreddit's primarily US audience. |
| Entity linking via fuzzy matching | "Napa" and "Napa Valley" must be counted together, not as separate regions. | MEDIUM | RapidFuzz for fuzzy string matching. Requires canonical name list. Domain inference rules for hierarchical relationships ("Margaux" → "Bordeaux" → "France"). |
| Export analytical datasets as CSV | The pipeline is the product; output must be consumable. | LOW | Polars to CSV. Standard format, no surprises. |
| Basic statistical summaries | Users need to see the shape of the data before digging deeper. | LOW | Counts, means, medians, standard deviations, min/max per dimension. |

### Differentiators (Competitive Advantage)

Features that set this pipeline apart from existing tools. Not required to be useful, but make the output uniquely valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Cross-tabulated analysis (region × varietal, price × score) | Single-dimension stats are shallow. Cross-tabs reveal patterns: "Does Napa Cabernet justify its price premium in community scores?" | MEDIUM | Polars groupby + pivot operations. Requires sufficient data volume per cell to be statistically meaningful. |
| Tasting note frequency analysis | What descriptors does the community actually use? "Bold" vs. "elegant," "oaky" vs. "mineral" — frequency reveals community vocabulary preferences. | MEDIUM | Tokenization + stopword removal + frequency counting. Requires wine-domain stopword list (removing generic words like "wine," "bottle," "drink"). |
| Reviewer/author frequency and consistency analysis | Who posts most? Do some reviewers consistently rate higher/lower? | LOW-MEDIUM | Author extraction from Reddit metadata. Score distribution per author. Privacy note: aggregate stats only, no profiling. |
| Value analysis (QPR — Quality-Price Ratio) | The question every wine drinker asks: "What's the best value?" Score/price ratio normalized and ranked. | LOW | Derived metric once scores and prices are extracted. Requires minimum review count per wine to be meaningful. |
| Vintage histogram analysis | What vintages are people drinking right now? Are older vintages over-represented in high scores? | LOW | Vintage extraction → histogram. Cross-tab with score to see age-quality relationship. |
| Interactive exploration via marimo + Plotly | Static CSVs are the baseline. Interactive filtering, zooming, and cross-filtering make exploration joyful. | MEDIUM | marimo notebook with Plotly express charts. Reactive cells update charts when filters change. |
| r/wine-specific community insights | General wine analysis tools exist. This pipeline captures *community* patterns: what the r/wine demographic values, what it ignores. | LOW (emergent) | Emerges naturally from r/wine data. Not a feature to build, but a lens to apply. |
| Pre-filter quality heuristics | Token gate + keyword gate tuned to r/wine's specific post patterns. Reduces noise that plagues generic scrapers. | LOW-MEDIUM | Domain-specific patterns: review posts tend to mention producer, vintage, tasting notes together. Questions and meta posts don't. |

### Anti-Features (Things to Deliberately NOT Build)

Features that seem appealing but create scope problems, technical debt, or misalignment with the project's identity as a local pipeline.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Web UI or hosted dashboard | "I want to share results without running code" | Transforms a local pipeline into a web app. Requires auth, deployment, ongoing maintenance. Explicitly out of scope per PROJECT.md. | Export CSV → load into any BI tool (Tableau, Google Sheets, Observable). marimo notebook can be shared as static HTML. |
| Real-time or automated periodic scraping | "I want fresh data automatically" | Reddit rate limits. Ethical concerns with unattended scraping. Pipeline stability dependency on Reddit API availability. Explicitly out of scope. | Manual triggered pipeline runs. Document how to run it; don't automate cron. |
| Wine recommendation engine | "Tell me what to buy based on the data" | Major ML undertaking. Cold-start problem. Crosses from analysis tool into consumer product territory. Requires personal taste profiles. | Let users draw their own conclusions from the data. The analysis is the recommendation. |
| Sentiment analysis on tasting notes (prematurely) | "Automatically tell if reviews are positive" | VADER/TextBlob do poorly on wine language ("aggressive tannins" sounds negative but is neutral/positive in context). Transformers are overkill for a first iteration. Not committed per PROJECT.md. | Keep method selection open. If added later, evaluate domain-specific approaches (fine-tuned model on wine corpus). |
| Multi-subreddit support | "Also scrape r/bourbon, r/scotch, r/beer" | Each community has different vocabulary, different entity types, different conventions. Dilutes focus before core works for r/wine. | Validate on r/wine first. Architecture can be parameterized later if successful. |
| Personal user profiles or auth | "Track my own tasting notes" | This is what Vivino/CellarTracker already do well. Adding it makes this a consumer app, not a data pipeline. | Export data; users can combine with their own tracking in whatever tool they prefer. |
| Social features (comments, voting, sharing) | "Let people discuss the analysis" | Adds moderation, auth, notification infrastructure. Reddit already IS the discussion platform — the pipeline analyzes it, it doesn't replace it. | Link back to original Reddit posts for discussion. |
| Label/image scanning | "Scan bottle labels to auto-populate data" | Requires computer vision pipeline (OpenCV, OCR, or ML model). Vivino already does this with 3.5B+ scans. No added value for a Reddit text analysis tool. | Not applicable — data source is text, not images. |
| ML-based wine quality prediction from reviews | "Predict scores from tasting notes" | Training data requirements, model maintenance, and the output ("predicted score") is less interesting than the actual community data. | Keep the pipeline descriptive, not predictive. Let the data speak for itself. |

## Feature Dependencies

```
Scrape Posts (old.reddit.com API)
    └──requires──> Rate Limiting (2 req/sec)
    └──requires──> Pagination Handling

Pre-Filter Non-Review Posts
    └──requires──> Scrape Posts
    └──enhances──> Entity Extraction (reduces noise)

Entity Extraction (spaCy EntityRuler + NER)
    └──requires──> Pre-Filter Non-Review Posts
    └──requires──> Gazetteer/Knowledge Base (region names, varietal names)
    ├──produces──> Region, Subregion, Varietal, Vintage, Price, Score

Entity Linking (RapidFuzz fuzzy matching)
    └──requires──> Entity Extraction
    └──requires──> Canonical Name List
    ├──produces──> Normalized Entities

Score Normalization (0–100 scale)
    └──requires──> Entity Extraction (score field)
    ├──enhances──> Cross-Tabulated Analysis

Price Normalization (USD)
    └──requires──> Entity Extraction (price field)
    ├──enhances──> Value Analysis (QPR)

Analytical Datasets (Polars → CSV)
    └──requires──> Entity Linking
    └──requires──> Score Normalization
    └──requires──> Price Normalization
    ├──produces──> All downstream analysis

Region Frequency Analysis ──requires──> Analytical Datasets
Price Statistics ──requires──> Analytical Datasets
Score Statistics ──requires──> Analytical Datasets
Cross-Tabulated Analysis ──requires──> Analytical Datasets
Vintage Histogram ──requires──> Analytical Datasets
Tasting Note Frequency ──requires──> Analytical Datasets
Value Analysis (QPR) ──requires──> Price Normalization + Score Normalization

Interactive Exploration (marimo + Plotly)
    └──requires──> Analytical Datasets
    └──enhances──> All Analysis Features (makes them interactive)
```

### Dependency Notes

- **Entity Extraction requires Gazetteer:** The EntityRuler approach depends on a curated list of known wine regions, subregions, AVAs, and varietals. This gazetteer is a project asset that grows over time. Without it, extraction is pure NER (lower precision, hallucinates).
- **Entity Linking requires Canonical Name List:** Fuzzy matching needs a target. The canonical list defines what "correct" entity names look like. Synonyms ("Napa" → "Napa Valley") and hierarchies ("Margaux" → "Bordeaux" → "France") must be encoded.
- **Score/Price Normalization is prerequisite for all comparative analysis:** Without a common scale, comparisons across reviews are meaningless. These must be built before any cross-tab or value metric.
- **Analytical Datasets are the integration point:** Every analysis feature reads from the same Polars DataFrames. This is both a strength (consistent data) and a risk (errors in the pipeline propagate to all analysis).

## MVP Definition

### Launch With (v1)

Minimum viable product — the smallest pipeline that produces useful, publishable analysis.

- [ ] **Scrape 2000+ r/wine posts** — Foundation. Without data, nothing else matters. Target volume gives statistically useful sample sizes for regional/varietal analysis.
- [ ] **Pre-filter non-review posts** — Token gate + keyword gate. Prevents entity extraction from wasting cycles on "What should I buy?" questions.
- [ ] **Extract core entities: region, varietal, vintage, price, score** — The extraction that defines the pipeline. Each entity type needs its own extraction strategy (regex for vintage, EntityRuler for regions, pattern matching for scores/prices).
- [ ] **Entity linking with fuzzy matching** — Without this, "Napa" and "Napa Valley" are separate entities, making frequency counts misleading.
- [ ] **Normalize scores to 0–100 and prices to USD** — Single scale makes all downstream comparisons valid.
- [ ] **Export analytical CSVs** — The pipeline's deliverable. Including: region frequency, varietal frequency, price stats (mean/median/min/max by category), score stats, vintage histogram.
- [ ] **Basic statistical summaries** — Count, mean, median, std per dimension. Callable as a function, printed or saved alongside CSVs.
- [ ] **marimo notebook with Plotly visualizations** — Interactive exploration. Charts: region frequency bar, price distribution histogram, score distribution, vintage histogram, price vs. score scatter.

### Add After Validation (v1.x)

Features to add once the core pipeline works and produces interesting results.

- [ ] **Cross-tabulated analysis (region × varietal, price × score)** — Trigger: Core pipeline produces consistent data. Add when single-dimension analysis shows patterns worth cross-referencing.
- [ ] **Tasting note frequency analysis** — Trigger: Entity extraction is stable. Add when the "what words does the community use?" question becomes interesting.
- [ ] **Value analysis (QPR — Quality-Price Ratio)** — Trigger: Sufficient data with both score and price (many posts lack one or both). Add when there are enough scored+priced reviews to compute meaningful ratios.
- [ ] **Reviewer frequency and consistency stats** — Trigger: Author data is clean and deduplicated. Add when community dynamics become interesting.

### Future Consideration (v2+)

Features to defer until the pipeline is proven valuable and stable.

- [ ] **Sentiment analysis** — Method TBD. Evaluate VADER/TextBlob (fast but inaccurate on wine language), then consider fine-tuned transformer if justified. Only add if community asks "are reviews getting more positive over time?"
- [ ] **Multi-subreddit support** — Parameterize the pipeline for r/bourbon, r/scotch, etc. Only if r/wine analysis proves the concept.
- [ ] **Automated report generation** — Generate a PDF/HTML summary report from the analytical datasets. Only if sharing results with non-technical audiences becomes a need.

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Scrape r/wine posts | HIGH | MEDIUM | P1 |
| Pre-filter non-review posts | MEDIUM | LOW-MEDIUM | P1 |
| Extract regions/subregions | HIGH | HIGH | P1 |
| Extract varietals | HIGH | MEDIUM | P1 |
| Extract vintage year | MEDIUM | LOW | P1 |
| Extract price | HIGH | MEDIUM | P1 |
| Extract score/rating | HIGH | MEDIUM | P1 |
| Entity linking (fuzzy match) | HIGH | MEDIUM | P1 |
| Score normalization | HIGH | LOW | P1 |
| Price normalization | HIGH | LOW | P1 |
| Analytical CSV export | HIGH | LOW | P1 |
| Basic statistical summaries | HIGH | LOW | P1 |
| marimo + Plotly exploration | HIGH | MEDIUM | P1 |
| Cross-tabulated analysis | HIGH | MEDIUM | P2 |
| Tasting note frequency | MEDIUM | MEDIUM | P2 |
| Value analysis (QPR) | MEDIUM | LOW | P2 |
| Reviewer consistency stats | LOW | LOW-MEDIUM | P3 |
| Sentiment analysis | MEDIUM | HIGH | P3 (deferred) |
| Multi-subreddit support | LOW | MEDIUM | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible after validation
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Vivino (commercial) | CellarTracker (commercial) | Wine-Review-EDA (open source) | Wine Cellar (open source) | This Pipeline |
|---------|---------------------|----------------------------|-------------------------------|---------------------------|---------------|
| Data source | User-submitted label scans + reviews | User-submitted tasting notes + inventory | WineEnthusiast Kaggle dataset (static) | User-submitted wine entries | Reddit r/wine community posts |
| Entity extraction | OCR from label images + structured forms | Structured input forms | Pre-extracted CSV columns | Structured input forms | Regex + spaCy from unstructured text |
| Score scale | 5-star | 100-point (user-defined) | 100-point (pre-extracted) | User-defined | Normalized to 0–100 from multiple scales |
| Price data | Retailer prices via marketplace | User-entered purchase price | Pre-extracted USD | User-entered | Regex extraction from text + USD normalization |
| Regional analysis | Yes (explore by region) | Limited (search/filter) | Yes (by country, province, region) | Limited (search/filter) | Yes (region, subregion, AVA/AOC/DOCG) |
| Varietal analysis | Yes (explore by grape) | Limited (search/filter) | Yes (by variety) | Limited | Yes (grape varietal with fuzzy linking) |
| Cross-tabulation | No (single-dimension browsing) | No | Partial (descriptive only) | No | Yes (region × varietal, price × score) |
| Tasting note frequency | Limited (flavor profile tags) | No (free-text notes) | Yes (word frequency analysis) | No | Yes (frequency analysis from community notes) |
| Value analysis (QPR) | No (price shown, no ratio) | No | No (separate analysis) | No | Yes (score/price ratio) |
| Vintage histogram | No | Limited (drink-by dates) | No (vintage not in dataset) | No | Yes |
| Interactive visualization | Yes (app) | Yes (web) | Static Jupyter Notebook | Yes (web) | Interactive marimo + Plotly |
| Web UI | Yes (native app + web) | Yes | No | Yes | No (by design) |
| Recommendation engine | Yes (personalized) | No | No | No | No (anti-feature) |
| Inventory/cellar management | No | Yes | No | Yes | No (anti-feature) |
| Community aspect | User reviews + ratings | Community tasting notes | N/A | Multi-user self-hosted | r/wine *is* the community |

## Sources

- **Vivino.com/about** (official) — Feature overview of the largest wine community platform. 78M+ users, 3.5B+ scans, 20.6M+ wines. Key features: label scanning, community ratings, personalized recommendations, marketplace. Provides the benchmark for consumer wine app features. Accessed 2026-11-27.
- **GitHub: the-broke-sommeliers/wine-cellar** (59 stars, actively maintained) — Leading open-source self-hosted wine management app. Features: wine tracking, inventory management, tasting notes, barcode scanning, ratings, food pairings, multi-user. Demonstrates what self-hosted wine tools prioritize. Accessed 2026-11-27.
- **GitHub: Hamza-Tahirr/Wine-Review-EDA** (2 stars) — Exploratory data analysis of 130k WineEnthusiast reviews using pandas, Seaborn, SciPy. Analysis dimensions: points/price by country, province, variety, winery, taster; description length analysis; outlier detection; scoring categories. Demonstrates the standard analytical questions wine data analysts ask. Accessed 2026-11-27.
- **GitHub: wine-reviews topic** (15 repos) — Landscape of wine review projects on GitHub. Dominated by Kaggle dataset EDA (6 Jupyter notebooks), cellar management apps (3 Java), and one-off projects. No reusable Reddit scraping pipelines exist — this is a gap. Accessed 2026-11-27.
- **GitHub: wine-analysistopic** (5 repos) — Smaller set focused specifically on wine data analysis. Mix of educational projects (Udacity), ML (TensorFlow on physicochemical properties), and dashboard experiments (Taipy). Confirms the open-source wine analysis ecosystem is thin and fragmented. Accessed 2026-11-27.
- **r/wine community rules and post patterns** (direct observation) — Subreddit requires tasting notes with bottle pictures. Posts contain producer names, vintages, regions, prices, and scores in semi-structured prose. Confirms data quality sufficient for entity extraction, and that pre-filtering review vs. non-review posts is tractable. Accessed 2026-11-27.
- **CelarTracker (via Wikipedia and open-source clones)** — Longest-running wine cellar management platform. Features: inventory tracking, tasting notes, community reviews, drink-by windows, barcode scanning. Defines the cellar-management feature set standard. Accessed 2026-11-27.
- **Wine-Searcher.com** — Commercial wine price comparison and critic score aggregation platform. Proves demand for aggregated wine data, but focuses on retail/professional critic data rather than community reviews. Accessed 2026-11-27.

---

*Feature research for: r/wine wine review aggregation and analysis pipeline*
*Researched:* 2026-11-27