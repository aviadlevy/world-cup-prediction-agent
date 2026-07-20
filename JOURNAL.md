# 📓 JOURNAL — building and running the World Cup Prediction Agent

A chronological devlog: how an office-pool prediction system got built with [Claude Code](https://claude.com/claude-code), and how it survived contact with a real tournament. First person is me (the author). Claude Code is the tool. Other players are anonymized — the pool was ~20 colleagues on a betting app at work.

The short version: I led 🥇 for most of the tournament on exact-score accuracy, got overtaken into 2nd after the semis, and I'm still live for 1st into the July 19 final. The longer version is below, pivots and mistakes included.

A theme runs through all of it: **I never took the agent for granted.** It was a tireless analyst and a flawless executor, but the calls that mattered — a bracket it got wrong, a strategy that depended on my risk tolerance, a scoring rule I'd forgotten to brief it on — came from me arguing with it, not rubber-stamping it. The wins came from the combination.

---

## 2026-06-10 — Build day: requirements, schedule, method

I didn't start by predicting anything. I started by grilling the requirements. What exactly is the pool? Exact scoreline per match, all 104 fixtures, plus two season-long futures (winner, top scorer). Prizes for 1st **and** 2nd. Scoring scales by stage.

Then the build, in layers:

- **The schedule** (`schedule.json`) — all 104 fixtures with Israel kickoff times, verified against multiple sources. This later mattered more than it sounds: Polymarket dates are UTC, so late-night Israel kickoffs land a calendar day earlier in a market slug. Guessing slugs produced 404s; the fix was to stop guessing and enumerate the market tag (`tag_id=102232`) instead.
- **The market recipe** (`polymarket.md`) — verified, no-auth API calls. Polymarket has a per-match 17-way exact-score market. Prices ≈ probabilities. This is the prior for everything.
- **The method** (`SKILL.md`) — pull the market, read prices as probabilities, take the highest-EV scoreline, log it. No vibes. One file, so every future session picks the same way.

The philosophy I locked in on day one: **a liquid market is a better forecaster than my opinion.** The whole system is built to defer to it — which is exactly what made the two places I *didn't* defer interesting.

## 2026-06-10 (evening) — Futures: the panel voted, I overruled it

For the two big one-off calls (tournament winner, top scorer) I didn't want a single model's guess. I ran a **4-model voting panel** — haiku, sonnet, opus, fable — same prompt, aggregate the votes.

The panel was **unanimous, 4-0: Spain to win, Mbappé top scorer.**

I overrode both. I entered **France (winner) and Harry Kane (top scorer)** on my own read — partly gut, partly for the sake of keeping it interesting to have skin in a different outcome. This is logged honestly in `predictions.json` with the note "User's call; 4-model panel voted Spain 4-0." I wanted the record to show where I departed from the machine, not to quietly launder my call as the system's.

(As of the semis: Spain reached the final and France is in the 3rd-place game. The panel had the winner-read closer than I did. Noted.)

## 2026-06-11 — Kickoff: 24 picks in

Generated all 24 matchday-1 exact scores through a 3-model panel, market-first. Some I let ride exactly as the method produced them. Several I overrode at entry:

- **Netherlands–Japan:** panel said 1-0; I entered **1-1**.
- **Iran–New Zealand:** panel said 1-0 (unanimous); I entered **0-0**.
- **England–Croatia:** panel majority said 1-0; I entered **2-1**.

Some of these hit, some missed. That's the point of logging them separately — I wanted to see, over 104 games, whether my interventions actually beat the method. Mostly the method won. But not always, and knowing *which* is only possible because the overrides are marked.

## 2026-06-12 — Automation: crons that don't nag

The system had to run itself while I slept. Two pieces:

1. A **daily 12:03 check** — find any pick kicking off in the next 24h, re-pull the market + ESPN + news, and **notify me only if a bet should actually change** past a threshold. No change, no ping.
2. A **chain of one-shot round reminders** — group → R32 → R16 → QF → SF → final. Each reminder resolves the next round's bracket and schedules the one after it.

The catch I hit immediately: Claude Code crons are **session-only**. Close the session, they're gone. And recurring jobs auto-expire after 7 days. So I saved every job definition to `cronjobs.json` and wired a `SessionStart` hook (`.claude/settings.json`) that checks for missing jobs and offers to recreate them. A fresh session rebuilds its own automation. (I still had to re-create the recurring jobs periodically as they expired — annoying, but at least visible.)

## 2026-06-14 to 06-17 — Group stage running; the daily check earns its keep

The daily cron started catching real bet changes, which is exactly what I built it for:

- **Côte d'Ivoire–Ecuador:** I'd picked 0-0; the market moved and 1-1 became the top scoreline (17% vs 14%). The cron flagged it, I firmed to **1-1**.
- **Iraq–Norway:** picked 1-2; the market shifted to a Norway clean sheet, so **0-2** became top (15.5% vs 8.5%). Revised.

These are the unglamorous wins — no drama, just a bet that would've been stale getting quietly corrected before kickoff.

## ~2026-06-17 — Adding the ESPN form layer

Mid-group-stage I realized the market price alone hides *how* a team is playing. A 3-0 favorite that just got held 0-0, and a 3-0 favorite that just won 7-1, look identical in a single exact-score number — but they're not the same bet.

So I added **ESPN** (`espn.md`, free `fifa.world` endpoints) as a **form layer**: actual results and standings, used as a cross-check, not an override. It breaks ties; it doesn't overrule the market. Concretely, it nudged picks like USA (4-1 opener → 2-0 over 1-0), Germany (7-1 opener → 2-0), and Netherlands–Sweden (both scoring freely → 1-1 scoring draw). Ties broken by form, priors kept.

## 2026-06-24 to 06-28 — Group finale and overrides

The final round of group games is where motivation matters more than raw quality — teams already through rotate, teams needing a result attack. I overrode the pure-market pick several times on that logic (Morocco running it up 3-0 vs an eliminated Haiti; Germany and USA winning 1-0 despite rotation risk; a couple of knife-edge groups I called 0-0). Logged, win or lose.

By the end of the group stage the method was working: I had the most exact-score "bullseye" hits in the pool, and since exact is worth 2–3× a correct outcome, that built a real lead.

---

## 2026-07-01 — Briefing the agent on the knockout scoring rule

A setup cleanup I should have done earlier. One pool rule I already knew but never wrote into the method: **in the knockouts only the 90-minute result counts** — a game won on penalties is scored as a *draw*. Because I hadn't briefed the agent on it up front, the early knockout picks had been over-optimizing the exact score.

The gap showed up concretely on **Belgium–Senegal** (R32): I'd firmed a **1-1** pick, the game finished **3-2 to Belgium after extra time** but was **2-2 at 90**, and my pick still scored — a draw pick cashes the direction points on any level-at-90 result. So I updated the method and the skill: **in knockouts, pick the most likely 90-minute outcome; the scoreline is just the entry form.** My omission, closed.

## 2026-07-02 — Re-scoring the bracket, and catching a wrong opponent

With direction scoring understood, I went back through the live R32 bracket. **Portugal–Croatia:** I'd had 1-1; under direction scoring, Portugal is a 56% favorite vs a 26% draw, so the value is the *Portugal win*, not the draw. Revised **1-1 → 2-1** (Portugal win). I confirmed it deliberately, now that I understood what I was betting.

Separately, I caught the agent resolving a bracket to the **wrong opponent**. It had a tie lined up against Australia — but Australia had gone out on penalties, and the team that actually advanced was **Egypt**. If I'd trusted the resolution blindly I'd have researched and bet the wrong match entirely. (This shows up in the ledger as the Argentina–Egypt note: "Opponent is EGYPT … not Australia.") The agent is fast and thorough; it is not infallible about which team is standing.

## 2026-07-04 to 07-07 — R16: direction-first, and the draw-trap

Every R16 pick now carries an explicit **`outcome`** (France win, Morocco win, Spain win …) with the scoreline demoted to "entry only." This is the method working the way it should once the knockout scoring rule was in play.

It also surfaced the corollary that would cost me later — the **draw-trap**. A heavy favorite dragged to 1-1 at 90 and winning on penalties is a *draw* for scoring, so a "favorite to win" bet **loses**. I started rating every tie for draw-trap risk (Switzerland–Colombia flagged HIGH, Portugal–Spain MED-HIGH, etc.) and, where two even or fatigued sides looked likely to still be level at 90, backing the draw outright.

The daily check kept flipping live toss-ups too: **USA–Belgium** moved to a USA top outcome (home in Seattle, Belgium's legs gone after 120'), so I revised Belgium 1-2 → USA 2-1 and confirmed it.

## 2026-07-09 to 07-12 — QF: the draw-trap bites

**France–Morocco:** I changed my entry 1-0 → **2-0** and it hit exact — France won 2-0 in regulation, direction *and* bullseye. Good day.

**Argentina–Switzerland** was the bad one. I'd flagged it as the **highest draw-trap on the board** — Argentina had been dragged close twice, Switzerland is a park-the-bus penalties specialist (they'd just knocked out Colombia 0-0). My note literally says "same profile that lost M96." I picked the Argentina win anyway. Result: **3-1 Argentina after extra time — level at 90 — MISS.** The trap I'd named in writing still caught me. Naming a risk and pricing it correctly are two different things.

