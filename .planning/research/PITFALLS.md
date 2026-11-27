# Pitfalls Research

**Domain:** Wine review NLP extraction and aggregation pipeline
**Researched:** 2026-11-27
**Confidence:** HIGH

## Critical Pitfalls

These are pitfalls that cause rewrites, silently corrupt data, or fundamentally break the pipeline.

### Pitfall 1: Silent Non-JSON Responses from old.reddit.com

**What goes wrong:**
The pipeline requests `https://old.reddit.com/r/wine/.json` expecting structured JSON, but Reddit sometimes returns HTML instead — a login page, a rate-limit interstitial, a "blocked" page, or a redirect. The `requests` library won't raise an error for a 200 OK with HTML body, and `response.json()` will raise a cryptic `JSONDecodeError` that looks like corrupted data, not a Reddit policy block. The pipeline silently fails or stores garbage.

**Why it happens:**
Reddit aggressively serves non-JSON responses via old.reddit.com in several scenarios: geolocation-based blocking (EU visitors get GDPR consent pages), VPN/proxy detection (returns "you've been blocked" HTML), and aggressive rate limiting beyond simple 429s (they serve HTML error pages at 200 status). The `.json` suffix is an unofficial API — it can break at any time with no announcement. Reddit's 2023 API pricing changes and 2024 API tightening have made the old interface less reliable. Additionally, deleted posts return `[removed]` text in `selftext` — the post object exists in the listing, but the content is gone, producing silent extraction failures rather than clear "no data" signals.

**How to avoid:**
1. Always validate `Content-Type` header before parsing — reject any response that isn't `application/json`
2. Wrap `response.json()` in try/except with specific detection for HTML bodies (look for `<html`, `<!DOCTYPE`, `<body` in response text)
3. For deleted/removed posts, check `selftext == "[removed]"` or `selftext == "[deleted]"` and skip them with a logged reason
4. For non-JSON responses, log the full URL, status code, headers, and first 500 chars of the body for debugging
5. Add a circuit breaker: if >5 consecutive non-JSON responses occur, pause scraping and alert
6. Test with real requests against the actual subreddit before committing to code; cache sample responses for offline testing
7. Use `raw_json=1` query parameter to opt out of Reddit's HTML entity encoding in JSON responses (prevents `&amp;` instead of `&`)

**Warning signs:**
- `JSONDecodeError` in scraped data without a clear rate-limit error first
- 200 OK responses that fail to parse
- Intermittent failures that correlate with time of day (peak Reddit traffic = more aggressive blocking)
- `selftext` content that is exactly `"[removed]"` — this means the post was once real but is now gone; you can scrape it but extraction will produce zero entities

**Phase to address:** Scraping phase — before any extraction code runs

---

### Pitfall 2: Non-Review Posts Contaminating the Dataset

**What goes wrong:**
r/wine contains a mix of review posts ("Just popped a 2019 Barbaresco..."), recommendation requests ("What should I pair with steak?"), bottle porn ("Look at this Chateau Lafite I found!"), questions ("Is this corked?"), news articles, and memes. A naive keyword gate (e.g., "must contain a year") lets through false positives (news articles mentioning vintages) and filters out false negatives (reviews without explicit vintages like "Had a great Beaujolais Nouveau last night"). Extraction then runs on non-review text, producing spurious entities — "casual Friday" gets "Friday" tagged as a wine, "steak dinner" gets analyzed.

**Why it happens:**
The review/non-review boundary is a spectrum, not a binary classification. Reviews vary enormously in structure: some are structured tasting notes with explicit scores/prices, others are narrative stories with embedded details. Token-gate approaches (must contain N wine-related terms) are too brittle. The project's stated approach of "token gate + keyword gate" pre-filtering requires careful calibration that's easy to get wrong.

**How to avoid:**
1. Instead of a binary gate, implement a **review-quality scoring pass**: posts score higher for having multiple signals (vintage mention, varietal mention, tasting descriptors, price mention, score mention). Posts above a threshold are "reviews." This is more robust than any single gate.
2. Build a rejection list of non-review subreddit flairs (e.g., "Discussion", "News", "Meme") — filter on `link_flair_text` in the JSON response
3. For posts that pass the gate, still include a `review_confidence` field in output (0-1) so analysis can filter post-hoc
4. Manually review the first 200 scraped posts to calibrate the filter before processing all 2000
5. Log all filtered-out posts with their reason codes so you can audit for false negatives
6. Consider a minimum character count gate — r/wine review posts tend to be longer (100+ words for a proper tasting note) while questions and memes are short

