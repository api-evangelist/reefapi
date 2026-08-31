---
name: reefapi-market-research-sweep
description: Run a multi-source reputation and demand sweep on a company or topic — reviews, social discussion, hiring signal and local presence — by fanning one question across several ReefAPI engines.
api: ReefAPI REST API
generated: '2026-08-31'
method: generated
source: openapi/reefapi-openapi.json (operationIds verified against the spec), https://reefapi.com/docs
operations:
  - trustpilot_company_search
  - trustpilot_company_reviews
  - google_maps_place_search
  - google_maps_place_reviews
  - reddit_search
  - reddit_post_comments
  - indeed_company
  - indeed_company_reviews
  - indeed_company_salaries
  - linkedin_jobs_jobs_by_company
---

# ReefAPI: cross-source sweep on one company

The point of a single gateway is that one question can be asked of many sources with one key
and one envelope. This skill fans a company name across four independent signal families and
reconciles them.

## Steps

1. **Resolve the company on each source first — never assume a slug.**
   - Trustpilot: `trustpilot_company_search` (`POST /trustpilot/v1/company/search`) with
     `{ "query": "<name>" }` → take the matched domain.
   - Indeed: `indeed_company_search` (`POST /indeed/v1/company/search`).
   - Google Maps: `google_maps_place_search` (`POST /google-maps/v1/place/search`) with the
     name and an optional lat/lng bias.
2. **Customer reputation.** `trustpilot_company_reviews`
   (`POST /trustpilot/v1/company/reviews`) for the star distribution and review text, and
   `google_maps_place_reviews` (`POST /google-maps/v1/place/reviews`) for local reviews —
   this action paginates to *all* reviews, so bound it before you start.
3. **Unfiltered discussion.** `reddit_search` (`POST /reddit/v1/search`) with
   `{ "q": "<company or product>", "sort": "new" }`; follow any high-signal thread with
   `reddit_post_comments` (`POST /reddit/v1/post_comments`). Where a thread is truncated,
   `reddit_load_more_comments` expands the collapsed nodes.
4. **Employer and demand signal.** `indeed_company` for the profile,
   `indeed_company_reviews` for employee sentiment, `indeed_company_salaries` for the pay
   breakdown by title, and `linkedin_jobs_jobs_by_company`
   (`POST /linkedin-jobs/v1/jobs/by-company`) for what they are currently hiring — open
   roles are the cheapest available read on where a company is investing.
5. **Reconcile before reporting.** Say which source each claim came from and when it was
   fetched. Where sources disagree (a 4.6 on Google and a 2.1 on Trustpilot is common and
   meaningful), report the disagreement rather than averaging it away.

## Rules

- **Resolve, then fetch.** Every one of these engines has a search/resolve action for a
  reason; a guessed company slug returns `NOT_FOUND` (which is free, but wastes a turn).
- **Pace the fan-out.** This skill issues 8–10 calls. At ~5 req/s per key that is fine
  sequentially; do not parallelise it wide or you will collect `RATE_LIMITED` (429), which
  is retryable but returns no `Retry-After` header to guide you.
- **Budget it.** Roughly one credit per call at the default rate, plus more for paginated
  review pulls — `google_maps_place_reviews` paginating to every review on a large listing
  is the expensive step. Check `meta.credits` on each response.
- **Never present scraped review text as verified fact.** These are public records of what
  people wrote, retrieved live; attribute them.
- **`completeness_pct` matters here.** A partial extraction across a review corpus will skew
  sentiment. Read `meta.completeness_pct` and `meta.record_count` and disclose them when the
  extraction was partial.

## Adjacent engines worth adding

`sitejabber`, `trustradius`, `yelp`, `glassdoor`, `ziprecruiter`, `enrich-company` and
`domain-intel` cover the same question from other angles. Use `search_engines` over MCP, or
`https://reefapi.com/apis`, to find them rather than hardcoding this list.
