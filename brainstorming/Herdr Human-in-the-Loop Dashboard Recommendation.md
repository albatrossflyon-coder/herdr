# Herdr Human-in-the-Loop Dashboard Recommendation

**Date:** 2026-08-30

## Recommendation

Yes, this is something you should build or adapt. You should not have to watch PowerShell panes to understand whether Herdr is functioning. The right interface is a **local web dashboard with optional desktop packaging**, connected directly to Herdr’s board and event stream.

I found two existing open-source projects that demonstrate much of the requested concept. **AgentPulse** provides a live dashboard for Claude Code and Codex sessions, observability-only and local-orchestration modes, hook/event ingestion, session timelines, a human-in-the-loop inbox, status classification, alerts, and scoped MCP control [1]. **Claude-Code-Agent-Monitor** provides a real-time React/WebSocket dashboard for Claude Code and Codex with session activity, tool usage, subagent orchestration, Kanban status, notifications, and macOS/Windows native-app support [2]. **LangSmith** is strong for model/tool tracing, online evaluation, cost, latency, and alerts, but it is not a direct Herdr pane-and-board console [3].

The best path is therefore **not to begin by inventing a native desktop application**. First make a Herdr-aware local web dashboard, either by adapting AgentPulse or by using the other project as a UI reference. If the resulting dashboard is useful, package the same web interface as a desktop app later. This avoids maintaining two separate products while still giving you a normal application window.

## What you should be able to see

The dashboard should answer, at a glance, five questions:

1. **What is the overall plan?** Show the current plan, milestones, blocked dependencies, and cards remaining.
2. **What is every agent doing?** Show Hermes, Observer/`flock-check`, each worker, and the outer supervisor with clear status and last-seen time.
3. **Is the status trustworthy?** Show the evidence source: Herdr push event, heartbeat, board event, process inspection, screen manifest, or reconciliation check.
4. **What needs human attention?** Show an inbox of stalled agents, contradictory signals, failed verification, risky proposed actions, and manager-replacement requests.
5. **What happened?** Show a timeline of dispatch, acknowledgement, status events, heartbeat, pane exit, requeue, verification, and recovery.

The dashboard should not be another terminal emulator with prettier colors. Its purpose is to present **control state and health evidence**, not merely reproduce screen output.

## Proposed dashboard layout

| Area | What it shows | Human action |
|---|---|---|
| **Command center** | Overall plan, cards by state, active incidents, Hermes health | Pause, approve, or escalate |
| **Agent roster** | Hermes, Observer, workers, outer Claude; status, role, card, heartbeat, confidence | Open details; approve recovery |
| **Board view** | Kanban cards, dependencies, leases, assignees, verification state | Inspect or approve plan changes |
| **Agent detail** | Event timeline, last heartbeat, latest Herdr event, board changes, sanitized transcript/tool summary | Investigate a specific agent |
| **Human inbox** | Requests for approval, manager stalled, conflicting signals, failed checks | Approve, decline, retry, replace Hermes |
| **Event timeline** | Chronological append-only events | Audit what actually happened |
| **Health panel** | Event-stream freshness, observer health, Hermes management progress, controller health | Diagnose system-level failure |

## The most important visual distinction

Every status badge should show both **state** and **confidence/evidence source**. For example:

```text
Hermes       MANAGING       heartbeat 8s ago       board advanced 22s ago
Observer     HEALTHY        push event 2s ago      sequence 8841
worker-01    WORKING        Herdr event 4s ago     card H-184
worker-02    IDLE           reconciled 11s ago     eligible for dispatch
worker-03    STALLED        process alive           Hermes heartbeat missing
Claude outer SUPERVISOR     session present         no manager commands issued
```

For Hermes and Claude Code, the screen manifest should be visually labeled as **weak evidence**. A green “process alive” dot must not look equivalent to “manager is making progress.” The interface should show the three separate signals:

```text
Runtime presence: confirmed / stale
Agent heartbeat: valid / missing / incoherent
Operational progress: board advanced / no movement / unknown
```

## How the dashboard connects to Herdr

The dashboard should consume Herdr events through a small adapter rather than trying to scrape terminals as its primary integration.

```text
Herdr board events ───────────────┐
Herdr pane.agent_status_changed ─┼─> Herdr adapter ─> event store ─> WebSocket/SSE ─> dashboard
Hermes heartbeat ────────────────┤
Observer projection ─────────────┤
flock-check reconciliation ──────┘
```

The adapter should translate Herdr-specific events into a stable dashboard schema while preserving the original event payload. The browser should not have direct authority to start arbitrary panes. Dashboard controls should call narrow, authenticated actions such as `request_dispatch`, `request_requeue`, `pause_plan`, `declare_incident`, or `replace_manager`.

The dashboard should consume `pane.agent_status_changed` as the primary live feed. `flock-check` should be exposed as a **Reconcile now** button and an automatic fallback when the event sequence has a gap, an event becomes stale, or board and runtime signals disagree.

