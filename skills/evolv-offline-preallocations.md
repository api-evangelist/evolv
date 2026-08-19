---
name: evolv-offline-preallocations
description: >-
  Pull a bulk set of Evolv preallocations for offline or server-side assignment, where the
  participant is not known at request time — email campaigns, batch jobs, kiosk and
  point-of-sale surfaces, or any experience assembled before a visitor exists.
api: Evolv Participant API
base_url: https://participants.evolv.ai/v1
operations:
  - GET /v1/{environment_id}/preallocations
  - GET /v1/{environment_id}/events
generated: '2026-08-13'
method: generated
source: postman/evolv-participant-api.postman_collection.json
---

# Pull preallocations for offline use

Every other Evolv flow starts with a participant. This one does not. Preallocations are
allocations minted **ahead of time** — the same `eid` / `cid` / `genome` triple you would
get from `/allocations`, but with no participant attached. You associate each one with a
user later, at your own discretion.

Use this when the visitor does not exist yet at the moment you need the variant: rendering a
batch of emails, precomputing a personalized export, seeding a kiosk, or any server-side job
that runs detached from a session.

## Step 1 — request the batch

Scoped to one experiment:

```
GET https://participants.evolv.ai/v1/{environment_id}/preallocations?eid=7e25506c1b&count=10
```

Across every experiment in the environment:

```
GET https://participants.evolv.ai/v1/{environment_id}/preallocations?count=10
```

| Parameter | Required | Behaviour |
|---|---|---|
| `count` | yes | How many preallocations to mint. |
| `eid` | no | If present, returns `count` preallocations **for that experiment**. If absent, returns an array of `count` preallocations **for each experiment** in the environment. |

That difference matters for sizing: without `eid`, the response is `count × experiments`,
not `count`. Read the array length rather than assuming it.

## Step 2 — read the response

Returns HTTP 200 with an array of preallocation records. Real shape, from Evolv's own saved
response (`examples/evolv-preallocations-basic-response-example.json`):

```json
[
  {
    "eid": "d8980de2ed",
    "cid": "d980e25485b719812720691415245159:d8980de2ed",
    "genome": { "pages": { "testing_page": { "header": "white" } } }
  }
]
```

There is no `uid`, no `sid` and no `excluded` flag — those only exist once a participant is
bound. `cid` is compound (`<hash>:<eid>`), so each record still resolves to its experiment.

## Step 3 — bind and confirm

Store each record against whichever user you assign it to, apply the `genome` when you
render, and then close the loop the normal way: send a `confirmation` event carrying that
user's `uid` together with the `eid` and `cid` from the preallocation, using the
`evolv-record-events` skill. Until that confirmation lands, the preallocation has not put
anyone into an experiment.

## Cautions

- **This route returns 500, not 4xx, on a bad environment id.** A probe of
  `/v1/test/preallocations?count=2` returned `500 {"msg":"Internal server error"}`, so a
  caller error surfaces here as a server error. Do not build an automatic retry on 500 for
  this endpoint — confirm the environment id first. See `errors/evolv-problem-types.yml`.
- **Preallocations are minted, not reserved.** Nothing in the published API expires or
  releases an unused one, and there is no read-back. Mint what you will use.
- **No idempotency key.** Re-issuing the same request mints a fresh batch; it does not
  return the previous one.
- **No pagination and no rate-limit headers.** `count` is the only sizing control you have
  (`rate-limits/evolv-rate-limits.yml`).
