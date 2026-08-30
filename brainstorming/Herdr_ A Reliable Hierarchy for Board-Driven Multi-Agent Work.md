# Herdr: A Reliable Hierarchy for Board-Driven Multi-Agent Work

**Prepared for the Herdr operator**  
**Date:** 2026-08-30

## Executive answer

The missing piece is not another manager prompt. It is a **separation of duties around the Herdr board**.

Your current design has been asking Hermes to be too many things at once: manager, observer, pane operator, planner, and sometimes builder. At the same time, the outer Claude Code session still has enough access to start or manipulate panes directly. That creates a competition for control. Hermes may be called the manager, but the system does not make it the only manager.

The hierarchy I recommend is:

> **You → Outer Claude Supervisor → Hermes Manager → Herdr Board and Dispatch Controller → Worker Panes**
>
> **Observer is a separate read-only service beside Hermes, not inside Hermes.**
>
> **Verifier is a separate completion role, not Hermes and not the worker.**

The central rule is:

> **The board is the source of task truth. The observer is the source of runtime truth. Hermes is the source of dispatch decisions. The controller is the only component allowed to change operational state. Workers do the implementation.**

This resembles established orchestrator-worker designs, but with an additional observer and durable event layer because Herdr is coordinating long-running terminal sessions. Anthropic describes an orchestrator-worker pattern in which a lead agent plans and delegates to specialized subagents, while also warning that coding work often has more dependencies and less parallelism than research work [1]. Anthropic’s managed-agent architecture further separates the reasoning “brain” from the execution “hands,” with durable events outside the agent so failed sessions can be replaced instead of manually nursed back to health [2]. LangGraph similarly separates thread checkpoints from durable application data [3], while Temporal treats the workflow event history as the durable record from which execution can resume after failure [4].

## 1. The proposed hierarchy

| Level | Component | Job | Must not do |
|---|---|---|---|
| 0 | **Human operator** | Set goals, approve major plan changes, authorize exceptional recovery | Micromanage panes during normal operation |
| 1 | **Outer Claude Code supervisor** | Translate your intent into a plan, review board-level reports, decide whether Hermes must be replaced | Start workers, wake panes, assign cards, read individual panes, or implement ordinary cards |
| 2 | **Hermes manager** | Convert the plan into board cards, select eligible workers, dispatch one card per worker, poll results, requeue failures | Build product code, serve as observer, mark work complete by assertion, or silently change the plan |
| 2A | **Observer / `flock-check`** | Read runtime facts: pane alive, process state, heartbeat, recent output, assigned card, idle/stalled/exited status | Dispatch, implement, alter the board, or give tactical instructions |
| 2B | **Verifier** | Check submitted work against tests, files, diffs, and acceptance criteria | Fix the work it verifies or close its own failed work |
| 3 | **Board / ledger** | Store cards, leases, dependencies, statuses, assignments, acceptance criteria, and event history | Accept arbitrary free-form status edits |
| 3A | **Dispatch controller** | Enforce leases, one-card-per-worker, valid state transitions, permissions, and complete task packets | Make architectural judgments or write source code |
| 4 | **Workers** | Implement one assigned card in an isolated worktree or branch and submit evidence | Select their own next card, manage other workers, or self-close cards |

### The critical correction

**`flock-check` should not be a spun-off Hermes subagent if it is intended to be an observer.** It should be a separate, read-only role or deterministic service. If Hermes owns both management and observation, Hermes can become blind, overloaded, or tempted to report its own interpretation as fact. The observer should report what it sees; Hermes should decide what to do about it.

The observer can still be powered by an agent if the runtime interpretation is difficult, but it must have read-only permissions and a separate identity. It should publish observations to the board or event stream, not send commands to workers.

## 2. What each layer should see

The system is currently failing because information and authority are mixed together. Use different views for different roles.

| Role | Board access | Runtime access | Write access |
|---|---|---|---|
| Outer Claude | Read summary and incidents; propose plan changes | Observer summary only | Plan/intention records; incident declaration through controller |
| Hermes | Read full operational board and observer feed | Observer API, not arbitrary pane scraping | Dispatch requests, requeue requests, structured status requests |
| Observer | Read panes/processes/heartbeats | Full runtime read access | Observation records only |
| Verifier | Read card, revision, artifacts, test environment | Clean verification environment | Verification verdict only |
| Worker | Read its card and assigned context | Its own pane/worktree | Code/artifacts and evidence submission |
| Controller | Full state and event log | Health-check interface | All authoritative state transitions |