**Warning signs:**
- Extracted entities include obviously non-wine terms appearing in dataset distributions
- Many posts with 0-1 extracted entities (indicating the post wasn't a review to begin with, but made it through the filter)
- Price extraction finds "$0" or absurd values (signs of non-review text being parsed)
- `review_confidence` scores cluster near 0 for >20% of extracted posts

**Phase to address:** Pre-filtering phase — runs between scraping and entity extraction

---

### Pitfall 3: Synonym Fragmentation — Same Entity, Different Names

**What goes wrong:**
The pipeline extracts "Syrah", "Shiraz", and "Petite Sirah" as three different entities. "Pinot Grigio" and "Pinot Gris" appear as separate varietals. "Tempranillo", "Tinto Fino", "Cencibel", and "Aragonez" are all the same grape but would appear as four distinct entries, splitting counts four ways. "Bordeaux", "Claret", and "Bordeaux Blend" fragment region statistics. The analytical output becomes misleading: Syrah appears to be much less popular than it actually is because half the mentions are under "Shiraz."

**Why it happens:**
Wine naming is famously inconsistent. Grapes have different names in different countries (Syrah/Shiraz), regions have historical aliases (Claret = Bordeaux), and some names are both a region and a style (Champagne, Port, Sherry). The EntityRuler matches exact patterns; without a synonym resolution layer, every spelling variant becomes a separate entity. The gazetteer may list canonical forms, but the extracted text uses whichever form the reviewer prefers.

**How to avoid:**
1. Build the gazetteer as a **synonym map**, not a flat list: each entry has a canonical name and a list of known aliases. All extracted entities resolve to the canonical name.
2. For the most common wine grapes, pre-populate alias groups from authoritative sources (Wine Grapes by Jancis Robinson, OIV grape variety database)
3. Include common misspellings in the alias list (e.g., "Cabarnet" → "Cabernet Sauvignon", "Chardonay" → "Chardonnay") — these appear surprisingly often in Reddit posts
4. Use fuzzy matching **after** exact matching as a fallback for unknown terms
5. Log every canonical resolution so you can audit for incorrect mappings
6. Maintain an "unresolved terms" output file for periodic human review of new terms that didn't match any known entity

**Warning signs:**
- Two or more entries in output dataset that are known synonyms (e.g., Syrah and Shiraz both appearing as top varietals)
- Entity frequency lists have suspicious gaps — a famous wine region has unusually low counts while an unknown alias has unexpectedly high counts
- Identical tasting note language clustered under different entity labels

**Phase to address:** Entity linking phase — the synonym map is the core of the linking step

---

### Pitfall 4: Region Hierarchy Collapse

**What goes wrong:**
When someone writes "2019 Chateau Margaux from Margaux, Bordeaux," the extractor captures "Bordeaux" but loses "Margaux" and "Pauillac." Or it captures "Pauillac" but drops the "Bordeaux" context, so the analyst can't aggregate subregions into their parent regions. The dataset shows 50 mentions of "Margaux," 30 of "Pauillac," and 200 of "Bordeaux" — but no way to know that Margaux and Pauillac are subregions of Bordeaux, so Bordeaux's true total (280) can't be computed.

**Why it happens:**
Entity extraction typically treats each entity as independent. But wine regions form a strict hierarchy (Country > Region > Subregion > Appellation > Vineyard). Napa Valley > Oakville > To Kalon Vineyard. The spaCy EntityRuler doesn't understand hierarchy — it just labels spans. Without a post-processing step that nests entities, the hierarchy is lost. Worse, a reviewer may only mention the subregion ("this Oakville Cab is fantastic"), expecting the reader to know that Oakville is in Napa. The region-level analysis then undercounts Napa.

**How to avoid:**
1. Build the gazetteer as a **tree structure**, not a flat list. Each entity knows its parent. "Pauillac" has parent "Médoc" which has parent "Bordeaux" which has parent "France."
2. When extracting entities from a post, resolve each match to its canonical node in the tree
3. For analytical aggregation, always include **all ancestor labels** in the output. A post mentioning "Pauillac" should increment counts for Pauillac, Médoc, Bordeaux, and France
4. When a post mentions only a subregion without the parent, propagate upward for aggregate stats but flag it as "inferred parent" so the analyst can filter or adjust
5. Handle the ambiguous case: "Bordeaux" mentioned without a specific subregion should only count Bordeaux at the region level, not be counted as a separate subregion
6. For the marimo visualization, build region drill-downs (click Bordeaux → see Médoc, Graves, etc.) to make the hierarchy explorable

**Warning signs:**
- Subregion counts that don't sum to anywhere near the parent region count
- Same post contributing to multiple levels incorrectly (double-counting)
- "Napa Valley" and "Napa" appearing as separate entities (synonym + hierarchy problem combined)

**Phase to address:** Entity linking phase — the gazetteer structure determines whether hierarchy works or not

---

### Pitfall 5: Fuzzy Matching Threshold Tuning — the False Positive Cliff

**What goes wrong:**
The RapidFuzz threshold is set too low (e.g., 70) to catch misspellings, and the pipeline starts linking unrelated terms. "Merlot" fuzzy-matches "Merlow" correctly but also "Marlboro" (completely unrelated). Short wine terms are especially dangerous: "Pinot" (5 chars) matches "Pinto" at 80% similarity, "Rum" matches "Rum" (right, but also matches "Run"), "Cava" matches "Java." The error rate grows exponentially as the string gets shorter. With a threshold set too high (e.g., 95), legitimate misspellings like "Chardonnay" → "Chardonnay" are missed, reducing recall.

**Why it happens:**
Fuzzy string matching doesn't understand semantics — it only sees characters. Wine terms span a huge length range: "Vin de Pays d'Oc" (17 chars) vs. "Rioja" (5 chars) vs. "Napa" (4 chars). A single threshold cannot serve all lengths well. The project spec says "fuzzy matching with domain inference rules," but if the rules aren't length-aware and frequency-aware, the tuning becomes a trial-and-error nightmare.

**How to avoid:**
1. Use a **length-adaptive threshold**: shorter strings need higher thresholds. 4-char terms → 95-100% threshold. 5-7 char terms → 90-95%. 8+ char terms → 85-90%. These are starting points to calibrate with real data.
2. **Never fuzzy-match against the full gazetteer** — first, exact-match against the gazetteer. Only fuzzy-match extracted terms that didn't get an exact hit.
3. Pre-filter fuzzy candidates: only consider gazetteer entries whose length is within ±30% of the extracted term. This eliminates "Napa" matching "Napa Valley" (which should be a hierarchical containment check, not fuzzy matching).
4. Token ratio is more reliable than partial ratio for wine: `rapidfuzz.fuzz.token_sort_ratio` handles word reordering in multi-word terms ("Valley Napa" matches "Napa Valley").
5. Log every fuzzy match decision (extracted term → matched canonical, score, threshold used) so you can audit false positives/negatives
6. Build a test suite of 50+ known difficult examples: misspellings that should match, confusables that shouldn't, borderline cases

**Warning signs:**
- Entity frequencies that seem "off" — a rare varietal suddenly appears in the top 10 (false positive match against a common varietal)
- Log reviews showing fuzzy matches with scores right at the threshold boundary
- Fuzzy-matched entities that have very few occurrences (1-2) — likely noise

**Phase to address:** Entity linking phase — threshold configuration and test suite must be built before processing all 2000 posts

---

### Pitfall 6: Score Scale Ambiguity

**What goes wrong:**
A reviewer writes "Score: 90" — the pipeline normalizes to 90/100 (correct). Another writes "Score: 17" — the pipeline also normalizes to 17/100 (wrong — this is a 17/20, which is 85/100). A third writes "Score: 4/5" — the pipeline can't handle fractional notation and records score=4, losing the scale entirely. A fourth writes "Best QPR wine I've had this year — 92 points" — the score is extracted but the qualitative context ("best QPR" meaning quality-price ratio) is lost. A reviewer gives "90 pts" for a wine that costs $15 (great value) and another 90 for a $200 wine (expected quality) — the scores are the same but the implications are different.

**Why it happens:**
The wine world uses multiple scoring systems, and reviewers on Reddit rarely state which scale they're using. Robert Parker's 100-point scale (actually 50-100 in practice, since no wine scores below 50) dominates, but European reviewers may use a 20-point scale. Some use 5-star ratings. Some just say "A-" or "Excellent." The project spec says "normalize scores to 0-100 scale," but normalization is impossible without detecting the original scale first.

**How to avoid:**
1. **Scale detection heuristics** (apply in order):
   - If score is a fraction like "17/20" or "4/5", parse the scale explicitly
   - If score > 50, assume 100-point scale (wine scores below 50 are essentially nonexistent)
   - If score ≤ 20, check for scale context clues: "out of 20", "Jancis Robinson" mention (she uses 20-point), European reviewer signals
   - If score ≤ 5, assume star rating (5-star) or 5-point scale
   - Letter grades (A, B, C, etc.) map to ranges: A+ ≈ 96-100, A ≈ 93-95, A- ≈ 90-92, etc.
2. **Extract the scoring phrase**, not just the number. The surrounding context ("17 out of 20", "17/20", "17 points") carries scale signal that the number alone doesn't.
3. Normalize all scores to 0-100 and **store the original scale** as a column in the dataset so the analyst can filter by scale reliability
4. For scores where scale is ambiguous (e.g., "15" could be 15/20 or a very low 15/100), flag with `score_confidence: low` and exclude from means/aggregates
5. Handle range scores: "90-92 pts" should extract both an average (91) and a range. Store as structured: `score: 91, score_min: 90, score_max: 92`
6. For scores embedded in text like "I'd give it a solid 89-90," use regex that captures both the number and surrounding qualifiers

**Warning signs:**
- Score distribution has a suspicious bimodal pattern (e.g., cluster at ~85 and another at ~17 — the 20-point scale scores weren't normalized)
- Mean score seems implausibly low (e.g., 65/100 for wines from a famous region) — signs of scale errors dragging down the mean
- Individual posts with extreme outlier scores (e.g., a Lafite at 18/100 instead of 18/20 → 90/100)

**Phase to address:** Entity extraction phase (score parsing) + normalization phase (scale detection and conversion). These should be sequential — parse raw scores first, then normalize in a separate pass.

---

### Pitfall 7: Price Format Proliferation

**What goes wrong:**
Reddit reviewers write prices in dozens of formats: "$25", "25 USD", "~$30", "$25-30", "under $20", "about fifteen bucks", "€15", "£12", "CAD 45", "$65 (on sale from $80)", "$25/bottle", "bought a case for $240" (which is $20/bottle, not $240/bottle). The pipeline extracts "$240" for a case purchase and the analysis shows someone paying $240 for a $20 wine — or worse, the case price skews the mean upward. You want visible flags for multi-bottle prices and non-USD prices so the analyst can exclude or convert them.

**Why it happens:**
Price extraction is deceptively complex because human price communication is informal and context-dependent. Regex alone can't distinguish between a per-bottle price and a total cost. The project spec says "normalize prices to USD" but doesn't address multi-bottle prices (case purchases, "2 for $30") or non-USD currencies that are common in r/wine (EUR, GBP, CAD, AUD).

**How to avoid:**
1. **Multi-pass price extraction**:
   - Pass 1: Extract all monetary amounts with their surrounding context (20 chars before/after)
   - Pass 2: Detect modifiers: "per bottle", "/bottle", "each", "a bottle" → single-unit price
   - Pass 3: Detect multi-bottle signals: "case", "6-pack", "3 for", "two bottles", "dozen" → divide total by quantity
   - Pass 4: Detect currency: "$" → USD, "€" → EUR, "£" → GBP, "CAD" → CAD, "AUD" → AUD
   - Pass 5: Convert all to USD using approximate exchange rates (hardcode a lookup table — no live FX API needed for research)
2. For text prices ("fifteen bucks"), use a number-word → numeric conversion (the `word2number` library or a small custom mapping for 1-200)
3. Handle price ranges: "between $25 and $35" → both bounds stored; "around $30" or "~$30" → single price with `price_precision: approximate` flag
4. Store the **original price string** and extracted **normalized_usd** as separate columns so the analyst can audit conversions
5. For non-dollar currencies, add `original_currency` and `original_amount` columns — don't discard the raw data
6. Log every price extraction decision at debug level so you can spot parsing errors in review

**Warning signs:**
- Price distribution has a long tail of extreme values ($500+ for wines that should be $20-50) — likely multi-bottle prices not divided
- Mean price for a region is 2-3x the expected market price — signs of currency or multi-bottle errors
- Many posts with price context words ("case", "bottle") but no price extracted — the regex is too restrictive

**Phase to address:** Entity extraction phase (price parsing) + normalization phase (currency conversion, multi-bottle math)

---

### Pitfall 8: EntityRuler Pattern Ordering and Overlap Resolution

**What goes wrong:**
Two EntityRuler patterns match overlapping spans: "Napa Valley" (region) vs. "Valley" (common word that shouldn't be extracted). Or "Chianti Classico" matches "Chianti" (subregion) AND "Chianti Classico" (the full DOCG name). The EntityRuler's default behavior (longest match wins, earliest pattern wins on ties) may select the wrong one. The NER model in `en_core_web_lg` may also produce entity spans that conflict with EntityRuler patterns. The `doc.ents` ends up with inconsistent, partially correct entities.

**Why it happens:**
The spaCy EntityRuler resolves overlaps by preferring longer matches and earlier pattern definitions. But for wine, the "correct" resolution isn't always the longest match. If someone writes "Napa Valley Cabernet," the EntityRuler should match "Napa Valley" (region) AND "Cabernet" (shorthand for Cabernet Sauvignon) as separate, non-overlapping entities. But if "Napa Valley Cabernet" is a phrase pattern, it might match as a single 3-token entity, swallowing the varietal. Pattern ordering matters enormously and is non-obvious.

**How to avoid:**
1. Define EntityRuler patterns in **priority order**: add exact multi-word patterns first (e.g., "Napa Valley" as REGION), then single-word patterns (e.g., "Cabernet" → "Cabernet Sauvignon" as VARIETAL), then fuzzy patterns last
2. Set `overwrite_ents=True` on the EntityRuler so it replaces NER predictions that overlap with rule-based matches (the rules should be more trustworthy than the general NER model for wine terms)
3. Add the EntityRuler **before** the `ner` pipeline component (not after) — this way the NER model respects the ruler's spans and doesn't try to re-label them
4. For the "partial match" problem (e.g., "Napa Valley Cabernet" where you want two entities, not one), use token patterns with `OP` quantifiers instead of phrase patterns that greedily consume tokens
5. Build a test suite: 30-50 wine review snippets with hand-annotated correct entities. Run the pipeline against them and assert exact matches. This catches pattern collision regressions immediately.
6. For overlapping matches, implement a custom `on_match` callback that inspects overlaps and makes domain-aware decisions (e.g., prefer REGION label over MISC for "Bordeaux")

**Warning signs:**
- Entity counts that don't match expectations (a known region appears 0 times despite being common in r/wine)
- Visual inspection of `doc.ents` shows correct labels on some posts but missing entities on structurally similar posts
- NER labels like ORG or PERSON appearing in `doc.ents` for wine terms (the NER model is hallucinating)

**Phase to address:** Entity extraction phase — pattern design and pipeline ordering must be right before processing production data

---

### Pitfall 9: Pipeline Data Provenance — You Can't Tell What You've Already Processed

**What goes wrong:**
The scraper runs, fetches 500 posts, and crashes on post 487. You fix the bug and re-run. Did you re-scrape from scratch (wasting rate limit and time)? Did you resume from post 488 (but what if the bug corrupted post 487's data)? Later, you add a new extraction rule for a region you missed. Do you need to re-extract all 2000 posts, or just the ones that mention that region? Without metadata tracking, the answer is "re-run everything to be safe," which wastes hours. Worse, different pipeline runs produce slightly different data, and you can't tell which version of the analysis came from which run.

**Why it happens:**
Script-based pipelines (vs. workflow engines like Airflow) don't natively track state. The project uses Python scripts with Click CLI — there's no built-in notion of "which posts have been extracted" or "which extraction version produced this data." Each pipeline stage produces a file, but the files don't carry lineage metadata. This is acceptable for a one-and-done research run, but becomes painful the moment you iterate on extraction rules (which you will — entity extraction is inherently iterative).

**How to avoid:**
1. Every pipeline stage writes a **manifest file** alongside its output: a JSON or YAML file recording what was processed, when, with what parameters, and what the input was
2. At minimum track per-post: `post_id`, `scraped_at`, `extracted_at`, `extraction_version` (a hash of the extraction rules/config), `extraction_success` (boolean), `entity_count` (how many entities found)
3. Use the manifest for incremental re-processing: before extracting, check if `extraction_version` matches current version and `extraction_success` is true — skip if so
4. Store raw scraped data (the original Reddit JSON response objects) separately from the extracted data — don't overwrite the raw source
5. Name output files with timestamps or run IDs: `reviews_extracted_2026-11-27T14-30-00.parquet` not `reviews.parquet`
6. The simplest version: a SQLite `.db` file in the output directory that tracks `(post_id, stage, status, run_id, timestamp)` — lightweight, zero-config, queryable

**Warning signs:**
- You can't answer "when was this data extracted?" without checking file modification times (which are unreliable)
- Re-running the pipeline produces a different number of extracted entities and you can't explain why
- You want to add a new extraction rule but don't know which posts to re-process

**Phase to address:** Pipeline architecture / scaffolding phase — this is foundational infrastructure that all subsequent phases depend on

---

### Pitfall 10: Gazetteer Bootstrap — Starting from Nothing

**What goes wrong:**
The pipeline's EntityRuler and fuzzy linker need a gazetteer (knowledge base) of wine regions, subregions, varietals, and aliases before they can extract anything. But building a comprehensive gazetteer from scratch is a massive manual effort — there are thousands of official appellations, grape varieties with multiple synonyms, and informal names. The project can't start extracting until the gazetteer exists, creating a chicken-and-egg problem: you need a gazetteer to extract, but you need extraction data to know what to put in the gazetteer.

**Why it happens:**
The project spec says "Wine region/varietal knowledge base needed — either built from public sources or iteratively grown." But the practical path isn't specified. Downloading a complete AOC/AVA list gives 500+ entries, most of which will never appear in r/wine (obscure French crus). Hand-curating a list is hours of work and inevitably misses the terms people actually use on Reddit. The iterative approach ("extract what you can, review what you missed, add it") works but the first iteration produces near-zero results without a decent seed.

**How to avoid:**
1. **Seed the gazetteer from high-signal public data**, not from scratch:
   - Scrape Wine-Searcher's "most popular" region/varietal lists (~200 entries each) — these are the terms most likely to appear in r/wine
   - Use Wikipedia's "List of wine-producing regions" for the hierarchy structure
   - Include the top 50 grape varieties by global planting (OIV data)
   - Add U.S.-focused regions since r/wine skews American: all AVAs, not just the famous ones
2. **Run an initial extraction pass with the seed gazetteer**, then analyze the "unmatched terms" output to identify what you're missing
3. **Prioritize by frequency, not completeness**: A gazetteer with 200 high-frequency entries that covers 80% of mentions is far more useful than a 2000-entry gazetteer where half are never used
4. Store the gazetteer as a version-controlled CSV or JSONL file (not a Python dict) so you can add entries without touching code
5. Build a "gazetteer review" marimo notebook as part of the analysis pipeline — it shows unmatched terms sorted by frequency, with a one-click "add to gazetteer" workflow
6. Set realistic expectations: the first pass will miss 15-30% of entities. That's acceptable. Plan for at least two extraction runs with gazetteer refinement between them.

**Warning signs:**
- First extraction run yields <5 entities per post on average (seed gazetteer is too sparse)
- Manual gazetteer curation takes more than 2 hours (you're over-scoping — build the MVP seed, run extraction, then expand)
- Unmatched terms list is >100 unique terms in the first 500 posts — the seed missed major categories

**Phase to address:** Entity extraction phase — gazetteer creation is the prerequisite that must be completed before any extraction code runs against real data

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcoding the gazetteer in Python | No file I/O, quick to start | Every gazetteer update requires a code change and re-running the full pipeline | Never — gazetteer changes will be frequent. Use a data file. |
| Using a single regex for price extraction | Fast to write, one pattern | Misses most formats; silently produces no price when the regex doesn't match | Only for a spike/prototype demo, never for real data |
| Skipping the manifest tracking system | Fewer files, simpler code | Can't resume, can't audit, can't reproduce results | Only for a one-shot analysis where you'll never re-run |
| Hardcoding the fuzzy threshold at 85 | No calibration work | False positives on short terms, false negatives on long terms; data quality degrades invisibly | Only for a prototype; calibrate before processing all data |
| Using `.json()` without Content-Type checking | One less line of code | Silent failures when Reddit returns HTML; wastes time debugging mysterious JSONDecodeErrors | Never |
| Storing all data in memory (lists of dicts) | Simpler than Polars DataFrames | Memory explosion at 2000+ posts with entity extraction data; analysis queries become slow loops | Only for first 10 test posts |
| Converting to pandas mid-pipeline for convenience | Familiar API | Nullifies Polars performance gains, requires `to_pandas()`/`from_pandas()` conversions, loses lazy evaluation | Never — stay in Polars throughout |
| `nlp(text)` processing one post at a time in a loop | Simple control flow | 20-50x slower than `nlp.pipe(batch_size=50, n_process=1)`; 2000 posts could take 10+ minutes instead of 30 seconds | Only for <10 posts during development |
| depending on reddit user agent hint for old reddit api scrapers | | | In the "old.reddit.com" JSON API, never set the User-Agent to the Python requests default; always provide something like "r-wine-review-analyzer/1.0 (research project)" — Reddit's bot detection blocks generic agents more aggressively |

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| old.reddit.com JSON API | Assuming `.json` endpoints always return JSON. Reddit returns HTML (login pages, blocks, GDPR consent) with 200 OK status. | Validate `Content-Type: application/json` header before parsing. Wrap `response.json()` in try/except with HTML detection. |
| old.reddit.com pagination | Hardcoding `limit=100` and assuming all 100 items are valid posts. Deleted posts still appear in listings with `selftext: "[removed]"`. | Use `after` parameter for pagination (not page numbers). Validate each post's `selftext` before processing. Reddit listings max at 1000 items per sort order. |
| spaCy en_core_web_lg | Passing raw HTML/Markdown to `nlp(text)`. Reddit selftext contains Markdown formatting that spaCy's tokenizer will tokenize into noise tokens. | Strip Markdown formatting before NLP processing. Use `re.sub(r'\[.*?\]\(.*?\)', '', text)` for links, `re.sub(r'[*_~`]', '', text)` for formatting characters. |
| spaCy EntityRuler | Adding patterns after the pipeline has already processed documents. Changes to the EntityRuler don't retroactively apply. | Add all EntityRuler patterns before calling `nlp(text)` or `nlp.pipe()`. If updating patterns, you must re-process all documents. |
| RapidFuzz | Using `fuzz.ratio()` on multi-word strings without normalization. "Napa Valley" vs. "Valley, Napa" gets a low score despite being the same entity. | Use `fuzz.token_sort_ratio()` for multi-word terms. Lowercase both strings before comparison. Strip punctuation. |
| Polars Parquet I/O | Writing `.parquet` without `use_pyarrow=True` or schema specification. Different runs may infer different types (string vs. categorical) for the same column. | Specify explicit schemas when writing: `df.write_parquet(path, use_pyarrow=True)`. Use `scan_parquet()` for lazy reading of large files. |
| Polars string columns | Using `pl.Utf8` dtype and hitting performance cliffs on string-heavy operations. Polars string operations are slower than categorical on repeated values. | Cast repeated string columns (regions, varietals) to `pl.Categorical` after extraction. This reduces memory 5-10x and speeds up groupby. |
| marimo | Using global variables for data that should be reactive. Changes to a variable won't trigger re-execution of dependent cells unless defined as `mo.state()` or computed reactively. | Define data-loading cells with `mo.cache()` for large datasets. Use `@mo.cell` decorator for reactive dependencies between cells. |

## Performance Traps

Patterns that work at small scale but fail as the dataset grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Single-threaded spaCy processing in a `for` loop | Pipeline takes 5+ minutes for 2000 posts; CPU sits at 20% | Use `nlp.pipe(texts, batch_size=50, n_process=1)` — even single-process batching gives 20-50x speedup vs. `for text in texts: nlp(text)` | ~50 posts (noticeable at 200) |
| Processing all extracted entities in memory as Python objects | Memory usage grows linearly with entity count; OOM at 50K+ entities | Store entities in Polars DataFrames from the moment of extraction. Avoid intermediate Python lists of dicts. | ~2000 posts × ~8 entities = 16K entities (approaching memory pressure) |
| `spacy.load("en_core_web_lg")` inside a pipeline stage function | Model loads from disk on every pipeline invocation (500MB, 2-5 seconds per load) | Load the model once at module level or via a singleton. Pipeline stages should receive the loaded `nlp` object, not load it themselves. | First run already too slow |
| Fuzzy matching against entire gazetteer for every extracted term | O(n × m) where n = extracted terms, m = gazetteer size. With 2000 terms and 500 gazetteer entries, that's 1M fuzzy comparisons, taking 10+ seconds | Exact-match first (hash lookup, O(1)). Only fuzzy-match the ~10% of terms that don't exact-match. Pre-filter gazetteer by length (±30%). | ~200 gazetteer entries × 500 extracted terms (noticeable slowdown) |
| Writing analysis output as CSV with string heavy columns | 2000 rows × 15 columns works fine. But CSV has no type information, so re-reading requires type inference or explicit schema. | Use Parquet format. It preserves types, loads 5-10x faster, and integrates natively with Polars. | ~500 rows (CSV is fine below this; Parquet wins above it) |
| Using `rapidfuzz.process.extractOne()` for every term | `extractOne` compares one term against the full gazetteer list for every term, doing redundant work | For a batch of extracted terms, batch them: pass all extracted terms at once to avoid per-term setup overhead. Or build an index if the gazetteer is large (>1000 entries). | ~2000 extracted terms × 500 gazetteer entries (1-2 seconds with RapidFuzz, but grows O(n²)) |

## Security and Ethical Concerns

Domain-specific issues beyond general security practices.

| Concern | Risk | Mitigation |
|---------|------|------------|
| Reddit rate limiting and blocking | Aggressive scraping (even within stated rate limits) can get your IP temporarily blocked or permanently banned. Reddit's anti-bot heuristics are more aggressive than their documented API limits. | Respect 2 req/sec **with jitter** (1.8-2.2 second random intervals). Use a descriptive User-Agent. If you get blocked, stop for 1 hour before retrying. Never scrape authenticated — use the unauthenticated old.reddit.com JSON API. |
| Personally identifiable information (PII) | Reddit usernames are collected as part of the listing JSON. These are public, but aggregating them alongside wine preferences could be considered profiling. | Strip author names from the final dataset. Only keep `post_id` for deduplication. The analysis is about wine patterns, not about individual users' drinking habits. |
| Reddit Terms of Service | Scraping Reddit data may violate ToS depending on usage. The old.reddit.com JSON API is a gray area — it's a public endpoint but not an official API. | Use only for personal research, not for a commercial product. Don't redistribute raw Reddit data. Respect robots.txt (though old.reddit.com doesn't serve one for JSON endpoints). |
| Data retention | Storing 2000+ Reddit posts locally creates a corpus that could be subpoenaed or leaked. The raw posts contain usernames, timestamps, and subreddit context. | Keep raw scraped data in a `.gitignore`d directory. Don't commit scraped data to git. Set a retention policy: delete raw data after the analysis is complete, keep only the aggregated/extracted dataset. |
| Copyright of tasting notes | Reddit users' tasting notes are their original content. Aggregating and analyzing them is fair use, but republishing them verbatim could be infringement. | Don't include the full `selftext` in published datasets. Extract and store only the structured entities. Provide aggregates, not individual reviews, in any public output. |

## "Looks Done But Isn't" Checklist

Things that appear complete but aren't actually ready for analysis.

- [ ] **Scraping:** Often missing pagination — scrapes the first 100 posts and stops. Verify: `len(posts) >= requested_limit` and post IDs are unique. Reddit listings max out at ~1000 posts per sort order; to get 2000 you need multiple sort strategies (hot, new, top, etc.).
- [ ] **Pre-filtering:** Often implemented as a simple keyword check. Verify: manually review 50 filtered-out posts to check for false negatives (real reviews that were rejected).
- [ ] **Region extraction:** Often missing subregion context. Verify: for a post mentioning "Pauillac," the output includes Pauillac AS subregion AND Bordeaux AS parent (via hierarchy propagation).
- [ ] **Varietal extraction:** Often maps "Cab" → "Cabernet Sauvignon" but doesn't handle "Cab Franc" → "Cabernet Franc" correctly (the short form "Cab" ambiguously refers to both). Verify: test on posts mentioning both. The pattern for "Cab" should require a lookahead to check if "Franc" follows.
- [ ] **Score normalization:** Often normalizes 20-point scores but doesn't tag them as `score_scale: "inferred_20pt"`, making post-hoc auditing impossible. Verify: every score row has a `score_scale` column populated.
- [ ] **Price normalization:** Often extracts "$25" but fails to detect "per bottle" context when the price was for a multi-bottle purchase. Verify: check that `price_unit` is populated for every price row, not just a single number.
- [ ] **Fuzzy linking:** Often links every unmatched term to the best fuzzy match, including noise terms. Verify: the unmatched terms log contains only true unknowns (not terms that should have matched but the gazetteer was missing).
- [ ] **Pipeline re-runs:** Often overwrite previous results without keeping history. Verify: re-running the pipeline produces a new output file, not a mutated old one.
- [ ] **Missing data handling:** Often silently drops posts with missing entities rather than recording them as "no entities extracted." Verify: the dataset has rows for every post that passed the pre-filter, even if `entity_count=0`.

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Reddit blocks your IP mid-scrape | LOW | Wait 1-24 hours for the block to expire (Reddit blocks are usually temporary). Implement exponential backoff from that point forward. Resume scraping from the last successful `after` cursor. Resume scraping from verified checkpoint. |
| Gazetteer misses a common term | LOW | Add the missing term to the gazetteer CSV, re-run only the extraction stage (not scraping). The manifest system enables incremental re-extraction of affected posts. |
| Fuzzy threshold is too low (false positives) | MEDIUM | Adjust the threshold, re-run fuzzy linking against the already-extracted entities (no need to re-scrape or re-run spaCy). Audit fuzzy match logs for new false positives. |
| Score normalization applied wrong scale | MEDIUM | If raw scores were stored alongside normalized scores, re-run only the normalization stage with corrected heuristics. If raw scores weren't stored, must re-run extraction (higher cost). Prevention: always store raw+normalized. |
| Pipe pipeline crashes mid-extraction | LOW | With manifest tracking, identify the last successfully processed post_id. Resume extraction from the next post. |
| EntityRuler pattern collision introduces wrong entity labels | MEDIUM | Fix the patterns, clear extraction cache for affected posts, re-extract. The test suite should catch this before it reaches production data. |
| Reddit shuts down old.reddit.com entirely | HIGH | This removes the primary data source. Options: (1) switch to the official Reddit API (requires OAuth app registration, has rate limits), (2) switch to Pushshift if available (has been unreliable since 2023 API changes), (3) accept the dataset as a static snapshot from the already-scraped data. Prevention: scrape all needed data in the first run — don't assume you can scrape more later. |

## Pitfall-to-Phase Mapping

How the pipeline phases should address these pitfalls. (Phases listed here are logical pipeline stages — the roadmap implementer maps these to formal GSD phases.)

| Pitfall | Prevention Stage | Verification |
|---------|------------------|--------------|
| Silent non-JSON responses (P1) | Scraping | Scraper unit tests with mock HTML responses; Content-Type validation exists in code |
| Non-review contamination (P2) | Pre-filtering | Manual audit of 50 filtered posts for false negatives; review_confidence field populated |
| Synonym fragmentation (P3) | Entity Linking | Gazetteer file has alias columns; output shows canonical names, not raw extracted strings |
| Region hierarchy collapse (P4) | Entity Linking | Gazetteer is tree-structured; region aggregations include parent counts; drill-down visualization works |
| Fuzzy threshold false positives (P5) | Entity Linking | Fuzzy match test suite passes; threshold is length-adaptive; fuzzy match logs auditable |
| Score scale ambiguity (P6) | Extraction + Normalization | Score distribution is unimodal around 85-90; score_scale column populated; no scores >100 after normalization |
| Price format proliferation (P7) | Extraction + Normalization | Price distribution has no extreme outliers; multi-bottle prices divided correctly; price_unit column populated |
| EntityRuler collision (P8) | Entity Extraction | EntityRuler test suite passes; visual spot-check of doc.ents on 20 sample reviews is clean |
| Data provenance (P9) | Pipeline Scaffolding | Manifest files exist; re-running pipeline produces identical results; can answer "when was this processed?" |
| Gazetteer bootstrap (P10) | Entity Extraction | Seed gazetteer has 200+ entries; first extraction pass yields >5 entities/post on average; unmatched terms log is manageable |

## Sources

- [spaCy Rule-Based Matching Guide](https://spacy.io/usage/rule-based-matching) — EntityRuler API, pattern resolution, overlap handling, pipeline integration (official docs, HIGH)
- [Reddit API Documentation](https://www.reddit.com/dev/api/) — Listing pagination (`after`/`before`), rate limits, `raw_json` parameter, `selftext` field for deleted posts (official docs, HIGH)
- Live test: `old.reddit.com/r/wine/.json` returned HTML instead of JSON (2026-11-27), confirming Pitfall 1 is active and requires Content-Type validation (direct observation, HIGH)
- SpaCy EntityRuler overlap resolution: longest span wins by default; `overwrite_ents` controls NER conflict resolution; pattern ordering matters for equally-long matches (official docs, HIGH)
- RapidFuzz API: `fuzz.ratio()` vs `fuzz.token_sort_ratio()` — token_sort handles word reordering in multi-word terms; Levenshtein distance is character-based, not semantic (official docs, HIGH)
- Reddit API post listing limits: maximum 1000 posts per sort order; deleted posts appear with `selftext: "[removed]"` or `selftext: "[deleted]"`; pagination uses `after` cursors, not page numbers (official docs, HIGH)
- Wine scoring scales: Robert Parker 100-point (practical range 50-100), Jancis Robinson 20-point, 5-star ratings, letter grades (A-F) — common knowledge in wine domain but requires explicit scale detection heuristics (domain expertise, HIGH)
- Wine naming conventions: grape synonyms from OIV and Wine Grapes by Robinson/Harding/Vouillamoz; regional hierarchy from AOC/AVA/DOCG classification systems; "Claret" = Bordeaux, "Syrah" = "Shiraz" are historically documented synonyms (domain expertise, HIGH)
- spaCy `nlp.pipe()` batch processing: `batch_size=50` provides 20-50x speedup over `for text in texts: nlp(text)` by amortizing pipeline setup costs (official docs, HIGH)
- Polars categorical optimization: `pl.Categorical` reduces memory 5-10x on repeated string columns and accelerates groupby operations (official docs, HIGH)
- Reddit rate limiting: documented at 60 requests/minute for OAuth API; unauthenticated old.reddit.com has undocumented but observed ~30 requests/minute before returning HTML block pages (community observation, MEDIUM)
- Pipeline provenance patterns: manifest files, incremental processing, content-addressed storage — common in data engineering ETL pipelines (industry practice, MEDIUM)

---
*Pitfalls research for: r/wine Review Analysis NLP data pipeline*
*Researched: 2026-11-27*
