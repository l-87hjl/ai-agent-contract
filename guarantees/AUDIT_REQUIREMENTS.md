# Audit Requirements

What must be logged. Logging is mandatory and cannot be disabled.

---

## What Must Be Logged

| Category | Events |
|----------|--------|
| Session | Start, pause, resume, complete, abort |
| Files | Read, write, delete |
| Git | Commit, push, branch, checkout |
| Validation | Every check (pass or fail) |
| State | STATE.json and TODO.json updates |
| Errors | All errors and exceptions |
| Violations | Prohibition blocks, rate limits |

---

## Mandatory Events

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

All entries must be machine-readable JSON:

```json
{
  "timestamp": "2026-02-01T14:30:00.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-abc123",
  "operation": "write",
  "repository": "/workspace/my-project",
  "path": "src/index.ts",
  "success": true,
  "duration_ms": 45,
  "details": {}
}
```

### Required Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `timestamp` | ISO 8601 | Yes | When event occurred (UTC) |
| `level` | Enum | Yes | DEBUG, INFO, WARN, ERROR |
| `event` | String | Yes | Event type from list above |
| `sessionId` | String | Yes | Current session ID |
| `operation` | String | Yes | Specific operation |
| `success` | Boolean | Yes | Whether operation succeeded |

### Log Levels

| Level | Use |
|-------|-----|
| DEBUG | Detailed diagnostics |
| INFO | Normal operations |
| WARN | Potential issues, rate limit warnings |
| ERROR | Failures, validation failures |

---

## Example Log Entries

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
    "workspace": "/workspace/my-project"
  }
}
```

### File Write
```json
{
  "timestamp": "2026-02-01T14:05:00.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-abc123",
  "operation": "write",
  "path": "src/feature.ts",
  "success": true,
  "details": {
    "size_bytes": 1234,
    "validations_passed": ["V-001", "V-002", "V-004"]
  }
}
```

### Validation Failure
```json
{
  "timestamp": "2026-02-01T14:10:00.000Z",
  "level": "ERROR",
  "event": "VALIDATION_FAIL",
  "sessionId": "session-abc123",
  "operation": "validate",
  "path": "STATE.json",
  "success": false,
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "Invalid status value"
  }
}
```

### Prohibition Block
```json
{
  "timestamp": "2026-02-01T14:15:00.000Z",
  "level": "ERROR",
  "event": "PROHIBITION_BLOCK",
  "sessionId": "session-abc123",
  "operation": "git_push_force",
  "success": false,
  "details": {
    "prohibition": "P-001",
    "severity": "critical"
  }
}
```

---

## Log Storage

### Location
```
workspace/
└── agent/
    ├── last-run.log          # Current session
    └── logs/
        ├── session-001.log   # Historical
        ├── session-002.log
        └── ...
```

### File Format
- **Filename**: `session-{sessionId}.log`
- **Encoding**: UTF-8
- **Format**: JSON Lines (one JSON per line)
- **Max Size**: 10 MB per file

---

## Retention Policy

| Log Type | Retention |
|----------|-----------|
| `last-run.log` | Until next session |
| Session logs | Last 10 runs |
| Error logs | 30 days |

### Rotation
1. At session start, move `last-run.log` to `logs/`
2. If >10 files in `logs/`, delete oldest
3. Create new `last-run.log`

---

## Viewing Logs

```bash
# Current session
cat workspace/agent/last-run.log | jq .

# Filter errors only
cat workspace/agent/last-run.log | jq 'select(.level == "ERROR")'

# Count by event type
cat workspace/agent/last-run.log | jq -s 'group_by(.event) | map({event: .[0].event, count: length})'
```

---

## Verification Checklist

After each session, verify:

- [ ] `last-run.log` exists
- [ ] All entries are valid JSON
- [ ] SESSION_START event present
- [ ] SESSION_END event present
- [ ] All file operations logged
- [ ] All validations logged
- [ ] Timestamps in ISO 8601 format

---

## What NOT to Log

- Actual file contents (metadata only)
- Passwords or secrets (even if detected)
- Personal identifiable information
- Sensitive error details

---

## Security

- Logs are append-only during session
- Previous entries cannot be modified
- Tampering attempts must be detected
- Recommended permissions: `640`