The outer Claude session must not merely be told not to use pane commands. Its available tools should be narrowed, and its `PreToolUse` hook should block direct pane-control commands. Claude Code’s official hooks support lifecycle events and can block a tool call before it runs [5]. That is the appropriate enforcement point for the outer boundary.

## 3. The Herdr board needs four distinct kinds of truth

The board should not be only a Kanban list with a status column. It needs to distinguish task intent, assignment, runtime observation, and completion evidence.

| Board record | Meaning | Example |
|---|---|---|
| **Card** | What needs to be accomplished | “Add retry handling to queue worker” |
| **Lease** | Which worker currently owns execution and until when | `worker-03`, lease expires in 20 minutes |
| **Observation** | What the observer sees in the runtime | “Pane alive, heartbeat 14s ago, process running” |
| **Evidence** | What the worker claims it produced | Commit SHA, changed files, test output |
| **Verification** | What an independent verifier confirmed | “Tests pass in clean worktree; acceptance criteria satisfied” |
| **Event** | What happened and when | `dispatch_requested`, `acknowledged`, `pane_exited`, `requeued` |

Do not collapse these into one `status` field. A pane can be alive while Hermes is stalled. A worker can claim `done` while the commit is missing. A card can be marked `working` while the worker process has exited. The board needs to show these as separate facts.

A recommended visible card might read:

```text
Card: H-184
Task: Add retry handling to queue worker
Board state: dispatched
Lease: worker-03 / lease-8bf / expires 14:32
Observer: process alive; last heartbeat 18s ago; last output 7s ago
Worker claim: none yet
Verification: not started
Next manager action: wait for acknowledgement, then poll
```

## 4. The operating loops

### Outer Claude supervisor loop

The outer session should operate slowly and at the board level:

1. Receive your goal and write one complete plan to the board.
2. Ask Hermes to execute the plan.
3. Read the observer summary and Hermes’s structured report.
4. Make strategic decisions only: clarify, reprioritize, pause, or replace Hermes.
5. If Hermes fails, invoke manager recovery. Do not operate the workers.

Its normal question should be **“Is Hermes managing?”**, not **“What is worker pane 4 doing?”**

### Hermes manager loop

Hermes should run a mandatory loop rather than waiting for the outer Claude session to prompt it repeatedly:

1. Read the current board snapshot.
2. Read the observer’s latest roster snapshot.
3. Find unblocked cards and eligible idle workers.
4. Dispatch exactly one complete task packet to each selected worker.
5. Confirm acknowledgement.
6. Poll for progress and update the board through the controller.
7. Route blocked or failed work to clarification, rework, or requeue.
8. Submit completed work to the verifier.
9. Continue until no actionable card remains or a structured blocker is raised.

A Hermes heartbeat should include the last observation timestamp, last dispatch attempt, cards under management, and the next intended action. A heartbeat that only says “still working” is not enough.

### Observer loop

The observer should run independently and repeatedly. It should inspect each pane and publish facts such as:

```text
pane: worker-03
agent: Codex
process: alive
assigned_card: H-184
runtime_state: working
last_heartbeat: 18 seconds ago
last_output: 7 seconds ago
board_claim: dispatched
confidence: high
```

The observer should use explicit rules for runtime states:

| State | Suggested condition |
|---|---|
| **Idle** | Process is alive but no active lease/card and no meaningful recent work |
| **Working** | Active lease, process alive, recent heartbeat or output |
| **Blocked** | Worker reports a structured blocker or controller records a dependency block |
| **Stalled** | Active lease but heartbeat/output exceeds the configured threshold |
| **Exited** | Pane or process is gone unexpectedly |
| **Unknown** | Signals disagree or the observer cannot obtain a reliable reading |

The observer should never infer “successful” from text such as “I finished.” It can report the text as an observation, but completion remains a verifier decision.

## 5. What successful external patterns contribute

The sources I found do not describe Herdr specifically, but they converge on several design principles that directly apply.

