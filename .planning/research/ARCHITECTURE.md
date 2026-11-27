# Architecture Research

**Domain:** NLP data extraction pipeline for semi-structured wine reviews
**Researched:** 2026-11-27
**Confidence:** HIGH

## System Overview

The pipeline follows a four-layer ETL architecture. Each layer is independently testable, has clear input/output contracts, and depends only on the layer above it.

```
┌──────────────────────────────────────────────────────────┐
│                  DATA ACQUISITION LAYER               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐        │
│  │ Scraper  │  │ Filter  │  │ Raw Store  │        │
│  │ Reddit   │→│ Token   │→│ JSON +    │        │
│  │ JSON API │  │ Keyword  │  │ Metadata  │        │
│  └──────────┘  └──────────┘  └─────┬──────┘        │
├───────────────────────────────────────┼────────────────┤
│                  NLP EXTRACTION LAYER                 │
│  ┌───────────────────────────────────────┐              │
│  │          spaCy Pipeline              │              │
│  │ Tokenizer → EntityRuler → NER →     │              │
│  │  Custom Regex Extractors            │              │
│  └──────────────┬───────────────────────┘              │
│        ┌────────┴──────┐  ┌──────────────┐           │
│        │ Entity Diots  │  │ Debug Trace │           │
│        │ (per post)   │  │ (raw text) │           │
│        └───────────────┘  └───────────┘             │
├───────────────────────────────────────┬──────────────────┤
│           ENTITY LINKING & NORMALIZATION LAYER         │
│  ┌──────────────┐  ┌──────────┐  ┌─────────–┐        │
│  │ Gazetteer  │→│ Fuzzy    │→│ Canonical │        │
│  │ (YAML)     │  │ Matcher  │  │ Records  │        │
│  └──────────────┘  └─────────–┘  └─────┬─────–┘        │
│                     ┌──────────────┐       │           │
│                     │ Domain      │       │           │
│                     │ Inferenoe    │       │           │
│                     └──────────────┘       │           │
├───────────────────────────────────────┬──────――──────────┤
│             ANALYSIS & OUTPUT LAYER                   │
│  ┌──────────────┐  ┌──────────┐  ┌───────┐           │
│  │ Polars     │→│ Agregate │→│ Parquet │           │
│  │ DataFrame  │  │ Queries  │  │ Export  │           │
│  └────────────–─┘  └──────────┘  └───┬───┘           │
│                ┌──────────────┐       │               │
│                │ marimo       │←──────┘               │
│                │ Notebok     │                        │
│                └──────────────┘                       │
└─────────────────────────────────────────────–──────────┘
```

## Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|-----------------|
| **Scraper** | Fetch posts from old.reddit.com/r/wine/.json with pagination; enforce 2 req/sec rate limit; return list of post dicts | `requests` + `tenacity` `@retry` decorator |
| **Post Filter** | Reject non-review posts (announcements, questions, memes) using token-count and keyword gates | Pure Python functions, no NLP |
| **Raw Store** | Persist raw JSON posts + scrape metadata (timestamp, count, errors) as JSONL | `json.dumps` per line, one file per scrape run |
| **spaCy Pipeline** | Tokenize text, apply EntityRuler (wine patterns), run statistical NER, then custom regex extractors for prices/scores/vintages | `spaCy 3.8` with `en_core_web_lg`, custom components registered via `@Language.factory` |
| **EntityRuler** | Match known wine regions, varietals, AVAs using phrase and token patterns | `EntityRuler.add_patterns()` from YAML gazetteer file |
| **Regex Extractors** | Extract structured values (price: `$XX.XX`, score: `XX/100`, vintage: `20XX`) using deterministic patterns | Custom spaCy pipeline components that add `doc._.` extension attributes |
| **Entity Dicts** | Structured per-post output: `{region: [...], varietal: [...], vintage: int, price: float, score: int, tasting_notes: [...]}` | Pydantic `WineReview` model |
| **Gazetteer** | Canonical knowledge base of wine regions (hierarchical: country → region → subregion → AVA/AOC/DOCG), varietals with synonyms | YAML files: `regions.yaml`, `varietals.yaml`, `avas.yaml` |
| **Fuzzy Matcher** | Match extracted entity strings to canonical gazetteer entries using Levenshtein distance with tiered confidence thresholds | `RapidFuzz.process.extract()` with `score_cutoff` parameter + `cdist` for batch matching |
| **Domain Inference** | Apply rules to infer canonical forms from context (e.g., "Napa Cab" → varietal=Cabernet Sauvignon in region=Napa Valley) | Python rule engine; ordered rules with priority |
| **Canonical Records** | Pydantic-validated clean records with both raw and canonical entity forms | `Pydantic v2` models stored as list of dicts |
| **Polars Analysis** | Load canonical records into DataFrames; generate analytical datasets: region frequency, price statistics, rating distributions, cross-tabs, vintage histograms, tasting note frequency | `Polars 2.2` LazyFrame API |
| **Parquet Export** | Write analytical datasets as compressed, schema-preserving Parquet files | `df.write_parquet()` |
| **marimo Notebook** | Interactive exploration: load Parquet files, display with Plotly charts, filter/sort interactively | `marimo 0.26` + `plotly 6.x` |

