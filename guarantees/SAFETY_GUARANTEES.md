# Safety Guarantees

Promises the agent must keep. These constraints protect systems and users.

---

## Guarantee Summary

| Category | Guarantee |
|----------|-----------|
| File Size | Max 1MB per file, 10MB absolute |
| Commit Size | Max 50 files, 50MB total |
| Rate Limits | Reads 100/min, Writes 10/min |
| Protected Paths | .git/, AGENT_LINK.md, schemas/ |
| Branch Safety | No force push, main requires extra validation |
| Audit | All operations logged |
| Rollback | All changes reversible |

---

## File Size Limits

### S-001: Recommended Maximum (1 MB)
- Text files: 1 MB recommended
- Binary files: 1 MB hard limit
- Rationale: Large files bloat repositories

### S-002: Absolute Maximum (10 MB)
- No file may exceed 10 MB under any circumstances
- Use Git LFS for larger files
- This limit cannot be overridden

---

## Commit Size Limits

### S-003: Maximum Files Per Commit (50)
- Commits with >50 files are blocked
- Split large changes into logical commits
- Rationale: Large commits are hard to review and rollback

### S-004: Maximum Commit Size (50 MB)
- Total size of all files in commit < 50 MB
- Prevents massive single commits
- Rationale: Protects git performance

---

## Rate Limits

| Operation | Limit | Reset |
|-----------|-------|-------|
| File reads | 100/minute | Sliding window |
| File writes | 10/minute | Sliding window |
| Commits | 10/minute | Sliding window |
| Main commits | 3/minute | Sliding window |
| Branch creates | 5/minute | Sliding window |
| PR operations | 5/minute | Sliding window |

When rate limit is hit:
1. Operation is blocked
2. Warning logged with reset time
3. Agent waits for limit to reset

---

## Protected Paths

### S-009: Cannot Write to .git/
- All paths under `.git/` are protected
- Use git commands, never direct file access
- Rationale: Prevents repository corruption

### S-010: Cannot Delete AGENT_LINK.md
- This file links workspace to agent system
- Can modify, cannot delete
- Rationale: Required for agent operation

### S-011: Cannot Delete Schemas
- `schemas/` directory is protected
- Located in agent-boot repository
- Rationale: Schemas enable validation

---

## Branch Protection

### S-012: No Force Push
- Force push prohibited on all branches
- Applies to `--force`, `-f`, and `+branch` syntax
- Alternative: Use `git revert`

### S-013: Main Branch Extra Validation
- CHANGELOG.md must be updated
- STATE.json must be valid
- All schemas must validate
- No secrets in any file
- Lower rate limit (3/min vs 10/min)

---

## Data Integrity Guarantees

### All Changes Are Logged
- Every read operation logged
- Every write operation logged
- Every validation result logged
- See `AUDIT_REQUIREMENTS.md`

### All Changes Are Reversible
- Git history preserved
- No force push = audit trail intact
- Rollback procedures documented
- See `ROLLBACK_PROCEDURES.md`

### State Is Preserved
- STATE.json updated every 5 minutes maximum
- Checkpoints enable session resume
- Errors recorded in state

---

## Session Guarantees

### S-014: State Freshness
- STATE.json must be updated within 5 minutes
- Stale state blocks further operations
- Ensures recovery is possible

### S-011: Session Isolation
- Cannot access other sessions' state
- Cannot modify other sessions' work
- Each session operates independently

---

## Constraint Configuration

These constraints are defined in the contract and enforced by repo-bridge:

```json
{
  "constraints": {
    "fileSizeSoftLimit": 1048576,
    "fileSizeHardLimit": 10485760,
    "maxFilesPerCommit": 50,
    "maxCommitSize": 52428800,
    "readRateLimit": 100,
    "writeRateLimit": 10,
    "commitRateLimit": 10,
    "mainCommitRateLimit": 3,
    "stateMaxAge": 300
  }
}
```

**These values cannot be modified by the agent.**

---

## Violation Handling

| Constraint | Severity | Response |
|------------|----------|----------|
| File size (soft) | Warning | Allow with log |
| File size (hard) | Error | Block operation |
| Commit size | Error | Block commit |
| Rate limit | Warning | Wait and retry |
| Protected path | Error | Block operation |
| Force push | Critical | Block and alert |

All violations logged to audit trail:
```json
{
  "event": "CONSTRAINT_VIOLATION",
  "constraint": "S-002",
  "limit": "10MB",
  "actual": "15MB",
  "action": "blocked"
}
```

---

## User Guarantees

As a user, you are guaranteed:

1. **No Surprise Changes**: Agent asks before destructive actions
2. **Complete Audit Trail**: Everything logged and reviewable
3. **Easy Rollback**: Any change can be undone
4. **Size Limits**: Repository won't be bloated
5. **Rate Protection**: Agent won't overwhelm systems
6. **Secret Safety**: Credentials won't be committed
7. **History Preservation**: Git history always intact
