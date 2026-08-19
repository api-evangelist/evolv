---
name: evolv-record-events
description: >-
  Record confirmations, contaminations and conversions against a participant's Evolv
  allocation — one at a time, as a batch, or on a link click with a redirect. Use when
  feeding outcome signal back into Evolv AI so its optimization can learn.
api: Evolv Participant API
base_url: https://participants.evolv.ai/v1
operations:
  - GET /v1/{environment_id}/events
  - POST /v1/{environment_id}/events
  - PATCH /v1/{environment_id}/events
generated: '2026-08-13'
method: generated
source: postman/evolv-participant-api.postman_collection.json
---

# Record events against an Evolv allocation

Allocation alone does nothing for the model. Evolv learns from **events** — this is the
half of the integration that closes the loop.

## The two reserved event types

`type` is free-form and can be any value, but two values have a special meaning in the
platform, stated in Evolv's own documentation:

- **`confirmation`** — registers the participant as *active* on a given experiment /
  candidate. Until you send this, an allocated participant is not counted as part of the
  experiment.
- **`contamination`** — *excludes* the specified participant from being counted in an
  experiment / candidate. Send this when you know the measurement is spoiled (the variant
  failed to render, the user hit an error path, a bot was detected).

Everything else — `conversion` and any custom name you choose — is ordinary outcome signal.

## Fields

| Field | Required | Meaning |
|---|---|---|
| `type` | yes | Event type. `confirmation` / `contamination` are reserved (above). |
| `uid` | yes | The participant. Always required. |
| `sid` | no | The participant's current session. |
| `eid` | contextual | Experiment id the event applies to. |
| `cid` | contextual | Candidate id, compound `<hash>:<eid>`. |
| `score` | no | Numeric score associated with the event. |
| `target` | GET only | URL to redirect to after recording — see the redirect flow below. |

## Step 1 — record a single event

Query-string form (the beacon shape the browser SDK uses):

```
GET https://participants.evolv.ai/v1/{environment_id}/events
      ?type=confirmation
      &eid=7e25506c1b
      &cid=2c3aefac54a4:7e25506c1b
      &uid={uid}
      &sid={sid}
      &score=1.0
```

Or as a form post:

```
POST https://participants.evolv.ai/v1/{environment_id}/events
Content-Type: application/x-www-form-urlencoded

type=conversion&uid={uid}&sid={sid}&score=1.0
```

Both return **HTTP 202 Accepted** with an empty body. 202 means *accepted for processing*,
not *recorded* — this is a fire-and-forget beacon. There is no receipt, no event id, and no
way to read an event back. Do not build a confirmation-of-write step; there isn't one.

## Step 2 — batch events

```
PATCH https://participants.evolv.ai/v1/{environment_id}/events
Content-Type: application/json

[
  { "type": "confirmation", "eid": "7e25506c1b", "cid": "2c3aefac54a4:7e25506c1b",
    "uid": "96109008_1657045713019", "sid": "21646621_1658149742929", "score": 0.0 },
  { "type": "conversion",   "eid": "7e25506c1b", "cid": "2c3aefac54a4:7e25506c1b",
    "uid": "96109008_1657045713019", "sid": "21646621_1658149742929", "score": 1.0 }
]
```

Note the verb: batch ingestion is **PATCH**, not POST — the one place this API departs from
what you would guess. The body is a JSON array of the same field set. Returns 202.

Prefer the batch route for server-side integrations: one request per flush instead of one
per event, and it is the only route that takes JSON rather than form encoding.

## Step 3 — record a conversion on a link click

```
GET https://participants.evolv.ai/v1/{environment_id}/events
      ?type=conversion&uid={uid}&sid={sid}&score=0.0
      &target=http%3A%2F%2Fwww.google.com
```

With `target` present the API records the event and returns **HTTP 302** to that URL. This
is how a plain `<a href>` can both convert and navigate with no JavaScript. Remember to
URL-encode `target`, and follow redirects in whatever client you use.

## Idempotency and retries — read this before you add a retry loop

There is **no idempotency key** on this API and no de-duplication contract. Every accepted
event is a new event. A blind retry after a timeout will double-count a conversion and skew
the optimization the platform is running. If you must retry, retry only on transport
failure where you have evidence the request never reached the edge, and never on a 202 you
simply did not read.

There are no rate-limit headers to pace against
(`rate-limits/evolv-rate-limits.yml`), so pick your own flush interval.

## Failure handling

| Status | Body | What to do |
|---|---|---|
| 202 | empty | Success. Nothing to parse. |
| 302 | empty | Success on the `target` flow; follow the `Location`. |
| 404 | `{"msg":"Not found"}` | Unknown environment id or unknown participant. Verify the environment id; do not retry the event. |
| 500 | `{"msg":"Internal server error"}` | May be a permanent caller error. See `errors/evolv-problem-types.yml`. |