## 2026-07-14 to 07-15 — SF: the deliberate draw, and getting overtaken

**France–Spain:** France a plurality favorite, both teams winning their knockouts in regulation (not draw-prone), France best-rested. France win, 2-1 entry. Correct read.

**England–Argentina** was my **first deliberate draw pick** of the tournament. Both sides came off 120-minute quarter-finals, both had repeatedly been level at 90 in the knockouts (Argentina twice, England once) — textbook draw-trap, so under direction scoring the *draw* is the value. Result: **Argentina won 1-2 in regulation — MISS.** The draw-trap logic was sound; this time the favorite just closed it out.

That semi is where I lost the lead. I'd been up by as much as +17. **Colleague 1**, who'd ridden Spain and Mbappé as futures (the exact pair my model's panel had voted for and I'd overruled) — hit a correct contrarian call and worked the side-bet pool, and I dropped to **2nd of ~20, −3, with the final to play.**

## 2026-07-16 — Bias-free agent, provisional 3rd-place pick, and strategy by position

Three things came together on this day.

**The bias-free agent.** For the final I *needed* one specific team (Argentina) to win — my only realistic path to catching 1st runs through that branch. That is precisely when your own judgment is worthless, and I didn't trust the agent either, because it had my standings in memory and could "helpfully" tell me what I wanted to hear. So I spawned a **clean sub-agent with zero knowledge of my standings or what I was rooting for** — just "predict this final objectively." It came back **Spain-favored (~41.5% vs Argentina ~27%)** — the opposite of what I needed. Painful, but it let me keep "what I hope" and "what I bet" in separate accounts.

