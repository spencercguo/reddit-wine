# API Coverage Decision — Reddit old.reddit.com JSON API

**Phase:** 1
**Generated:** 2026-11-27

## Context

Phase 1 scrapes r/wine posts via Reddit's unofficial `old.reddit.com/.json` endpoints. These are public, unauthenticated JSON feeds — not the official OAuth API. The unofficial API has no rate-limit headers, no documentation guarantees, and no versioning. This document records which endpoints Phase 1 integrates and which it opts out of.

## Endpoint Decisions

| Endpoint | Decision | Rationale |
|---|---|---|
| `GET /r/{subreddit}/new.json` | **INTEGRATE** | Primary data source for Phase 1. Returns paginated listing of recent posts in JSON format. Pagination via `?after={fullname}` cursor parameter. Used by `src/scrape.py` fetch loop. |
| `GET /r/{subreddit}/comments/{post_id}.json` | **INTEGRATE** | Used only for image posts (empty selftext) — fetches the OP's first top-level comment to merge into `review_text`. Called per image post within the scrape loop (CONTEXT.md D-02). Limited to `?limit=1` to minimize requests. |
| Pagination via `after` cursor | **INTEGRATE** | Reddit listing responses include `data.after` (a base-36 fullname like `t3_1abc23`). The scraper passes this as `?after={value}` to fetch the next page. Empty/null `after` signals end-of-listing. |
| `GET /r/{subreddit}/hot.json` | **OPT-OUT** | Not needed for Phase 1. The `new.json` sort provides a chronological feed suitable for a representative sample. Multi-sort strategies (hot, top, new) are deferred to Phase 5 orchestration if listing exhaust becomes a concern. |
| `GET /r/{subreddit}/top.json` | **OPT-OUT** | Same rationale as `hot.json`. Deferred to Phase 5. |
| `GET /r/{subreddit}/search.json` | **OPT-OUT** | Search is not needed for the pipeline's goal of sampling community reviews chronologically. |
| `GET /r/{subreddit}/about.json` | **OPT-OUT** | Subreddit metadata (subscriber count, description) is not needed for scraping. |
| `GET /user/{username}/.json` | **OPT-OUT** | User profile scraping is explicitly out of scope (REQUIREMENTS.md: no user profiling). Author names are stored in raw data but not analyzed. |
| OAuth endpoints (`oauth.reddit.com`) | **OPT-OUT** | The pipeline uses the unauthenticated JSON API. OAuth would require app registration, client secrets, and token management — unnecessary complexity for a local research tool. |
| Pushshift API | **OPT-OUT** | Historical archive alternative. Not needed for current data; old.reddit.com `.json` provides sufficient recency. Could serve as fallback if Reddit blocks the unofficial API in the future (RESEARCH.md assumption A1). |

## Endpoint Shape (Key Fields Used)

### `GET /r/{subreddit}/new.json` Response

```json
{
  "kind": "Listing",
  "data": {
    "after": "t3_1abc23",
    "children": [
      {
        "kind": "t3",
        "data": {
          "id": "1abc23",
          "name": "t3_1abc23",
          "title": "...",
          "selftext": "...",
          "author": "...",
          "created_utc": 1732745600.0,
          "score": 42,
          "num_comments": 15,
          "permalink": "/r/wine/comments/1abc23/...",
          "post_hint": "image",
          "stickied": false,
          "url": "https://i.redd.it/..."
        }
      }
    ]
  }
}
```

### `GET /r/{subreddit}/comments/{post_id}.json?limit=1` Response

Returns a list of two elements: `[post_data, comment_listing]`. The scraper reads `[1]["data"]["children"]` and matches `author == op_author` on the first top-level comment (`kind == "t1"`).

## Known Limitations

1. **Listing exhaust at ~1000 posts.** Reddit caps listing indexes at approximately 1000 items per sort order. A `--limit 2000` request may return fewer posts than requested. The manifest tracks `posts_requested` vs `posts_returned` to make the discrepancy visible.
2. **No rate-limit headers.** The unofficial API does not return `X-RateLimit-*` headers. Rate limiting is enforced client-side via tenacity's `wait_exponential + wait_random`.
3. **HTML responses at 200 OK.** Reddit may serve HTML (login pages, blocks, GDPR interstitials) with HTTP 200 instead of proper error codes. Content-Type validation catches these before JSON parsing.
4. **No SLA.** The unofficial JSON API may change or be removed at any time. Assumption A1 in RESEARCH.md tracks this risk; Pushshift is the documented fallback.
