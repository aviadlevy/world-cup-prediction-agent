# The methodology: how the agent makes a pick

This is the method in depth — the reasoning the [`/mondial` skill](../.claude/skills/mondial/SKILL.md) encodes and applies the same way in every session, days apart. The one-paragraph version lives in the [README](../README.md); this is the long version, with the math.

The whole thing rests on one bet: **a liquid prediction market is a better forecaster than my opinion.** Everything below is scaffolding around that idea — how to read the market, when to let anything else touch the pick, and how the pool's scoring rules bend "highest probability" into "highest expected value," which is not always the same thing.

---

## 1. The market is the prior

For every fixture the agent pulls [Polymarket](../.claude/skills/mondial/polymarket.md) and reads its prices as probabilities. Two markets per match matter:

- **The 1X2 market** — three binary markets ("Will *home* win?", draw, "Will *away* win?"). Yes-prices sum to ~1 plus a small overround; normalize by dividing each by the sum. This gives the **outcome** distribution: `P(home win) / P(draw) / P(away win)`.
- **The `-exact-score` market** — 17 binary markets, every scoreline from 0-0 through 3-3 plus an "Any Other Score" bucket. This gives the **scoreline** distribution. We rank these by raw price and *don't* normalize the set — the ranking is what we use, not a calibrated probability.

Prices are USD in `[0,1]` and ≈ implied probability. The plumbing (JSON-encoded strings inside the JSON, `active/closed` filters, per-match slug quirks, the UTC-vs-Israel date offset that caused real slug bugs) is all documented in [`polymarket.md`](../.claude/skills/mondial/polymarket.md) — that file exists so the agent never re-derives the API or re-makes the same date mistake.

Why a market and not a model? Because the market has already eaten every model. Opta's supercomputer, the bookmakers, the injury news, the sharp money — all of it is priced in by the time I look. My job isn't to out-forecast that; it's to read it correctly and translate it into the pool's scoring.

---

## 2. Pure EV — no cleverness

The pool's stated stance, encoded in [`CONTEXT.md`](../CONTEXT.md) and the skill: **every pick is the highest-expected-value option, full stop.** No contrarian "pick something no one else will" logic, no differentiation against what the other players might do, no gut. In the group stage that means the highest-priced exact scoreline consistent with the most likely outcome.

