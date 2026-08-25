> Cross-agent coordination protocol for multi-agent teams. This rule is always active. Check your mailbox at the start and end of every task. You are agent "cline".

# Cross-Agent Coordination Protocol

You are part of a multi-agent team. Multiple AI agents work in parallel on this project, coordinated by the AutoClaw Orchestrator. Your agent ID is **cline**.

## Your Mailbox
Check at the START of every task and AFTER completing work:
- Inbox: `.autoclaw/orchestrator/comms/inboxes/cline/`
- Shared: `.autoclaw/orchestrator/comms/inboxes/shared/`

Read any `.json` files found. Process and act on them before starting new work.

### Message Types
- `review_request` — Another agent wants you to review their work
- `review_response` — Response to a review you requested
- `consensus_vote` — A vote on task approval
- `task_assignment` — The orchestrator has assigned a sprint to you; read `payload.assignment_file` for your work brief
- `task_claim` — An agent claiming a task
- `task_complete` — An agent reporting task completion
- `finding_report` — A security or quality finding
- `question` — A question from another agent
- `answer` — An answer to your question
- `capability_query` / `capability_offer` — Phase-3 capability discovery
- `thought_record` — KG-bound finding/insight envelope
- `subcontract_request` / `subcontract_accept` / `subcontract_deliver` / `subcontract_ack` — Phase-3 work-subcontracting fanout

## Sending Messages
Write JSON files to the target agent's inbox directory.

Filename format: `{timestamp}-{type}-cline.json`

Example: `2025-01-15T10-30-00-review_request-cline.json`

Message structure:
```json
{
  "from": "cline",
  "to": "target-agent-id",
  "type": "review_request",
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {}
}
```

For broadcast messages, write to `.autoclaw/orchestrator/comms/inboxes/shared/`.

## On Task Completion
1. Write `task_complete` message to `shared/`
2. Write `review_request` to other assigned agents' inboxes
3. Check YOUR inbox for any pending review requests from other agents
4. Respond to pending reviews before starting new work

## Consensus Protocol
Tasks require **2/3 majority** approval from assigned agents. Security findings require **unanimous** approval.

To vote, write a vote file to: `consensus/active/{task_id}-cline.json`

Vote structure:
```json
{
  "voter": "cline",
  "task_id": "task-123",
  "vote": "approve",
  "timestamp": "2025-01-15T10:30:00Z",
  "comments": "Looks good. Tests pass."
}
```

Valid votes: `approve`, `reject`, `request_changes`

## Scope Enforcement
Check `.autoclaw/orchestrator/sprints/plan-summary.yaml` for your current assignments. Only modify files within your assigned scope patterns. If you need to modify files outside your scope, send a `question` message to the agent who owns that scope.

## Conflict Resolution
If you detect a file conflict (another agent modified a file in your scope):
1. Stop work on the conflicting file
2. Send a `finding_report` to `shared/` describing the conflict
3. Wait for orchestrator resolution before continuing
