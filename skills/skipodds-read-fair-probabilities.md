---
name: skipodds-read-fair-probabilities
description: >-
  Fetch de-vigged consensus win probabilities for upcoming or live sports fixtures from SkipOdds, in
  any of 13 sports, and read them correctly — including the market-shape polymorphism the OpenAPI does
  not model.
api: SkipOdds REST API
generated: '2026-08-11'
method: generated
source: openapi/skipodds-openapi.yml + https://skipodds.com/docs/fixtures
operations:
  - listFixtures
  - listSportFixtures
  - getFixture
  - getSportFixture
---

# Read fair win probabilities from SkipOdds

The `skipodds` object on every fixture is the SkipOdds Index: every surveyed bookmaker price averaged
and stripped of its margin, so the probabilities sum to exactly 1.

## Before you start

- Base URL is `https://skipodds.com`. Every path is prefixed `/v1`.
- Send the key as `Authorization: Bearer <key>`. `x-api-key: <key>` is accepted for existing
  integrations. **Never put the key in a query string** — the provider states this explicitly.
- With no key you fall onto the shared demo key `skipodds-demo-2026`: 100 requests/day shared with
  every other anonymous caller. A free personal key (250/day) is at https://skipodds.com/#free.
- On the free tiers you **must** render a visible credit linking to https://skipodds.com wherever the
  numbers appear. This is a licence term, not a courtesy.

## Steps

### 1. Choose the right operation for the sport

Soccer is the unprefixed default; every other sport names itself in the path.

- Soccer → `listFixtures` — `GET /v1/fixtures`
- Anything else → `listSportFixtures` — `GET /v1/{sport}/fixtures`, where `{sport}` is one of
  `tennis`, `cricket`, `rugby`, `baseball`, `football` (American), `basketball`, `hockey`,
  `college-football`, `college-basketball`, `mma`, `boxing`.

Golf is not a fixture sport — use `listGolfTournaments` (see the outrights skill).

### 2. List

```
GET /v1/fixtures?competition=premier-league-2026&limit=25
Authorization: Bearer <key>
```

- `limit` is 1–50, default 25. The OpenAPI declares only the default; the range is in the docs.
- `competition` is a soccer-only slug and is ignored for other sports. **You cannot look slugs up** —
  call once without it and read the slugs off the fixtures that come back.
- An empty `fixtures` array is a normal result, not an error: nothing is scheduled in range.

### 3. Read the envelope before the payload

Every 200 carries `source`, `attribution`, `generated_at`, `requests_remaining_today` and `tier`.

**`requests_remaining_today` is the only rate-limit signal there is.** SkipOdds sends no
`RateLimit-*`, no `X-RateLimit-*`, and no `Retry-After`. If you wait for a header you will never see
the wall coming — watch this field and back off before it reaches zero.

### 4. Read the `skipodds` object for the right market shape

The key set changes by sport, and **only the 3-way variant is in the OpenAPI schema**. Branch on what
is present, never on what the spec says:

| Market | Keys |
|---|---|
| Team sports pricing a draw (soccer, cricket, rugby) | `home`, `draw`, `away` |
| Moneyline sports (baseball, football, basketball, hockey, college) | `home`, `away` |
| Tennis, MMA, boxing | `p1`, `p2` |
| Golf | `players[]` — each `{name, p, fair_odds}` |

Every variant also carries `fair_odds` (decimal odds implied by the fair probabilities, keyed the same
way), `books_surveyed`, and `margin_removed` (the overround that was stripped out).

Probabilities are fractions summing to 1 — `0.6789` is 67.9%, not 0.68%.

### 5. Fetch a single fixture

```
GET /v1/fixtures/{id}
GET /v1/{sport}/fixtures/{id}
```

`id` is an opaque UUID. The spec says it plainly: **do not construct or guess one.** Take it verbatim
from a listing. Ids also expire — a fixture rolls off the board 6 hours after kickoff for soccer and
tennis, 8 hours for basketball, baseball, hockey, American football, MMA and boxing, and 12 hours for
cricket and rugby. If an id stops resolving, re-list rather than retrying.

## Errors

The envelope is flat JSON, not RFC 9457: `{"error": "<slug>"}`.

- `401 invalid_api_key` — key missing, malformed, or revoked. Do not retry; fix the header.
- `429 quota_exceeded` — daily quota gone. There is no `Retry-After` and no published reset time.
  Back off until the next day; do not hammer.
- `404` — unknown fixture. Documented in prose only, absent from the OpenAPI, and its slug is
  unpublished, so match on the status code, not on a body value.

No 5xx and no 400 validation shape is declared anywhere. Treat any other status as opaque.

## Caching and retries

Responses are edge-cached about 60 seconds (`cache-control: public, s-maxage=60, max-age=60`). Polling
faster than that spends quota for identical bytes. All four operations here are GETs and are safe to
retry.

## Do not

- Do not present these numbers as betting advice. The provider states throughout that the data is
  informational only, carries no warranty of fitness for wagering, and that it does not accept bets,
  hold funds, or settle wagers. It is 18+.
- Do not bulk re-sell or redistribute raw responses — the terms permit display, not resale.
- Do not expect bookmaker names. None are exposed, by design.
