> Persistent always-on background agent with automatic memory consolidation. Trigger on "start background agent", "enable kdream", "/kdream start", "persistent daemon", or "auto-dream memory".

# KDream — Persistent Background Agent

## Operating Rules (read before any sub-command)

1. **Use file tools, not shell, for directories and files.** Create folders and files with the host's file/write tool (e.g. Write, create_file, edit_file). Do NOT use `mkdir -p`, `touch`, `New-Item`, or shell redirection — they fail across the Bash/PowerShell/cmd.exe mix you may be running on. If you must shell out, detect the platform first and use `mkdir` (no `-p`) on Windows cmd, or `New-Item -ItemType Directory -Force` in PowerShell.
2. **Always use forward slashes in paths** (e.g. `.autoclaw/kdream/state.json`). Node, git, and every supported shell accept them.
3. **Be idempotent.** Before running `start`, read `.autoclaw/kdream/state.json`. If `status == "running"`, do NOT recreate directories or rewrite state — just run a fresh tick and report current status.
4. **Output discipline.** When confirming an action, output ≤3 short lines: what was done, current counts (ticks/TODOs/follow-ups), next step. Do NOT narrate your reasoning, repeat headings, or invent style rules ("titles must be gerunds", etc.). No emojis unless the user asked.
5. **Never invent files, follow-ups, or commits.** Only report what you actually read from disk or git.

## On Invocation

Determine the sub-command from the user's message:

- `start` / no sub-command → **Start the daemon**
- `ps` / `status` → **Report status**
- `logs` → **Show recent log entries**
- `stop` / `kill` → **Shut down**
- `dream` → **Run an autoDream cycle now**
- `add <note>` → **Add a task or note to MEMORY.md**
- `todo` → **List all open TODO/FIXME items found in workspace**
- `work <item>` → **Actively work on a specific TODO or follow-up item**

---

## start — Launch Daemon

0. **Idempotency check.** Read `.autoclaw/kdream/state.json`. If it exists and `status == "running"`, skip steps 1–4, jump to step 5 (run a tick), then report: "KDream already running (tick N). Ran fresh tick."
1. Create directories if missing using the host's file/write tool (NOT shell `mkdir -p`):
   - `.autoclaw/kdream/logs/`
   - `.autoclaw/kdream/memory/`
2. Write `.autoclaw/kdream/state.json` using the file/write tool:
   ```json
   { "status": "running", "started": "<ISO timestamp>", "tick": 0, "lastDream": null, "todos": [] }
   ```
3. Create `.autoclaw/kdream/memory/MEMORY.md` if missing with this structure:
   ```markdown
   # KDream Memory
   
   ## Follow-ups
   <!-- KDream checks this section on every tick. Add tasks here. -->
   
   ## Facts
   <!-- Consolidated knowledge about this workspace. -->
   
   ## Observations
   <!-- Notable events and patterns observed over time. -->
   ```
4. Append to today's log (`.autoclaw/kdream/logs/YYYY-MM-DD.md`):
   ```
   [HH:MM:SS] KDream started. Workspace: <cwd>
   ```
5. Run the first **tick** immediately (see Tick Cycle below).
6. Inform the user with a single concise block — for example:
   ```
   KDream started. Tick 1.
   Git: <N> uncommitted, <M> commits today.
   TODOs: <K>. Follow-ups: <J>. No autoDream yet.
   Add tasks with /kdream add <note> or in MEMORY.md ## Follow-ups.
   ```
   Adapt counts to reality. No extra prose, no headings, no style commentary.

---

## Tick Cycle

On each tick:

### 1. Check git status
If a git repo exists: run `git status` and `git log --oneline -5`.
- If there are uncommitted changes older than 1 hour: log `[WARN] Stale uncommitted changes: <files>` and notify user.
- If recent commits exist: log them silently.

### 2. Scan TODO/FIXME items
Glob all source files for lines matching `TODO`, `FIXME`, `HACK`, `XXX`, `BUG`.
- For each match: record file path, line number, and comment text.
- Compare against previous tick's list (stored in `state.json` under `"todos"`).
- New items since last tick → log `[NEW TODO] <file>:<line> — <text>` and notify user.
- Resolved items (present last tick, gone now) → log `[RESOLVED] <file>:<line>` and update memory.
- Update `state.json` with current todo list.

### 3. Check MEMORY.md follow-ups
Read `.autoclaw/kdream/memory/MEMORY.md`, find all lines under `## Follow-ups`.
- Lines starting with `- [ ]` are open tasks → report them to the user if any exist.
- Lines starting with `- [x]` are done → move to `## Observations` during next autoDream.
- If the user asks KDream to act on a follow-up: work on it, then mark `- [x]`.

