<!-- GSD:project-start source:PROJECT.md -->

## Project

**r/wine Review Analysis**

A data pipeline that scrapes wine review posts from Reddit's r/wine community, extracts structured entities (wine regions, varietals, vintages, prices, scores) using regex and NLP, and produces analytical datasets to uncover patterns in what people drink, what they pay, and what they think. Built as a local Python tool — one person's pipeline, not a hosted product.

**Core Value:** Turn thousands of unstructured community wine reviews into queryable, analyzable data — surfacing the hidden patterns in r/wine's tasting notes, price distributions, and regional preferences that no single post reveals alone.

### Constraints

- **Rate limit**: 2 requests/second to old.reddit.com
- **Compute**: CPU-only, no GPU required (spaCy `en_core_web_lg` is adequate)
- **Language**: Python only, single codebase
- **Scope**: Single subreddit (r/wine), English-language posts only
- **Gazetteer**: Wine region/varietal knowledge base needed — either built from public sources or iteratively grown
- **Environment**: macOS (darwin), local development

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Python | 3.13 | Runtime | Latest stable with GIL improvements and free-threading support. Required by Polars 2.x (>=3.11) and RapidFuzz 3.x (>=3.11). macOS default. |
| spaCy | 3.8.18 | NLP engine: tokenization, EntityRuler, NER | Industry standard for rule-based + statistical NLP. `EntityRuler` gives deterministic extraction for known wine entities. `en_core_web_lg` model provides word vectors and NER without GPU. |
| Polars | 2.2.0 | Data analysis: groupby, aggregation, cross-tabs | Rust-based engine 10-50x faster than pandas for analytical queries. Apache Arrow backing enables zero-copy interop. Expressive API with lazy evaluation and query optimization. |
| RapidFuzz | 3.14.6 | Fuzzy entity linking | C++ backed Levenshtein distance matching at 10-100x the speed of FuzzyWuzzy. `process.extractOne()` with threshold tuning is ideal for matching extracted entities to gazetteer entries. |
| marimo | 0.26.0 | Reactive notebook exploration | Stores as pure `.py` files (Git-friendly), deterministic execution order, built-in Plotly integration, interactive dataframes. Replaces Jupyter notebook entirely. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| requests | 2.x | HTTP client for Reddit JSON API | Standard library for simple synchronous HTTP. Reddit's old.reddit.com/.json endpoint returns structured JSON that requests parses natively. |
| tenacity | 9.x | Retry logic with exponential backoff | Enforce 2 req/sec rate limit with automatic retry on 429/503. `@retry(wait=wait_exponential(...))` decorator keeps scraping code clean. |
| Pydantic | 2.x | Data models and validation | Define structured `WineReview` model with typed fields (vintage: int, price: float, score: int). Auto-validates extracted entities. |
| plotly | 7.2.0 | Interactive visualizations | Rich chart types (histograms, scatter, choropleth maps for wine regions). Integrated natively with marimo for reactive dashboards. Version 7.x is the current stable; use `plotly.express` for declarative charts. |
| click | 8.x | CLI interface for pipeline scripts | Clean argument parsing for pipeline stages (e.g., `pipeline scrape --subreddit wine --limit 2000`). Lighter than argparse for script-based pipelines. |
| spacy-lookups-data | latest | Lemmatization and normalization data for spaCy | Required for en_core_web_lg model's lemmatizer. Install with `spacy[lookups]` extra. |
| pyarrow | bundled (Polars) | Columnar data format for Parquet I/O | Polars uses Apache Arrow natively. Explicit import only if working with Arrow tables directly. |
| pytest | 9.2.0 | Testing framework | Standard Python test runner. Fixtures for sample review data. Parameterized tests for extraction edge cases. Pytest 9.x is current stable. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| uv | Package management and virtual environments | Fast pip alternative. Creates isolated `.venv`. `uv pip install` for deterministic installs. |
| ruff | Linting and formatting | Fast Python linter (Rust-based). Handles both linting and formatting. Configure via `pyproject.toml`. |
| mypy | Static type checking | Ensure type safety on Pydantic models and Polars expressions. Opt-in via `pyproject.toml`. |
| pre-commit | Git hook automation | Run ruff + mypy before commits. Ensures pipeline scripts remain clean. |

## Installation

# Create virtual environment

# Core

# spaCy model (downloads ~500MB word vectors)

# Dev dependencies

## Alternatives Considered

