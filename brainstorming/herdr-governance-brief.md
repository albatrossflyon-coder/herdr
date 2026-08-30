# Herdr Governance: The Architecture Was Already Decided. Why Isn't It Being Followed?

## Context

`herdr` is a terminal multiplexer that runs multiple AI coding agents (Claude Code, Codex, Pi, Copilot, Antigravity, and ~10 others) as tenants in one shared workspace, coordinated through a shared board. This is not a "how should we design this" question — a real architecture was already decided six days before the incident below, converged on independently three ways (ChatGPT, Gemini, and herdr's own project docs). It has not held up in practice, and the gap turns out to be specific and fixable, not a vague "AI agents are unreliable" problem.

## The decided architecture (2026-08-24, six days before the incident below)

The question asked at the time: is "one Hermes manager supervising N agents" really one task, or N+1 tasks in disguise? Answer, converged three ways: **it's N+1 unless state is externalized.** The ceiling isn't agent count, it's how much live state the manager holds and re-reasons over in its own context. Four pieces:

1. **Ledger** — `herdr-board` (external kanban plugin, not the manager's own memory). One task = one card. **Built, installed, verified working.**
2. **Observer** — `flock-check`, read-only, answers "who's working/idle/blocked" in one call instead of manual pane-by-pane polling. **Built and installed — but see the flaw below.**
3. **Manager (Hermes)** — exception-driven only: poll the board digest, pull raw pane output only when a card goes blocked/failed, never hold full live-roster-state in its own conversation. **Design is right; not consistently followed.**
4. **Dispatcher** — explicitly scoped as: extend the existing `~/.local/herdr-watchdog.py` into a claim/heartbeat/stale-worker-reclaim/retry role. This is the one piece meant to make the rules above mechanically enforced rather than dependent on an LLM remembering them under pressure. **Logged as "not yet done" on 2026-08-24. Confirmed still not done: the live file on disk is 65 lines, does one thing (flag an agent whose `state_change_seq` hasn't moved in 60s to a log file), and has zero claim/heartbeat/reclaim/retry logic.**

## A real, newly-identified flaw in piece 2: the observer is coupled to the thing it's supposed to watch

`flock-check` was installed via `npx skills add tonbistudio/flock-check -g -a hermes-agent -y` — registered as a **skill inside Hermes's own agent process**, not a standalone pane or process. That means the tool meant to detect "Hermes has stalled/crashed/gone off-script" is only reachable *through Hermes itself*. If Hermes stalls, the observer stalls with it. **This should run independently of Hermes** — its own pane, or invoked directly by the dispatcher/supervisor — so it keeps functioning precisely when the thing it's watching fails.

## A real, herdr-native limitation on signal quality (not a process failure — a tooling ceiling)

herdr detects agent state two ways: real lifecycle events (accurate, native) for Pi/OMP, Kimi, OpenCode, Kilo, MastraCode — versus a screen-manifest visual guess (herdr screenshots the pane bottom, pattern-matches against TOML rules) for everything else, **including both Claude Code and Hermes Agent itself.** herdr's own docs say blocked-detection is deliberately conservative ("intentionally strict to avoid misclassifying a running agent as blocked") — it leans toward *not* flagging "waiting for your approval" when ambiguous.

Consequence: decoupling the observer from Hermes (the fix above) solves the single-point-of-failure problem, but not the signal-quality problem — an independent observer watching Hermes's pane still only gets a screen-inferred guess about Hermes, not a confirmed lifecycle event, because Hermes itself is in the low-accuracy bucket. **One candidate fix, offered for the crew to critique, not yet decided:** have Hermes (and any other screen-manifest-only manager/supervisor role) write its own heartbeat timestamp to a file on every poll cycle, checked externally by the dispatcher/observer for staleness — a self-reported liveness signal that doesn't depend on herdr's visual guess at all. Open question: is this sufficient, or does a self-reported heartbeat have the same "the patient takes their own pulse" problem as self-reported task completion (see failure mode 3 below)?

## What actually happened last night (2026-08-30) — traced against the plan above, not a vague "it went badly"

Three specific, already-documented rules were violated, continuing patterns that predate this incident:

1. **Flock-check-first was skipped.** `herdr.md` §1a states explicitly: the first roster-status check of any session must be flock-check, never a direct read/dispatch to an individual agent — a rule already written down after two *earlier* incidents (2026-08-24, then again 2026-08-25) specifically because it had already recurred once. Last night, an existing hermes-manager was found and dispatched to directly, a **third** occurrence of the same skip.
2. **No health check before dispatching to a found agent, then the manager was told to implement directly.** Both are documented rule categories from earlier incidents (§1b, §5 in herdr.md), both broken again. The found hermes-manager turned out to be a stale session roughly a week and a half old — genuinely stalled, not idle — and was dispatched to before that was checked.
3. **Underneath all of it: no deterministic dispatcher exists**, so nothing mechanically enforced "don't resume a session this old," "check key balance before assigning work" (a real insufficient-funds error on Hermes's own model mid-session required a manual key swap), or "reclaim a stalled claim automatically." Every one of these stayed a prose rule an LLM had to remember correctly under time pressure, for the third time running on the flock-check rule specifically, and it didn't hold.

**The honest root cause: this was not "the agent ignored the plan."** The plan's enforcement layer — the one piece designed specifically to convert prose rules into mechanical ones — was never built. Everything since has been a variation on the same gap.

## What's already decided — don't re-litigate these with the crew

- herdr-board as external ledger, one task = one card. Working.
- Hermes as exception-driven manager, not holding full roster state. Right design, inconsistently followed due to the missing enforcement layer, not a wrong design.
- flock-check as the roster-status mechanism (needs decoupling from Hermes per above, not redesigning).
- The N+1 diagnosis itself — three independent sources already converged on this; don't re-derive it.

## The actual ask for the brainstorm crew

This is a review/harden pass on a specific, already-scoped design — not a from-scratch "how should multi-agent orchestration work" question:

1. **Dispatcher spec review.** The unbuilt piece needs: claim (an agent can't be assigned work without registering the claim externally), heartbeat (liveness must be provable, not inferred), stale-worker-reclaim (a threshold-based rule for "this session/claim is too old, don't resume it — start fresh"), and retry (with backoff, on transient failures like a 429/insufficient-funds error). What's missing from that four-part spec? What's the right staleness threshold, and should it differ by agent kind given the lifecycle-hook accuracy gap above?
2. **Observer independence.** Is "its own pane, polled by the dispatcher" sufficient, or does a truly independent observer need to run outside herdr's own process tree entirely (a genuinely separate OS-level process) to survive a herdr server crash, not just a Hermes crash?
3. **The heartbeat-for-unobservable-agents idea above** — sufficient, or does self-reported liveness need independent corroboration the same way self-reported task completion turned out to need it?
4. **Verification gate for "done."** Separate from the dispatcher: multiple confirmed cases of a worker or the manager marking a card done when the actual artifact doesn't exist, or the board's own `status` field lagging reality. What's a cheap, structural check before a card can move to Done that doesn't depend on any agent's own claim?
5. **Enforcement mechanism generally.** Given a written rule has now failed to prevent its own repeat violation *three times* on the same specific rule (flock-check-first), should the highest-risk rules be moved out of prose entirely and into something the dispatch call itself can't skip (a hook, a wrapper, a gate) — and if so, which of the four dispatcher pieces above is the highest-leverage one to build first?
