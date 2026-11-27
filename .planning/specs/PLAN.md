# Plan: r/wine Review Analysis

---

## Project Structure

```
reddit_wine/
├── pyproject.toml
├── data/
│   ├── gazetteer/
│   │   ├── regions.json        # WINE_REGION, WINE_SUBREGION, WINE_AVA, WINE_AOC, WINE_DOCG
│   │   ├── varietals.json      # GRAPE_VARIETAL
│   │   └── aliases.json        # Canonical alias table for entity linking
│   ├── raw/                    # Raw JSON from scraper
│   └── processed/              # Clean CSVs, unmapped_spans.csv
├── src/
│   ├── scrape.py               # Fetch posts + OP comments from old.reddit.com JSON
│   ├── extract.py              # Deterministic regex + spaCy EntityRuler (gazetteer)
│   ├── extract_nlp.py          # spaCy NER discovery pass
│   ├── link.py                 # Entity linking, fuzzy matching, inference rules
│   ├── normalize.py            # Score/price normalization, sentiment fallback
│   └── analyze.py              # Distributions, cross-tabs, overrepresentation (Polars)
├── scripts/
│   ├── run_sample.py           # End-to-end pipeline on ~100 posts
│   └── run_full.py             # End-to-end pipeline on 2000+ posts
└── notebooks/
    └── explore.py              # marimo notebook (exploratory — sample data, Plotly)
```

---

## Stage 0 — Scraping & Thread Assembly (`src/scrape.py`)

**Source**: `old.reddit.com/r/wine/new.json` with `?limit=100&after=t3_...`

- **Sample**: ~100 posts. **Full**: paginate to 2000+
- Per post: `title`, `selftext`, `author`, `created_utc`, `score`, `num_comments`, `permalink`, `link_flair_text`, `post_hint` (detect image posts), `id`

**OP comment fetch** — only for image posts (empty `selftext`):
- GET `old.reddit.com/r/wine/comments/{post_id}.json?limit=1`
- Match `author == OP` on the first top-level comment
- Merge into `review_text = title + "\n" + selftext + "\n" + op_comment`

Rate limit: 2 req/sec. Save raw JSON to `data/raw/`.

---

## Stage 1 — Pre-Filtering (in `src/extract.py`)

Lightweight heuristic to drop non-review posts before NLP:

- **Token gate**: `review_text` ≥ 30 tokens
- **Keyword gate**: ≥1 match from a review-indicator keyword set:
  - Sensory: *tannin, oak, acidity, finish, palate, nose, aroma, bouquet, body*
  - Structure cues: *price, paid, bought, \$|€|£, retail*
  - Score cues: *points, pts, /100, /5, score, rating*
  - Varietal/region: common grape and region names
- Failing posts are logged as skipped

**Rationale**: Filters "Is this ruined?", "What wine for a wedding?", meta posts, article links.

---

## Stage 2 — Deterministic Extraction (`src/extract.py`)

High-precision pass: regex + spaCy `EntityRuler` loaded from `data/gazetteer/`.

**Regex extractors**:

| Field | Pattern | Guardrail |
|---|---|---|
| Vintage | `\b(19[5-9]\d\|20[0-2]\d)\b` | Must be near wine terms; not "I started drinking in 2018" |
| Price | `[$€£]\s?\d+(\.\d{2})?` + `~40 bucks`, `sub-$30`, `under \$X` | Currency symbol or shorthand required |
| Explicit score | `\b([89]\d(\.\d)?\|100)\s*(?:/100\|pts?\|points?)` | 100-point scale |
| Explicit score | `\b([3-5](\.\d{1,2})?)\s*/\s*5\b` | 5-point scale |
| Vivino/CT | `Vivino`, `CellarTracker`, `[Vv]ivino \d\.\d` | External rating mention |

**spaCy EntityRuler** — loaded from gazetteer:
- Labels: `WINE_REGION`, `WINE_SUBREGION`, `WINE_AVA`, `WINE_AOC`, `WINE_DOCG`, `GRAPE_VARIETAL`
- Alias table supports: `"CDR"` → `Côtes du Rhône`, `"Napa"` → `Napa Valley`, `"Gewurz"` → `Gewürztraminer`
- **Homonym guardrails**: require capitalization + wine-context proximity; skip lowercase `port`, `sherry`, `orange`, `rose`, `washington`

---

## Stage 3 — Discovery NER (`src/extract_nlp.py`)

For sentences/posts that missed coverage in Stage 2:

- spaCy NER (`en_core_web_lg` — CPU-friendly, no GPU needed): extract `ORG`/`PERSON` → candidate producers, `GPE`/`LOC` → candidate regions
- Proximity scoring: how close is a candidate span to known wine terms in the same sentence?
- Low-confidence spans are flagged for fuzzy linking in Stage 4

---

## Stage 4 — Entity Linking & Normalization (`src/link.py`)

