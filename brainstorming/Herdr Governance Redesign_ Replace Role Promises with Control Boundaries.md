# Herdr Governance Redesign: Replace Role Promises with Control Boundaries

**Prepared for:** Herdr operator  
**Purpose:** Make the roster-management process reliable under pressure by moving critical controls from agent instructions into the orchestration layer.

## Executive conclusion

You are not facing a documentation problem or primarily a Hermes problem. You have a **control-system problem**: the same LLMs are both asked to follow the process and allowed to bypass it, declare success, and repair exceptions through undocumented side channels. When an incident creates urgency, the supervisor retains the capability to circumvent the manager, and the manager retains the capability to assert work is complete without independently checkable evidence. In that design, a good written rule competes with a faster shortcut—and the shortcut will recur.

The recommended structural change is a **split-control model**. Make the manager role a small deterministic workflow controller, not Hermes or Claude Code. Hermes may remain the *tactical planner/dispatcher brain*, but it must act only by producing structured commands that the controller validates and executes. Claude Code should be an **exception authority**, not the routine manager and not an implementation backstop. Workers remain the only principals allowed to modify task deliverables. A separate verifier promotes work to `Done` only when reproducible evidence passes declared acceptance checks.

> **Operating rule:** An LLM may propose state changes. Only the controller may commit them. A worker may claim completion. Only an independent verifier may close the card.

This preserves your settled rules—one worker per card, board as ledger, Hermes does not implement, paid Claude is for judgment—while making the dangerous violations impossible or conspicuous rather than merely forbidden.

| Component | Recommended holder | Authority | Explicitly cannot do |
|---|---|---|---|
| **Controller** | Deterministic service / CLI wrapper | Reserve cards, validate health, dispatch, record transitions, issue leases, enforce gates | Write product code; infer success from prose; bypass board |
| **Tactical manager** | Hermes, replaceable | Read sanitized roster state; choose worker; create a complete task packet; request dispatch; triage failures | Directly modify deliverables; mark work done; dispatch outside the controller |
| **Workers** | Codex, Pi, Copilot, Cline, etc. | Execute exactly one leased card and submit evidence | Change card to `Done`; take another card; direct other workers |
| **Verifier** | Separate agent or deterministic CI/test runner, never the worker | Run acceptance checks; inspect diff and artifacts; issue `verified` / `failed` verdict | Implement the fix it is verifying; dispatch work |
| **Supervisor / exception authority** | Claude Code, paid key used sparingly | Declare incident state; replace a failed manager; revise acceptance criteria; authorize termination/requeue | Directly patch ordinary task work; directly correct workers; routinely poll roster |

## 1. The key structural decision: manager is a capability boundary, not a personality

Do **not** promote Claude Code to be the always-on manager. It may reason more reliably than Hermes in some cases, but the principal failure is not Hermes’s identity. It is that an LLM manager and an LLM supervisor are given overlapping, un-gated powers under time pressure. Moving the title to Claude merely concentrates authority and increases cost while preserving the ability to improvise around the ledger.

Likewise, do not make an LLM the sole manager. Keep Hermes only as a replaceable tactical intelligence layer. The stable “manager” should be a deterministic controller with a narrow state machine. It is responsible for checking liveness, acquiring a task lease, assigning a specific worker, recording the full briefing hash, and refusing invalid transitions. Hermes asks; the controller decides whether the request is valid.

This separation means a Hermes stall becomes an ordinary failover event, not a reason for the outer session to start reading panes and patching code. The supervisor replaces Hermes or gives a new manager a complete snapshot. The controller preserves the authoritative state throughout.

### Required state transitions

The board should not permit free-form status edits. Its backend should accept only the following transitions, each with required evidence.

| From state | To state | Actor allowed | Mandatory guard |
|---|---|---|---|
| `Ready` | `Reserved` | Controller | No active lease for the card; worker is eligible |
| `Reserved` | `Dispatched` | Controller | Fresh health check passed; immutable complete briefing recorded; one worker/card constraint holds |
| `Dispatched` | `Working` | Worker through controller | Worker acknowledges correct card, lease, and acceptance criteria |
| `Working` | `Submitted` | Worker through controller | Commit/artifact references plus command output submitted |
| `Submitted` | `Verifying` | Controller | Required evidence bundle is present |
| `Verifying` | `Done` | Verifier through controller | Acceptance checks pass; artifacts exist; verdict is signed/recorded |
| `Verifying` | `Rework` | Verifier through controller | Failure record names a reproducible check and failure output |
| `Any active` | `Blocked` | Worker or controller | A machine-readable blocking reason and last heartbeat are present |
| `Any active` | `Cancelled` or `Requeued` | Exception authority through controller | Incident reason, replacement action, and audit event recorded |

## 2. Turn each repeated failure into a gate

The governing idea is simple: if violating a rule can be cheaper than following it, it is still a suggestion. Controls should be placed where the action occurs—not in a briefing that must be remembered later.

