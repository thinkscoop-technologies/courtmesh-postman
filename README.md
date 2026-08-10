# CourtMesh API Postman Collection

A Postman Collection v2.1 for the CourtMesh Indian court cases API. All 12 production endpoints, organised into six folders, with request bodies, saved example responses and per request documentation.

Base URL: `https://research.courtmesh.ai/api/v1/prod`

File: [`CourtMesh.postman_collection.json`](./CourtMesh.postman_collection.json)

## What is in it

| folder | requests |
| --- | --- |
| Search | `POST /search/cases`, `POST /search/cases/semantic` |
| Cases | `GET /cases/{id}`, `GET /cases/{id}/related`, `GET /cases/{id}/pdf` |
| Analysis | `GET /cases/{id}/analysis`, `POST /cases/{id}/analyze`, `POST /cases/{id}/analyze-consolidated` |
| Judges | `GET /judges/search` |
| Timeline | `POST /request-timeline`, `GET /get-timeline/{requestId}` |
| System | `GET /health` |

30 saved example responses covering the success paths and the realistic failure paths: validation errors, 404s, rate limits, and the semantic endpoint's habit of returning errors with a 200 status.

## Import and run it locally

1. Open Postman, click **Import**, and select `CourtMesh.postman_collection.json`.
2. Create an environment (top right, **Environments**, then **Create**) with a single variable:
   - `apiKey` set to your key, for example `cm-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XXXX`
3. Select that environment, then send **System / Health**. It needs no key, so it confirms the base URL is reachable.
4. Send **Judges / Search judges**. It is the cheapest authenticated call, so it confirms your key works.
5. Send **Search / Search cases (keyword)**, copy an `_id` from the results into the `caseId` collection variable, then work through the Cases and Analysis folders.

Collection variables you may want to set: `baseUrl`, `apiKey`, `caseId`, `requestId`. Keep `apiKey` in an environment rather than in the collection so it never lands in source control.

Auth is configured once at the collection level as a bearer token bound to `{{apiKey}}`, so every request inherits it. `GET /health` overrides that with no auth. The API also accepts `X-API-Key: <key>` if you prefer that header.

## Run it from the command line

```bash
npm install -g newman
newman run CourtMesh.postman_collection.json \
  --env-var apiKey=cm-your-key-here \
  --folder System
```

Note the rate limit before you run the whole collection: 10 requests per minute per API key. Add `--delay-request 7000` for a full run.

## Publishing this as public Postman documentation

The public documenter page is itself the asset here. It is indexed by search engines, it ranks for developer queries, and it links back to the API. Publish it once and keep it updated.

Do this in the Postman web or desktop app while signed into the CourtMesh team workspace.

1. **Import into a team workspace.** Personal workspaces cannot publish public docs. Open the CourtMesh team workspace first, then **Import** the JSON file.
2. **Move the collection to the team workspace** if it landed anywhere else. Use the collection's **...** menu, then **Move**.
3. **Check the collection description renders.** Click the collection name, then the documentation icon on the right. The overview you see there is what visitors read first. It carries the auth instructions, the envelope shape, the rate limit and the endpoint table.
4. **Publish.** Collection **...** menu, then **View documentation**, then the **Publish** button at the top right. This opens the publishing settings page in a browser.
5. **Configure the published page:**
   - Environment: choose **No environment**. Never publish an environment that contains a real key.
   - Custom domain: leave as the default `documenter.getpostman.com` unless the team has a verified domain configured.
   - Styling: set the brand colours and the CourtMesh logo.
   - Optional: enable **Run in Postman** button and the language snippets for cURL, Python requests, Node fetch, Go and PHP.
6. **Click Publish Collection.** You get a public URL of the form `https://documenter.getpostman.com/view/<userId>/<collectionSlug>/<versionTag>`.
7. **Verify the published page** in a private browsing window while signed out. Confirm no key is visible anywhere, that all six folders appear, and that the saved examples render.
8. **Submit the URL for indexing.** Add it to Google Search Console, link to it from the CourtMesh docs and website footer, and use it as the documentation link in every developer directory listing.

To publish an update later: re import or edit the collection, then **View documentation**, then **Publish** again. The URL stays the same as long as you republish the same collection.

## Publishing this repository to GitHub

Nothing here has been pushed anywhere. To publish it, run these exact commands from this directory:

```bash
gh repo create courtmesh/courtmesh-postman --public \
  --description "Postman collection for the CourtMesh Indian court cases API" \
  --source . --remote origin --push
```

Or without the GitHub CLI:

```bash
git remote add origin git@github.com:courtmesh/courtmesh-postman.git
git branch -M main
git push -u origin main
```

## Endpoint notes worth reading before you build

These come from the API's actual behaviour, not from an idealised spec.

- **Response envelope.** Everything except `GET /health` returns `{ "success": true, "data": ..., "meta": ..., "pagination": ... }`. Failures return `{ "success": false, "error": "..." }`, plus a `details` array when request validation failed. Authentication and rate limit failures come from middleware and are a bare `{ "error": "..." }` with no `success` key.
- **Two pagination shapes.** `POST /search/cases` returns `{ total, hasMore, page, limit, nextCursor }` with no `totalPages`, and `page` disappears once you paginate by cursor. `POST /search/cases/semantic` returns `{ page, limit, total, totalPages, hasMore }` where `total` and `totalPages` are estimates that only become exact on the last page.
- **`caseNumber` on `POST /search/cases` does not filter.** It passes validation and is echoed back in `meta.filters`, but the handler never forwards it to the search backend. Put the case number in `query` instead.
- **`sortBy` on `POST /search/cases`** is validated as `relevance` or `date`, while the search backend expects `relevance`, `recent` or `oldest`. Only `relevance` is meaningful today.
- **The semantic endpoint ignores top level filters.** `court`, `year`, `caseType`, `judgeName` and the date fields pass validation and are then dropped. Put filters inside the `filters` object.
- **The semantic endpoint can fail with HTTP 200.** It writes the status line before it starts work, so a downstream failure arrives as `200` with `{ "success": false, "error": "..." }`. Check `success`, not the status code.
- **`GET /cases/{id}/pdf` returns ciphertext.** `data.pdfUrl` is an encrypted wrapper around a presigned S3 link that expires in one hour, not a URL you can fetch directly.
- **Path ids are flexible.** Every `{id}` accepts a Mongo ObjectId string or a case number. `POST /request-timeline` is the exception: its `case_id` must be an ObjectId.
- **Rate limit.** 10 requests per minute per API key. 429 bodies carry `retryAfter` in seconds and `resetTime` as an ISO timestamp.

## Official SDKs

If you would rather not hand roll HTTP calls:

- Python: `pip install courtmesh`
- Node and TypeScript: `npm install @courtmesh/sdk`

## Licence

MIT. Copyright Thinkscoop Technologies LLP.