Map every raw extracted span to a canonical ID:

1. **Exact + alias lookup**: match against `aliases.json` (e.g., `"CDR"` → `appellation: Côtes du Rhône, region: Rhône Valley, country: France, canonical_id: cotes_du_rhone`)

2. **Fuzzy fallback**: RapidFuzz (Jaro-Winkler ≥ 0.90) against all canonical labels. Auto-link above threshold; flag below for review

3. **Domain inference rules** (`is_inferred=True`):
   ```
   IF appellation == "Barolo"  AND varietal missing  → varietal: Nebbiolo, country: Italy, region: Piedmont
   IF appellation == "Chablis" AND varietal missing  → varietal: Chardonnay, country: France, region: Burgundy
   IF appellation == "Sancerre" AND varietal missing AND color ≈ white → varietal: Sauvignon Blanc
   IF appellation == "Champagne" AND varietal missing → varietal: [Chardonnay, Pinot Noir, Pinot Meunier] (set)
   ...(extend for Bordeaux blends, Rioja, Chianti, etc.)
   ```

4. **Candidate queue**: unmapped spans appearing ≥3 times → `data/processed/unmapped_spans.csv` for periodic gazetteer updates

---

## Stage 5 — Score & Price Normalization (`src/normalize.py`)

| Output field | Logic |
|---|---|
| `raw_score` + `scale_type` | e.g., `(92, "100pt")`, `(4.2, "5pt")`, `None` for prose-only |
| `normalized_0_100` | `5pt → score × 20`, `10pt → score × 10`, `20pt → score × 5`, Vivino percentile-mapped 5pt → ~89-90 range |
| `price_usd` | Normalize `€/£/CAD/AUD` to approximate USD; strip `~` and `sub-$` |
| `sentiment_score` | **TBD** — method for sentiment analysis left open (VADER, TextBlob, or lightweight transformer under consideration) |
| `qpr_flag` | `sentiment > 0.5 AND price <= 30` → standout quality-price-ratio |

---

## Stage 6 — Analysis (`src/analyze.py`)

Uses **Polars** for all data processing. Output to `data/processed/`:

1. `region_freq.csv` — count, percentage, over/under-representation vs. expected (e.g., by global production share)
2. `price_stats.csv` — histogram bins, mean, median, q1, q3, by region
3. `rating_stats.csv` — mean/median rating by region, by price bracket, by vintage
4. `cross_tab_region_price.csv` — pivot: regions × price buckets
5. `cross_tab_region_rating.csv` — pivot: regions × rating buckets
6. `vintage_hist.csv` — vintage year distribution
7. `tasting_note_freq.csv` — top-N descriptors, filtered for wine vocabulary

---

## Stage 7 — marimo Notebook (`notebooks/explore.py`)

Exploratory visualization of sample data using **Plotly**:

- Region bar chart (top 15 by count)
- Price histogram (USD, log or trimmed at 99th percentile)
- Rating histogram + rating-by-region boxplots (top N regions)
- Scatter: price vs. rating, colored by region
- Vintage timeline (bar chart by year)
- Tasting note word frequencies (horizontal bar, top 20)
- Data quality overview: % posts with extracted scores, % with prices, % inferred vs. explicit fields

---

## Orchestrators (`scripts/`)

**`run_sample.py`** — scrape ~100 → extract → link → normalize → analyze → print summary

**`run_full.py`** — same pipeline at 2000+ scale with tqdm progress bars

---

## Dependencies (`pyproject.toml`)

```
requests, beautifulsoup4, spacy, en_core_web_lg, polars,
plotly, marimo, rapidfuzz
# sentiment method TBD — not yet committed to vaderSentiment
```

---

## 🚩 Open Question: Gazetteer

Before any code can run, we need to decide the gazetteer. Two paths:

| Path | Description | Effort |
|---|---|---|
| **A) Build from public sources** | Compile AVAs (TTB), AOCs (Wikipedia), DOCGs, major wine regions, top ~150 varietals. Build alias table by hand for common abbreviations/short forms. This is the "production" approach — high coverage but takes time to assemble. | Higher upfront |
| **B) Lightweight starter** | Begin with ~40 top regions + ~60 top varietals + common aliases. Run the sample, let the candidate queue (`unmapped_spans.csv` from Stage 4) tell us what to add. Iterate. Faster to ship but noisier on early runs. | Faster to ship |

**Questions to resolve:**

1. Path A or B? (Or a hybrid — start B, plan to graduate to A)
2. Do you have existing wine region/varietal data sources you want me to use, or should I curate from scratch?
3. How deep should the hierarchy be? e.g., `France > Burgundy > Côte de Nuits > Vosne-Romanée` vs. just `country + appellation`?
4. Should the gazetteer include non-European regions (Australia, NZ, South Africa, Chile, Argentina, US) from the start?