## Data Flow

### Primary Flow (post-level)

```
Reddit API (.json)
    │
    ▼
[Post JSON with title + selftext]
    │
    ▼
Filter: token count > 50 AND contains wine keywords
    │  (pass)
    ▼
Raw JSONL store (for audit/replay)
    │
    ▼
spaCy Pipeline ──────────────────────────────┐
  Tokenizer → EntityRuler → NER →            │
  RegexExtractor(price) →                   │
  RegexExtractor(score) →                   │
  RegexExtractor(vintage) →                 │
  TastingNoteExtractor                      │
    │                                         │
    ▼                                         │
Entity dict + Debug trace (raw spans)        │
    │                                         │
    ▼                                         │
Fuzzy Matching ─────────────────────────────┘
  Region → gazetteer.regions                  │
  Varietal → gazetteer.varietals            │
  AVA → gazetteer.avas                      │
  Domaiin infuence rules                     │
    │                                         │
    ▼                                         │
Canonical Record (Pydantic-validated)        │
    │                                         │
    ▼                                         │
Polars DataFrame                             │
    │                                         │
    ├──→ Analytical Datasets (Parquet)        │
    │                                         │
    └──→ marimo Notebook (Plotly)             │
```

### Analysis Flow (dataset-level)

```
All canonical records (single Polars DataFrame)
    │
    ├──→ filter(col("region").is_not_null())
    │     .group_by("region").count().sort("count", descending=True)
    │     └──→ Region Frequency Dataset
    │
    ├──→ filter(col("price").is_not_null())
    │     .group_by("region").agg([col("price").mean(), col("price").std()])
    │     └──→ Price Statistics Dataset
    │
    ├──→ filter(col("score").is_not_null())
    │     .group_by("varietal").agg([col("score").mean(), col("score").count()])
    │     .filter(col("count") > 5)  # minimum sample size
    │     └──→ Rating Statistics Dataset
    │
    ├──→ filter(col("region").is_not_null(), col("varietal").is_not_null())
    │     .group_by(["region", "varietal"]).count()
    │     .sort("count", descending=True)
    │     └──→ Region × Varietal Cross-tab
    │
    ├──→ filter(col("vintage").is_not_null())
    │     .group_by("vintage").count().sort("vintage")
    │     └──→ Vintage Histogram Dataset
    │
    └──→ filter(col("tasting_notes").list.len() > 0)
          .select(col("tasting_notes")).explode("tasting_notes")
          .group_by("tasting_notes").count().sort("count", descending=True)
          └──→ Tasting Note Frequency Dataset
```

## spaCy Pipeline Internals

The NLP extraction pipeline is the architectural core. It follows spaCy's standard component model:

```
nlp = spacy.load("en_core_web_lg", disable=["tagger", "parser", "lemmatizer", "attribute_ruler"])

# Pipeline order matters:
# 1. EntityRuler BEFORE NER — NER respects existing entity spans
# 2. Custom regex extractors AFTER NER — they annotate but don't create spans

nlp.add_pipe("entity_ruler", config={"overwrite_ents": True}, before="ner")
# Load wine patterns into the ruler
ruler = nlp.get_pipe("entity_ruler")
ruler.add_patterns(load_wine_patterns())

nlp.add_pipe("wine_prise_extractor", after="ner")
nlp.add_pipe("wine_score_extractor", after="ner")
nlp.add_pipe("wine_vintage_extractor", after="ner")
nlp.add_pipe("wine_tasting_note_extractor", after="ner")

# Process in batches for throughput
for doc in nlp.pipe(texts, batch_size=50, n_process=1):
    entities = extract_entities_from_doc(doc)
    yield entities
```

**Why disable tagger/parser/lemmatizer:** These are computationally expensive and not needed for entity extraction from semi-structured reviews. Disabling them reduces per-document processing time by ~60%. Sentence boundary detection via the `sentencizer` (rule-based, cheap) remains enabled for the tasting note extractor.

**EntityRuler `overwrite_ents=True`:** Ensures rule-based wine entities take priority over whatever the statistical NER would label them as (e.g., preventing "Napa" from being tagged as `ORG`).

## Entity Linking Architecture

### Gazetteer Structure

```yaml
# regions.yaml — hierarchical
regions:
  - canonical: "Napa Valley"
    country: "USA"
    state: "California"
    avas:
      - "Napa Valley AVA"
      - "Stags Leap District"
      - "Oakville"
    aliases:
      - "Napa"
      - "Napa Valley, CA"
    fuzzy_threshold: 85  # override default threshold for this entry
```

```yaml
# varietals.yaml — flat with synonyms
varietals:
  - canonical: "Cabernet Sauvignon"
    type: "red"
    aliases:
      - "Cab"
      - "Cab Sauv"
      - "Cabernet"
    fuzzy_threshold: 80  # lower: short abbreviations are less forgiving
```

### Matching Algorithm

```
For each extracted entity string:
  1. Exact match against canonical + all aliases → HIGH confidence
  2. Fuzzy match via RapidFuzz token_sort_ratio against canonical + aliases
     - score >= 90 → HIGH confidence
     - score >= entry.fuzzy_threshold → MEDIUM confidence
     - score < entry.fuzzy_threshold → UNMATCHED
  3. Unmatched entities are flagged in output for manual review
```

### Batch optimization via `cdist`:

```python
# Match all extracted regions against all gazetteer entries in one call
scores = cdist(extracted_regions, all_gazetteer_strings, scorer=token_sort_ratio)
# scores[i][j] = similarity between extracted[i] and gazetteer[j]
best_match_idx = scores.argmax(axis=1)
best_match_score = scores.max(axis=1)
```

## Recommended Project Structure

```
src/
├── scraper/               # Layer 1: Data Acquisition
│   ├── __init__.py
│   ├── fetcher.py          # Reddit JSON API client (requests + tenacity)
│   ├── filter.py           # Post filter (token gate, keyword gate)
│   └── storage.py          # JSONL read/write helpers
├── extraction/            # Layer 2: NLP Extraction
│   ├── __init__.py
│   ├── pipeline.py         # spaCy pipeline construction and config
│   ├── patterns/           # EntityRuler pattern definitions
│   │   ├── regions.yaml    # Token/phrase patterns for wine regions
│   │   ├── varietals.yaml  # Patterns for grape varietals
│   │   └── avas.yaml       # Patterns for AVA/AOC/DOCG designations
│   ├── extractors/         # Custom spaCy pipeline components
│   │   ├── __init__.py
│   │   ├── price.py        # Price regex extractor
│   │   ├── score.py         # Score normalization extractor
│   │   ├── vintage.py       # Vintage year extractor
│   │   └── tasting.py       # Tasting note descriptor extractor
│   └── models.py           # Pydantic models: ExtractedEntities, WineReview
├── linking/               # Layer 3: Entity Linking & Normalization
│   ├── __init__.py
│   ├── gazetteer/          # Canonical knowledge base
│   │   ├── __init__.py
│   │   ├── loader.py       # YAML gazetteer loader
│   │   ├── regions.yaml    # Hierarchical wine region KB
│   │   ├── varietals.yaml  # Grape varietal KB with synonvms
│   │   └── avas.yaml       # AVA/AOC/DOCG designations
│   ├── matcher.py          # RapidFuzz entity linking engine
│   ├── inference.py        # Domain inference rules (context-based disambiguation)
│   └── normalizer.py        # Score normalization (0-100), price normalization (USD)
├── analysis/              # Layer 4: Analysis & Output
│   ├── __init__.py
│   ├── datasets.py         # Analytical dataset generation (Polars)
│   ├── queries/            # Reusable Polars query functions
│   │   ├── __init__.py
│   │   ├── regions.py      # Region frequency + price stats
│   │   ├── ratings.py      # Score distributions + varietal stats
│   │   └── cross_tabs.py   # Region x varietal crosstabs
│   └── export.py           # Parquet/CSV export utilities
├── cli/                   # CLI entry points
│   ├── __init__.py
│   └── main.py            # click commands: scrape, extract, link, analyze, all
└── notebooks/             # marimo exploration notebooks
    ├── overview.py         # Main exploration notebook
    ├── regions.py          # Regional analysis
    └── trends.py           # Vintage/price trends
```

