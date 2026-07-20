# Polymarket WC2026 API Reference (verified 2026-06-10)

Read-only, no auth. Gamma API for discovery + price snapshots; CLOB API for freshest mid-price.

## Working curl commands

```bash
# Search events by keyword
curl -s "https://gamma-api.polymarket.com/public-search?q=World%20Cup%20Winner&limit_per_type=8"

# List ALL open World Cup events (tag_id 102232 = FIFA World Cup; paginate, ~515 open events)
curl -s "https://gamma-api.polymarket.com/events?tag_id=102232&closed=false&limit=100&offset=0"

# Event + all its markets/prices by slug (returns array)
curl -s "https://gamma-api.polymarket.com/events?slug=world-cup-winner"
curl -s "https://gamma-api.polymarket.com/events?slug=world-cup-golden-boot-winner"
curl -s "https://gamma-api.polymarket.com/events?slug=fifwc-mex-rsa-2026-06-11"
curl -s "https://gamma-api.polymarket.com/events?slug=fifwc-mex-rsa-2026-06-11-exact-score"

# CLOB live price (token_id from market's clobTokenIds field; [0]=Yes, [1]=No)
curl -s "https://clob.polymarket.com/midpoint?token_id=<TOKEN_ID>"
```

Responses are large (exact-score events are tens of KB). Standard parsing pattern — works without assuming `jq` flags beyond basics:

```bash
# Scoreline prices from an exact-score event, sorted
curl -s "https://gamma-api.polymarket.com/events?slug=fifwc-mex-rsa-2026-06-11-exact-score" | \
  jq -r '.[0].markets[] | select(.active and (.closed|not)) | [.question, (.outcomePrices|fromjson)[0]] | @tsv' | sort -t$'\t' -k2 -rn
```

## Key event slugs

| Market | Event slug | Event ID |
|---|---|---|
| Tournament winner | `world-cup-winner` | 30615 |
| Golden Boot (top scorer) | `world-cup-golden-boot-winner` | 413862 |
| Group winners | `world-cup-group-{a..l}-winner` | — |
| Reach final / SF / QF / R16 | `world-cup-nation-to-reach-{final,semifinals,quarterfinals,round-of-16}` | — |
| Most assists / goal contributions / clean sheets | `world-cup-most-{assists,goal-contributions,clean-sheets-gk}` | — |

## Per-match slug pattern

`fifwc-{home3}-{away3}-{YYYY-MM-DD}[-{suffix}]` — lowercase 3-letter FIFA-ish codes (`mex`, `rsa`, `usa`, `fra`; quirks: `kr` = Korea Republic, `cdr` = DR Congo, `cvi` = Cabo Verde).

Event variants per match:
- **(no suffix)** — 1X2 as 3 binary markets ("Will {home} win on {date}?", draw, away)
- **`-exact-score`** — 17 binary markets: every score 0-0 through 3-3 + "Any Other Score" bucket
- **`-halftime-result`** — HT 1X2
- **`-more-markets`** — O/U total goals 0.5–5.5, spreads, BTTS, per-team O/U
- Near matchday: `-first-to-score`, `-total-corners`, `-player-props`

Knockout-match events appear once pairings are known — same pattern. If a slug guess 404s, find the match via the tag_id listing or `public-search`.

## Price → probability

- Prices are USD in [0,1]; price ≈ implied probability. `outcomes` / `outcomePrices` / `clobTokenIds` are JSON-encoded strings inside the JSON — parse them.
- Multi-outcome events are sets of independent binary markets; Yes-prices sum to ~1 plus small overround. Normalize: divide each by the sum.
- Filter on `active:true, closed:false` — search results include dead markets.
- Winner-market slugs carry arbitrary numeric suffixes (`will-spain-win-the-2026-fifa-world-cup-963`) — resolve markets via the parent event, never by guessing market slugs.
