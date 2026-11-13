# CourtMesh API Postman Collection

A Postman Collection v2.1 for the CourtMesh Indian court cases API. All 61 production endpoints (plus the deep variant of `GET /health`), organised into fourteen folders, with request bodies, saved example responses and per request documentation.

Base URL: `https://research.courtmesh.ai/api/v1/prod`

File: [`CourtMesh.postman_collection.json`](./CourtMesh.postman_collection.json)

## What is in it

| folder | requests |
| --- | --- |
| Search | `POST /search/cases`, `POST /search/cases/semantic` |
| Cases | `GET /cases/{id}`, `GET /cases/{id}/related`, `GET /cases/{id}/pdf`, `GET /cases/{id}/orders`, `GET /cases/{id}/orders/{documentId}/text` |
| Analysis | `GET /cases/{id}/analysis`, `POST /cases/{id}/analyze`, `POST /cases/{id}/analyze-consolidated` |
| Judges | `GET /judges/search` |
| CNR Lookup | `GET /cases/by-cnr/{cnr}`, `POST /search/cases` with `cnr`, `POST` and `GET /cases/by-cnr/{cnr}/refresh`, `POST /cases/refresh/batch` |
| Monitoring | watchlists (`/watchlists`, `/watchlists/{id}`, `/watchlists/{id}/items`, `/watchlists/{id}/items/{cnr}`), the change feed (`GET /changes`, `GET /watchlists/{id}/events`), webhooks (`/webhooks`, `/webhooks/{id}`, `rotate-secret`, `test`, `deliveries`, delivery `retry`) |
| Acts and Sections | `GET /acts/search`, `GET /acts/resolve`, `GET /acts/{slug}`, `GET /acts/{slug}/history`, `GET /acts/{slug}/sections`, `GET /acts/{slug}/sections/{sectionSlug}`, `GET /acts/{slug}/sections/{sectionSlug}/citing-cases`, `GET /sections/search`, `GET /sections/lookup` |
| Cause lists | `GET /cause-lists/listings`, `GET /matters/{matterId}/cause-listings`, `GET /cause-lists/court`, `POST /cause-lists/check` |
| Timeline | `POST /request-timeline`, `GET /get-timeline/{requestId}` |
| Litigation Check | `POST /party/screen`, `POST /party/screen/batch`, `POST /party/profile` (quick and full mode), `GET /party/profile/{requestId}` |
| Account | `GET /usage`, `GET /me`, `GET /audit`, `DELETE /audit` |
| Reference | `GET /reference/courts`, `GET /reference/case-types` |
| Coverage | `GET /coverage` |
| System | `GET /health`, `GET /health?deep=1` |

164 saved example responses covering the success paths and the realistic failure paths: validation errors, 404s, rate limits, insufficient credits, tier gates (`API_TIER_NOT_ALLOWED`, `SEMANTIC_NOT_ALLOWED`, `LIVE_FETCH_NOT_ALLOWED`, `REMOTE_FETCH_NOT_ALLOWED`), a degraded upstream search (`PARTY_SCREEN_SEARCH_DEGRADED`), an unavailable entitlement lookup (503), a batch party screen mixing successful and failed items, and a full mode party profile from the 202 answer through running, completed and failed.

## Import and run it locally

1. Open Postman, click **Import**, and select `CourtMesh.postman_collection.json`.
2. Create an environment (top right, **Environments**, then **Create**) with a single variable:
   - `apiKey` set to your key, for example `cm-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XXXX`
3. Select that environment, then send **System / Health**. It needs no key, so it confirms the base URL is reachable.
4. Send **Judges / Search judges**. It is the cheapest authenticated call, so it confirms your key works.
5. Send **Search / Search cases (keyword)**, copy an `id` from the results into the `caseId` collection variable, then work through the Cases and Analysis folders.

Collection variables you may want to set: `baseUrl`, `apiKey`, `caseId`, `requestId`, `profileRequestId`, `userId`, `matterId`, `courtKey`, and for the Acts and Sections folder `actSlug` and `sectionSlug`. Keep `apiKey` in an environment rather than in the collection so it never lands in source control.

Auth is configured once at the collection level as a bearer token bound to `{{apiKey}}`, so every request inherits it. `GET /health`, `GET /health?deep=1`, `GET /reference/courts` and `GET /reference/case-types` override that with no auth. The API also accepts `X-API-Key: <key>` if you prefer that header.

