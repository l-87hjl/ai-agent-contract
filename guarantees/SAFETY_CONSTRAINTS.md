# Safety Constraints

This document defines the safety limits that protect systems and users from runaway operations, resource exhaustion, and unintended modifications.

## Constraint Summary

| Category | Constraint | Limit |
|----------|------------|-------|
| File Size | Maximum per file | 1 MB |
| File Size | Absolute maximum | 10 MB |
| Commit Size | Maximum files | 50 files |
| Commit Size | Maximum total size | 50 MB |
| Rate Limit | File reads | 100/minute |
| Rate Limit | File writes | 10/minute |
| Rate Limit | Commits | 10/minute |
| Rate Limit | Main branch commits | 3/minute |
| Protected Paths | .git/ directory | Cannot write |
| Protected Paths | AGENT_LINK.md | Cannot delete |
| Protected Paths | Schemas directory | Cannot delete |
| Branch Protection | Force push | Prohibited on all branches |
| Branch Protection | Main branch | Additional validation required |

---

## File Size Limits

### S-001: Maximum File Size (1 MB Recommended)

**Limit**: 1 MB per file (soft limit)

**Rationale**: Large files bloat repositories, slow operations, and often indicate data that belongs elsewhere.

**Enforcement**:
```python
def check_file_size(content: bytes, path: str) -> bool:
    size_mb = len(content) / (1024 * 1024)

    if size_mb > 1.0:
        log_warning(f"File {path} is {size_mb:.2f}MB, exceeds recommended 1MB")
        return False  # Block write

    return True
```

**Exceptions**:
- Text files may exceed 1 MB up to 10 MB with warning
- No exceptions for binary files

### S-002: Absolute Maximum File Size (10 MB)

**Limit**: 10 MB per file (hard limit)

**Rationale**: Files larger than 10 MB should use Git LFS or external storage.

**Enforcement**:
```python
def check_absolute_file_size(content: bytes, path: str) -> bool:
    size_mb = len(content) / (1024 * 1024)

    if size_mb > 10.0:
        log_error(f"File {path} is {size_mb:.2f}MB, exceeds absolute limit 10MB")
        raise FileSizeExceeded(path, size_mb)

    return True
```

**No Exceptions**: This is a hard limit that cannot be overridden.

---

## Commit Size Limits

### S-003: Maximum Files Per Commit (50 files)

**Limit**: 50 files maximum per commit

**Rationale**: Large commits are hard to review, debug, and rollback.

**Enforcement**:
```python
def check_commit_file_count(staged_files: List[str]) -> bool:
    if len(staged_files) > 50:
        log_error(f"Commit has {len(staged_files)} files, limit is 50")
        raise TooManyFilesInCommit(len(staged_files))

    return True
```

**What to Do Instead**:
- Split changes into logical commits
- Commit related files together
- Use separate branches for large refactors

### S-004: Maximum Commit Size (50 MB)

**Limit**: 50 MB total size per commit

**Rationale**: Massive commits strain git operations and storage.

**Enforcement**:
```python
def check_commit_total_size(staged_files: List[str]) -> bool:
    total_size = sum(get_file_size(f) for f in staged_files)
    total_mb = total_size / (1024 * 1024)

    if total_mb > 50:
        log_error(f"Commit total size is {total_mb:.2f}MB, limit is 50MB")
        raise CommitSizeExceeded(total_mb)

    return True
```

---

## Rate Limits

### S-005: File Read Rate Limit (100/minute)

**Limit**: Maximum 100 file reads per minute

**Rationale**: Prevents runaway read operations and ensures fair resource usage.

**Implementation**:
```python
class RateLimiter:
    def __init__(self):
        self.read_times = []

    def check_read_limit(self) -> bool:
        now = time.time()
        minute_ago = now - 60

        # Remove old entries
        self.read_times = [t for t in self.read_times if t > minute_ago]

        if len(self.read_times) >= 100:
            wait_time = self.read_times[0] - minute_ago
            log_warning(f"Read rate limit reached, wait {wait_time:.1f}s")
            return False

        self.read_times.append(now)
        return True
```

**Recovery**: Rate limit resets on sliding window (oldest operation ages out after 60 seconds).

### S-006: File Write Rate Limit (10/minute)

**Limit**: Maximum 10 file writes per minute

**Rationale**: Writes are more impactful than reads; limiting prevents runaway modifications.

**Implementation**:
```python
def check_write_limit(self) -> bool:
    now = time.time()
    minute_ago = now - 60

    self.write_times = [t for t in self.write_times if t > minute_ago]

    if len(self.write_times) >= 10:
        wait_time = self.write_times[0] - minute_ago
        log_warning(f"Write rate limit reached, wait {wait_time:.1f}s")
        return False

    self.write_times.append(now)
    return True
```

### S-007: Commit Rate Limit (10/minute)

**Limit**: Maximum 10 commits per minute

**Rationale**: Prevents commit spam and ensures each commit is intentional.

### S-008: Main Branch Commit Rate Limit (3/minute)

**Limit**: Maximum 3 commits to main branch per minute

**Rationale**: Main branch changes are high-impact; extra rate limiting provides safety margin.

---

## Protected Paths

### S-009: Cannot Write to .git/ Directory

