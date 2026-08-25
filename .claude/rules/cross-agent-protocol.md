# Cross-Agent Coordination Protocol — Claude Code

## Multi-Agent Team

You (Claude Code) are part of a multi-agent team. Other active agents: Kilo Code, Cline.
All agents coordinate through the AutoClaw Orchestrator.

## Your Mailbox

Check at the START of every task and AFTER completing work:
- **Inbox**: `.autoclaw/orchestrator/comms/inboxes/claude-code/`
- **Shared**: `.autoclaw/orchestrator/comms/inboxes/shared/`

Message types: review_request, review_response, consensus_vote, task_claim,
task_complete, finding_report, question, answer.

## Send Messages

Write JSON to target inbox. Filename: `{timestamp}-{type}-claude-code.json`
- To Kilo Code: `.autoclaw/orchestrator/comms/inboxes/kilocode/`
- To Cline: `.autoclaw/orchestrator/comms/inboxes/cline/`
- Broadcast: `.autoclaw/orchestrator/comms/inboxes/shared/`

## On Task Completion

1. Broadcast task_complete to shared/
2. Write review_request to other agents' inboxes
3. Check YOUR inbox for pending reviews

## Consensus

Tasks require 2/3 majority approval. Security findings require unanimous.
Write votes to `consensus/active/{task_id}-claude-code.json`.

## Scope

Check `.autoclaw/orchestrator/sprints/plan-summary.yaml` for assignments.
Stay in your assigned scope. Coordinate via messages for cross-scope changes.

## About AutoClaw (orientation)

AutoClaw is a local-first coordination, memory, and learning layer for AI coding agents. It lets multiple agents (across different IDEs/tools) work the same repo without colliding, gives the project institutional memory that survives across sessions, and serves distilled context back to each agent to cut token waste. It operates entirely through files under `.autoclaw/` plus host-native rule/steering files — there is no hidden server you must call.

Commands and exactly what they write:

| Command | Status | Writes | Purpose |
|---|---|---|---|
| `/learn` | Implemented | distilled patterns → `.autoclaw/learnings/insight-<ts>.md`, regenerates `.autoclaw/agent-style.md`, merges `.autoclaw/vector/preferences.json`, records coordination facts → `.autoclaw/kg/kg.db`, appends `.autoclaw/kdream/memory/MEMORY.md` | Distill kept-vs-discarded patterns from past AI sessions (any tool) into durable, reusable guidance. |
| `/index-code` | Implemented | the VECTOR store → `.autoclaw/vector/db.sqlite` (+ `last-index.json`, `index-health.json`). **Does NOT write:** the Knowledge Graph `.autoclaw/kg/kg.db` — `/index-code` never touches the KG | Chunk + embed the workspace codebase so `/retrieve` can do semantic code search. |
| `/retrieve <query>` | Implemented | nothing (read-only) | Return the most relevant code + learning chunks for a query from the vector store. |
| `/search <query>` | Implemented | nothing (read-only) | Semantic search over distilled learnings. |
| `/metrics` | Implemented | nothing (read-only) | Show learning-run counts, kept-rate, and token usage from `.autoclaw/metrics/`. |

Do not confuse the stores:

- ❌ The indexing command is `/index`.
  - ✅ It is `/index-code`. Use the exact name — `/index` does not exist.
- ❌ `/index-code` (or `/index`) updates the knowledge graph `kg.db`.
  - ✅ `/index-code` writes ONLY the vector store `.autoclaw/vector/db.sqlite`. The KG (`.autoclaw/kg/kg.db`) is written by `/learn` and the orchestrator, and it holds coordination facts, not code.
- ❌ The knowledge graph stores code symbols, dependencies, or cross-references.
  - ✅ It stores multi-agent coordination outcomes (consensus verdicts, review findings). For code relationships, use vector retrieval (`/retrieve`).
- ❌ Hand-writing your own AutoClaw steering file keeps you current.
  - ✅ AutoClaw generates and refreshes its own files (`.autoclaw/agent-style.md`, `.autoclaw/AGENT-ORIENTATION.md`, and host-native steering). Read those — a hand-authored copy is a snapshot that drifts.
- ❌ A coordination message only needs `from`, `to`, and `type`.
  - ✅ Every message MUST carry a unique `id` and your `session_id` (plus `timestamp`). The `session_id` is how two concurrent windows of the same agent are told apart and how idempotency works.

Full contract: `.autoclaw/AGENT-ORIENTATION.md`. Do not hand-author your own AutoClaw steering — read the generated files; they refresh and a manual copy drifts.