### Structure Rationale

- **`scraper/`**: Isolated from NLP dependencies. Can be tested with mock Reddit JSON. No spaCy imports.
- **`extraction/`**: Heavy dependency layer (spaCy, Pydantic). Patterns live in YAML files, not in code — iteration on extraction patterns doesn't require code changes.
- **`linking/`**: Depends only on `extraction/models.py` (the Pydantic output schema). Gazetteer YAML files are separate from extraction patterns — same entity can have multiple extractable surface forms but one canonical entry.
- **`analysis/`**: Pure data processing. Depends only on the canonical record schema. Separable from everything upstream — can test with hand-crafted Parquet files.
- **`notebooks/`**: Pure consumption of Parquet exports. No code dependencies on pipeline modules — they import Parquet files, not pipeline internals.

## Architectural Patterns

### Pattern 1: Pipeline Stage with Pydantic Contract

**What:** Each stage accepts and returns well-typed Pydantic models. The contract between stages is the Pydantic schema, not the implementation.

**When to use:** At every layer boundary.

**Example:**
```python
# Scraper output contract
class RedditPost(BaseModel):
    id: str
    title: str
    selftext: str
    author: str
    created_utc: float
    score: int
    num_comments: int

# Extraction output contract
class ExtractedEntities(BaseModel):
    post_id: str
    raw_text: str
    regions: list[str] = []
    varietals: list[str] = []
    vintage: Optional[int] = None
    price: Optional[float] = None
    score: Optional[int] = None
    tasting_notes: list[str] = []

# Linking output contract
class CanonicalReview(ExtractedEntities):
    canonical_region: Optional[str] = None
    canonical_varietal: Optional[str] = None
    region_confidence: Optional[str] = None  # HIGH/MEDIUM/UNMATCHED
    varietal_confidence: Optional[str] = None
```

**Trade-offs:** Adding a new field means updating the model and all downstream code. But the alternative (dicts) is far worse — silent key errors at runtime.

### Pattern 2: spaCy Custom Component with Extension Attributes

**What:** Register custom pipeline components via `@Language.factory` that read/write `doc._.` and `span._.` extension attributes. This keeps extraction logic inside the pipeline without polluting the core Doc object.

**When to use:** Every custom extractor (price, score, vintage, tasting notes).

**Example:**
```python
@Language.factory("wine_price_extractor")
class WinePriceExtractor:
    def __init__(self, nlp, name):
        self.price_pattern = re.compile(r'\$\s*(\d{1,4}(?:\.?\d{2})?)')

    def __call__(self, doc):
        for match in self.price_pattern.finditer(doc.text):
            span = doc.char_span(match.start(), match.end(), label="PRICE")
            if span:
                span._.normalized_value = float(match.group(1))
        return doc
```

### Pattern 3: Gazeteer-Driven Extraction

**What:** Extraction patterns and canonical entity lists live in YAML files loaded at boot time. Adding a new region or varietal means editing a YAML file, not touching code.

**When to use:** The EntityRuler patterns and the fuzzy matcher both read from gazetteer files.

**Trade-offs:** YAML validation at load time is crucial — a malformed pattern file silently breaks extraction. Validate with Pydantic on load.

