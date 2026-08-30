# Agent Dashboard Research Findings

## AgentPulse (open source)
Source: https://github.com/jstuart0/agentpulse

AgentPulse is an existing real-time monitoring dashboard for AI coding-agent sessions, including Claude Code and Codex CLI. Its README describes observability-only mode, full local orchestration mode, and remote dashboard mode. It receives hook events, shows active sessions, current work, status, and scrollable chat/tool timelines. It includes a human-in-the-loop inbox, session watcher, deterministic health classifier (healthy/blocked/stuck/risky/complete candidate), alerts, summaries, and an MCP server with observe-only versus orchestration scopes. This is unusually close to the desired Herdr dashboard concept and should be evaluated before building from scratch.

Important limitation for Herdr: the repository is designed around its own session/event model and would likely need an adapter for Herdr’s `pane.agent_status_changed`, board cards, Hermes heartbeat, Observer projection, leases, and worker-pane controls. The existing project should not automatically be treated as authoritative; Herdr remains the source of pane and board truth.

## LangSmith Observability
Source: https://www.langchain.com/langsmith/observability

LangSmith offers agent tracing, real-time monitoring, online evaluations, cost tracking, trajectory monitoring, webhooks/PagerDuty alerts, and OpenTelemetry integration. It is useful for detailed model/tool traces and production quality metrics. It is not a direct terminal-pane dashboard and does not inherently understand Herdr’s board, pane status events, or Hermes/Observer hierarchy. It could complement Herdr for model-level traces, but it is not the primary UI for the requested human operator view.

## Initial conclusion

An existing tool does exist that is close to the requested interface: AgentPulse. The likely best path is not a new desktop application first. It is to evaluate AgentPulse as a local web dashboard and add a Herdr adapter/event ingestion path. A browser-based dashboard is preferable initially because it can show cards, roster health, timelines, approvals, and live events in one place, while remaining usable from a desktop. A native desktop wrapper could be added later if notifications, tray behavior, or always-on local access justify it.


## Claude-Code-Agent-Monitor
Source: https://github.com/hoangsonww/Claude-Code-Agent-Monitor

This open-source project is even closer to the requested form factor: a real-time dashboard for Claude Code and Codex with Node.js, Express, React, SQLite, WebSockets, live session/activity/tool views, subagent orchestration, Kanban status, notifications, and native macOS/Windows app support. It confirms that both a web dashboard and desktop wrapper are practical patterns for this use case.

Its stated integration is through Claude Code and Codex native hooks, so Herdr would still need an adapter for Herdr pane events, the Herdr board, Hermes, and Observer. It may be useful as a UI/reference implementation, but its supported-agent model and control assumptions need validation before adopting it.

## Refined conclusion

There are at least two existing projects demonstrating the requested concept. AgentPulse appears closer to Herdr’s observer/approval/orchestration needs, while Claude-Code-Agent-Monitor appears stronger as a polished multi-session UI with WebSockets and native desktop packaging. LangSmith is better suited to model/tool traces than terminal-pane operations. The recommended first step is to evaluate AgentPulse and Claude-Code-Agent-Monitor side by side, then either adapt one or build a Herdr-specific local web dashboard. A native desktop app should be a wrapper around that dashboard rather than the first implementation.