**Position-dependent strategy.** Being overtaken flipped the whole calculus, and it took a few corrections to the agent's framing to get it right:

- I had to point out to the agent that the final is the **last game** — you can't "wait and see" the winner-future before betting it. No information is coming.
- I clarified the **side-bet ("challenges") mechanic**: it's zero-sum, needs someone to take the opposite side, it's open to the whole team, and you wager your own points. Consensus bets can't close a gap — only being right where the crowd (and the leader) is wrong does.
- The one that reframed everything: **2nd place also pays.** So the plan became *protect the secure prize, and only chase 1st with moves that can't cost me points.* Leading → low variance, take the chalk. Chasing → you need variance. But when 2nd is money, you don't burn it gambling on 1st.

**The provisional 3rd-place pick.** France–England for 3rd is my **exact-score clawback lever** — 3rd-place exact is worth +10, and I'm chasing −3. The exact market wasn't live yet, so the agent recorded a **provisional** pick (France win, both score) flagged to firm to the market modal the moment it opens (~Jul 18). This is the provisional-loop pattern the whole system uses: never leave a bet stale, never fire before the market exists.

**The final bet, entered honestly:** Spain 1-0, the pure EV-max exact play (Spain's signature scoreline — one goal conceded all tournament). It's logged with the note that it **does not correlate with my title path** — my points only matter in the world where Argentina wins, but the objective max-EV bet is Spain. I bet the EV and left the title chase to the pool bets, deliberately. Hope and EV, kept apart.

---

## 2026-07-19 — Final day: 3rd of ~20

I went in **2nd of ~20, −3**, still live for 1st, with 2nd already paying. Both games broke against me, and I finished **3rd of ~20 on 122.5 points.**

**3rd place — France 2-1 → England 6-4 France.** My clawback lever misfired in the loudest way possible. I'd read England's defense as leaky (they conceded in every knockout game) and firmed France 2-1. The "leaky" read was right — the game was a **10-goal shootout** — but I had the winner backwards. England won 6-4. Right about the goals, wrong about the direction. Miss.

**The final — Spain 1-0 → 0-0 at 90, Spain champions in extra time.** The EV-max pick was the *right* pick and still scored nothing: Spain–Argentina sat 0-0 through 90 and Spain **settled it in extra time**, so a 90-minute scoreline bet had nothing to grade against — the draw-trap in its purest form, on the biggest stage. My one companion challenge, **Under 2.5 goals, won (+7.5)** — the only points I banked on the night, and exactly the shape the theory predicts: the correlated-with-my-world bets were longshots by design, so what paid was the modest, high-conviction side-bet, not the main pick. Argentina never scored; Messi didn't find the net. Kept stakes small throughout to protect the paying spot — but slipped to 3rd anyway, on variance I couldn't have safely covered.

**How I lost the top two, honestly.** Not by picking worse — the opposite. Strip the zero-sum side-bets out of everyone's total and I finished **1st on the prediction ledger (116)**, ahead of Colleague 1 (113) and Colleague 2 (95). Both players who passed me on the final total did it *entirely* in the challenge lane I sat out:

1. **Colleague 2's one-night heater.** Around 86 points, he banked **+32.5 in a single night** (+38.5 total in side-bets) on the final's zero-sum challenge duels and vaulted to 2nd (133.5). The variance lane doing exactly what it does for a chaser — I was leapfrogged, not out-predicted.
2. **Colleague 1 worked the same lane to 1st.** The Spain + Mbappé futures I'd passed on both cashed (+20) — but even with them, his prediction total (113) sat *below* mine. His **+45 in side-bets** is what actually carried him to 158. The futures stung; the side-bets decided it.

And the near-miss on 2nd: my final pick was **Spain 1-0**, and Spain *won* 1-0 — just in extra time, so the 90-minute rule (0-0) scored it nothing. In regulation that's **+10 → 132.5** for me and **−5 → 128.5** for Colleague 2 (who'd bet the 1-1 draw and took +5 on the 0-0): I'd have taken 2nd outright. Right pick, ~15 minutes late.

**The record stands, and I'd take it again.** A market-as-prior method, applied boringly and consistently, gave me the **most exact-score hits in the field by a distance — 19, next best 14** — and led the pool wire-to-wire for weeks. The process was the best in the room. The lesson that landed hard: in a pool where a challenge duel can swing 30+ points in a night, a late leader **can't fully sit still** — pure defense protected my points and still left the door open in the one lane I wasn't playing. Missing my June futures made the final night sting, but the ledger is clear — even with them dead I still had the most prediction points in the pool. I was leapfrogged in the side-bet lane, not out-predicted. Best process, third place — and I'm proud of the process.
