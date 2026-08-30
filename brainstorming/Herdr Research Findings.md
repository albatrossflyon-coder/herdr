# Herdr Research Findings

## Anthropic: multi-agent research system
Source: https://www.anthropic.com/engineering/multi-agent-research-system (published 2025-06-13)

Anthropic describes an orchestrator-worker architecture: a lead agent plans, decomposes work, and creates specialized subagents that operate in parallel; results return to the lead for synthesis. Their engineering lessons include giving each subagent a clear objective, output format, tool guidance, and task boundaries; scaling effort to task complexity; using explicit tool purpose and descriptions; persisting the lead's plan in memory; and evaluating systems with small realistic test cases and observability. They also note that multi-agent systems cost substantially more and are not a good fit for tasks with many dependencies or where all agents need the same context. This supports using a narrow manager/observer hierarchy for Herdr rather than combining multiple duties in one LLM.

## Anthropic: managed agents
Source: https://www.anthropic.com/engineering/managed-agents (published 2026-04-08)

Anthropic describes decoupling the "brain" (agent/harness) from the "hands" (execution container). The harness interacts with the container through a small stable interface such as execute/provision/wake/getSession, while durable session events live outside the harness. Failed containers can be replaced instead of nursed back to health; failed harnesses can be rebooted from the external event log. They also emphasize structural security boundaries and proxying access through narrow tools rather than exposing broad credentials. This supports making Hermes and the observer communicate with Herdr through narrow controller operations, keeping durable board/event state outside any one LLM session, and replacing failed sessions rather than having the outer Claude session take over worker operations.

## Direct implications for Herdr

1. The board should be durable state outside Hermes, not merely a document Hermes is expected to remember.
2. Hermes should be the lead/orchestrator and should create precise worker assignments, but it should not also be the observer, executor, and verifier.
3. The observer should be a separate role/process that reads the board and pane/runtime signals and reports facts; it should not plan or implement.
4. Worker operations should go through narrow actions (dispatch, heartbeat, pause, requeue, report) rather than broad pane control exposed to every agent.
5. A manager can be restarted from durable event state; the outer supervisor should replace a failed Hermes rather than silently becoming the manager.
6. Herdr should use a small set of realistic failure drills/evaluations to test whether the controls hold, not rely on prompt wording alone.

## LangGraph persistence documentation
Source: https://docs.langchain.com/oss/python/langgraph/persistence

LangGraph distinguishes thread-scoped checkpoints from durable application data stores. Checkpoints preserve graph state for conversation continuity, human-in-the-loop workflows, time travel, and fault tolerance; stores hold longer-lived shared information across threads. The implication for Herdr is to separate live run state from durable board/project state and to make both resumable outside an LLM context.

## Temporal workflow execution documentation
Source: https://docs.temporal.io/workflow-execution

Temporal treats a workflow execution as durable and recoverable. State persists through failures and the workflow resumes from the latest recorded event; replay checks newly generated commands against an existing event history. The implication for Herdr is to maintain an append-only event history for plan, dispatch, heartbeat, pane exit, requeue, and recovery events, so a new Hermes or observer can resume from recorded state rather than guessing from pane appearance or conversational memory.

## Claude Code hooks documentation
Source: https://code.claude.com/docs/en/hooks-guide

Claude Code hooks run at lifecycle points such as SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, SubagentStart/Stop, TaskCreated/Completed, TeammateIdle, and SessionEnd. Crucially, PreToolUse hooks can block a tool call, while post-tool hooks can notify or record results. This means an outer Claude session can be technically prevented from using forbidden Herdr pane-control commands by a pre-tool hook, not merely reminded not to use them. Hooks are event-triggered; they do not replace a periodic observer/watchdog that polls current pane state.

## Claude Code subagents documentation
Source: https://code.claude.com/docs/en/sub-agents

Claude Code custom subagents have separate context, descriptions that guide delegation, independently scoped tools/permissions, optional persistent memory, and hooks for subagent events. The documentation distinguishes subagents within one session from independent background sessions and agent teams. This supports making the observer a genuinely separate, narrowly permissioned agent/session rather than combining observer, manager, and implementer responsibilities inside Hermes. It also supports explicitly restricting which tools a manager or observer can access.