### Idempotency-Key

`POST /party/screen`, `POST /party/screen/batch`, `POST /party/profile`, `POST /cases/{id}/analyze`, `POST /cases/{id}/analyze-consolidated` and `POST /request-timeline` each carry an `Idempotency-Key` header, as does `POST /cause-lists/check`, defaulted to `{{$guid}}`, Postman's dynamic variable for a fresh UUID v4 on every send. A replayed request (same key, same body, within 24 hours) returns the original response again with no new charge (`Idempotency-Replayed: true` on the response); the same key with a different body is refused with 409 `IDEMPOTENCY_KEY_REUSED`. Because `{{$guid}}` regenerates every send, re-sending one of these requests from the Postman UI is always a fresh, separately charged call; paste a fixed value into the header to test the replay behaviour on purpose.

## Run it from the command line

```bash
npm install -g newman
newman run CourtMesh.postman_collection.json \
  --env-var apiKey=cm-your-key-here \
  --folder System
```

Note the rate limit before you run the whole collection: 10 requests per minute per API key. Add `--delay-request 7000` for a full run.

## Endpoint notes worth reading before you build

These come from the API's actual behaviour, not from an idealised spec.

- **Response envelope.** Everything except `GET /health` returns `{ "success": true, "data": ..., "meta": ..., "pagination": ... }`. Failures return `{ "success": false, "error": "..." }`, plus a `details` array when request validation failed. Authentication and rate limit failures come from middleware and are a bare `{ "error": "..." }` with no `success` key. Every response, success or failure, also carries an `X-Request-Id` response header; some endpoints (`GET /usage`, `POST /party/screen/batch`) echo it in `meta.requestId` too.
- **Two pagination shapes.** `POST /search/cases` returns `{ total, hasMore, page, limit, nextCursor }` with no `totalPages`, and `page` disappears once you paginate by `cursor` (or the legacy `searchAfter`). `POST /search/cases/semantic` returns `{ page, limit, total, totalPages, hasMore }` where `total` and `totalPages` are estimates that only become exact on the last page.
- **`caseNumber` on `POST /search/cases` does filter results now,** to a single value only: send one string, or a one element array (a second array element is rejected with 400, never silently dropped). It is echoed back in `meta.filters`.
- **`sortBy` on `POST /search/cases`** accepts `relevance`, `recent` or `oldest`, all real sort orders; `date` is still accepted too, as a deprecated alias applied as `recent` (the response's `meta` notes the substitution).
- **Pagination cursors on `POST /search/cases`.** `cursor` is the current mechanism: an opaque signed string, echoed as `pagination.nextCursor`, bound to the exact query and filters it was issued for. Sending a cursor for a different query, or a malformed one, is rejected with 400 `CURSOR_INVALID`. `searchAfter` is the older mechanism; prefer `cursor` for new integrations. Each plan tier also caps `limit` and total pagination depth (Free tier: `limit` up to 20); going over either returns 400 `PAGE_LIMIT_EXCEEDED` or `PAGINATION_DEPTH_EXCEEDED`.
- **The semantic endpoint's top level filters are real.** `court`, `year`, `caseType`, `caseNumber`, `judgeName` (and aliases `judges`/`judge`), `fromDate` and `toDate` are all mapped onto the AI search's filter keys and echoed back in `meta.appliedFilters`. The nested `filters` object addresses the AI search's own filter keys directly and takes precedence when the same key is set in both places. Not available on the Free tier at all (403 `SEMANTIC_NOT_ALLOWED`).
- **`GET /cases/{id}/pdf` returns ciphertext.** `data.pdfUrl` is an encrypted, time limited link that expires in one hour, not a URL you can fetch directly. Its two distinct 404s are `CASE_NOT_FOUND` (no such case) and `PDF_NOT_STORED` (case exists, no stored document, with a `hint` pointing at `POST /request-timeline` with `refresh: true`).
- **`POST /cases/{id}/analyze` needs `allowRemoteFetch: true`** before it will fetch a case's source document from a remote court host that is not already stored; without it, that case returns 403 `REMOTE_FETCH_NOT_ALLOWED`. A remote fetch adds a 20 credit surcharge on top of the base 100.
- **`POST /request-timeline` defaults to a stored read** (1 credit, `meta.liveFetch: false`). Send `refresh: true` for a live court fetch instead (also 1 credit, `meta.liveFetch: true`, PAYG tier or above only, 403 `LIVE_FETCH_NOT_ALLOWED` on Free, 429 `LIVE_FETCH_LIMIT_REACHED` once the tier's daily cap is exhausted).
- **Monitoring is free to set up and run.** Watchlist and webhook management, the daily check of every watched CNR and webhook delivery cost 0 credits; reading `GET /changes` or `GET /watchlists/{id}/events` costs 1 credit per non-empty page of up to 100 events (an empty page is free). `GET /cases/by-cnr/{cnr}` costs 1 credit when a record is returned. Webhook deliveries are signed: verify `CourtMesh-Signature` with the Node or Python SDK helper. A daily add allowance per account applies on top of the watched CNR cap (Free 50, Pay As You Go 5,000, Scale 50,000, Enterprise 500,000; items over it come back `cap_exceeded`). A case taken down at a party's request shows as an `unavailable` item and sends no events. If the court portal had nothing for a CNR in the last 24 hours it is not asked again: adds come back `not_found` and `GET /cases/by-cnr/{cnr}?fetch=true` answers 404 `CASE_NOT_FOUND`.
- **Acts and Sections.** Nine `GET` requests, each 1 credit when something is returned and free otherwise, for API keys with the `legislation` scope. Slugs are lower case letters, digits and hyphens; get them from `GET /acts/resolve` or `GET /sections/lookup`. Every response carries repeal status, so check it before relying on a provision. `GET /acts/{slug}/history` is at act level and does not map old sections to new ones. The citing judgments list is not exhaustive, some matters are never listed, a page can be short or empty while `hasMore` is true (keep following `nextCursor`), and an empty list does not mean no judgment applied the provision. Search, resolve, lookup and citing list requests have a lower per minute limit and answer 429 above it; a failed check on the citing list is 502 `UPSTREAM_UNAVAILABLE`, safe to retry once. New error codes: `ACT_NOT_FOUND`, `SECTION_NOT_FOUND`.
- **Cause lists need the `causelist` API key scope** (keys without explicit scopes have full access) and an account with cause lists enabled (403 `CAUSE_LISTS_NOT_ENABLED` otherwise). `GET /cause-lists/listings` and `GET /matters/{matterId}/cause-listings` cost 1 credit per successful call; `GET /cause-lists/court` costs 1 credit only when `status` is `ok` or `partial` and 0 for the other statuses. `POST /cause-lists/check` takes 1 to 100 CNRs you watch and costs 1 credit per CNR with status `listed` or `no_entry_found`; `list_not_read`, `cannot_check` and `not_watched` are free. It never contacts a court. Only listings that were found are reported: `no_entry_found` and an empty result do not show that a case is absent from a list. A temporary outage is 502 `UPSTREAM_UNAVAILABLE` and is not charged. The one exception to "failed calls are not charged" is a 503 `SETTLEMENT_UNCONFIRMED` on the check: the answer was ready but the charge outcome is unknown and no results were returned, so check `GET /usage` before retrying (repeating the same call with the same `Idempotency-Key` is refused with 409 `IDEMPOTENCY_IN_PROGRESS` for up to 24 hours). A 503 `ENTITLEMENT_UNAVAILABLE` ran nothing and is safe to retry.
- **Path ids are flexible.** Every `{id}` accepts a 24 character hexadecimal case id or a case number. `POST /request-timeline` is the exception: its `case_id` must be a 24 character hexadecimal case id.
- **Rate limit.** 10 requests per minute per API key by default (this varies per plan and per endpoint). 429 bodies carry `retryAfter` in seconds and `resetTime` as an ISO timestamp.
- **`POST /party/screen` requires `purpose`.** It is the DPDP lawful basis recorded against the request, not an optional label. A match in the response is a case record bearing the screened name, not a verified identity, check `confidence` (`{ band, score, calibrated, engine }`, band one of `confirmed`/`probable`/`possible`/`unlikely`) and `evidence` (`signals` is an array of `{ name, status, weight, evidence }` objects) on each match. `data.summary.verdict` is `no_matches_found` only when `data.coverage.exhaustive` is true and nothing was withheld; sending `since` always forces `inconclusive`, and `matchCount` can read 0 while `verdict` still reads `matches_found` when `displayThreshold` hides everything found. The top level `notice` field carries the case removal policy text (there is no `coverage.note`). Costs 100 credits with matches, 20 credits with none, plus 80 credits when `adjudicate` is true AND at least one candidate was actually sent to the model (`adjudicationsRun > 0`); a 402 response carries `required`, `balance`, `shortfall`, `wallet`, `walletOwner` and either `topUpUrl` or `contactAdmin`. On the Free tier, `adjudicate: true` is a hard 403 `API_TIER_NOT_ALLOWED`, not a silent no-op.
- **`POST /party/profile` builds a litigation profile of one person or company** on top of the same screen, grouped into identities (an individual is never confirmed on name alone), not available on the Free tier. Quick mode answers in one call and is metered: at least 100 credits even when nothing is found, never more than 304, itemised in `meta.creditsBreakdown`. Full mode (the default where available) answers 202 with a `requestId`, refreshes every case of the first identity from its court, analyses each case and the party, and is charged by actual usage up to `maxCredits` (100 to 5000, default 1000); read it free from `GET /party/profile/{requestId}` or pass a `webhook`. At most 3 full profiles may run at once per account (429 `PARTY_PROFILE_TOO_MANY_ACTIVE`), and a 503 `PARTY_PROFILE_FULL_UNAVAILABLE` means send `mode: "quick"` instead.
- **New optional fields on existing responses.** Case detail can carry `bench` (tribunals), `courtRecordActs` and `courtRecordFir` (district cases), as the court record states them. Supreme Court search and screen hits carry `sameMatter`, the other orders of the same matter. `POST /search/cases` with `cnr` for a case not held yet returns an empty list plus a top level `cnrLookup` with the court check's status. `GET /cases/{id}/orders` rows without text carry `unavailableReason`, and a case whose text is not offered carries `ordersTextWithheld: true` with `withheldReason`. A change feed event for a case removed later keeps its place with `withheld: "case_restricted"` and empty `data`.
- **`GET /coverage` needs no key** and is cached server side for several hours, see `meta.cacheTtlSeconds`. Each row's count is `records`, not `total`.
- **`POST /party/screen/batch` screens 1 to 25 names in one call,** not available on the Free tier. Each item is independently priced and can independently fail: `data.results[i]` is either `{ ok: true, screen: <same shape as POST /party/screen's data> }` or `{ ok: false, error: { code, message } }`, so check `ok` per item rather than treating the whole call as one pass/fail. `adjudicate` is not supported at the batch level (only `false`/omitted); use `POST /party/screen` directly for adjudication.
- **`GET /usage`, `GET /me`, `GET /audit` and `DELETE /audit` are unmetered account calls.** `GET /usage` reports tier, wallet balance, the full per tier limits object and per endpoint call volume for the current Asia/Kolkata month. `GET /me` returns your own userId/organizationId, useful before calling `GET /audit` with `userId` or `organizationId` (you may only read your own data, or your organization's as an org admin). `DELETE /audit?before=<date and time>` deletes the calling key's own logged calls before that time and cannot be undone; other keys on the account are not touched, and charges stay on invoices, balance and usage totals. It needs the `account` scope, and when `data.complete` is false you send the same request again to finish.
- **`GET /reference/courts` and `GET /reference/case-types` need no key.** They serve the exact same taxonomy the `court` and `caseType` filters on the search and screen endpoints accept, cached for 1 hour server side (`ETag`/`Cache-Control`).
- **`Idempotency-Key`** on the six job/charge-triggering POST endpoints (`POST /party/screen`, `POST /party/screen/batch`, `POST /party/profile`, `POST /cases/{id}/analyze`, `POST /cases/{id}/analyze-consolidated`, `POST /request-timeline`) makes a retried call with the same key and body a no-op replay (`Idempotency-Replayed: true`, no new charge) instead of running the job or charging twice; the same key with a different body is a 409 `IDEMPOTENCY_KEY_REUSED`. See the Idempotency-Key section above.

## Official SDKs

If you would rather not hand roll HTTP calls:

- Python: `pip install courtmesh`
- Node and TypeScript: `npm install @courtmesh/sdk`

## Licence

MIT. Copyright Thinkscoop Technologies LLP.
