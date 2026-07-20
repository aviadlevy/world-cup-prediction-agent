# ESPN WC2026 results & standings (verified 2026-06-17)

Free, no auth. Use as the **form/standings cross-check layer** on top of Polymarket (Polymarket prices the future; ESPN gives the verifiable past — actual scores, points, goal difference, qualification state). League slug: `fifa.world`.

## Standings (points, GD, goals for/against) — all 12 groups

```bash
curl -s "https://site.api.espn.com/apis/v2/sports/soccer/fifa.world/standings" | jq -r '
  .children[] | .name as $g | "\n## \($g)",
  (.standings.entries[]
   | "  \(.stats[]|select(.name=="points").displayValue)pts  \(.stats[]|select(.name=="overall").displayValue)  GF\(.stats[]|select(.name=="pointsFor").displayValue)/GA\(.stats[]|select(.name=="pointsAgainst").displayValue) (GD\(.stats[]|select(.name=="pointDifferential").displayValue))  \(.team.displayName)")'
```

Stat `name`s on each entry: `points` (3/win), `pointDifferential` (GD), `pointsFor`/`pointsAgainst` (goals — NOT points), `wins`/`ties`/`losses`, `overall` (W-D-L string), `gamesPlayed`, `rank`, `advanced`.

## Results for a given day (actual scores)

`dates` is `YYYYMMDD` in the match's local (US) date — same one-day-behind-Israel offset as Polymarket slugs.

```bash
curl -s "https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard?dates=20260614" | jq -r '
  .events[]? | "\(.status.type.state)  \(.name): \(.competitions[0].competitors[0].team.abbreviation) \(.competitions[0].competitors[0].score) - \(.competitions[0].competitors[1].score) \(.competitions[0].competitors[1].team.abbreviation)"'
```

`status.type.state`: `pre` (not started), `in` (live), `post` (final). Date range: loop the dates you need.

## How to use it in a prediction

1. Pull standings once per session — gives every team's GD, points, and (from round 3 on) whether they're already qualified/eliminated.
2. Read each team's prior result(s): a 5-1 or 0-7 is a strong form/finishing signal the market's single price flattens.
3. **It does not override Polymarket** (pure-EV prior). It (a) cross-checks the market pick, (b) breaks near-ties toward the in-form / higher-scoring side, and (c) from round 3 flags rotation risk for already-qualified sides.
