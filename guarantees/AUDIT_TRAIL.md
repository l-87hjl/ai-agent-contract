# Audit Trail Requirements

This document specifies what must be logged, the log format, where logs are stored, and retention policies. Audit logging is mandatory and cannot be disabled.

## Logging Requirements

### What Must Be Logged

Every agent operation must be logged, including:

| Category | Operations |
|----------|------------|
| **File Operations** | Read, write, delete, move, copy |
| **Git Operations** | Commit, push, pull, branch, checkout, merge |
| **Validation** | Every validation check (pass or fail) |
| **State Changes** | STATE.json updates, TODO.json updates |
| **Errors** | All errors, exceptions, and failures |
| **Session Events** | Start, pause, resume, complete, abort |

### Mandatory Log Events

These events must always generate log entries:

```
SESSION_START      - Agent session begins
SESSION_END        - Agent session completes or aborts
FILE_READ          - Any file read operation
FILE_WRITE         - Any file write operation
FILE_DELETE        - Any file deletion
GIT_COMMIT         - Any git commit
GIT_PUSH           - Any git push
VALIDATION_PASS    - Validation check passed
VALIDATION_FAIL    - Validation check failed
STATE_UPDATE       - STATE.json modified
ERROR              - Any error or exception
PROHIBITION_BLOCK  - Prohibited action attempted
RATE_LIMIT_HIT     - Rate limit reached
```

---

## Log Format

All log entries must follow this JSON format for machine readability:

### Standard Log Entry

```json
{
  "timestamp": "2026-02-01T14:30:00.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-abc123",
  "operation": "write",
  "repository": "/workspace/my-project",
  "path": "src/components/Button.tsx",
  "success": true,
  "duration_ms": 45,
  "details": {
    "size_bytes": 1234,
    "validations_passed": ["V-001", "V-002", "V-003"]
  }
}
```

### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `timestamp` | ISO 8601 | Yes | When the event occurred (UTC) |
| `level` | Enum | Yes | Log level: DEBUG, INFO, WARN, ERROR |
| `event` | String | Yes | Event type from mandatory list |
| `sessionId` | String | Yes | Current session identifier |
| `operation` | String | Yes | Specific operation performed |
| `repository` | String | No | Repository path if applicable |
| `path` | String | No | File path if applicable |
| `success` | Boolean | Yes | Whether operation succeeded |
| `duration_ms` | Number | No | Operation duration in milliseconds |
| `details` | Object | No | Additional context |
| `error` | Object | No | Error details if success=false |

### Log Levels

| Level | When to Use |
|-------|-------------|
| `DEBUG` | Detailed diagnostic information |
| `INFO` | Normal operations (reads, writes, commits) |
| `WARN` | Potential issues, rate limit approaches |
| `ERROR` | Operation failures, validation failures |

---

## Log Entry Examples

### Session Start

```json
{
  "timestamp": "2026-02-01T14:00:00.000Z",
  "level": "INFO",
  "event": "SESSION_START",
  "sessionId": "session-abc123",
  "operation": "start",
  "success": true,
  "details": {
    "contractVersion": "1.0.0",
    "bootVersion": "1.0.2",
    "workspace": "/workspace/my-project"
  }
}
```

### File Read

```json
{
  "timestamp": "2026-02-01T14:00:05.000Z",
  "level": "INFO",
  "event": "FILE_READ",
  "sessionId": "session-abc123",
  "operation": "read",
  "repository": "/workspace/my-project",
  "path": "src/index.ts",
  "success": true,
  "duration_ms": 12,
  "details": {
    "size_bytes": 2048,
    "encoding": "utf-8"
  }
}
```

### File Write with Validation

```json
{
  "timestamp": "2026-02-01T14:00:10.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-abc123",
  "operation": "write",
  "repository": "/workspace/my-project",
  "path": "src/components/NewFeature.tsx",
  "success": true,
  "duration_ms": 89,
  "details": {
    "size_bytes": 3456,
    "validations": [
      {"id": "V-001", "name": "path_allowlist", "passed": true},
      {"id": "V-002", "name": "file_size", "passed": true},
      {"id": "V-004", "name": "secret_scan", "passed": true}
    ]
  }
}
```

### Validation Failure