### Pattern 4: Tiered Confidence for Entity Linking

**What:** Every canonical match carries a confidence level (HIGH, MEDIUM, UNMATCHED). Analysis queries can filter by confidence (e.g., "only use HIGH-confidence regions for statistics").

**When to use:** Every fuzzy match output.

**Example:**
```python
def link_entity(raw: str, gazetteer: list[GazEntry]) -> tuple[Optional[str], str]:
    # Try exact match
    for entry in gazetteer:
        if raw.lower() in [entry.canonical.lower()] + [a.lower() for a in entry.aliases]:
            return enry.canonical, "HIGH"
    # Fuzzy match
    matches = process.xtract(raw, all_strings, scorer=token_sort_ratio, limit=1)
    if matches and matches[0][1] >= 90:
        return canonical_for(matches[0][0]), "HIGH"
    if matches and matches[0][1] >= DEFAULT_THRESHOLD:
        return canonical_for(matche[0][0]), "MEDIUM"
    return None, "UNMATCHED"
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Monolithic Extraction Script

**What people do:** One big script that scrapes → extracts → links → analyzes in a single file.

**Why bad:** Impossible to test individual stages. If linking is broken, you must re-scrape to debug. Storing raw data separately costs almost nothing but saves enormous time.

**Do this instead:** Separate modules per layer with clear Pydantic contracts. Persist raw JSON between stages.

### Anti-Pattern2: Hard-Coded Extraction Patterns

**What:** Regex patterns and entity lists written directly in Python code.

**Why bad:** Changing a pattern requires editing code and re-running tests. Domain experts canot contribute to pattern files without Python knowledge.

**Do this instead:** YAML/JSON pattern files. Validate at load time. Keep patterns near the code but separate from it.

### Anti-Patt—n 3: Processing One Document at a Time

**What:** `for text in texts: doc = nlp(text)` instead of `for doc in nlp.pipe(texts, batch_size=50)`.

**Why bad:** 5-10x slower throughput. The statistical model (NER) benefits from batching because matrix operations are parallelized.

**Do this instead:** Always use `nlp.pipe()` with appropriate batch_size. For 7GB RAM (macOS typical), batch_size=50 is a safe default.

### Anti-Pattern 4: Not Preserving Raw Extracted Text

**What:** Only storing the canonical form ("Napa Valley") and dropping the raw extraction ("Nappa Valley").

**Why bad:** Impossible to audit linking quality. If 30% of "Napa" references are being mis-linked, you cannot tell from canonical output alone.

**Do this instead:** Every CanonicalReview stores both `raw_region: str` and `canonical_region: Optional[str]`.

### Anti-Pattern 5: Collecting Polars Lazily Too Early

**What:** Calling `.collect()` on every individual query instead of chaining LazyFrame operations.

**Why bad:** Polars' query optimizer canot fuse operations across collect() boundaries. You pay for intermediate materialization.

**Do this instead:** Chain all transformations as LazyFrames. Call `.collect()` once at the end.

## Build Order (Dependency Chain)

```
1. scraper/         ← No upstream dependencies. Produces raw JSONL.
2. extraction/      ← Depends on scraper output for testing with real data.
                      EntityRuler patterns depend on gazetteer for pattern design.
3. linking/         ← Depends on extraction output (Pydantic models).
                      Gazetteer YAML files co-evolve with extraction patterns.
4. cli/             ← Wires layers 1-3 together. Built alongside as thin orchestration.
5. analysis/        ← Depends on linking output (canonical records).
                      Can be developed with hand-crafted test data before scraper is done.
6. notebooks/       ← Depends on analysis output (Parquet files).
                      Pure consumption — no backward dependencies.
