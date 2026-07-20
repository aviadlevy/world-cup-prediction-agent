# The automation: making a tournament run itself

A 104-fixture tournament runs for five weeks. Bets need re-checking as lineups leak and markets move; brackets need resolving as rounds finish; picks made a week early need firming when their market opens. None of that should require me to remember to do it. So the [`/mondial` skill](../.claude/skills/mondial/SKILL.md) is wrapped in an automation layer built entirely out of [Claude Code](https://docs.claude.com/en/docs/claude-code) primitives — cron jobs, a session hook, sub-agents, and persistent memory.

This doc is how those pieces fit, including the honest wart: **Claude Code crons are session-only**, and most of the design below exists to work around that.

---

## The pieces

| Piece | File | Job |
|---|---|---|
| Daily bet-revision check | [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) | Re-check upcoming picks every day; notify only on a real change |
| Round-reminder chain | [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) | Resolve each round's bracket, schedule the next reminder |
| Restore hook | [`../.claude/settings.json`](../.claude/settings.json) | On session start, offer to recreate any missing cron |
| Memory | (outside the repo) | Standings/scoring/strategy notes so fresh sessions have no cold start |
| Sub-agents | (spawned on demand) | Parallel per-tie research; a bias-free agent for the final |

---

## 1. The daily bet-revision cron — notify only on change

The workhorse. A recurring job at **12:03 every day** (`cron: "3 12 * * *"` in [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json)). Its whole design principle is **quiet by default** — it should be invisible unless it has something worth interrupting me for. The flow:

1. Get the current Israel time (`TZ=Asia/Jerusalem date`).
2. Read [`schedule.json`](../.claude/skills/mondial/schedule.json) and [`predictions.json`](../.claude/skills/mondial/predictions.json).
3. Find every match kicking off in the **next 24 hours that already has a recorded pick.** If there are none, reply with a single line and stop.
4. For each such match, re-pull the current [Polymarket](../.claude/skills/mondial/polymarket.md) prices (1X2 + exact-score), the current [ESPN](../.claude/skills/mondial/espn.md) standings and results, and a quick web search for fresh team news (lineups, injuries, rotation for already-qualified sides).
5. **Compare against the recorded pick.** A change is warranted *only* if the pick's scoreline is no longer consistent with the market's most-likely outcome, **or** a materially better scoreline now leads by more than 3 points, **or** major team news invalidates the pick's premise. (The job carries the knockout scoring rule with it: for knockout picks it compares the pick's *implied outcome* to the market's top 1X2 outcome, not the exact scoreline — see [methodology §3](methodology.md).)
6. **If nothing changed:** one line — `"World Cup check: picks still good for <matches>"`. No notification.
7. **If something changed:** fire a **push notification** (via the `PushNotification` tool, loaded on demand through `ToolSearch`) titled *"World Cup: bet change recommended"*, and print a short table of match / old pick / new pick / reason.

The threshold in step 5 is the point. Markets jitter constantly; a job that pinged me on every 1-point wiggle would be noise I'd learn to ignore. Notifying **only when a bet should actually change** is what makes the automation trustworthy — a ping *means* something. It caught real revisions during the group stage (see [`predictions.json`](../.claude/skills/mondial/predictions.json) matches 9, 18, 94, where picks flipped when the market moved) and stayed silent the rest of the time.

Crucially, **the cron never edits `predictions.json` itself.** It recommends; I confirm. Human-in-the-loop is a design invariant, not an accident — the agent proposes the change and I make the call. (The overrides that came *from* those confirmations are the human-judgment thread in the [README](../README.md).)

---

## 2. The round-reminder chain

A separate mechanism from the daily check: a **chain of one-shot reminders** that walks the tournament round by round — group round 2 → round 3 → Round of 32 → R16 → QF → SF → final. Each reminder, when it fires, does two things:

1. **Resolves the next round's bracket** — the knockout fixtures carry placeholders (`W74`, `1A`) in [`schedule.json`](../.claude/skills/mondial/schedule.json) until the qualifying games are played; the reminder web-searches the actual pairings and writes the real team names back (keeping the placeholder in a `was` field, per the skill).
2. **Schedules the next reminder** in the chain.

A chain rather than one big recurring job because the rounds are irregularly spaced and each one's work depends on the previous round's results existing. It's a self-propagating to-do list: each link does its task and creates the next link. Resolving brackets ahead of time is also where the agent is most fallible — one entry in [`predictions.json`](../.claude/skills/mondial/predictions.json) records a wrong opponent resolved for a knockout tie (Australia vs. Egypt), caught by me, not the agent. Another reason the human stays in the loop.

The live tail of the chain is the **final-day prep reminder** in [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) (`cron: "55 9 18 7 *"`, a one-shot for the eve of the last two games). It's the most elaborate single job in the system: firm the 3rd-place exact-score pick from the now-live market, re-verify the final pick, and — because standings math is now everything — proactively raise the side-bet ("challenges") pool and advise which contrarian wagers to join or open. It encodes the full endgame reasoning from [methodology §8](methodology.md).

---

## 3. The session-only limitation, and the restore pattern

Here's the wart. **Claude Code cron jobs live only as long as the session that created them.** Close the session, the jobs are gone. For a five-week project spanning dozens of sessions, that's fatal to any "set it and forget it" plan — the daily check would evaporate the first time I closed my laptop.

The workaround is a **save + restore** pattern:

**Save.** Every cron's full definition — `name`, `cron`, `recurring`, `createdAt`, and the complete `prompt` — is written to [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json). This file is the durable source of truth for "what jobs *should* exist." It's version-controlled and survives anything.

**Restore.** A `SessionStart` hook in [`../.claude/settings.json`](../.claude/settings.json) fires on every new session. It checks that `cronjobs.json` exists and, if so, injects a context note telling the agent to:

> Compare `CronList` against `.claude/skills/mondial/cronjobs.json` and offer to restore any missing job via `CronCreate` with the stored cron/recurring/prompt.

So the loop on every session start is: the hook reminds the agent, the agent lists live crons, diffs them against the saved definitions, and offers to recreate whatever's missing. The definitions in `cronjobs.json` are written to be **re-createable verbatim** — that's why each job's entire prompt is stored inline, not summarized. The skill itself also documents this restore procedure ([`SKILL.md`](../.claude/skills/mondial/SKILL.md), the `cronjobs.json` bullet), so even a session that skips the hook can recover.

The hook is deliberately a *pure notifier* — it shells out to `printf` an `additionalContext` payload and nothing more. It never creates a cron itself (a hook can't reliably drive the `CronCreate` tool), and it never acts without offering — the recreate is proposed to me, not done silently.

---

## 4. The 7-day expiry

A second wrinkle stacked on the first: **recurring crons auto-expire 7 days after their `createdAt`.** So even inside a single long-lived session, the daily check would quietly die after a week. Both [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) and the skill call this out explicitly, and the restore procedure treats it as first-class: when recreating a recurring job you **update its `createdAt`** (resetting the 7-day clock), and you **skip one-shots whose date already passed** (a fired reminder shouldn't be resurrected). In practice this meant the daily check had to be re-created periodically over the tournament — which the save/restore loop made a one-line confirmation rather than a from-scratch rebuild.

---

## 5. The provisional-pick firming loop

Exact-score markets for later fixtures don't open until a few days before kickoff — but I want a pick *on record* for every match well ahead of time, both to enter it in the app and so nothing is ever left blank. So the two automations combine into a firming loop:

1. When a match's market isn't live yet, the agent records a **provisional** pick — the most-likely outcome by team strength plus a heuristic scoreline — and flags it `"provisional": true` in [`predictions.json`](../.claude/skills/mondial/predictions.json).
2. The **daily cron** (§1) then firms it to the live market the moment that market opens, and — per its notify-on-change rule — pings me only if the firmed pick differs from the provisional.

You can watch this happen in the ledger: matches 45–48 were entered before their group's opener was even played ("Group L opener not yet played at entry; daily-check to revisit"); matches 80, 84, 85 carry notes like "Market went live 2026-06-30/07-02: confirmed as market top." Match 103 (the 3rd-place game) is a provisional still waiting for its market as of the last update. Bets are never stale, and I'm never spammed for a firming that didn't change anything.

---

## 6. Memory — no cold starts

Cron jobs handle *doing*; memory handles *knowing*. Persistent memory notes hold the state a fresh session would otherwise have to re-derive: the current standings, the scoring structure, and the strategic situation ("2nd of ~20, −3, here's the plan"). A session days later opens already knowing where I stand and what the plan is, instead of re-reading the whole tournament. (These memory files live outside the repo, so they're referenced but not published here — the final-day reminder in [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) points at `mondial-standings.md` for exactly this reason.) Memory is what turns a pile of separate sessions into one continuous project.

---

## 7. Sub-agents — for breadth, and for honesty

Two distinct uses, and the second is the interesting one.

**Parallel per-tie research (breadth).** For a round of knockout ties, the agent spawns one sub-agent per tie, each doing the full deep dive — form, injuries/suspensions, market state — concurrently. Fan-out research: each agent owns one fixture, results come back in parallel, and the main thread assembles the picks. It's a throughput trick — eight independent fixtures researched at once instead of in sequence.

**A bias-free agent for the final (honesty).** This is the one I'm proudest of as a piece of design. Going into the final I *needed* a specific result — Argentina had to win for my title path to stay alive ([methodology §8](methodology.md), [`predictions.json`](../.claude/skills/mondial/predictions.json) match 104). That's precisely the moment your own judgment is worthless: I *wanted* an answer, so I couldn't trust myself — or an agent primed with my standings — to read the game objectively. So I spawned a **clean sub-agent that knew nothing** about my position, what I was rooting for, or the pool at all — just *"predict this match objectively."* It came back Spain-favored (~42% vs. Argentina ~27%) — the opposite of what I wanted to hear. Painful, and exactly the point: isolating the objective forecast from the interested party let me keep "what I hope" and "what I bet" cleanly separate. Context is usually the thing you want to give an agent; here the *absence* of context was the feature.

---

## What made it hold together

The automation isn't clever individually — a daily cron, a reminder chain, a hook, some sub-agents. What made it work across five weeks and dozens of sessions is that each fragile primitive was backed by a durable file: session-only crons backed by [`cronjobs.json`](../.claude/skills/mondial/cronjobs.json) + the [restore hook](../.claude/settings.json), transient session state backed by memory, and the method itself backed by [`SKILL.md`](../.claude/skills/mondial/SKILL.md). The primitives can all evaporate; the files bring them back. And every automated recommendation stopped one step short of acting — the agent proposed, I confirmed. That single invariant is what let me trust an automated system with something I actually cared about winning.
