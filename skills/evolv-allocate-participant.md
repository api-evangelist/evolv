---
name: evolv-allocate-participant
description: >-
  Boot a participant into Evolv's running experiments and read back the variant
  configuration (genome) to apply to the experience. Use when integrating Evolv AI
  server-side or in a runtime that cannot run the browser snippet.
api: Evolv Participant API
base_url: https://participants.evolv.ai/v1
operations:
  - GET /v1/{environment_id}/configuration
  - POST /v1/{environment_id}/allocations
  - GET /v1/{environment_id}/allocations
generated: '2026-08-13'
method: generated
source: postman/evolv-participant-api.postman_collection.json
---

# Allocate a participant into Evolv experiments

Evolv's Participant API is a runtime edge API. It answers one question — *which variant
should this visitor see* — and it answers it as a **genome**: a nested object of the
configuration values the experiment is mutating. There is no schema for the genome; its
shape is defined by the customer's own Evolv project.

## Before you start

- You need an **environment id**. It is the second path segment of every call
  (`/v1/{environment_id}/...`) and is a publishable, client-side value — it is not a secret
  credential. See `authentication/evolv-authentication.yml`.
- You need a **uid** for the participant. Evolv does not mint it; the SDK does, and if you
  are calling the API directly you mint and persist it yourself. The format Evolv's own
  examples use is `<random>_<epoch-millis>`, e.g. `96109008_1657045713019`.
- A **sid** (session id) is optional on every operation that accepts it.
- Base host is `https://participants.evolv.ai`, version is the path segment `v1`. A staging
  host, `participants-stg.evolv.ai`, is shipped as `TEST_DOMAIN` in the official SDKs — see
  `sandbox/evolv-sandbox.yml`.

## Step 1 — read the environment configuration

```
GET https://participants.evolv.ai/v1/{environment_id}/configuration
```

Returns the whole environment document: `_experiments[]`, each with `sample_rate` and
`_candidates[]`, each candidate carrying `allocation_probability` and its `genome`. See
`examples/evolv-configuration-basic-response-example.json` for a real response body.

> **Parse defensively on this route.** Configuration is served from an S3 origin behind
> CloudFront, not from the routed API. An unknown or unauthorized environment id returns
> **HTTP 403 with an XML** `<Error><Code>AccessDenied</Code>` body — not the JSON `{"msg":…}`
> envelope the other endpoints use. Branch on `Content-Type`, not on status alone.

## Step 2 — allocate the participant

```
POST https://participants.evolv.ai/v1/{environment_id}/allocations
Content-Type: application/x-www-form-urlencoded

uid=96109008_1657045713019&sid=21646621_1658149742929
```

`uid` is always required. `sid` is optional. This attempts to update the participant's
allocations based on the values in the body.

Evolv is explicit that allocation is **candidate**, not membership: for every experiment a
participant is eligible for, this returns the candidate allocation they *would* receive —
they are **not yet considered part of the experiment**. Membership only starts when you
record a `confirmation` event (see the `evolv-record-events` skill). This endpoint also
respects **experiment throttling and audience filters**, so a participant can legitimately
come back excluded.

## Step 3 — read the participant's allocations

```
GET https://participants.evolv.ai/v1/{environment_id}/allocations?uid={uid}&sid={sid}
```

Returns an array of allocation records. Real shape, from Evolv's own saved response
(`examples/evolv-allocations-get-participants-allocations-example.json`):

```json
[
  {
    "excluded": false,
    "uid": "test_user",
    "sid": "test_session",
    "eid": "experiment_id",
    "cid": "candidate_id",
    "genome": { "pages": { "testing_page": { "header": "white" } } }
  }
]
```

Apply the `genome` of every record where `excluded` is `false`. Skip any record where
`excluded` is `true` — that participant was filtered out by throttling or an audience rule
and must be served the control experience.

Note `cid` is compound: `<allocation-hash>:<eid>`. The experiment id is the suffix, so a
candidate id resolves back to its experiment without a second call.

## Failure handling

| Status | Body | What it means | What to do |
|---|---|---|---|
| 404 | `{"msg":"Not found"}` | Unknown environment id **or** unallocated participant — the API does not distinguish them | Treat as "no configuration available" and serve control. Verify the environment id before retrying. |
| 403 (XML) | `<Error><Code>AccessDenied</Code>` | Configuration route only; S3 origin rejected the environment id | Fix the environment id. Do not retry. |
| 500 | `{"msg":"Internal server error"}` | Can be a permanent caller error, not a transient fault | Do not retry blindly. See `errors/evolv-problem-types.yml`. |

There is no `Retry-After` and there are **no rate-limit headers of any family** on this API
(`rate-limits/evolv-rate-limits.yml`), so back off on your own schedule and never assume a
budget signal will arrive.

## What this API does not give you

- **No idempotency key.** Repeating `POST /allocations` is not protected by a key; the
  operation's safety comes from it being a state-refresh for one uid, not from a contract.
- **No pagination.** Every response is a whole document for one participant or environment.
- **No error codes.** The only machine-readable signal is the HTTP status; the body carries
  one free-text `msg` string.
