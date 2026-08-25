> Multi-agent coordinator that spawns specialized sub-agents working in parallel. Trigger on "/mateam launch", "spawn agents", "multi-agent", or "coordinate team".

# MAteam — Multi-Agent Coordinator

## Operating Rules (read first)

1. **Use file tools, not shell, for the scratchpad.** Create `.autoclaw/mateam/scratch/<session>/...` and its files with the host's file/write tool — never `mkdir -p`/`touch`/`New-Item`.
2. **Forward slashes in paths.** Always.
3. **Roles are sequential by default**, but Researcher steps that read independent files MAY be issued as parallel tool calls. Coder waits for Researcher; Reviewer waits for Coder; Verifier waits for Reviewer.
4. **One scratchpad per session.** Re-using a session ID without `cancel`-ing the prior session is an error — append `-2`, `-3`, etc., or pick a new slug.
5. **Output discipline.** The final report is ≤6 lines: what shipped, key review concerns, test/build result, scratchpad path. No reasoning narration, no per-role transcripts.

## Host detection & dispatch

Before spawning anything, decide which dispatch path applies:

- **If the host exposes a tool literally named `Agent`** (Anthropic Claude Code subagents): each role (Researcher, Coder, Reviewer, Verifier) MUST be invoked as a separate `Agent` tool call with a self-contained prompt that names the role, its responsibilities, the scratchpad files it must read/write, and acceptance criteria. Roles run in sequence (Researcher → Coder → Reviewer → Verifier). Within a single role, independent file reads MAY fan out as parallel tool calls.
- **Otherwise** (GitHub Copilot Chat, Cursor, Continue, Cline, Kilo Code, Antigravity, Windsurf, Kiro, etc.): there is no subagent primitive. Play each role yourself, in turn, in this same context, writing the handoff to the scratchpad files between roles. This is the correct and expected fallback — do NOT fabricate an `Agent` tool call.
- **Hard rule:** if you invent an `Agent` invocation when the host has no such tool, that is a critical failure. Halt and tell the user the host lacks subagents and you ran in-session instead.

## On Invocation

Determine the sub-command from the user's message:

- `launch "<task>"` / no sub-command + task → **Spawn a team**
- `status` → **Show active agents**
- `list-peers` → **List all agents in session**
- `cancel` / `stop` → **Halt all agents**
- `result` / `merge` → **Collect and merge outputs**

---

## launch — Spawn Agent Team

### Step 1 — Decompose the Task
Break the user's task into parallel workstreams. Standard roles:

| Role | Responsibility |
|---|---|
| **Researcher** | Gathers context: reads relevant files, searches codebase, identifies dependencies |
| **Coder** | Implements changes based on Researcher's findings |
| **Reviewer** | Audits Coder's output for correctness, security, and style |
| **Verifier** | Runs tests, checks build, confirms acceptance criteria are met |

Assign only the roles the task requires. Small tasks may need only Researcher + Coder.

### Step 2 — Create Scratchpad
Create `.autoclaw/mateam/scratch/<session-id>/` with:
- `plan.md` — task decomposition and role assignments
- `context.md` — shared findings (Researcher writes here)
- `output.md` — Coder's deliverables
- `review.md` — Reviewer's notes
- `verify.md` — Verifier's results

Write the task and role breakdown to `plan.md`.

### Step 3 — Execute Roles in Order

**Researcher:**
- Under Claude Code: dispatch via `Agent` tool with the prompt below. Otherwise: simulate this role in-session and write handoff to `context.md`.
- Read `plan.md` to understand scope.
- Search the codebase for relevant files, functions, and patterns.
- Write findings to `context.md`: file paths, key functions, existing patterns, potential conflicts.

**Coder:**
- Under Claude Code: dispatch via `Agent` tool with the prompt below. Otherwise: simulate this role in-session and write handoff to `output.md`.
- Read `plan.md` and `context.md`.
- Implement the required changes.
- Write a summary of changes made (files modified, functions added/changed) to `output.md`.

**Reviewer:**
- Under Claude Code: dispatch via `Agent` tool with the prompt below. Otherwise: simulate this role in-session and write handoff to `review.md`.
- Read `output.md` and the actual changed files.
- Check for: logic errors, security issues, style inconsistencies, missing edge cases.
- Write findings to `review.md`. If blockers found, flag them clearly.

**Verifier:**
- Under Claude Code: dispatch via `Agent` tool with the prompt below. Otherwise: simulate this role in-session and write handoff to `verify.md`.
- Read `review.md`. If blockers exist, halt and report to user.
- Run the project's test suite and/or build command.
- Write results (pass/fail, test output summary) to `verify.md`.

### Step 4 — Report
Summarize all four role outputs to the user:
- What was done (from `output.md`)
- Any review concerns (from `review.md`)
- Test/build result (from `verify.md`)
- Location of full scratchpad for inspection

## Reporting

The final user-facing report MUST state which dispatch path was taken — e.g. "Ran 4 subagents via Agent tool" (Claude Code) vs "Simulated 4 roles in-session" (every other host). This is non-optional: the user needs to know whether real isolation happened or not.

---

## status — Show Active Agents

Read `.autoclaw/mateam/scratch/` and list all active sessions with their current phase and last update time.

## list-peers — List All Agents

List each role active in the current session, their assigned task segment, and current state (pending / running / done / blocked).

## cancel — Halt All Agents

1. Write `{ "cancelled": true }` to each session's scratchpad.
2. Append cancellation notice to `plan.md`.
3. Confirm to user.

## result / merge — Collect Outputs

Read `output.md` from the most recent session and present the final merged result to the user.

---

## Parallel Execution Note

When running multiple independent sub-tasks, execute Researcher and any non-dependent Coder segments in parallel by issuing simultaneous tool calls. Sequential dependencies (Reviewer must wait for Coder) must be respected. Always document handoff points in `plan.md`.

---

## Session ID Format

Use `<YYYY-MM-DD>-<task-slug>` as the session ID, e.g. `2026-04-01-refactor-auth`.
