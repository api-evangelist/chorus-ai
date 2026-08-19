---
name: chorus-ai-upload-recording
description: >-
  Upload an external call or meeting recording into Chorus by ZoomInfo for transcription and AI
  analysis, and subscribe to the recording_done webhook so you know when processing has finished.
  Use when conversation media originates outside Chorus and must be ingested.
api: Chorus API
base_url: https://chorus.ai
generated: '2026-08-13'
method: generated
source: openapi/_original/chorus-ai-openapi.json
operations:
  - post-v3-upload
  - post-v3-webhook
  - get-v3-webhook
  - delete-v3-webhook
  - post-api-v1-conversations:bulk
  - post-api-v1-conversations:validate
---

# Upload a recording into Chorus

## Authentication

```
Authorization: <CHORUS_API_TOKEN>
```

Raw token, no `Bearer` prefix. The token's user must have a role permitting API access, and the
uploaded recording is attributed to the `user` you name in the request.

## Step 1 — Upload the media (`post-v3-upload`)

```
POST https://chorus.ai/v3/upload
Content-Type: multipart/form-data
```

Form fields:

| Field | Required | Purpose |
|---|---|---|
| `data` | **yes** | The media file |
| `name` | **yes** | Name of the conversation |
| `user` | **yes** | The Chorus user the recording is attributed to |
| `meeting_id` | no | External meeting identifier |
| `enforce_unique_meeting_id` | no | Reject the upload if `meeting_id` already exists |
| `crm_account_id` | no | CRM account to associate |
| `crm_opportunity_id` | no | CRM opportunity to associate |

Supported formats, verbatim from the contract: **3GPP, AU, AVI, FLV, HLS, MKV, MP3, MP4, Ogg,
WAV, WebM**.

### Duplicate protection — read this before retrying

**Chorus supports no idempotency key.** There is no `Idempotency-Key` header anywhere in the
API, so a retried upload after a timeout can create a second copy of the same call.

The only guard the API gives you is `enforce_unique_meeting_id`. **Always send a stable
`meeting_id` and set `enforce_unique_meeting_id` to true.** That turns a duplicate retry into a
`409 Conflict` you can safely ignore, instead of a duplicate conversation you have to find and
delete.

```
meeting_id=<your-stable-external-id>
enforce_unique_meeting_id=true
```

If you cannot supply a stable id, treat a timed-out upload as *unknown*, not as *failed* — search
`get-v3-engagements` for the recording before retrying.

## Step 2 — Know when processing finishes (`post-v3-webhook`)

Transcription and AI analysis are asynchronous. Rather than polling, register a callback:

```
POST https://chorus.ai/v3/webhook
Content-Type: application/json

{
  "event": "recording_done",
  "hook_url": "https://your-endpoint.example.com/chorus"
}
```

`recording_done` is the **only** event Chorus enumerates. An optional `type` field selects
presets with predefined filters (e.g. `"zapier"`).

List your registrations with `GET /v3/webhook`. Remove one with `DELETE /v3/webhook`, which is
keyed on the **URL**, not an id:

```
DELETE https://chorus.ai/v3/webhook
Content-Type: application/json

{"hook_url": "https://your-endpoint.example.com/chorus"}
```

### Receiver warnings

- **The callback payload is undocumented.** Chorus specifies the registration request but never
  describes the body it POSTs to `hook_url`. Write your receiver defensively: accept any JSON,
  log the first real delivery, and derive your parser from what actually arrives.
- **There is no signing mechanism.** No signature header, shared secret or verification
  procedure is published, so you cannot cryptographically prove a callback came from Chorus. Use
  an unguessable `hook_url` path, require HTTPS, and re-fetch the conversation through the API
  before trusting anything in the payload.
- **No retry or ordering guarantees are documented.** Make your handler idempotent on your side.

## Step 3 — Bulk ingest (`post-api-v1-conversations:bulk`)

For many conversations at once, use the bulk endpoint on the JSON:API surface:

```
POST https://chorus.ai/api/v1/conversations:bulk
Content-Type: application/vnd.api+json
```

The request accepts a `create` array of conversation objects with `attributes` including
`activity_type`, `content`, `external_id`, `related_external_id`, and a `crm` block
(`account_name`, `opportunity_id`, …), plus ZoomInfo graph keys `zi_person_id` / `zi_company_id`.

Validate first where you can:

```
POST https://chorus.ai/api/v1/conversations:validate
```

Bulk operations answer **202 Accepted** with a task handle — they are asynchronous. Supply
`external_id` values so you can reconcile results, and remember that without idempotency a
retried bulk create is a duplicate risk on every row.

## Error handling

JSON:API envelope, not RFC 9457:

- **400** — validation failure; `errors[].source.pointer` names the bad field.
- **409** — conflict. On upload with `enforce_unique_meeting_id=true`, this is the *expected,
  benign* outcome of a duplicate retry. Treat it as success-already-recorded.
- **404** — resource missing or not visible to the token user.

`401`/`403`/`429`/`5xx` are not declared in the contract but can occur; handle them.

## Rate limits

Undocumented — no `429` declared, no rate-limit or `Retry-After` headers published. Media
uploads are heavy: upload serially, cap concurrency, and back off exponentially with jitter on
any unexpected error.
