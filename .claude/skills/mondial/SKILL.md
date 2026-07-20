---
name: mondial
description: Use when asked about World Cup 2026 matches, predictions, or what to bet in the office Tiko pool — upcoming games, a specific match or team, exact-score picks, futures (winner / top scorer), or pool standings strategy.
---

# Mondial — World Cup 2026 Office Pool Predictor

Recommends exact-score predictions for the office Tiko pool. Domain terms and scoring: see [CONTEXT.md](../../../CONTEXT.md) at repo root.

## Files (all in `.claude/skills/mondial/` from repo root)

- `schedule.json` — all 104 fixtures, kickoff times in Israel time (`Asia/Jerusalem`). The single source of truth for "what's next".
- `polymarket.md` — verified API commands, slugs, and parsing patterns. Read it before any Polymarket call.
- `espn.md` — free, no-auth ESPN endpoints for actual results and live group standings (points/GD/goals). The form & qualification cross-check layer on top of Polymarket.
- `predictions.json` — picks the user has confirmed (match picks and futures). Created in this same directory on first confirm.
- `cronjobs.json` — definitions of the session-only CronCreate jobs (daily bet-revision check + reminders). If the session restarted, ask the user whether to restore them: run CronList, and re-create any missing entry with CronCreate using the stored cron/recurring/prompt (skip expired one-shots; recurring jobs expire 7 days after createdAt).

## Answering schedule questions

Run `TZ=Asia/Jerusalem date '+%Y-%m-%dT%H:%M'` for current time, then read `schedule.json`. "Next game" / "next 5 games" = first matches with `kickoffIsrael` after now. Filter by team or date when asked. Always show kickoff in Israel time.

**Knockout placeholder resolution:** when a queried knockout match still has placeholders (`1A`, `W74`) but the qualifying games have been played, web-search the actual pairing and write the real team names back into `schedule.json` (keep the placeholder in a `was` field).

## Making a match prediction

For each requested match:

1. **Polymarket first** (commands in `polymarket.md`):
   - 1X2 event → W/D/L probabilities, normalized (divide by the sum)
   - `-exact-score` event → rank scorelines by raw price; report raw % (don't normalize the 17-way set — ranking is what matters)
   - Gamma snapshot prices are fine; use CLOB `/midpoint` only when kickoff is <2h away
2. **Form & standings** (`espn.md`): pull current standings (points/GD) once per session and each team's prior result(s) this tournament. A lopsided scoreline (5-1, 0-7) is a finishing/defensive signal the market's single price flattens. From group round 3 on, read each team's qualification state (already through, eliminated, or needing a specific result) — this drives rotation and game-state, and is the dominant factor that round.
3. **Web research** (2–3 searches): confirmed lineups, injuries/suspensions, motivation (must-win vs. already-qualified — critical in group round 3 and for likely rotation), and any published model forecast (e.g. Opta supercomputer) as a cross-check.
4. **Pick**: the highest-priced exact score consistent with the most likely outcome. Polymarket exact-score prices take priority; if unavailable (market not yet live), use bookmaker correct-score odds found via search, else heuristic: clear favorite → 2-0/2-1, slight favorite → 1-0/2-1, even → 1-1.
   - **Near-tie tiebreak**: top two scorelines within 2 points of each other → break with the O/U 2.5 lean from `-more-markets`, the in-form/higher-scoring side per ESPN form, and the model-forecast scoreline; say in "Why" that it was a tiebreak.
   - **Cross-check disagreement**: if Opta/press consensus points to a different scoreline or its W/D/L differs from the market by >10 points, keep the market pick but say so in "Why".
5. Knockout rounds — **scoring is DIRECTION, not exact score.** Points are for correctly calling the **regular-time result (W/D/L at 90 min)**, regardless of the exact scoreline. So the pick is the **most-likely outcome from the 1X2 market** — NOT the exact-score top. A "draw" pick (e.g. 1-1) cashes on ANY scoreline level after 90 (including ties that go to extra time), so only pick a draw when the **draw is the top *outcome*** (genuinely even ties) — not merely because 1-1 is the top exact *scoreline*. Enter any scoreline consistent with the chosen outcome (the modal winning scoreline is fine; it doesn't affect points). This is the opposite of the group stage, which is exact-score (Bull). **Suspensions:** group-stage yellow cards are wiped before the Round of 32 (FIFA 2026) — only red cards and injuries carry; don't hunt phantom yellow suspensions; do check injuries. Markets for later knockout ties often go live only ~1-3 days pre-kickoff — if not live, record a provisional pick (most-likely outcome by team strength) and let the daily check firm it.

## Output format

One table for all requested matches:

| # | Match (stage) | Kickoff (IL) | W / D / L | Bet (Bull) | 2nd choice | Why |
|---|---|---|---|---|---|---|

Probabilities as percentages. "Why" = one line. **Flag a pick as low-confidence when the top scoreline is priced under 15% or within 2 points of the 2nd choice.** After the table, ask whether to record the picks; on confirm, append to `predictions.json` (`enteredAt` = Israel-time datetime):

```json
{"match": 1, "pick": "1-0", "enteredAt": "2026-06-11T09:30", "kickoff": "2026-06-11T22:00", "teams": "Mexico vs South Africa"}
```

Futures entries use `{"future": "winner" | "topScorer", "pick": "...", "enteredAt": "..."}`.

Never re-recommend a match already in `predictions.json` — show the recorded pick instead (unless the user asks to revise; then update the entry).

## Strategy stance (pure EV)

Every pick — per-game and futures — is the highest-probability option, full stop. Ignore what other pool participants might pick; no contrarian or differentiation logic.

## Common mistakes

- Don't trust memory for fixtures or prices — read `schedule.json`, call the API.
- Don't average Polymarket with weaker sources — Polymarket is the prior; news only shifts it when materially fresh (lineup leaks, injuries).
- Don't recommend 0-0 Bulls unless the market genuinely prices it top (it rarely does).
- Don't forget `outcomePrices` is a JSON-encoded string inside the JSON response.