| Repeated failure | Why prose fails | Enforced replacement |
|---|---|---|
| Reading panes one by one instead of `flock-check` | The supervisor has direct access and wants fast local context | Put individual-pane reads behind an **incident capability**. Default UI/API exposes only `flock-check`; controller grants time-limited read access only after an incident is opened. Log every grant. |
| Dispatching to a stale or stalled Hermes/worker | Existence of a session is mistaken for readiness | Controller requires a nonce-based health check no older than, for example, 60 seconds, plus a current heartbeat and a short readiness acknowledgement tied to the pending lease. |
| Patched briefing | Work starts before all exceptions have been embedded | Dispatch accepts a single immutable **task packet** with version, card ID, inputs, constraints, acceptance tests, known-bad board facts, and escalation route. Any change invalidates the lease and requires reissue before work continues. |
| Manager implements directly | The manager has filesystem/write access, or the prompt permits an exception | Run the manager with no workspace write credential and no shell capable of changing deliverables. Its only mutation endpoint is a controller request endpoint. |
| Board/agent says `Done` when work is absent or broken | Status and prose are assertions, not evidence | `Done` is not an agent-writable state. Require an evidence bundle and an independent verification verdict. |
| Supervisor makes scattered direct fixes | The urgency path has no defined alternative | Create an explicit **incident mode**: suspend/revoke the manager lease, preserve a snapshot, requeue or appoint a replacement manager, and dispatch a separate repair card. No informal direct correction path exists. |

## 3. Use a cheap, layered definition of “done”

You do not need heavyweight review for every card. You need a standard that is cheaper than a false completion. The minimum completion gate should be a structured **evidence bundle**, verified by a separate identity.

| Evidence layer | Minimum required item | Typical automation |
|---|---|---|
| **Existence** | Commit SHA or artifact path exists on disk and changes the expected files | `git diff --exit-code <base>..<sha>` plus path allowlist/required-file check |
| **Traceability** | Card ID appears in commit, branch, or submitted manifest | Pre-commit/CI check that links card and revision |
| **Build/test** | Declared commands run and return success, with captured output | CI or a clean verifier worktree executes the card-specific command list |
| **Acceptance** | Objective acceptance assertions pass | Card contains machine-runnable tests, a smoke script, or a reviewer checklist with required evidence |
| **Independent verdict** | A verifier records `pass` or `fail`, including command outputs and artifact hashes | Separate verifier identity calls the sole `Done` transition endpoint |

For tiny, low-risk cards, the verifier can be deterministic CI plus a simple changed-file check. For judgment-heavy cards, use a separate low-cost LLM verifier to inspect the diff after deterministic checks, escalating only ambiguous or high-impact verdicts to Claude. The critical restriction is **non-self-verification**: the same worker or manager cannot write, verify, and close the card.

A suggested completion record is:

```json
{
  "card_id": "H-184",
  "lease_id": "lease_8bf...",
  "worker": "codex-3",
  "revision": "a1b2c3d4",
  "changed_paths": ["src/queue.ts", "tests/queue.test.ts"],
  "commands": [
    {"cmd": "npm test -- queue", "exit_code": 0, "output_hash": "..."},
    {"cmd": "npm run lint", "exit_code": 0, "output_hash": "..."}
  ],
  "acceptance_evidence": {"required_test": "passes", "artifact_hash": "..."},
  "submitted_at": "2026-08-30T...Z"
}
```

The verifier must recreate or inspect these facts instead of trusting this record blindly. The record makes the claim auditable; it does not make it true.

## 4. A real escalation boundary

The supervisor needs authority to repair the *system*, but not permission to silently become a worker. Define two modes before the next incident.

| Condition | Normal mode response | Incident mode response | Prohibited shortcut |
|---|---|---|---|
| Worker is blocked but manager is healthy | Manager requests clarification or requeues through controller | Not needed | Supervisor messaging/patching worker directly |
| Worker has failed verification | Verifier creates `Rework`; manager sends revised complete task packet | Not needed unless repeat failure threshold is crossed | Manager implements the correction |
| Manager is stale, unresponsive, or breaches role boundary | Controller marks manager unavailable when heartbeat/health policy fails | Supervisor opens incident, freezes manager lease, replaces manager from a state snapshot, and requeues affected cards | Supervisor individually operates the roster or patches product code |
| Board/controller is suspect or unavailable | Stop new dispatch; preserve append-only event log | Supervisor declares total outage and may authorize recovery procedure | Treating scrollback or a worker’s report as the new ledger |
| Production/security emergency | Use your explicitly approved break-glass policy | A named emergency implementer can act only with incident ID, bounded scope, and mandatory post-incident reconciliation | Unlogged direct fixes framed as “just this once” |

The distinction is: **the supervisor may change principals, permissions, and policies; the supervisor should not change ordinary task output.** If an emergency genuinely requires an immediate code change, make it a formal break-glass path with a temporary role, incident identifier, audit log, and mandatory verification. This is safer than pretending emergencies never happen and thereby inviting invisible bypasses.

