# Invalid Session Examples

This document provides examples of sessions that encounter errors and how the agent handles them according to contract requirements.

---

## Example 1: Missing AGENT_LINK.md

### Scenario

Agent attempts to start a session in a workspace that lacks the required `AGENT_LINK.md` file.

### Initial State

**Workspace**: `/workspace/unlinked-project`

**Directory Contents**:
```
/workspace/unlinked-project/
├── src/
│   └── index.ts
├── package.json
└── README.md
```

**Note**: No `AGENT_LINK.md` file exists.

### Session Attempt

```
Agent: Starting session in /workspace/unlinked-project
Agent: Checking for AGENT_LINK.md...
Agent: ERROR - AGENT_LINK.md not found
```

### Agent Behavior

**Step 1**: Check for AGENT_LINK.md
```python
agent_link_path = os.path.join(workspace, 'AGENT_LINK.md')
if not os.path.exists(agent_link_path):
    raise AgentLinkMissing(workspace)
```

**Step 2**: Log error and abort
```json
{
  "timestamp": "2026-02-01T10:00:00.000Z",
  "level": "ERROR",
  "event": "SESSION_ABORT",
  "sessionId": "session-attempt-001",
  "operation": "initialize",
  "success": false,
  "error": {
    "code": "AGENT_LINK_MISSING",
    "message": "AGENT_LINK.md not found in workspace",
    "workspace": "/workspace/unlinked-project"
  }
}
```

**Step 3**: Return error to user

```
ERROR: Cannot start agent session

The workspace /workspace/unlinked-project is not linked to the agent system.

Required file missing: AGENT_LINK.md

To fix this:
1. Run the agent setup command in this workspace
2. Or copy AGENT_LINK.md from a linked workspace and update it

Session aborted.
```

### Why This Happens

The `AGENT_LINK.md` file serves as the contract between the workspace and the agent system. Without it:
- Agent cannot verify workspace authorization
- Agent cannot determine allowed operations
- Agent cannot locate schemas and configuration

### Resolution

```bash
# Option 1: Initialize workspace with agent
agent init /workspace/unlinked-project

# Option 2: Create AGENT_LINK.md manually
cat > /workspace/unlinked-project/AGENT_LINK.md << 'EOF'
# Agent Link

This workspace is connected to the AI Agent system.

## Configuration

- **Boot Repository**: ../agent-boot
- **Contract Version**: 1.0.0
- **Created**: 2026-02-01T10:00:00Z
EOF
```

---

## Example 2: Corrupted STATE.json

### Scenario

Agent attempts to start a session but finds a corrupted or invalid `STATE.json` file.

### Initial State

**File**: `/workspace/my-project/STATE.json` (corrupted)

```json
{
  "sessionId": "session-prev-001",
  "status": "running",
  "lastUpdated": "invalid-date-format",
  "currentTask": 12345,
  "checkpoint": "not-an-object"
}
```

**Issues**:
1. `lastUpdated` is not a valid ISO 8601 timestamp
2. `currentTask` should be a string, not a number
3. `checkpoint` should be an object or null, not a string

### Session Attempt

```
Agent: Starting session in /workspace/my-project
Agent: Loading STATE.json...
Agent: ERROR - STATE.json validation failed
Agent: Attempting recovery...
```

### Agent Behavior

**Step 1**: Attempt to load and validate STATE.json
```python
try:
    state = json.load(open(state_path))
    validate_state_schema(state)
except jsonschema.ValidationError as e:
    log_error("STATE.json validation failed", e)
    attempt_state_recovery(state_path)
```

**Step 2**: Log validation error
```json
{
  "timestamp": "2026-02-01T10:00:00.000Z",
  "level": "ERROR",
  "event": "VALIDATION_FAIL",
  "sessionId": "session-recovery-001",
  "operation": "validate_state",
  "path": "STATE.json",
  "success": false,
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "STATE.json does not match schema",
    "errors": [
      {
        "path": "lastUpdated",
        "message": "'invalid-date-format' does not match ISO 8601 format"
      },
      {
        "path": "currentTask",
        "message": "12345 is not of type 'string'"
      },
      {
        "path": "checkpoint",
        "message": "'not-an-object' is not of type 'object'"
      }
    ]
  }
}
```

