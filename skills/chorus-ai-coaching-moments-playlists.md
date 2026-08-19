---
name: chorus-ai-coaching-moments-playlists
description: >-
  Build sales coaching material in Chorus by ZoomInfo — clip notable segments of a call as
  moments, assemble them into playlists, and pull scorecards and team rosters. Use when an agent
  is producing enablement content or reviewing rep performance.
api: Chorus API
base_url: https://chorus.ai
generated: '2026-08-13'
method: generated
source: openapi/_original/chorus-ai-openapi.json
operations:
  - get-api-v1-moments
  - post-api-v1-moments
  - put-api-v1-moments-id
  - delete-api-v1-moments-id
  - get-api-v1-playlists
  - post-api-v1-playlists
  - post-api-v1-playlists-moments
  - patch-api-v1-playlists-moments-id
  - post-api-v1-smart_playlists
  - get-api-v1-scorecards
  - get-api-v1-teams
  - get-v3-users
---

# Build coaching moments and playlists in Chorus

This skill lives entirely on the `/api/v1` **JSON:API** surface. Every request and response uses
`Content-Type: application/vnd.api+json` and the `data` / `type` / `attributes` document shape.
Getting the `type` string exactly right matters — Chorus enumerates it per endpoint.

## Authentication

```
Authorization: <CHORUS_API_TOKEN>
```

Raw token, no `Bearer` prefix. Coaching operations respect the same role and data-access rules as
the rest of the API — a user who cannot see a call cannot clip it.

## Step 1 — Find the people (`get-v3-users`, `get-api-v1-teams`)

```
GET https://chorus.ai/v3/users
GET https://chorus.ai/api/v1/teams
```

Users carry `manager_id` (self-referential hierarchy) and `team_ids`, so you can build a coaching
org chart from these two calls. `GET /api/v1/users/me` returns the token user.

## Step 2 — Create a moment (`post-api-v1-moments`)

A moment is a timestamped clip of a conversation.

```
POST https://chorus.ai/api/v1/moments
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "moment_create_request",
    "attributes": {
      "callid": "<conversation id>",
      "moment_time": 512,
      "moment_duration": 45,
      "subject": "Objection: pricing vs incumbent",
      "note": "Rep handled the discount ask without conceding.",
      "expiration_date": "2027-01-01T00:00:00Z",
      "password": null
    }
  }
}
```

- Required attributes: **`callid`, `moment_time`, `moment_duration`**.
- `moment_time` and `moment_duration` are numbers in seconds — offset into the call and clip
  length.
- `type` must be exactly `moment_create_request`.
- `expiration_date` and `password` control external sharing. If you are creating a shareable
  link for anyone outside the company, set both deliberately — an unset expiration means the
  clip stays reachable indefinitely.

List moments with `GET /api/v1/moments`, which **requires** the `filter[shared_on]` query
parameter. Update with `PUT /api/v1/moments/{id}`, remove with `DELETE /api/v1/moments/{id}`.

## Step 3 — Create a playlist (`post-api-v1-playlists`)

```
POST https://chorus.ai/api/v1/playlists
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "playlist",
    "attributes": {
      "name": "Q3 objection handling",
      "description": "Best examples of pricing objection handling.",
      "parent": null
    }
  }
}
```

- Required attribute: **`name`**. `type` must be exactly `playlist`.
- `parent` is an integer playlist id — playlists nest.

Browse existing playlists with `GET /api/v1/playlists`, which is the richest read in this skill:

`filter[name]`, `filter[type]`, `filter[conversation]`, `filter[parent]`, `filter[subscribed]`,
`top_level`, `recursive`, `include_parent`, `sort`, `page[number]`, `page[size]`.

Note this surface uses **`page[number]`/`page[size]`**, unlike `/v3/engagements`, which uses a
`continuation_key` cursor. Do not mix the two grammars.

## Step 4 — Add a moment to a playlist (`post-api-v1-playlists-moments`)

```
POST https://chorus.ai/api/v1/playlists/moments
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "playlist_moment_request",
    "attributes": {
      "playlist": 1234,
      "callid": "<conversation id>",
      "moment_duration": 45,
      "moment_time": 512,
      "description": "Objection handling example"
    }
  }
}
```

- Required attributes: **`callid`, `moment_duration`, `playlist`**.
- `playlist` is an **integer** id; `callid` is the conversation id.
- `type` must be exactly `playlist_moment_request`.

Adjust position or detail with `PATCH /api/v1/playlists/moments/{id}`.

## Step 5 — Smart playlists (`post-api-v1-smart_playlists`)

For a rule-driven playlist that populates itself, use `POST /api/v1/smart_playlists`. Smart
playlists scope by `user_ids`, `team_ids` and `account_ids` rather than by explicit moments.
Update with `PUT /api/v1/smart_playlists/{id}`.

## Step 6 — Scorecards (`get-api-v1-scorecards`)

```
GET https://chorus.ai/api/v1/scorecards
```

Filters: `filter[recipients]`, `filter[reviewers]`, `filter[initiative]`, `filter[submitted]`,
plus `page[number]` / `page[size]`.

Export asynchronously with `POST /api/v1/scorecards:export`, which answers **202** with a task
handle.

## Error handling

JSON:API envelope, not RFC 9457. The most common failures in this flow:

- **400 Bad request** — almost always a wrong `type` string or a missing required attribute.
  Read `errors[].source.pointer`; it points at the offending member of the document.
- **404 Not found** — bad `callid`/`playlist` id, *or* the token user cannot see that
  conversation. Check permissions before assuming the id is wrong.
- **409 Conflict** — the moment or playlist entry already exists, or a concurrent edit collided.

## Idempotency and retries

Chorus supports **no** `Idempotency-Key`. Creating a moment or playlist entry is not replay-safe:
a retried `POST` after a timeout produces a second clip. Before retrying a create, read back with
`GET /api/v1/moments?filter[shared_on]=...` or `GET /api/v1/playlists` and check whether the
first attempt actually landed.

## Rate limits

None documented — no `429`, no rate-limit headers, no `Retry-After`. Keep concurrency modest and
back off exponentially with jitter on unexpected errors.