## 5. Fit the paid-Claude constraint to the right work

Your existing cost decision supports this architecture. Routine polling, liveness checks, transition validation, and test execution should be deterministic and therefore consume no high-quality reasoning budget. Hermes or free-pooled agents can plan ordinary assignments and triage expected failures. Claude Code should be invoked for bounded judgment calls: ambiguous verifier disputes, repeated systemic failure, unsafe task decompositions, incident declaration, manager replacement, and changes to policy or acceptance criteria.

| Work class | Recommended execution | Claude use? |
|---|---|---|
| Roster polling and lease expiry | Controller plus `flock-check` | No |
| Routine dispatch request | Hermes submits structured packet | No |
| Deterministic completion checks | CI/verifier script | No |
| Standard diff/acceptance review | Separate verifier agent, then deterministic gates | Usually no |
| Ambiguous acceptance or high-impact risk | Supervisor judgment | Yes |
| Manager breach, systemic stall, or control failure | Supervisor declares/coordinates incident | Yes |

This gives Claude a sharper job with less polling burden: it becomes the person who protects the rules of the game, not another player required to remember them while rushing.

## 6. The next implementation sequence

Do not attempt a major rewrite first. Establish the smallest controls that remove the repeated failure paths, then harden them using observed incidents.

| Priority | Change | Definition of success |
|---|---|---|
| **P0 — immediately** | Remove `Done` permission from workers and managers. Add `Submitted` and `Verifying`; require commit/artifact reference plus verifier result. | A false “done” claim cannot close a card. |
| **P0 — immediately** | Create a dispatch wrapper that requires a recent nonce health check, unexpired worker lease, and a complete immutable task packet. | No dispatch command succeeds without all three items. |
| **P0 — immediately** | Remove manager write access to the workspace and limit its tools to board/controller operations. | Hermes cannot implement even if prompted to do so. |
| **P1 — next** | Make `flock-check` the sole normal roster-read interface; require incident capability for individual pane reads. | Pane-by-pane monitoring is visible and exceptional. |
| **P1 — next** | Implement manager heartbeats, lease TTLs, and automatic `manager-unavailable` state. | A stale manager cannot receive or retain work. |
| **P1 — next** | Add incident-mode commands: freeze manager, snapshot board/events, revoke leases, replace manager, requeue. | A failing manager can be replaced without the supervisor taking over implementation. |
| **P2 — harden** | Run weekly failure-injection drills: stale manager, false board status, missing artifact, failed tests, and task-packet amendment. | Each invalid action is rejected or becomes a logged incident—not an improvised workaround. |
| **P2 — harden** | Review the append-only event log after incidents and change controls, not prose, when the same class repeats. | Recurrence produces a new gate, test, or permission change. |

### First milestone: a one-day viable control plane

If you only implement one slice, make it this: all dispatches flow through `herdr dispatch`; it calls `flock-check`, obtains a fresh challenge-response health check from the selected worker, stores a content-addressed task packet, assigns a single lease, and records an event. Work completion flows through `herdr submit`; it requires revision and test evidence, then places the card in `Verifying`. Only `herdr verify` can set `Done` after re-running required checks. Manager and supervisor prompts are then advisory layers around a path that refuses invalid operations.

That single slice directly addresses the documented chain: dispatch-without-health-check becomes rejected; patched briefing becomes lease-invalidating; manager implementation becomes credential-denied; self-reported completion becomes `Submitted`, not `Done`; and supervisor intervention becomes an explicit manager-replacement incident instead of an untracked bypass.

## 7. A concise operating contract to retain in prompts

Prompts should become short reminders of the controller’s contract—not the primary line of defense.

> **Manager contract:** I may inspect controller-provided state, produce a complete task packet, request dispatch, and triage exceptions. I may not write deliverables, read individual worker panes outside an incident capability, alter `Done`, or amend an active task packet. If I cannot proceed, I raise a structured escalation.
>
> **Supervisor contract:** I protect the control plane. I may declare an incident, suspend/replace a manager, revise policy, and authorize a documented break-glass event. I do not implement routine cards or directly direct workers.
>
> **Worker contract:** I own one leased card. I submit reproducible evidence; I do not self-close work.

## Final recommendation

Keep **Hermes as the tactical manager**, but demote its authority from direct operator to structured decision-maker. Keep **Claude Code as an escalation-only supervisor**, explicitly tasked with protecting the control plane rather than patching the product. Introduce a **deterministic controller** as the actual holder of dispatch and state-transition authority, and introduce an **independent verification gate** as the only route to `Done`.

The measure of success is not that the next manager prompt says the right words. It is that, during the next stressful failure, the easy path through the tooling is still the compliant path—and the old shortcut is denied, lease-invalidated, or turned into an auditable incident.

## Basis

This memo is a governance and systems-design analysis grounded in the incident history and constraints supplied in *Herdr Governance: Who Should Manage the Roster, and How Do We Make It Stick?* (provided by the operator, 2026-08-30). No external factual claims are necessary for the recommendations above.
