---
name: reefapi-engine-discovery
description: Find the right ReefAPI engine and action for a data need, learn its exact input contract and credit price, then make a correct first call — without guessing an endpoint.
api: ReefAPI
generated: '2026-08-31'
method: generated
source: mcp/reefapi-mcp-tools.json (live tools/list probed 2026-08-31), openapi/reefapi-openapi.json, https://reefapi.com/mcp, https://reefapi.com/docs
operations:
  - search_engines
  - get_catalog
  - get_engine_schema
  - get_action_schema
  - call_engine
---

# ReefAPI: discovery-first engine selection

ReefAPI fronts **183 engines / 1,528 actions** behind one contract. Do **not** guess a path —
the catalog changes weekly. Discover, then call. Discovery is **keyless and free**; only the
final call spends credits, and a failed call costs zero.

## When to use this

You need live web data the model cannot know from memory (prices, listings, posts, reviews,
job postings, availability, profiles), or a direct fetch failed because the site is
captcha-walled, login-walled or JS-heavy.

## Steps

1. **Find the engine.** Call `search_engines` with English keywords or a one-line use case
   ("company reviews", "rental listings in Austin", "is this domain free"). The catalog is
   indexed in English — translate a non-English need before searching.
   If the keyword search misses, call `get_catalog` (no arguments) and pick from the full
   menu yourself; it is grouped into 14 categories.
2. **List the engine's actions.** Call `get_engine_schema` with `{ "engine": "<slug>" }`.
   You get every action with its description, required params and return shape — kept
   deliberately compact so a 90-action engine stays token-cheap.
3. **Read the exact input contract.** Call `get_action_schema` with
   `{ "engine": "<slug>", "action": "<action>" }`. This is the authoritative source: every
   parameter with type, required flag, allowed values, default, min/max, plus the action's
   **credit price** and a ready-to-run `example_params`. Read the credit price before you
   call — most actions cost 1 credit, heavier ones cost more.
4. **Call it.** `call_engine` with `{ "engine", "action", "params" }`. This is the only tool
   that needs the user's key and the only one that spends credits.
5. **Branch on `ok`.** Every response is `{ ok, data, meta, error }`. On `ok: true` read
   `data`; also read `meta.record_count` and `meta.completeness_pct` to judge whether the
   extraction is good enough before acting on it, and `meta.credits` for what it cost.

## Rules

- **Never skip step 3.** The published OpenAPI declares every request property as an untyped
  string with no enum or example. `get_action_schema` is the only place the real enums,
  defaults and limits live. Building params from the OpenAPI alone will produce
  `INVALID_PARAM`.
- **Never hardcode an engine roster into a prompt.** Discover at call time.
- **Do not assume pagination.** Paging is per-engine and inherited from the upstream source:
  nine different parameter names appear across the catalog (`page`, `limit`, `cursor`,
  `per_page`, `max_results`, `offset`, `page_size`, `count`, `start`). Take the one
  `get_action_schema` names for this action.
- **Errors are cheap.** `ok: false` costs 0 credits, so a probing call is safe. Retry only
  when `error.retryable` is `true` (`RATE_LIMITED`, `TARGET_BLOCKED`, `UPSTREAM_TIMEOUT`).
  Never retry `AUTH_FAILED`, `QUOTA_EXCEEDED`, `MISSING_PARAM` or `INVALID_PARAM`.
- **Back off on 429.** Defaults are ~5 req/s per key and ~10 req/s per IP over a 3-second
  rolling window. No `Retry-After` or `RateLimit-*` header is returned — pace yourself from
  these documented numbers, do not wait for a header that never arrives.

## Equivalent over plain HTTP

There is **no REST endpoint for the catalog** — steps 1–3 exist only on MCP and in the
markdown docs. Over HTTP, read `https://reefapi.com/docs/{engine}.md` for the same parameter
and return detail, then:

```
POST https://api.reefapi.com/{engine}/v1/{action}
x-api-key: <key>
content-type: application/json

{ ...params }
```

Note the header differs by surface: REST uses `x-api-key`, MCP uses `Authorization: Bearer`.
