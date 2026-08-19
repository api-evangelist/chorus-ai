---
name: chorus-ai-retrieve-conversations
description: >-
  Search, retrieve and download recorded sales calls and meetings from Chorus by ZoomInfo —
  filter engagements by date, duration, participant, owner or team, fetch a single conversation
  with sparse fieldsets, and resolve its media file. Use when an agent needs conversation
  intelligence data out of Chorus.
api: Chorus API
base_url: https://chorus.ai
generated: '2026-08-13'
method: generated
source: openapi/_original/chorus-ai-openapi.json
operations:
  - get-v3-engagements
  - get-api-v1-conversations-id
  - get-api-v1-conversations-id-media
  - get-api-v1-conversations-live
  - post-conversations-export
---

# Retrieve conversations from Chorus

## Authentication

Every request carries a Chorus API token in the `Authorization` header. Send the **raw token
with no `Bearer` prefix** — this is what the provider's own example demonstrates, even though
the OpenAPI declares an `http/bearer` scheme:

```
Authorization: <CHORUS_API_TOKEN>
```

Tokens are minted per user from Personal Settings inside the Chorus application, and the user's
role must permit API access. There is no self-serve API signup; during the documented early
access period a token is obtained through a Chorus customer success manager.

**Permission model matters for correctness.** The token inherits its user's data access:
recordings marked private are never returned, and if data-access-control is enabled the user
must have access to a recording for the API to return it. A `404` may therefore mean "not
permitted" rather than "does not exist" — never treat a 404 here as proof a conversation is gone.

## Step 1 — Search engagements (`get-v3-engagements`)

```
GET https://chorus.ai/v3/engagements
```

All parameters are optional query parameters:

| Parameter | Purpose |
|---|---|
| `min_date` / `max_date` | Bound the engagement date range |
| `min_duration` / `max_duration` | Bound meeting duration, in **seconds** |
| `engagement_type` | Type of engagement |
| `content_type` | Type of content |
| `engagement_id` | Comma-separated list of engagement ids |
| `user_id` | Comma-separated owner user id(s) |
| `team_id` | Comma-separated owner team id(s) |
| `participants_email` | A participant's email address |
| `compliance` | Call-recording compliance flag |
| `disposition_connected`, `disposition_voicemail`, `disposition_gatekeeper`, `disposition_tree` | Chorus call dispositions |
| `with_trackers` | Return tracker information with results |
| `continuation_key` | Pagination cursor |
| `additional_parameters` | JSON object escape hatch for extra params |

**Pagination on this surface is a continuation cursor, not page numbers.** Read
`continuation_key` from the response and pass it back on the next call until the API stops
returning one. Do not send `page[number]`/`page[size]` here — those belong to the `/api/v1`
surface, which uses a different grammar.

## Step 2 — Fetch one conversation (`get-api-v1-conversations-id`)

```
GET https://chorus.ai/api/v1/conversations/{id}
```

This is the JSON:API surface — responses are `application/vnd.api+json` with a
`data`/`attributes` document.

Useful query parameters:

- `fields`, `fields[conversations]`, `fields[recordings]` — JSON:API sparse fieldsets. Request
  only what you need; the full conversation document is large.
- `include_meeting_metadata` — attach meeting metadata.
- `force_regeneration` — force regeneration of derived output.
- `skip_summary_generation` — skip AI summary generation.

Be deliberate with the last two: `force_regeneration` causes real work server-side, and
`skip_summary_generation` changes what you get back. Prefer the defaults unless you have a
reason.

## Step 3 — Resolve media (`get-api-v1-conversations-id-media`)

```
GET https://chorus.ai/api/v1/conversations/{id}/media
```

Pass `info` to request metadata instead of the file itself.

**Expect a redirect.** Media and export downloads answer `302`/`307` to a signed storage URL
rather than streaming inline. Follow redirects, and do not log or persist the signed URL — treat
it as a short-lived credential.

## Step 4 — Live calls (`get-api-v1-conversations-live`)

```
GET https://chorus.ai/api/v1/conversations/live
```

Fetches an in-progress conversation by meeting uuid or user id. Use this only for real-time
scenarios; for historical analysis use `get-v3-engagements`.

## Step 5 — Bulk export (`post-conversations-export`)

For large pulls, do not paginate the search endpoint — use the export:

```
POST https://chorus.ai/api/v1/conversations:export
Content-Type: application/json

{
  "start": "<ISO datetime>",
  "end": "<ISO datetime>",
  "callback_url": "https://your-endpoint.example.com/chorus-export",
  "email_on_complete": true,
  "add_usage": false
}
```

This is asynchronous — it answers `202 Accepted` with a task handle. Poll or supply
`callback_url`. Do not block an agent turn waiting on it.

## Error handling

Errors are **JSON:API**, not RFC 9457 problem+json:

```json
{"errors":[{"id":"...","code":"...","status":"404","title":"Not found",
            "detail":"...","source":{"pointer":"...","parameter":"..."}}]}
```

Only three failures are declared across the contract:

- **400 Bad request** — read `errors[].source.parameter` / `source.pointer` to find the offending
  input, fix it, retry. Not retryable unchanged.
- **404 Not found** — the id is wrong *or* the token user cannot see it. Check permissions before
  concluding absence.
- **409 Conflict** — server state conflicts with the request.

`errors[].code` is present but Chorus publishes no code registry, so branch on HTTP status, not
on `code`.

**Undeclared but real:** the contract declares no `401`, `403`, `429` or `5xx` on any operation,
yet the live API returns `401` for a missing token. Handle those defensively.

## Rate limits

Chorus documents none — no `429` in the contract, no `X-RateLimit-*` / `RateLimit-*` /
`Retry-After` headers. There is no runtime signal to read, so throttle yourself: keep
concurrency low, add exponential backoff with jitter on any unexpected failure, and prefer the
bulk export over tight pagination loops.

## Idempotency

Not supported. No `Idempotency-Key` header exists. The reads in this skill are naturally safe to
retry; be careful with the export in step 5, since a retried request may start a second job.