| External pattern | Source lesson | Herdr adaptation |
|---|---|---|
| Orchestrator-worker | A lead agent decomposes work and delegates to specialized agents [1] | Hermes owns decomposition and dispatch; workers receive bounded cards |
| Separate context and tools | Subagents need clear objectives, output formats, and tool boundaries [1] | Give Hermes manager tools, observer read tools, and workers implementation tools |
| Brain/hands separation | The reasoning harness should call execution resources through a stable interface; failed resources can be replaced [2] | Hermes requests dispatch; a controller operates panes; replacement does not require improvisation |
| Durable checkpoints and stores | Current run state and longer-lived shared data should be persisted separately [3] | Keep live Hermes/observer run state separate from the durable board and event history |
| Event-history replay | Durable workflows resume from recorded events after failure [4] | A new Hermes should reconstruct assignments from board events, not from pane appearance or memory |
| Pre-tool enforcement | Claude Code hooks can block tool calls before execution [5] | Block outer Claude from direct worker-pane operations during normal mode |
| Scoped subagents | Claude Code supports separate context, tool controls, permissions, and hooks for subagents [6] | Give Observer read-only tools and Hermes no implementation tools |

Anthropic also reports that vague subtask descriptions lead to duplicated work and gaps, and recommends explicit objectives, output format, tool guidance, and task boundaries [1]. Therefore, every card dispatched by Hermes should include acceptance criteria and a required report format, not just a conversational instruction.

## 6. How the process should work in practice

Suppose you give the outer Claude session this direction: “Implement the queue retry plan using the Herdr team.” The correct flow is:

| Phase | Responsible component | Action |
|---|---|---|
| Plan | Outer Claude | Writes the strategic plan and asks Hermes to execute it |
| Decompose | Hermes | Creates cards H-184, H-185, and H-186 with dependencies and acceptance tests |
| Observe | Observer | Reports which panes are alive, idle, working, or stalled |
| Dispatch | Hermes through controller | Assigns H-184 to one eligible worker with a lease and complete packet |
| Execute | Worker | Implements only H-184 and submits revision/evidence |
| Monitor | Observer | Reports runtime facts; does not interfere |
| Coordinate | Hermes | Polls, handles blockers, and dispatches the next eligible card |
| Verify | Verifier | Runs tests and inspects artifacts independently |
| Close | Controller | Moves H-184 to `Done` only after verification |
| Report | Hermes | Summarizes board progress to outer Claude |

If the selected worker is idle, Hermes handles it. If the selected worker is stalled, Hermes requeues it. If Hermes itself stops managing, the observer/watchdog reports **manager stalled** to the outer Claude session. The outer session restarts Hermes from the durable board state. It does not start the worker itself.

## 7. Hooks, watchdogs, and the board: how they fit together

These are complementary, not competing, mechanisms.

| Mechanism | Role in Herdr |
|---|---|
| **Board** | Durable task and assignment ledger |
| **Observer / `flock-check`** | Point-in-time runtime observation |
| **Watchdog** | Repeatedly checks observer and Hermes heartbeats; detects stale or broken components |
| **Hooks** | React to lifecycle/tool events; block forbidden outer-session commands and record events |
| **Controller** | Enforces permissions, leases, and valid transitions |
| **Hermes** | Makes dispatch and recovery decisions |
| **Outer Claude** | Makes strategic decisions and replaces a failed Hermes |

A hook cannot replace `flock-check`, because a hook usually fires on an event while `flock-check` answers “what is the state of every pane right now?” Conversely, `flock-check` cannot enforce authority by itself. It can report that workers are idle, but only Hermes should be allowed to respond by dispatching them.

## 8. The implementation game plan

### Phase 1: Stop the role collision

Immediately define two permission profiles. The **outer Claude profile** can read the board summary, read observer reports, write plans, and invoke manager recovery. It cannot invoke pane-control commands. The **Hermes profile** can read the full board and observer feed and request dispatch/requeue operations. It cannot write implementation files or run worker commands. The **observer profile** can inspect panes and processes but cannot dispatch or modify cards.

Add a `PreToolUse` hook to the outer Claude session that blocks direct commands matching pane creation, pane selection, pane sending, worker restart, or worker filesystem repair. The block message should say: **“Direct worker control is disabled in supervisor mode. Ask Hermes or invoke manager recovery.”** This converts the most important role rule from a prompt preference into a tool-level refusal [5].

### Phase 2: Separate `flock-check` from Hermes

Run the observer as its own process or independent agent identity. Give it a read-only runtime interface. It should publish a roster snapshot with timestamps and confidence. Hermes consumes this feed. Hermes no longer has to be both “the manager” and “the observer.”

If you retain an LLM-powered observer, restrict its tools to pane/process inspection and structured observation writes. Do not let it send messages to workers. The simplest and most reliable observer may be deterministic shell/process inspection plus a small parser, with an LLM used only when a runtime state is ambiguous.

### Phase 3: Make Hermes a loop, not a conversation