**Step 3**: Attempt recovery from git history
```python
def attempt_state_recovery(state_path):
    # Find last valid STATE.json in git history
    result = subprocess.run(
        ['git', 'log', '--oneline', '-20', '--', 'STATE.json'],
        capture_output=True, text=True
    )

    for commit in result.stdout.strip().split('\n'):
        commit_hash = commit.split()[0]
        try:
            # Try to restore from this commit
            old_state = get_file_from_commit(commit_hash, 'STATE.json')
            validate_state_schema(old_state)

            # Found valid state, restore it
            log_info(f"Restoring STATE.json from commit {commit_hash}")
            restore_state_from_commit(commit_hash)
            return True
        except Exception:
            continue

    # No valid state found in history
    create_fresh_state()
    return False
```

**Step 4**: Restore from backup
```json
{
  "timestamp": "2026-02-01T10:00:05.000Z",
  "level": "INFO",
  "event": "STATE_RECOVERY",
  "sessionId": "session-recovery-001",
  "operation": "restore",
  "success": true,
  "details": {
    "recoveredFrom": "commit abc123",
    "previousSessionId": "session-prev-001"
  }
}
```

**File**: `/workspace/my-project/STATE.json` (after recovery)

```json
{
  "sessionId": "session-recovery-001",
  "status": "running",
  "lastUpdated": "2026-02-01T10:00:05Z",
  "currentTask": null,
  "checkpoint": {
    "recoveredFrom": "abc123",
    "recoveryReason": "corrupted_state"
  },
  "errors": [
    {
      "timestamp": "2026-02-01T10:00:00Z",
      "code": "STATE_CORRUPTED",
      "message": "Previous STATE.json was corrupted and recovered from backup"
    }
  ]
}
```

### User Notification

```
WARNING: STATE.json was corrupted and has been recovered

Previous state was invalid due to:
- Invalid date format in lastUpdated
- Wrong type for currentTask (expected string)
- Wrong type for checkpoint (expected object)

Recovered from: commit abc123 (2026-01-31)

The previous session's work may have been interrupted.
Please review the workspace state before continuing.

Session started with recovery state.
```

---

## Example 3: Schema Validation Fails

### Scenario

Agent attempts to write a file that should match a schema, but the content is invalid.

### Context

Agent is trying to update `TODO.json` with a new task, but provides invalid data.

**Attempted Content**:
```json
{
  "version": "1.0.0",
  "tasks": [
    {
      "id": "task-001",
      "title": "Fix bug",
      "status": "in progress",
      "priority": "urgent"
    }
  ]
}
```

**Issues**:
1. `status` value "in progress" should be "in_progress" (underscore, not space)
2. `priority` value "urgent" is not in the allowed enum (should be "high", "medium", "low")
3. Missing required field: `lastUpdated`
4. Missing required fields in task: `description`, `createdAt`, `updatedAt`

### Agent Behavior

**Step 1**: Validate content against schema before write
```python
def write_file(path: str, content: str):
    # Check if file has registered schema
    schema = get_schema_for_path(path)

    if schema:
        try:
            data = json.loads(content)
            jsonschema.validate(data, schema)
        except jsonschema.ValidationError as e:
            log_error("Schema validation failed", e)
            raise SchemaValidationError(path, e)

    # Only write if validation passes
    with open(path, 'w') as f:
        f.write(content)
```

**Step 2**: Log validation failure
```json
{
  "timestamp": "2026-02-01T10:00:00.000Z",
  "level": "ERROR",
  "event": "VALIDATION_FAIL",
  "sessionId": "session-task-001",
  "operation": "write",
  "path": "TODO.json",
  "success": false,
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "Content does not match schema",
    "schema": "todo.schema.json",
    "errors": [
      {
        "path": "tasks[0].status",
        "message": "'in progress' is not one of ['pending', 'in_progress', 'completed', 'failed', 'blocked']"
      },
      {
        "path": "tasks[0].priority",
        "message": "'urgent' is not one of ['high', 'medium', 'low']"
      },
      {
        "path": "",
        "message": "'lastUpdated' is a required property"
      },
      {
        "path": "tasks[0]",
        "message": "'description' is a required property"
      }
    ]
  }
}
```

**Step 3**: Stop operation and report
```
ERROR: Cannot write TODO.json - schema validation failed

Validation errors:
1. tasks[0].status: 'in progress' is not valid
   Valid values: pending, in_progress, completed, failed, blocked

2. tasks[0].priority: 'urgent' is not valid
   Valid values: high, medium, low

3. Missing required field: lastUpdated

4. tasks[0] missing required field: description

The file was NOT written. Please fix the data and try again.
```

