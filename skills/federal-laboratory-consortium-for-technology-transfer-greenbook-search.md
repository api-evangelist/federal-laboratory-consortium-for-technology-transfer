---
name: flc-greenbook-search
description: Answer a question about US federal technology transfer legislation and policy using the FLC Green Book retrieval-augmented search API, and return the answer with the Green Book sections it was grounded in.
api: FLC Greenbook API
base_url: https://api.federallabs.org
operations:
  - SectionsController_searchSections
generated: '2026-09-09'
method: generated
source: openapi/federal-laboratory-consortium-for-technology-transfer-greenbook-openapi.yml
---

# FLC Green Book — grounded policy search

The Federal Laboratory Consortium publishes *The Green Book* (Federal Technology Transfer
Legislation and Policy), the reference for the statutes and executive policies that frame the
federal technology transfer program. The FLC Green Book mobile app is backed by a public API with
exactly one published operation, and it does retrieval-augmented answering: semantic retrieval over
Green Book sections, an LLM rerank, then a generated answer plus the source sections.

## Before you start

**Access is not self-service.** The operation requires an `app-key` request header, described in the
contract as the "Mobile app key". There is no published sign-up flow, no developer portal, and no
pricing page — `https://federallabs.org/pricing` and `https://federallabs.org/developer-api-documentation`
both redirect to `/page-not-found`. Ask FLC at <https://federallabs.org/contact>. Without a key the
endpoint returns `401` and nothing else works.

## Step 1 — Ask a question

`SectionsController_searchSections` — `POST /v1/sections/search`

Headers:

- `app-key: <your key>` — required. Missing or wrong gives `401`.
- `x-session-id: <opaque id>` — the contract marks this **required** in `parameters`, while the
  operation description calls it optional and says it is used for metrics and queries-per-session.
  Treat it as required and send a stable per-conversation identifier.
- `Content-Type: application/json`

Body (`SectionSearchQueryDto`):

- `query` (string, required, max 2000 characters) — the user's question or topic.
- `limit` (number, optional, default 10, range 1–50) — how many source sections to return.

## Step 2 — Read the answer and its sources

A `200` returns `SectionSearchResponseDto`:

- `answer` (string) — the generated answer.
- `sections` (array of `SectionSearchResultItemDto`) — the sources the answer was built from,
  reranked by the LLM and ordered by relevance. Each carries `section` (id and title only) and
  `score` (similarity, 0–1, higher is more similar).
- `cacheId` (number) — a cache entry id. The operation description says it is used to submit
  feedback via `POST /v1/sections/query-cache/:id/feedback`. **That operation is not declared in the
  published contract**, so do not call it blind — confirm with FLC first.

**Always show the sections.** This is a RAG surface over legal and policy text: the `answer` is
generated, the `sections` are the citation, and an answer presented without them is an unsourced
claim about federal law. If `sections` is empty or every `score` is low, say the Green Book did not
cover the question rather than passing the generated text along.

## Errors and limits

- `401` — `{"message":"Invalid or missing app-key header...","error":"Unauthorized","statusCode":401}`.
  The envelope is `{message, error, statusCode}`; it is **not** RFC 9457 problem+json.
- Rate limit — responses carry `X-RateLimit-Limit: 100`, `X-RateLimit-Remaining` and
  `X-RateLimit-Reset: 60` (seconds). Budget for roughly 100 requests per minute and back off on
  `X-RateLimit-Remaining: 0`. No `Retry-After` is sent.
- There is no idempotency mechanism and none is needed: the operation writes nothing you own, so it
  is safe to retry. There is likewise nothing to reverse.

See `conventions/`, `errors/` and `rate-limits/` in this repository for the full observed behaviour.