```json
{
  "timestamp": "2026-02-01T14:00:15.000Z",
  "level": "ERROR",
  "event": "VALIDATION_FAIL",
  "sessionId": "session-abc123",
  "operation": "validate",
  "repository": "/workspace/my-project",
  "path": "STATE.json",
  "success": false,
  "details": {
    "validation": "V-003",
    "name": "json_schema",
    "schema": "state.schema.json"
  },
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "'invalid_status' is not one of ['idle', 'running', 'paused', 'completed', 'failed']",
    "schemaPath": ["properties", "status", "enum"],
    "instancePath": ["status"]
  }
}
```

### Git Commit

```json
{
  "timestamp": "2026-02-01T14:00:20.000Z",
  "level": "INFO",
  "event": "GIT_COMMIT",
  "sessionId": "session-abc123",
  "operation": "commit",
  "repository": "/workspace/my-project",
  "success": true,
  "duration_ms": 234,
  "details": {
    "commitHash": "a1b2c3d4e5f6",
    "branch": "feature/add-auth",
    "message": "Add authentication module",
    "filesChanged": 3,
    "insertions": 150,
    "deletions": 20
  }
}
```

### Prohibition Blocked

```json
{
  "timestamp": "2026-02-01T14:00:25.000Z",
  "level": "ERROR",
  "event": "PROHIBITION_BLOCK",
  "sessionId": "session-abc123",
  "operation": "git_push_force",
  "repository": "/workspace/my-project",
  "success": false,
  "details": {
    "prohibition": "P-001",
    "name": "force_push",
    "attemptedCommand": "git push --force origin main"
  },
  "error": {
    "code": "PROHIBITED_OPERATION",
    "message": "Force push is prohibited. Use git revert instead.",
    "severity": "critical"
  }
}
```

### Rate Limit Hit

```json
{
  "timestamp": "2026-02-01T14:00:30.000Z",
  "level": "WARN",
  "event": "RATE_LIMIT_HIT",
  "sessionId": "session-abc123",
  "operation": "write",
  "success": false,
  "details": {
    "limit": "10 writes per minute",
    "current": 10,
    "resetAt": "2026-02-01T14:01:00.000Z"
  },
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Write rate limit exceeded. Wait 30 seconds."
  }
}
```

### Session End

```json
{
  "timestamp": "2026-02-01T14:30:00.000Z",
  "level": "INFO",
  "event": "SESSION_END",
  "sessionId": "session-abc123",
  "operation": "complete",
  "success": true,
  "details": {
    "status": "completed",
    "duration_minutes": 30,
    "summary": {
      "filesRead": 25,
      "filesWritten": 8,
      "commits": 3,
      "validationsPassed": 45,
      "validationsFailed": 2,
      "errors": 0
    }
  }
}
```

---

## Log Storage

### Location

Logs are stored in the workspace at:

```
workspace/
└── agent/
    ├── last-run.log          # Current/most recent session log
    ├── logs/
    │   ├── session-abc123.log    # Historical session logs
    │   ├── session-def456.log
    │   └── ...
    └── metrics/
        └── summary.json       # Aggregated metrics
```

### File Format

- **Filename**: `session-{sessionId}.log`
- **Encoding**: UTF-8
- **Format**: JSON Lines (one JSON object per line)
- **Max Size**: 10 MB per file (rotates if exceeded)

### Example Log File

```jsonl
{"timestamp":"2026-02-01T14:00:00.000Z","level":"INFO","event":"SESSION_START","sessionId":"session-abc123",...}
{"timestamp":"2026-02-01T14:00:05.000Z","level":"INFO","event":"FILE_READ","sessionId":"session-abc123",...}
{"timestamp":"2026-02-01T14:00:10.000Z","level":"INFO","event":"FILE_WRITE","sessionId":"session-abc123",...}
{"timestamp":"2026-02-01T14:30:00.000Z","level":"INFO","event":"SESSION_END","sessionId":"session-abc123",...}
```

---

## Retention Policy

### Retention Rules

| Log Type | Retention Period | Action After |
|----------|------------------|--------------|
| `last-run.log` | Until next session | Moved to logs/ |
| Session logs | Last 10 runs | Oldest deleted |
| Error logs | 30 days | Archived then deleted |
| Metrics | 90 days | Aggregated then deleted |