### Correct Content

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-01T10:00:00Z",
  "tasks": [
    {
      "id": "task-001",
      "title": "Fix bug",
      "description": "Fix null pointer exception in auth module",
      "status": "in_progress",
      "priority": "high",
      "assignedSession": null,
      "createdAt": "2026-02-01T10:00:00Z",
      "updatedAt": "2026-02-01T10:00:00Z"
    }
  ]
}
```

### Why This Matters

Schema validation ensures:
- Data consistency across sessions
- Compatibility with other tools reading these files
- Early error detection before corrupting shared state
- Clear contracts for data format

---

## Example 4: Attempt to Write to Boot Repo

### Scenario

Agent attempts to write a file to the protected boot repository.

### Context

Agent incorrectly tries to modify a schema file in the boot repository to "fix" a perceived issue.

**Attempted Operation**:
```python
# Agent tries to modify schema
write_file('/workspace/agent-boot/schemas/state.schema.json', modified_schema)
```

### Agent Behavior

**Step 1**: Path validation check
```python
def validate_path_allowlist(path: str) -> bool:
    absolute_path = os.path.abspath(path)

    # Check against protected repositories
    if '/agent-boot/' in absolute_path:
        raise ProtectedPathError(
            path=path,
            reason="Boot repository is read-only"
        )

    return True
```

**Step 2**: repo-bridge intercepts
```json
{
  "timestamp": "2026-02-01T10:00:00.000Z",
  "level": "ERROR",
  "event": "PROHIBITION_BLOCK",
  "sessionId": "session-modify-001",
  "operation": "write",
  "path": "/workspace/agent-boot/schemas/state.schema.json",
  "success": false,
  "details": {
    "prohibition": "P-003",
    "name": "modify_boot_repository"
  },
  "error": {
    "code": "PROTECTED_PATH",
    "message": "Cannot write to boot repository",
    "severity": "critical",
    "action": "blocked"
  }
}
```

**Step 3**: Block operation completely
```
ERROR: Operation blocked by repo-bridge

Attempted to write to protected path:
  /workspace/agent-boot/schemas/state.schema.json

This path is protected because:
  The boot repository contains critical system files that ensure
  agent safety and correct operation. Modifying these files could
  disable safety validations or corrupt the agent system.

Prohibition: P-003 (Modify Boot Repository)
Severity: Critical

What you can do instead:
1. If you found a bug in the schema, report it to the maintainers
2. If you need custom validation, create a local override file
3. If you need different behavior, request a new contract version

This incident has been logged.
```

### Escalation

Because this is a critical prohibition, additional actions are taken:

**Step 4**: Alert generated
```json
{
  "timestamp": "2026-02-01T10:00:00.000Z",
  "level": "ALERT",
  "event": "CRITICAL_PROHIBITION",
  "sessionId": "session-modify-001",
  "details": {
    "prohibition": "P-003",
    "attemptedPath": "/workspace/agent-boot/schemas/state.schema.json",
    "agentId": "agent-001",
    "recommendation": "Review agent behavior and task instructions"
  }
}
```

### Why This Is Blocked

The boot repository is protected because:
1. **Safety**: Contains validation schemas that protect users
2. **Integrity**: Modifications could disable safety checks
3. **Trust**: Other workspaces depend on these files being unchanged
4. **Audit**: Changes to boot would affect audit trail reliability

---

## Summary: Error Handling Patterns

| Error Type | Detection | Response | Recovery |
|------------|-----------|----------|----------|
| Missing AGENT_LINK.md | Initialization check | Abort session | Manual setup required |
| Corrupted STATE.json | Schema validation | Recover from backup | Automatic from git history |
| Schema validation fail | Pre-write validation | Block write | Fix data and retry |
| Protected path write | Path allowlist check | Block + alert | Use alternative approach |

### Common Recovery Steps

1. **Check audit log**: `cat workspace/agent/last-run.log | jq .`
2. **Review git history**: `git log --oneline -10`
3. **Restore from backup**: `git checkout <commit> -- <file>`
4. **Reset state**: Delete STATE.json and reinitialize
5. **Contact support**: If automated recovery fails
