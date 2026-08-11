---
name: skipodds-tournament-outrights
description: >-
  Get normalised tournament-winner probabilities from SkipOdds — soccer competitions and golf fields —
  and understand why the numbers differ from a raw outright board.
api: SkipOdds REST API
generated: '2026-08-11'
method: generated
source: openapi/skipodds-openapi.yml + https://skipodds.com/docs/outrights
operations:
  - listOutrights
  - listGolfTournaments
  - getGolfTournament
---

# Read tournament outrights from SkipOdds

Outright ("who wins the whole thing") markets exist for **soccer and golf only**. Every other sport in
the catalogue is fixture-level — there is no outright endpoint for tennis, cricket, the ball sports or
combat sports, and asking for one is a modelling mistake, not a missing feature.

## Soccer competitions

```
GET /v1/outrights?competition=premier-league-2026        # listOutrights
Authorization: Bearer <key>
```

`competition` is a slug. There is no endpoint that lists slugs — call `listFixtures` without a filter
and read them off the fixtures that come back.

What makes these numbers different from a bookmaker's outright board: the field is **normalised across
the entrants that can still win**. Eliminated teams are excluded rather than left sitting at a stale
price, so the surviving probabilities sum to 1 over the live field. A raw board keeps knocked-out teams
on it at whatever number they last traded, which is why raw outright markets overround so badly.

## Golf

Two operations, and the second is where the field lives.

```
GET /v1/golf/tournaments                                 # listGolfTournaments
GET /v1/golf/tournaments/{key}                           # getGolfTournament
```

- `listGolfTournaments` returns the tournaments currently priced, each with field size, margin removed,
  books surveyed and the current favourite. Call it first — `{key}` is a slug like
  `golf_the_open_championship_winner` and you should take it from here rather than composing it.
- `getGolfTournament` returns the full priced field:

```json
{
  "sport": "golf",
  "tournament": {
    "key": "golf_the_open_championship_winner",
    "title": "The Open Winner", "status": "scheduled",
    "skipodds": {
      "players": [{ "name": "Scottie Scheffler", "p": 0.091, "fair_odds": 10.99 }],
      "books_surveyed": 2, "margin_removed": 0.543
    }
  }
}
```

Golf is the one place the `skipodds` object is a **`players[]` array** rather than named outcome keys.
Each entry is `{name, p, fair_odds}`; the whole field is normalised to exactly 100% and dust quotes are
filtered out.

**Read `books_surveyed` and `margin_removed` before you trust a golf number.** The example the provider
publishes itself shows `books_surveyed: 2` and `margin_removed: 0.543` — a 54% overround stripped from
a two-book sample. That is a thin, wide market by nature, not an error, but a consensus across two
books is a very different object from one across 69, and a caller that renders both identically is
misleading its users. Surface the book count next to the probability.

Coverage: the docs state the four majors are on the current feed, with weekly tour events planned.

## Shared mechanics

- Auth, envelope, quota and errors are exactly as in `skipodds-read-fair-probabilities`. Watch
  `requests_remaining_today`; there are no rate-limit headers.
- Responses are edge-cached ~60s. Outright markets move slowly — poll far less often than fixtures.
- Free tiers must credit SkipOdds with a link to https://skipodds.com wherever the numbers render.
- All three operations are GETs and safe to retry.

## Do not

- Do not build an outright board for a sport other than soccer or golf — the data does not exist.
- Do not compare a SkipOdds outright probability against a bookmaker's implied probability without
  saying that one has the margin removed and the other does not. That difference is the whole product.
- Do not present any of it as betting advice. Informational only, no warranty of fitness for wagering,
  18+.