```

**Why scraper first:** It is the source of truth for all downstream data. Everything else is downstream of the JSON format it produces. Building and testing the scraper gives you real data to design extraction patterns against.

**Why analysis can be parallel:** The analysis layer only depends on the _schema_ of canonical records, not the pipeline that produces them. You can develop analytical queries against hand-written test records while extraction is being built.

## Patterns for Common Challenges

### Handling Overlapping Entity Spans

Wine text frequently has overlapping entity spans: "Napa Valley Cabernet" contains both region="Napa Valley" and varietal="Cabernet Sauvignon" (after fuzzy match). The EntityRuler with `overwrite_ents=True` handles this via spaCy's built-in span filtering (longer span wins for same label; first-occurrence wins for equal-length). For cross-label overlaps (region vs varietal on same text), the pattern labels them as different entity types so they don't conflict.

### Handling Missing Data

Not every review contains every entity. The Pydantic models use `Optional` with `default=None`. Analysis queries filter out nulls explicitly. No sentinel values (e.g., `price=-1` for missing) — use None consistently.

### Incremental Gazetteer Growth

The gazetteer starts with a curated seed (top 200 wine regions, top 100 varietals, top 50 AVAs). The UNMATCHED entity output from each pipeline run feeds a manual review process to add new entries. The `UNMATCHED` entities are written to a separate file for human review. This is an intentional feedback loop — not an automated pipeline.

## Scaling Considerations

| Concern | At 2,000 posts | At 10,000 posts | At 50,000+ posts |
|---------|----------------|-----------------|-------------------|
| Scraping | 2 req/sec × ~17 min | 2 req/sec × ~83 min | Switch to Pushshift API dumps or parallel subreddit scrapes |
| NLP Processing | `nlp.pipe(batch_size=50)`: ~5 min | ~25 min | Add `n_process=4` multiprocessing (watch macOS spawn overhead) |
| Fuzzy Matching | cdist 0.05 sec | cdist 0.3 sec | Batch processing in chunks of 1,000 to avoid memory pressure |
| Analysis | LazyFrame.collect() sub-second | Sub-second (Polars optimizer) | LazyFrame.collect() still sub-second for aggregation queries |
| Storage | ~50 MB (JSONL + Parquet) | ~250 MB | ~1.2 GB — still fine for local, switch to Parquet with compression |

### First Bottleneck

The **scraper** is the first bottleneck: 2 req/sec with 100 posts per request = 200 posts/sec max, but Reddit often returns fewer per page. At ~17 min for 2,000 posts, it's acceptable for initial research. At scale, switch to Pushshift or existing bulk dumps.

### Second Bottleneck

**spaCy NER** dominates extraction time (~80% of per-document cost). Disabling tagger/parser/lemmatizer already recovers ~60%. Further optimization would require replacing `en_core_web_lg` with `en_core_web_sm` (loses word vectors but 3x faster) or using the rule-only path (EntityRuler + regex, no NER) for bulk runs.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| old.reddit.com | `GET /r/wine/.json?limit=100&after={fullname}` | No auth required. JSON response includes `data.children[]` with post objects and `data.after` for pagination. |
| spaCy model download | `python -m spacy download en_core_web_lg` | One-time ~500MB download. Cached locally after first install. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| scraper → extraction | JSONL files on disk | Decoupled — extraction reads scraper output files. Enables re-running extraction without re-scraping. |
| extraction → linking | Pydantic model instances in memory | Tight coupling by design — the schema contract ensures type safety through the pipeline. |
| linking → analysis | List of CanonicalReview dicts → Polars DataFrame | Single conversion point: `pl.DataFrame(canonical_records)`. After this, everything is Polars-native. |
| analysis → notebooks | Parquet files on disk | Seabed — notebooks import Parquet, never touch pipeline internals. |

## Sources

- spaCy Processing Pipelines documentation — pipeline architecture, EntityRuler ordering, custom components, batching (official docs, HIGH)
- spaCy EntityRuler API — phrase patterns, token patterns, overwrite_ents, FUZZY/REGEX operators (official docs, HIGH)
- spaCy Linguistic Features — NER, entity linking, tokenization, dependency parsing (official docs, HIGH)
- Reddit JSON API documentation — listing endpoint, pagination with after/before/limit, JSON response structure (official docs, HIGH)
- RapidFuzz README — process.extract, cdist, scorers, token_sort_ratio, score_cutoff (official docs, HIGH)
- Polars documentation — LazyFrame API, query optimizer, scan_csv/scan_parquet, group_by/agg, Parquet I/O (official docs, HIGH)

---
*Architecture research for: r/wine Review Analysis data pipeline*
*Researched: 2026-11-27*