This felt almost too boring to be an edge. It was the edge. Applied without deviation across the group stage, market-as-prior produced the **most exact-score ("bullseye") hits of anyone in the pool by a distance — 19, next best 14** — and since an exact score is worth 2–3× a merely-correct outcome, that quiet accuracy is what built and held the lead for weeks. Twenty-one people predicting from vibes, one process predicting from the market. (The final standing was 3rd, not 1st — but on the underlying process, nobody was close; how a pool-best process still finishes third is the story in the [README results](../README.md#results).)

(Early on there *was* a differentiation analysis — "pick the high-probability team the crowd underrates." It's preserved in [`FUTURES.md`](../FUTURES.md) for the record, but I dropped it before the tournament and went pure-EV. More on when differentiation *does* come back in §8.)

---

## 3. The scoring rule: exact vs. direction

Every stage pays two tiers (full table in [`CONTEXT.md`](../CONTEXT.md)):

| Stage | Direction (correct outcome) | Bull (exact score) |
|---|---|---|
| Group / R32 | +1 | +3 |
| Round of 16 | +2 | +4 |
| Quarter-final | +3 | +6 |
| Semi-final | +4 | +8 |
| 3rd place / Final | +5 | +10 |

Two facts sit underneath every knockout decision:

1. **Exact is worth 2–3× direction.** Being precise is where the points are.
2. **Knockouts are scored on the 90-minute result.** Extra time and penalties don't count. A game won on penalties is, for scoring, a **draw**. (Group stage is scored on its final — always 90 minutes — so it's straightforwardly exact.)

Fact 2 is one I knew but hadn't written into the method for the first two rounds — so early knockout picks over-optimized the exact score until I briefed it in. It's straightforward once stated: a draw pick cashes the **direction** points on *any* level-at-90 scoreline, so a knockout that finishes 3-2 after extra time but was 2-2 at 90 rewards a "1-1" pick. See [`predictions.json`](../.claude/skills/mondial/predictions.json) match 82 (Belgium–Senegal) for the ledger entry.

The consequence: **in knockouts you don't optimize the scoreline, you optimize the outcome.** The scoreline you type into the app is just the entry form — pick the modal scoreline for the outcome you want, but it barely moves your EV.

---

## 4. Worked example — why a favorite-win scoreline beats the top scoreline

Here's the calculation that makes §3 concrete. Take the semi-final, France vs. Spain ([`predictions.json`](../.claude/skills/mondial/predictions.json) match 101). SF scoring is **exact +8 / direction +4**.

The two markets disagreed about what "most likely" means:

- **1X2 (outcome):** France win **42.3%**, draw 29.4%, Spain win 28.4%. → France win is the top *outcome*.
- **Exact score:** **1-1 was the single most-likely scoreline (~16%)** — a draw. → 1-1 is the top *scoreline*.

A naive exact-score picker types 1-1. Let's compute the EV of each candidate under direction scoring (you get +8 only if your exact score is the 90-min result; +4 if just the outcome is right):

**Pick 1-1 (a draw):**
```
EV = P(exactly 1-1 at 90) × 8  +  P(draw but not 1-1) × 4
   = 0.16 × 8  +  (0.294 − 0.16) × 4
   = 1.28  +  0.54  =  1.82
```

**Pick 2-1 (the modal France-win scoreline, ~12%):**
```
EV = P(exactly 2-1 at 90) × 8  +  P(France win, other score) × 4
   = 0.12 × 8  +  (0.423 − 0.12) × 4
   = 0.96  +  1.21  =  2.17
```

The France-win scoreline wins **even though its exact probability is lower than 1-1's**, because the direction term dominates: France win (42.3%) is a far bigger target than draw (29.4%), and it's the direction term that most of your EV lives in. The single most-likely scoreline being a draw is a trap for the naive picker. The pick that maximizes EV is the modal scoreline *of the most likely outcome*.

(This is exactly the logic in `SKILL.md` step 5: "the pick is the most-likely **outcome** from the 1X2 market — NOT the exact-score top." The example numbers are illustrative of the market state recorded at entry; the point is structural, not the second decimal.)

---

## 5. The draw-trap

The corollary to §4, and it cost me real games (see [`predictions.json`](../.claude/skills/mondial/predictions.json) matches 96, 100, 102 — quarter/semi losses).

A heavy favorite that gets dragged to a **1-1 at 90 and then wins on penalties is a draw for scoring.** So a "favorite to win" bet *loses*, even though the favorite advanced. The knockouts are full of these: two fatigued or evenly-matched sides, a cagey 120 minutes, level at 90, decided from the spot. Match 100 (Argentina–Switzerland) is the textbook case — I flagged it "HIGHEST draw-trap," picked the Argentina win anyway, and it finished 3-1 *after* extra time but was level at 90. Miss.

The fix is a discipline, not a formula: **rate every knockout tie for draw-trap risk** (evenness in the 1X2 + fatigue from a prior 120-minute game + a low over/under 2.5), and be willing to *bet the draw* when the draw is genuinely the top outcome. Match 102 (England–Argentina SF) was my first deliberate draw pick for exactly this reason — both sides fresh off 120-minute quarters and both repeatedly level at 90 in the knockouts. (It missed — Argentina won in regulation — but it was the right *process* on the information available.)

The trap only springs one way, which is worth saying plainly: a draw pick is safe against the favorite-drags-to-penalties scenario *and* the actual-draw scenario. A favorite-win pick is only safe if the favorite wins inside 90. When you're unsure between "narrow favorite" and "even," the draw is often the higher-EV *and* lower-variance call.

---

## 6. ESPN form — a tie-breaker, never an override

Polymarket prices the future; it collapses a lot of information into one number. What it flattens is *how* a team got there. A side can be a 60% favorite whether it's grinding out 1-0s or winning 7-1, and those imply very different scorelines. So the agent adds a **form layer** from the free [ESPN `fifa.world` endpoints](../.claude/skills/mondial/espn.md): actual results, live group standings (points, goal difference), and — from group round 3 on — each team's qualification state.

Its role is strictly bounded, and the boundary is the whole point:

- **It cross-checks** the market pick (does a 3-0 make sense for a team that's scored one goal in two games?).
- **It breaks near-ties.** When the top two scorelines are within ~2 points, form tips it toward the in-form / higher-scoring side. Examples in [`predictions.json`](../.claude/skills/mondial/predictions.json): match 33 (Germany 7-1 in the opener → nudged 1-0 to 2-0), match 34 (opponent had shipped 7 → 3-0), match 35 (both sides high-scoring → the scoring-draw 1-1).
- **From round 3 it flags rotation and game-state** — an already-qualified side will rest players; a must-win side will chase. That becomes the dominant factor that round, more than raw strength.

What it explicitly does **not** do is overrule the market. Form is a lens on the prior, not a competing prior. The rule in the skill: *"It does not override Polymarket. It breaks near-ties."* Averaging Polymarket with weaker sources is listed as a common mistake for a reason — the market already saw the form.

---

## 7. Multi-model voting panels for the one-off calls

Per-match picks are cheap and self-correcting — there are 104 of them and a daily cron re-checks the live ones. The **futures** (tournament winner, top scorer) are the opposite: two picks, locked once before the opener, worth +10 each, with no second chance. High stakes, single shot — exactly where you want an ensemble, not a single sample.

So for the futures I ran a **4-model voting panel**: Haiku, Sonnet, Opus, and Fable, each given the *identical* pure-EV prompt and each doing its own live research (Polymarket + Opta + bookmakers + injuries), then aggregated with a 3/2/1 weighting of their rankings. The full panel is in [`FUTURES.md`](../FUTURES.md). It came back **unanimous**: Spain for the winner (4/4, agreeing with Polymarket, Opta, and the books), Mbappé for the golden boot (4/4). A voting panel does two things a single call can't: it surfaces disagreement as signal (here, there was none, which is itself information), and it launders out any one model's idiosyncratic error.

The same "run the prompt through several models and aggregate" pattern was used for the opening weekend's 24 exact-score picks (a 3-model panel). It's reserved for the high-leverage, hard-to-redo calls — it's overkill for a routine group-stage fixture the market has already priced cleanly.

(What I did with the panel's unanimous answer is a separate story — I overrode it, backing France + Kane on my own read. That's the human-judgment thread in the [README](../README.md) and [`JOURNAL.md`](../JOURNAL.md), not the methodology. The method's job was to give me the honest EV-max answer; the decision to deviate was mine, and logged as mine.)

---

## 8. Strategy is a function of your position

Pure EV picks the best *scoreline*. It does not tell you how much *variance* to take, and variance is where standings are actually won and lost. That's a function of where you sit.

- **Leading?** Play **low variance** — take the chalk, stay correlated with the field. If you and your nearest colleague both bet the obvious favorite, a shared result doesn't change the gap, and you keep your lead. When you're ahead, the field's mistakes help you; you don't need your own heroics.
- **Chasing?** You *need* variance. Being right *with* the crowd doesn't close a gap — if everyone nails the same pick, everyone moves together and you stay behind. You only gain by being **right where the crowd (and specifically the leader) is wrong.** That means seeking the contrarian-correct spot, not the consensus one. This is the one place the pure-EV stance bends: a chaser rationally trades a sliver of EV for the variance that makes catching up *possible at all*.

Two refinements from the actual endgame (I was overtaken into 2nd after the semis, −3 with the final to play — see the [README](../README.md) results):

- **When 2nd place also pays, the calculus flips again.** My 2nd was fairly secure (a comfortable margin over 3rd). So the frame became: **protect the secure prize, and chase 1st only with moves that can't cost me the money I've already got.** Don't gamble away a paying position for a swing at a better one — unless the swing is genuinely free.
- **Bet correlated with the world where you can still win.** This is the subtle one, and the final made it vivid ([`predictions.json`](../.claude/skills/mondial/predictions.json) match 104). My points only mattered in the branch where the underdog (Argentina) won the final — because the leader held a Spain-champion future that only I could offset by Spain *not* winning. So the max-EV bet (Spain 1-0, the objective best scoreline) and the bet that served my *title* pointed in opposite directions. The resolution isn't to make one bet do both jobs — it's to recognize they're different objectives. So I ran **two tracks at once on the final**: the *objective EV pick* (Spain 1-0, for the points it was worth) and *title-aligned longshots* (rooting for Argentina, an "Argentina scores first" challenge lean), with **challenge stakes kept deliberately modest** so a swing at 1st could never cost me the guaranteed-paying 2nd. Conditioning your bets on *"the branch in which I can still win exists"* is a different optimization than maximizing unconditional EV — and when it collides with "protect the paying position," the paying position wins.
  - **How it resolved:** the EV pick was the *right* pick and still missed — Spain–Argentina was 0-0 at 90 and Spain **won in extra time**, so a 90-minute scoreline bet had nothing to grade against (the [draw-trap](#5-the-draw-trap) in its purest form: a final settled after 90). The one place I *did* score on the night was the low-stakes companion challenge — **Under 2.5 goals, +7.5** — which is exactly the shape the theory predicts: the correlated-with-my-world bets were longshots by design, and the points came from the modest, high-conviction side-bet, not the main pick. The endgame double-threat also sharpened the bar: once France was eliminated, Mbappé became the **outright top-scorer leader**, so the leader (holding *both* Spain-winner and Mbappé futures) had **two live +10s** into the final — only a Messi goal could flip both, and it didn't come. No low-variance move catches a colleague with two independent bets that only one specific event can kill.

### The side-bet pool is zero-sum

The pool has a "challenges" feature — player-vs-player wagers of your *own* points, open to the whole team, winners splitting the losers' stakes. This is a genuinely different game from the prediction ledger, and it took a correction to see why: **a consensus bet closes no gap.** If I and the leader both take the favorite side of an obvious challenge (Over 2.5, both-teams-to-score), we both win points and the gap is unchanged. Gains against a specific colleague come **only** from being contrarian-*correct* — taking the side the leader took wrongly. And because it wagers real points while 2nd place pays, the correct posture from behind-but-in-the-money is *narrow*: only high-conviction contrarian spots, ideally directly opposite a colleague whose side is visible, and never enough to gamble away the paying position. The endgame reasoning for this is spelled out in the final-day reminder inside [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json).

---

## The method in one breath

Read the market as the prior (1X2 for the outcome, exact-score for the scoreline). Take the highest-EV option — which in the group stage is the top scoreline, but in the knockouts is the **modal scoreline of the top outcome**, because knockouts score the 90-minute result and direction points dominate the EV. Watch for the draw-trap. Use ESPN form only to break ties. Ensemble the rare high-stakes one-off calls across models. And let your **standings position**, not just the probabilities, decide how much variance to buy — because a prediction pool is won on expected *points against a field*, not on being right in a vacuum.