| Recommended | Alternative | Why Not |
|-------------|-------------|---------|
| Polars | pandas | Polars is 10-50x faster for analytical queries (groupby, aggregation, cross-tabs). Better memory profile. Project already decided against pandas. |
| spaCy EntityRuler | Custom regex-only extraction | Regex alone can't handle entity disambiguation (e.g., "Merlot" as varietal vs. winery name). EntityRuler combines regex with linguistic context. |
| RapidFuzz | FuzzyWuzzy | FuzzyWuzzy is deprecated, GPL-licensed, and 10-100x slower. RapidFuzz is MIT-licensed, actively maintained, and has a cleaner API. |
| marimo | Jupyter Notebook | Jupyter stores as JSON (bad for git), has hidden state, non-deterministic execution. marimo solves all of these. |
| requests + tenacity | httpx/async | Async adds complexity with no benefit here: pipeline is single-machine, CPU-bound on NLP, not I/O-bound. Synchronous requests with tenacity retries is simpler and sufficient. |
| Pydantic v2 | dataclasses | Pydantic provides built-in validation (e.g., vintage: 1900-2030), JSON schema generation, and better error messages for malformed extracted entities. |
| click | argparse | click has cleaner decorator-based API, better help text, and native support for subcommands (scrape/extract/analyze). |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| pandas | Slower than Polars for analytical queries. Already ruled out by project decisions. Heavier memory footprint. | Polars 2.2.0 |
| NLTK | Academic library with slower performance. Overkill for a pipeline that needs tokenization + NER. spaCy does both with a cleaner API. | spaCy 3.8.18 |
| transformers / HuggingFace BERT | Requires GPU for reasonable performance. EntityRuler + regex achieves sufficient accuracy for structured wine review data. | spaCy EntityRuler + regex |
| BeautifulSoup / Scrapy | Reddit JSON API returns structured data — no HTML parsing needed. Adding a DOM parser adds unnecessary complexity. | requests (raw JSON) |
| FuzzyWuzzy | Deprecated, GPL-licensed, 10-100x slower than RapidFuzz | RapidFuzz 3.14.6 |
| Jupyter Notebook | Hidden state bugs, JSON storage (bad for git), non-deterministic cell execution order | marimo 0.26.0 |
| Streamlit / Dash | Overkill for local research-grade analysis. marimo provides reactive interactivity without building a web app. | marimo + Plotly |
| SQLite / PostgreSQL | No database needed — output is analytical Parquet/CSV files explored in marimo. Adding a DB adds schema management overhead for research pipeline. | Polars with Parquet I/O |
| Docker | Local macOS pipeline, no containerization needed. Adds complexity without benefit for single-machine research tool. | `uv venv` + `pyproject.toml` |

## Stack Patterns by Variant

- Use `spacy-langchain` or `vaderSentiment` for lightweight sentiment (no GPU)
- Avoid HuggingFace transformers for this specific use case (CPU constraint)
- VADER is adequate: rule-based, fast, good on social media text
- Switch from synchronous `requests` to `httpx` with async for parallel API calls (still respecting 2 req/sec)
- Use Polars `LazyFrame` with `scan_parquet` for streaming analysis
- Consider `spaCy nlp.pipe(batch_size=...)` for batched processing
- `spacy[transformers]` with `en_core_web_trf` (requires CUDA)
- Polars `[gpu]` extra for NVIDIA-accelerated queries

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| spaCy 3.8.18 | Python <3.16, >=3.10 | macOS ARM64/Intel wheels available. RegEX/FUZZY operators added in v3.5. |
| Polars 2.2.0 | Python >=3.11 | Major version 2.x changed some APIs from 1.x. Use 2.x docs for reference. |
| RapidFuzz 3.14.6 | Python >=3.11 | Requires Visual C++ 2019 redist on Windows only. macOS has prebuilt wheels. |
| Plotly 7.2.0 | Python >=3.10 | Use `plotly.express` API (px.histogram, px.scatter, px.bar) not graph_objects for declarative charts. 7.x is current stable with API compatibility from 6.x.
| marimo 0.26.0 | Python >=3.11 | Install `marimo[recommended]` for SQL + AI features. |
| en_core_web_lg | spaCy 3.8.x | `lg` = large model (~500MB). Provides word vectors and NER. `sm`/`md` alternatives lack word vectors needed for semantic similarity. |

## Sources

- [PyPI: spaCy 3.8.18](https://pypi.org/project/spacy/) — version, Python compatibility, release date (context7, MEDIUM)
- [spaCy Rule-Based Matching Guide](https://spacy.io/usage/rule-based-matching) — EntityRuler API, phrase patterns, token patterns, REGEX/FUZZY operators (official docs, HIGH)
- [spaCy Models & Languages](https://spacy.io/usage/models) — en_core_web_lg availability and download instructions (official docs, HIGH)
- [PyPI: Polars 2.2.0](https://pypi.org/project/polars/) — version, Python >=3.11 requirement, release date (context7, MEDIUM)
- [Polars Python API Reference](https://docs.pola.rs/api/python/stable/reference/index.html) — DataFrame/LazyFrame APIs, group_by, agg, expressions (official docs, HIGH)
- [PyPI: RapidFuzz 3.14.6](https://pypi.org/project/rapidfuzz/) — version, Python >=3.11, API surface, performance benchmarks (context7, MEDIUM)
- [PyPI: marimo 0.26.0](https://pypi.org/project/marimo/) — version, Python >=3.11, reactive notebook features (context7, MEDIUM)
- [marimo Official Documentation](https://docs.marimo.io/) — Plotly integration, interactive dataframes, reactivity model (official docs, HIGH)
- Reddit JSON API scraping patterns, wine regex extraction patterns, ETL pipeline architecture, wine gazetteer construction (websearch, LOW — used for pattern guidance, not as authoritative sources)

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-fast` for a trivial task inline, with no subagents and no PLAN.md
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
