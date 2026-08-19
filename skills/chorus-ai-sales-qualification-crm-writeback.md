---
name: chorus-ai-sales-qualification-crm-writeback
description: >-
  Run Chorus by ZoomInfo AI sales-qualification analysis on a recorded call, read the framework
  fields it extracts, and write the results back to a CRM opportunity. Use when an agent must
  turn call content into structured deal data in Salesforce or another CRM.
api: Chorus API
base_url: https://chorus.ai
generated: '2026-08-13'
method: generated
source: openapi/_original/chorus-ai-openapi.json
operations:
  - get-api-v1-sales_qualification_configuration
  - put-api-v1-sales_qualification_configuration-framework_id
  - post-api-v1-sales-qualifications
  - get-api-v1-sales-qualifications-recording_id
  - post-api-v1-sales-qualifications-actions-writeback-crm
---

# Sales qualification analysis and CRM writeback

This is the highest-consequence flow in the Chorus API: its final step **mutates records in your
CRM**. Read the guardrails at the bottom before wiring it into an autonomous agent.

## Authentication

```
Authorization: <CHORUS_API_TOKEN>
```

Raw token, no `Bearer` prefix. All requests and responses on this flow use
`application/vnd.api+json`.

## Step 1 — Read the framework configuration (`get-api-v1-sales_qualification_configuration`)

```
GET https://chorus.ai/api/v1/sales_qualification_configuration
```

Returns the tenant's qualification framework — the field set the AI will populate. **Always read
this first.** The framework is per-tenant configuration, so field names are not universal; an
agent that hardcodes them will silently write the wrong things.

To change the configuration:

```
PUT https://chorus.ai/api/v1/sales_qualification_configuration/{framework_id}
```

Treat that as an administrative action, not something an agent does mid-flow.

## Step 2 — Request analysis of a recording (`post-api-v1-sales-qualifications`)

```
POST https://chorus.ai/api/v1/sales-qualifications
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "sales_qualifications",
    "attributes": {
      "recording_id": "<recording id>"
    }
  }
}
```

- Required attribute: **`recording_id`**. `type` must be exactly `sales_qualifications`.

## Step 3 — Read the analysis (`get-api-v1-sales-qualifications-recording_id`)

```
GET https://chorus.ai/api/v1/sales-qualifications/{recording_id}
```

The returned analysis carries `field_id`, `opportunity_id`, `customer_id` and `user_id`
alongside the extracted framework values. Note that it is addressed by **`recording_id`**, not by
an analysis id.

**Review the output before acting on it.** This is model-generated inference about a sales
conversation, not observed fact.

## Step 4 — Write back to CRM (`post-api-v1-sales-qualifications-actions-writeback-crm`)

```
POST https://chorus.ai/api/v1/sales-qualifications/actions/writeback-crm
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "crm-field-update",
    "attributes": {
      "meeting_id": "<meeting id>",
      "opportunity_id": "<crm opportunity id>",
      "object_type": "Opportunity",
      "crm_changes": [
        {
          "field_name": "Budget_Confirmed__c",
          "previous_value": null,
          "new_value": true
        }
      ]
    }
  }
}
```

- Required attributes: **`meeting_id`** and **`crm_changes`**.
- `type` must be exactly `crm-field-update`.
- `object_type` defaults to `Opportunity`.
- Each entry in `crm_changes` requires **`field_name`**; `previous_value` and `new_value` accept
  any JSON type and may be null.

### Send `previous_value` — it is your only concurrency control

`previous_value` is optional in the schema, but populate it from what you actually read in step 3.
Chorus publishes no ETag, no `If-Match`, and no optimistic-concurrency mechanism, so
`previous_value` is the only record of what you believed the field held when you decided to
change it. Without it, a blind overwrite of a field a human edited seconds earlier is
indistinguishable from a legitimate update.

## Guardrails for autonomous use

This flow writes to a system of record, and the Chorus contract provides unusually little
protection:

1. **No idempotency.** There is no `Idempotency-Key` header. A retried writeback after a timeout
   may apply the same field change twice. Before retrying, re-read step 3 and compare.
2. **No 401/403/429/5xx are declared** on any operation in the contract, so failure modes are
   under-specified. Do not assume a non-200 means nothing was written — verify in the CRM.
3. **No rate limits are published**, so there is no throttling signal telling you to slow a bulk
   writeback loop.
4. **The analysis is AI-generated.** Gate writeback behind human review, or restrict it to a
   narrow allowlist of low-consequence fields. Do not let an agent write deal stage, amount or
   close date unsupervised.
5. **Scope creep is silent.** `crm_changes` is an open array of arbitrary `field_name` values —
   nothing in the API constrains which CRM fields an agent may touch. Enforce that allowlist in
   your own code; the API will not.

## Error handling

JSON:API envelope, not RFC 9457:

- **400** — wrong `type` string, missing `meeting_id`/`crm_changes`, or an unknown
  `field_name`. `errors[].source.pointer` locates it.
- **404** — unknown `recording_id`, or the token user lacks access to that recording.
- **409** — conflicting state; re-read the analysis and the CRM record before retrying.

`errors[].code` exists but is not enumerated by Chorus, so branch on HTTP status.
