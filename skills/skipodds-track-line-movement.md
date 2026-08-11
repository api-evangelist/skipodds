---
name: skipodds-track-line-movement
description: >-
  Detect how the SkipOdds Index has moved on a fixture over a time window, and set up push alerts when
  it moves past a threshold — the poll path and the webhook path, with the caveats on both.
api: SkipOdds REST API
generated: '2026-08-11'
method: generated
source: openapi/skipodds-openapi.yml + https://skipodds.com/docs/movement + https://skipodds.com/docs/alerts
operations:
  - listFixtures
  - getFixtureMovement
  - getSportFixtureMovement
  - listAlertWebhooks
  - createAlertWebhook
---

# Track line movement on SkipOdds

Two ways to see the consensus move: pull a window on demand, or register a webhook and be pushed. Pull
works on every tier; push is paid tiers only.

## Path A — pull a movement window

### 1. Get a fixture id

Movement is addressed through the parent fixture, so start with `listFixtures` (soccer) or
`listSportFixtures` (everything else) and take the opaque `id`.

### 2. Ask for the window

```
GET /v1/fixtures/{id}/movement?hours=24            # listFixtures ids  → getFixtureMovement
GET /v1/{sport}/fixtures/{id}/movement?hours=24    # sport ids         → getSportFixtureMovement
Authorization: Bearer <key>
```

`hours` defaults to 24. It is capped at **72** on Demo, Free, Starter and Pro, and **336** (14 days)
only on the Scale tier. Neither cap is expressed in the OpenAPI — asking for 336 on a Pro key is a
tier problem, not a parameter problem.

### 3. Read the result

```json
{
  "from":      { "home": 0.52,  "draw": 0.24,  "away": 0.24  },
  "to":        { "home": 0.561, "draw": 0.23,  "away": 0.209 },
  "delta_pct": { "home": 4.1,   "draw": -1.0,  "away": -3.1  },
  "span_hours": 22.5
}
```

- `from` and `to` are the earliest and latest captures **inside the window you asked for** — not the
  opening line and not now-vs-kickoff. Change `hours` and both endpoints move.
- `delta_pct` is a signed change in probability **points**, not a percentage change: `4.1` means
  52.0% → 56.1%. Do not divide it by anything.
- `span_hours` is the real elapsed time between the two captures and will be less than `hours` when
  the fixture has not existed that long. Use it, not `hours`, when annotating a chart.
- The key set follows the same market polymorphism as the fixture payload (`home/draw/away`,
  `home/away`, `p1/p2`).

Responses are edge-cached ~60s, so polling faster than a minute returns the same bytes and spends
quota. Watch `requests_remaining_today` in the envelope — it is the only rate-limit signal, since no
`RateLimit-*` or `Retry-After` headers are sent.

## Path B — push, via alert webhooks

Paid tiers only (Starter and above). SkipOdds evaluates upcoming events in your chosen sport every 15
minutes and posts to your endpoint when the Index moves past your threshold.

### 1. Register

```
POST /v1/alerts/webhooks                          # createAlertWebhook
Authorization: Bearer <key>
Content-Type: application/json

{ "url": "https://example.com/hook", "sport": "soccer", "threshold_points": 3 }
```

**Field-name caveat:** the docs call this field `threshold_points`; the OpenAPI `requestBody` calls it
`threshold`. The two published sources disagree — send what the docs show and verify with
`listAlertWebhooks` that the registration took the value you meant.

`threshold_points` is the minimum move in probability points, 1–20, default 3. A move from 58.0% to
61.5% is +3.5 points.

**This POST is the only mutating operation in the API and there is no idempotency mechanism.** No
`Idempotency-Key`, no de-duplication. If the call times out, do **not** blind-retry — call
`listAlertWebhooks` first and check whether the registration already exists, or you will end up with
duplicates burning your slots.

Slots per tier: Starter 2, Pro 5, Scale 20.

### 2. Confirm and manage

```
GET /v1/alerts/webhooks                           # listAlertWebhooks
```

`DELETE /v1/alerts/webhooks/{id}` is documented at https://skipodds.com/docs/alerts but is **not** in
the published OpenAPI, so a generated client will not have it. Call it directly if you need it.

### 3. Receive

A Discord webhook URL gets a pre-formatted Discord message. Any other HTTPS endpoint gets JSON:

```json
{
  "event": "index_move", "sport": "soccer",
  "market": "Norway v England", "outcome": "Norway",
  "from": 0.231, "to": 0.268, "delta_points": 3.7,
  "ts": "2026-07-11T18:15:00.000Z"
}
```

Two things to plan around:

- **There is no signature and no shared secret.** No HMAC header, no timestamp signature, no published
  source IP range. You cannot verify that a delivery came from SkipOdds. Treat the endpoint as
  untrusted input: use an unguessable URL, validate the payload shape, and never let it trigger
  anything with a side effect you would regret.
- **The payload carries no fixture id** — `market` is a human-readable string like `"Norway v England"`.
  You cannot join an event back to a fixture programmatically. If you need the fixture, re-list and
  match on team names yourself.

Retry and replay behaviour is not documented. Endpoints failing **10 deliveries in a row are
deactivated automatically**, so a receiver that is down for an hour may come back unsubscribed —
re-check `listAlertWebhooks` after any outage.

## Do not

- Do not read a move as a signal to bet. The data is informational only and carries no warranty of
  fitness for wagering. 18+.
- Do not resell raw responses; display is permitted, bulk redistribution is not.