## Existing options compared

| Option | Strengths | Weaknesses for Herdr | Recommendation |
|---|---|---|---|
| **AgentPulse** | Closest match to live coding-agent monitoring, inbox, watcher, statuses, MCP, local orchestration | Needs Herdr adapter and validation of its assumptions; supports fewer agent types directly | First project to evaluate and potentially adapt |
| **Claude-Code-Agent-Monitor** | Strong visual dashboard, WebSockets, Kanban, activity/tool views, desktop packaging | Appears centered on Claude Code/Codex hooks; Herdr event/board integration still needed | UI/reference alternative; evaluate second |
| **LangSmith** | Excellent traces, metrics, online evals, alerts, OTel | Not a Herdr pane/board control room; external-service/privacy and integration overhead | Optional complement, not primary console |
| **Build from scratch** | Exact Herdr roles, events, board, confidence, leases, and controls | More engineering and maintenance; easy to recreate existing work | Only after evaluating existing projects |
| **Native desktop first** | Dedicated window, tray icon, notifications | Duplicates frontend/backend effort and slows iteration | Defer; package local web dashboard later |

## What should be built versus reused

Reuse the **dashboard shell, event timeline, WebSocket/SSE update pattern, inbox, and desktop wrapper ideas** from existing projects where possible. Build or adapt the **Herdr integration layer**, because Herdr’s semantics are specific: its board is authoritative for work, `pane.agent_status_changed` is its push event, `flock-check` is its reconciliation observer, Hermes is its manager, and the screen-manifest accuracy differs by agent family.

Do not let an existing dashboard silently become a second orchestration authority. The first version should be **observe-only plus human-approved recovery**. It may show buttons for “ask Hermes to dispatch,” “reconcile,” “pause,” and “replace Hermes,” but it should not offer an unrestricted “send text to any pane” button. That would recreate the same outer-Claude bypass problem in graphical form.

## Suggested build sequence

### Stage 1: Observe-only command center

Create the Herdr adapter and show the board, roster, push-event timeline, `flock-check` results, Hermes heartbeat, and evidence confidence. Add no automatic agent control. This stage answers whether the dashboard accurately represents reality.

### Stage 2: Human-in-the-loop actions

Add authenticated buttons for `reconcile`, `pause`, `resume`, `request Hermes dispatch`, `requeue card`, and `replace Hermes`. Every button should create a pending action record and display the result. High-impact actions should require confirmation.

### Stage 3: Guarded automation

Allow the watchdog/controller to take low-risk automatic actions such as marking an observation stale, expiring a lease, or notifying Hermes. Keep worker restart, manager replacement, and plan changes human-approved until the failure policy has been tested.

### Stage 4: Desktop packaging

If you want a dedicated application window, wrap the local web dashboard with Tauri, Electron, or the desktop packaging approach already used by the selected project. The desktop shell should be a presentation and notification layer; the Herdr adapter and event store should remain the same.

## First version acceptance criteria

The first usable version should pass these tests:

| Test | Expected result |
|---|---|
| A worker emits `pane.agent_status_changed` | Dashboard updates without manual refresh |
| No event arrives for a configured interval | Dashboard shows `stale_observation`, not falsely `idle` |
| `flock-check` disagrees with the event projection | Contradiction is visible and reconciliation is offered |
| Hermes process exists but heartbeat is absent | Dashboard says `manager_ineffective`, not simply “online” |
| Hermes heartbeat is fresh but board does not advance | Dashboard says `manager_stalled` |
| Outer Claude attempts a worker-control action | Action is blocked or routed to Hermes; event is logged |
| Human approves manager replacement | Hermes is replaced from durable board/event state |
| Worker claims completion with no artifact | Card remains unverified and cannot become `Done` |

## Final answer

Yes, this is a realistic application to build, and people have already built close versions. It should start as a **local web application**, not a separate native desktop system. The browser dashboard can give you the visual command center you are missing, and a desktop wrapper can follow later.

My recommendation is to inspect and potentially adapt **AgentPulse first**, compare it with **Claude-Code-Agent-Monitor**, and retain LangSmith only as an optional model-trace system. The Herdr-specific work is the adapter and authority model: push events into an Observer projection, board state into the task view, Hermes heartbeats into manager health, `flock-check` into reconciliation, and human approvals into a controlled inbox.

The application should make it easy for you to see what is happening without giving the outer Claude session—or the dashboard itself—permission to bypass Hermes.

## References

[1]: https://github.com/jstuart0/agentpulse "AgentPulse: real-time monitoring dashboard for AI coding agent sessions"

[2]: https://github.com/hoangsonww/Claude-Code-Agent-Monitor "Claude-Code-Agent-Monitor: real-time monitoring dashboard for Claude Code and Codex"

[3]: https://www.langchain.com/langsmith/observability "LangSmith Observability"