### Rotation Process

```bash
# Automatic rotation at session start
1. Move last-run.log to logs/session-{previousSessionId}.log
2. Count files in logs/
3. If > 10 files, delete oldest
4. Create new last-run.log for current session
```

### Manual Cleanup

```bash
# View log files
ls -la workspace/agent/logs/

# Check total log size
du -sh workspace/agent/

# Manually remove old logs (keep last 10)
cd workspace/agent/logs/
ls -t | tail -n +11 | xargs rm -f
```

---

## Log Analysis

### Viewing Current Session

```bash
# View last-run.log in real-time
tail -f workspace/agent/last-run.log

# Pretty print JSON
cat workspace/agent/last-run.log | jq .

# Filter by level
cat workspace/agent/last-run.log | jq 'select(.level == "ERROR")'
```

### Searching Logs

```bash
# Find all validation failures
cat workspace/agent/last-run.log | jq 'select(.event == "VALIDATION_FAIL")'

# Find all file writes to specific path
cat workspace/agent/last-run.log | jq 'select(.event == "FILE_WRITE" and .path | contains("src/"))'

# Count operations by type
cat workspace/agent/last-run.log | jq -s 'group_by(.event) | map({event: .[0].event, count: length})'
```

### Session Summary

```bash
# Generate session summary
cat workspace/agent/last-run.log | jq -s '
{
  sessionId: .[0].sessionId,
  start: (map(select(.event == "SESSION_START")) | .[0].timestamp),
  end: (map(select(.event == "SESSION_END")) | .[0].timestamp),
  totalEvents: length,
  byLevel: (group_by(.level) | map({level: .[0].level, count: length})),
  byEvent: (group_by(.event) | map({event: .[0].event, count: length})),
  errors: [.[] | select(.level == "ERROR")]
}
'
```

---

## Compliance Verification

To verify audit trail compliance:

### Checklist

- [ ] `last-run.log` exists after session
- [ ] All entries are valid JSON
- [ ] All entries have required fields
- [ ] SESSION_START and SESSION_END events present
- [ ] All file operations logged
- [ ] All validation results logged
- [ ] Timestamps are in ISO 8601 format
- [ ] Session IDs are consistent

### Automated Verification Script

```bash
#!/bin/bash
# verify-audit-trail.sh

LOG_FILE="${1:-workspace/agent/last-run.log}"

echo "Verifying audit trail: $LOG_FILE"

# Check file exists
if [ ! -f "$LOG_FILE" ]; then
  echo "FAIL: Log file does not exist"
  exit 1
fi

# Check all lines are valid JSON
if ! cat "$LOG_FILE" | jq -e . > /dev/null 2>&1; then
  echo "FAIL: Log contains invalid JSON"
  exit 1
fi

# Check for required events
SESSION_START=$(cat "$LOG_FILE" | jq -s '[.[] | select(.event == "SESSION_START")] | length')
SESSION_END=$(cat "$LOG_FILE" | jq -s '[.[] | select(.event == "SESSION_END")] | length')

if [ "$SESSION_START" -lt 1 ]; then
  echo "FAIL: Missing SESSION_START event"
  exit 1
fi

if [ "$SESSION_END" -lt 1 ]; then
  echo "WARN: Missing SESSION_END event (session may be in progress)"
fi

# Check required fields
MISSING_FIELDS=$(cat "$LOG_FILE" | jq -s '
  [.[] | select(.timestamp == null or .level == null or .event == null or .sessionId == null)]
  | length
')

if [ "$MISSING_FIELDS" -gt 0 ]; then
  echo "FAIL: $MISSING_FIELDS entries missing required fields"
  exit 1
fi

echo "PASS: Audit trail is compliant"
```

---

## Security Considerations

### What NOT to Log

- Actual file contents (only metadata)
- Passwords or secrets (even if detected)
- Personal identifiable information
- Full error stack traces with sensitive data

### Log Integrity

- Logs should be append-only during session
- Previous log entries cannot be modified
- Log tampering attempts must be detected
- Consider log signing for sensitive environments

### Access Control

- Logs should be readable by user and system
- Logs should not be world-readable
- Recommended permissions: `640` (owner read/write, group read)