**Protected Path**: `.git/` and all contents

**Rationale**: Direct modification of git internals can corrupt repository state.

**Enforcement**:
```python
PROTECTED_PATHS = [
    r'\.git(/|$)',  # .git directory and contents
]

def is_path_protected(path: str) -> bool:
    for pattern in PROTECTED_PATHS:
        if re.search(pattern, path):
            return True
    return False
```

**What to Do Instead**: Use git commands to modify repository state.

### S-010: Cannot Delete AGENT_LINK.md

**Protected File**: `AGENT_LINK.md`

**Rationale**: This file links the workspace to the agent system. Deleting it would break agent functionality.

**Enforcement**:
```python
UNDELETABLE_FILES = [
    'AGENT_LINK.md',
    'STATE.json',  # Can modify, cannot delete
]

def can_delete_file(path: str) -> bool:
    filename = os.path.basename(path)
    if filename in UNDELETABLE_FILES:
        log_error(f"Cannot delete protected file: {filename}")
        return False
    return True
```

**What to Do Instead**: Modify contents if needed, but never delete.

### S-011: Cannot Delete Schemas Directory

**Protected Path**: `schemas/` directory (in boot repository)

**Rationale**: Schemas are essential for validation. Deleting them would disable safety checks.

**Enforcement**: Write access to boot repository is blocked entirely.

---

## Branch Protection

### S-012: Cannot Force Push to Any Branch

**Protection**: Force push is prohibited on all branches

**Rationale**: Force pushing rewrites history and can destroy work.

**Enforcement**:
```python
BLOCKED_GIT_COMMANDS = [
    r'git push.*--force',
    r'git push.*-f\b',
    r'git push.*\+\w',  # git push origin +branch
]

def is_command_blocked(command: str) -> bool:
    for pattern in BLOCKED_GIT_COMMANDS:
        if re.search(pattern, command):
            log_error(f"Blocked command: {command}")
            return True
    return False
```

**What to Do Instead**: Use `git revert` to undo changes safely.

### S-013: Main Branch Additional Validation

**Protection**: Commits to main require additional validation

**Additional Requirements**:
1. CHANGELOG.md must be updated
2. STATE.json must be valid
3. All schemas must validate
4. No secrets in any file
5. Rate limit: 3 commits/minute (vs 10 for other branches)

**Enforcement**:
```python
def validate_main_branch_commit() -> bool:
    validations = [
        check_changelog_updated(),
        check_state_valid(),
        check_schemas_valid(),
        check_no_secrets(),
        check_main_rate_limit(),
    ]

    return all(validations)
```

---

## Resource Constraints

### S-014: Session Duration Limit

**Limit**: No hard limit, but STATE.json must be updated every 5 minutes

**Rationale**: Ensures state is preserved and session can be recovered.

**Enforcement**:
```python
def check_state_freshness() -> bool:
    state = load_state()
    last_updated = datetime.fromisoformat(state['lastUpdated'])

    if datetime.now() - last_updated > timedelta(minutes=5):
        log_warning("STATE.json is stale, must update before continuing")
        return False

    return True
```

### S-015: Maximum Open Files

**Limit**: 100 simultaneous file handles

**Rationale**: Prevents file descriptor exhaustion.

**Enforcement**: Implemented at system level, logged if limit approached.

---

## Constraint Violation Handling

### Violation Response Matrix

| Constraint | Severity | Response |
|------------|----------|----------|
| File size (soft) | Warning | Allow with log |
| File size (hard) | Error | Block operation |
| Commit size | Error | Block commit |
| Read rate limit | Warning | Wait and retry |
| Write rate limit | Warning | Wait and retry |
| Protected path | Error | Block operation |
| Force push | Critical | Block and alert |

### Violation Log Format

```json
{
  "timestamp": "2026-02-01T14:30:00.000Z",
  "level": "ERROR",
  "event": "CONSTRAINT_VIOLATION",
  "sessionId": "session-abc123",
  "constraint": "S-002",
  "name": "absolute_file_size",
  "limit": "10MB",
  "actual": "15.5MB",
  "path": "/workspace/project/large-file.bin",
  "action": "blocked"
}
```

---

## Constraint Configuration

Constraints are defined in the contract and cannot be modified by the agent. However, system administrators can adjust limits by deploying a new contract version.

### Current Configuration

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
  },
  "protectedPaths": [
    "\\.git(/|$)",
    "^agent-boot/",
    "^schemas/"
  ],
  "undeletableFiles": [
    "AGENT_LINK.md",
    "STATE.json"
  ]
}
```

---

## Monitoring and Alerts

### Constraint Metrics

Track these metrics for operational monitoring:

```
agent_file_size_bytes{path}          - Size of files written
agent_commit_file_count{session}     - Files per commit
agent_read_rate{session}             - Current read rate
agent_write_rate{session}            - Current write rate
agent_constraint_violations{type}    - Violation count by type
```

### Alert Conditions

| Condition | Alert Level |
|-----------|-------------|
| File size > 5 MB | Warning |
| Commit > 30 files | Warning |
| Rate limit at 80% | Warning |
| Protected path access | Error |
| Force push attempt | Critical |
| Repeated violations | Critical |