Hermes should start with a complete briefing containing the plan, known board facts, current observer snapshot, card schema, dispatch rules, worker roles, and escalation policy. It should then be required to produce one of two outputs after each cycle:

```text
DISPATCHED: card=H-184 worker=worker-03 lease=...
```

or:

```text
MANAGER_BLOCKED: reason=No healthy eligible worker; next_check=60s
```

An output such as “I’ll monitor the situation” is not a valid manager result. The watchdog should flag Hermes if it has not produced a dispatch, board update, or structured blocker within the agreed interval.

### Phase 4: Add board leases and event history

Every assignment should have a lease ID, worker ID, card ID, start time, expiry time, and current observer snapshot. Record events such as `plan_received`, `card_created`, `dispatch_requested`, `dispatch_acknowledged`, `heartbeat`, `blocked`, `pane_exited`, `requeued`, `verification_started`, and `manager_replaced`.

Keep the event history outside Hermes. This follows the durable-state pattern in which a failed orchestration session can be replaced and resumed from events [2] [3] [4].

### Phase 5: Add verification after orchestration is stable

Once Hermes is consistently dispatching and the outer Claude session is staying out of the panes, add the independent `Submitted → Verifying → Done` gate. This is important, but it should not distract from the more immediate problem: workers are idle because the manager loop is not reliably running and the outer supervisor is competing with it.

## 9. The failure-recovery policy

The recovery policy should be written as a small decision table and implemented in the controller.

| Detection | Who detects it | Automatic action | Outer Claude action |
|---|---|---|---|
| Worker pane exited | Observer | Mark lease failed; preserve card; notify Hermes | None unless Hermes fails to recover |
| Worker stalled | Observer/watchdog | Notify Hermes; do not start replacement automatically unless policy allows | None |
| Hermes has no heartbeat | Watchdog | Mark manager stale; stop new dispatches | Replace Hermes from board/event state |
| Hermes has heartbeat but no dispatch activity | Watchdog | Emit `manager_ineffective` event | Restart or rebrief Hermes; do not touch workers |
| Outer Claude attempts direct pane control | Pre-tool hook | Block command and record violation | Continue in supervisor mode |
| Board/controller unavailable | Controller monitor | Freeze dispatch; preserve local event buffer | Declare system incident and restore control plane |

The rule should be: **worker failures are Hermes’s problem; Hermes failures are the outer Claude supervisor’s problem; controller failures are the system’s incident problem.** No layer should solve a lower layer’s problem by silently becoming that layer.

## 10. Success criteria for a two-day trial

Run a small controlled trial with perhaps three cards and two workers. Do not judge success by whether the agents sound confident. Measure the control flow.

| Metric | Target for trial |
|---|---|
| Outer Claude direct worker-control attempts | Zero successful attempts; blocked attempts are logged |
| Dispatches with a fresh observer snapshot | 100% |
| Cards with exactly one active worker lease | 100% |
| Hermes cycles ending in dispatch/update/blocker | 100% |
| Worker panes left idle while eligible work exists | Explained by a board/observer state, not unexplained |
| Hermes replacement from durable state | Successful without manual worker operation |
| Cards closed without verifier evidence | Zero |

Use real failure drills: kill Hermes, leave a worker idle, make a worker exit, provide a stale board status, and attempt a forbidden outer pane command. If the process only succeeds when everyone remembers the rules, it is not yet reliable.

## Final hierarchy in one sentence

> **The outer Claude Code session sets direction and supervises the manager; Hermes owns the board-driven dispatch loop; the independent Observer/`flock-check` reports runtime facts; the controller enforces permissions and leases; workers implement cards; and a separate verifier closes them.**

The most important change is to stop treating Hermes and `flock-check` as one personality with many duties. Hermes should manage. The observer should observe. The board should remember. The controller should enforce. The outer Claude session should supervise—and when Hermes fails, replace Hermes rather than taking over its job.

## References

[1]: https://www.anthropic.com/engineering/multi-agent-research-system "Anthropic, How we built our multi-agent research system"

[2]: https://www.anthropic.com/engineering/managed-agents "Anthropic, Scaling Managed Agents: Decoupling the brain from the hands"

[3]: https://docs.langchain.com/oss/python/langgraph/persistence "LangGraph documentation, Persistence"

[4]: https://docs.temporal.io/workflow-execution "Temporal documentation, Workflow Execution overview"

[5]: https://code.claude.com/docs/en/hooks-guide "Claude Code documentation, Automate actions with hooks"

[6]: https://code.claude.com/docs/en/sub-agents "Claude Code documentation, Create custom subagents"