### 4. Decide and act
- If ≥1 notification-worthy item: surface a concise summary to the user with options to act.
- If nothing notable: append a silent heartbeat to log only. Do not disturb the user.

### 5. Update state
Increment `tick` in `state.json`. Save current todo list snapshot.
If `tick % 20 == 0` or last dream >24h ago → trigger **autoDream**.

---

## add — Add a Follow-up

When the user runs `/kdream add <note>`:
1. Append `- [ ] <note>` under `## Follow-ups` in `MEMORY.md`.
2. Confirm: "Added to KDream follow-ups: `<note>`"

This is the fastest way to give KDream something to watch or act on.

## todo — List Open Items

1. Read current `todos` array from `state.json`.
2. Read open `- [ ]` items from `MEMORY.md ## Follow-ups`.
3. Report both lists clearly, grouped by source (code TODOs vs manual follow-ups).

## work — Act on an Item

When the user runs `/kdream work <item description or number>`:
1. **Dispatch.** If the host exposes a tool literally named `Agent` (Claude Code), spawn a coder subagent scoped to this single item with a short, self-contained prompt and the relevant file paths. Otherwise (Copilot, Cursor, Cline, Kilo, Continue, Antigravity, Windsurf, Kiro, etc.) work the item in-session yourself — do NOT fabricate an `Agent` call. Small items (one-file edits, doc tweaks) MAY be done in-session even when `Agent` is available, to avoid spawn overhead; items spanning ≥3 files or requiring a research pass SHOULD use `Agent`.
2. Identify the matching TODO/FIXME or follow-up item.
3. Read the relevant file(s) and context.
4. Implement or resolve the item using available tools.
5. Mark the follow-up as `- [x]` in `MEMORY.md` or confirm the code change.
6. Log the action taken.

Steps 2–5 apply regardless of which dispatch path was taken.

---

## ps — Status

Read `.autoclaw/kdream/state.json` and report:
- Running / stopped, start time, tick count, last dream timestamp
- Number of open TODOs tracked
- Number of open follow-ups in MEMORY.md
- Last log entry

## logs — Show Logs

Read the last 30 lines of today's log at `.autoclaw/kdream/logs/YYYY-MM-DD.md`.

## stop — Shutdown

1. Update `state.json`: `{ "status": "stopped", "stopped": "<ISO timestamp>" }`
2. Append to log: `[HH:MM:SS] KDream stopped.`
3. Confirm to user.

---

## autoDream Cycle (Memory Consolidation)

Triggered automatically (tick % 20 or 24h elapsed) or via `/kdream dream`.

### Phase 1 — Orient
List all files in `.autoclaw/kdream/memory/`. Note current MEMORY.md line count.

### Phase 2 — Gather
Read last 7 days of log files. Extract:
- `[NEW TODO]` entries → add to Facts if not already there
- `[RESOLVED]` entries → move matching Follow-ups to Observations
- `[WARN]` entries → surface any recurring patterns

### Phase 3 — Consolidate
- Merge gathered items into appropriate MEMORY.md sections.
- Remove contradictions (keep newer fact).
- Convert relative dates to absolute ISO dates.
- Deduplicate identical entries.
- Move `- [x]` completed follow-ups from Follow-ups to Observations.

### Phase 4 — Prune
If MEMORY.md exceeds 200 lines or 25KB:
- Archive oldest 20% of Observations to `.autoclaw/kdream/memory/archive-YYYY-MM-DD.md`.
- Remove them from MEMORY.md.

### Phase 5 — Finalize
Update `state.json` `"lastDream"`. Append: `[HH:MM:SS] autoDream complete. Memory: <N> lines.`

---

## Provenance — a fact without provenance is a guess (MEM-1)

Distilled facts follow the continual-learning discipline: **Fail → Investigate
→ Verify → Distill → Consult.** A lesson is only worth keeping once it has been
*verified*, and a verified fact records *how* it was checked.

When consolidating, a fact MAY carry a `verified_by` provenance stamp:

```json
"verified_by": { "method": "command", "evidence": "npm run compile exited 0", "verified_at": "<ISO>" }
```

- `method` is one of `session` | `tool_result` | `command` | `user` | `unverified`.
- Omitting `verified_by` is fine — readers treat an absent stamp as
  `unverified` (a guess), so existing memory keeps working unchanged.
- Prefer stamping facts you confirmed by a command, tool result, or the user
  over ones merely observed in chatter; readers can then rank verified facts
  above guesses.
