# Agent Prohibitions: Forbidden Operations

This document defines operations the agent is **explicitly forbidden** from performing. These prohibitions are absolute and cannot be overridden by user requests, configuration, or any other mechanism.

## Prohibition Index

| ID | Prohibition | Severity | Category |
|----|-------------|----------|----------|
| P-001 | Force Push | Critical | Git Safety |
| P-002 | Delete Repositories | Critical | Destructive |
| P-003 | Modify Boot Repository | Critical | System Integrity |
| P-004 | Bypass Validation | Critical | Security |
| P-005 | Commit Secrets | Critical | Security |
| P-006 | Write Large Files | High | Resource |
| P-007 | Execute Binaries | Critical | Security |
| P-008 | Delete Protected Files | High | System Integrity |
| P-009 | Write to .git Directory | Critical | Git Safety |
| P-010 | Disable Audit Logging | Critical | Accountability |
| P-011 | Modify Other Sessions | High | Isolation |
| P-012 | Exceed Rate Limits | Medium | Resource |

---

## P-001: Force Push

**Prohibition**: Agent must NEVER force push to any branch.

**Rationale**: Force pushing rewrites history and can destroy work from other contributors. It breaks the audit trail and makes rollback unreliable.

**What to Do Instead**:
- Create a new commit that reverts unwanted changes
- Use `git revert` to undo specific commits
- Create a new branch if major restructuring is needed
- Ask user to manually resolve if rebase is truly required

**Detection**:
```bash
# These commands are blocked:
git push --force
git push -f
git push --force-with-lease
git push origin +branch
```

**Enforcement**: repo-bridge blocks all force push commands.

---

## P-002: Delete Repositories

**Prohibition**: Agent must NEVER delete repositories, whether local clones or remote repositories.

**Rationale**: Repository deletion is irreversible and could destroy critical project history, configuration, and user work.

**What to Do Instead**:
- Archive repositories instead of deleting (if supported)
- Remove specific files or branches as needed
- Request user to delete manually if truly required
- Document why deletion was considered

**Detection**:
```bash
# These operations are blocked:
rm -rf .git
rm -rf /path/to/repo
gh repo delete
```

**Enforcement**: File system protection and API restrictions.

---

## P-003: Modify Boot Repository

**Prohibition**: Agent must NEVER write to, commit to, or modify the agent-boot repository in any way.

**Rationale**: The boot repository contains critical initialization schemas and validation logic. Modifications could compromise system integrity and disable safety mechanisms.

**What to Do Instead**:
- Report issues with boot repository to maintainers
- Document workarounds in workspace
- Use configuration overrides where explicitly supported
- Request boot repository updates through proper channels

**Detection**:
```
# Any write operation targeting boot repository is blocked:
/path/to/agent-boot/*
*/agent-boot/*
```

**Enforcement**: repo-bridge maintains allowlist of writable repositories.

---

## P-004: Bypass Validation

**Prohibition**: Agent must NEVER skip, disable, or circumvent validation requirements.

**Rationale**: Validations exist to ensure data integrity, prevent corruption, and maintain security. Bypassing them undermines all safety guarantees.

**Prohibited Actions**:
- Skipping schema validation
- Ignoring file size checks
- Bypassing path allowlist
- Disabling secret detection
- Modifying validation configuration
- Using flags like `--no-verify` on git commands

**What to Do Instead**:
- Fix the data to pass validation
- Report validation bugs if validation is incorrect
- Request user intervention if validation cannot be passed
- Document why validation is failing

**Detection**:
```bash
# These flags are blocked:
git commit --no-verify
git push --no-verify
--skip-validation
--force-validation
```

**Enforcement**: Command parsing and validation wrapper.

---

## P-005: Commit Secrets

**Prohibition**: Agent must NEVER commit files containing secrets, credentials, or sensitive authentication data.

**Rationale**: Secrets in version control are a critical security vulnerability. They persist in git history even after removal and can be extracted by malicious actors.

**Prohibited Content Patterns**:
```
# API Keys
API_KEY=...
APIKEY=...
api_key=...

# AWS
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AKIA[A-Z0-9]{16}

# Private Keys
-----BEGIN RSA PRIVATE KEY-----
-----BEGIN OPENSSH PRIVATE KEY-----
-----BEGIN PGP PRIVATE KEY-----

# Tokens
github_pat_...
ghp_...
gho_...
Bearer ...
token=...

# Passwords
password=...
passwd=...
pwd=...

# Database
DATABASE_URL=...://user:pass@...
mongodb://...
postgres://...
mysql://...
```

**What to Do Instead**:
- Use environment variables for secrets
- Reference `.env.example` with placeholder values
- Use secret management systems (Vault, AWS Secrets Manager)
- Add secret files to `.gitignore`
- Use git-crypt or similar for encrypted secrets

**Enforcement**: Pre-commit secret scanning on all staged files.

---

## P-006: Write Large Files

**Prohibition**: Agent must NEVER write files larger than 10MB or commit binary files larger than 1MB.

**Rationale**: Large files bloat repository size, slow clone/fetch operations, and may indicate binary artifacts that should use external storage.

**Size Limits**:
| File Type | Max Size |
|-----------|----------|
| Text files | 1 MB |
| JSON/config | 1 MB |
| Any binary | 1 MB |
| Single file absolute max | 10 MB |
| Total commit size | 50 MB |

**What to Do Instead**:
- Use Git LFS for large binary files
- Store artifacts in external storage (S3, artifact registry)
- Split large files into smaller chunks
- Compress files before storage
- Reference external URLs instead of embedding

**Detection**: File size check on all write operations.

**Enforcement**: Write operations blocked if size exceeds limits.

---

## P-007: Execute Binaries

**Prohibition**: Agent must NEVER download and execute binary files or scripts from untrusted sources.

**Rationale**: Executing untrusted code is a critical security vulnerability that could compromise the system, steal data, or enable further attacks.

**Prohibited Actions**:
- Downloading and running executables
- Executing scripts from URLs
- Running binaries not in approved paths
- Installing software from untrusted sources
- Running `curl | bash` or similar patterns

**What to Do Instead**:
- Use package managers with verified sources
- Verify checksums before execution
- Use pre-approved tool installations
- Request user to install required software
- Document software requirements

**Detection**:
```bash
# These patterns are blocked:
curl ... | bash
wget ... | sh
chmod +x && ./...
```

**Enforcement**: Command parsing and execution restrictions.

---

## P-008: Delete Protected Files

**Prohibition**: Agent must NEVER delete files that are essential for system operation or user safety.

**Protected Files**:
```
AGENT_LINK.md        # Links workspace to agent
STATE.json           # Session state
schemas/*            # Validation schemas
.git/                # Git internals
CHANGELOG.md         # Change history (modify only, never delete)
```

**Rationale**: These files are critical for agent operation, audit trail, and system integrity. Deleting them would break functionality or hide history.

**What to Do Instead**:
- Modify protected files through allowed operations
- Archive deprecated content within files
- Request user intervention for special cases
- Document why deletion was considered

**Enforcement**: File deletion blocked for protected paths.

---

## P-009: Write to .git Directory

**Prohibition**: Agent must NEVER write directly to the `.git` directory.

**Rationale**: The `.git` directory contains repository internals. Direct modification could corrupt the repository, break git operations, or create security vulnerabilities.

**Prohibited Paths**:
```
.git/config
.git/hooks/*
.git/objects/*
.git/refs/*
.git/HEAD
.git/index
.git/*
```

**What to Do Instead**:
- Use git commands to modify repository state
- Use approved git configuration commands
- Do not modify hooks directly
- Use git-managed methods for references

**Enforcement**: Path validation blocks all `.git/*` writes.

---

## P-010: Disable Audit Logging

**Prohibition**: Agent must NEVER disable, bypass, or tamper with audit logging.

**Rationale**: Audit logs are essential for accountability, debugging, and security forensics. Disabling them would hide agent actions and break trust guarantees.

**Prohibited Actions**:
- Disabling log output
- Deleting log files
- Modifying past log entries
- Redirecting logs to /dev/null
- Changing log file permissions to prevent writing

**What to Do Instead**:
- Always log all operations
- Use log rotation for space management
- Archive old logs instead of deleting
- Report logging issues to maintainers

**Enforcement**: Logging is mandatory and cannot be configured off.

---

## P-011: Modify Other Sessions

**Prohibition**: Agent must NEVER access or modify state from other agent sessions.

**Rationale**: Sessions must be isolated to prevent interference, ensure reproducibility, and maintain clear accountability for each session's actions.

**Prohibited Actions**:
- Reading other session's STATE.json
- Modifying other session's checkpoint data
- Accessing other session's working directory
- Impersonating another session's identity

**What to Do Instead**:
- Work only within current session's scope
- Use proper session resumption for continuing work
- Request user to transfer relevant data between sessions
- Document dependencies between sessions

**Enforcement**: Session ID validation on all state operations.

---

## P-012: Exceed Rate Limits

**Prohibition**: Agent must NEVER intentionally exceed rate limits or attempt to circumvent rate limiting.

**Rationale**: Rate limits protect shared resources, prevent runaway operations, and ensure fair access for all processes.

**Rate Limits** (from SAFETY_CONSTRAINTS.md):
| Operation | Limit |
|-----------|-------|
| File reads | 100/minute |
| File writes | 10/minute |
| Commits | 10/minute |
| Main branch commits | 3/minute |
| PR operations | 5/minute |

**Prohibited Actions**:
- Batching operations to exceed limits
- Using delays to just avoid detection
- Spawning sub-processes to bypass limits
- Modifying rate limit configuration

**What to Do Instead**:
- Plan operations to stay within limits
- Break large tasks across multiple sessions
- Request limit increases through proper channels
- Optimize to reduce unnecessary operations

**Enforcement**: Rate limiter tracks all operations with hard cutoff.

---

## Violation Handling

When a prohibition violation is detected:

1. **Block**: Operation is immediately blocked
2. **Log**: Violation is logged to audit trail with details
3. **Alert**: Critical violations trigger immediate alerts
4. **Abort**: Session may be aborted for critical violations
5. **Report**: Violation report generated for review

### Severity Levels

| Severity | Response |
|----------|----------|
| Critical | Block + abort session + alert |
| High | Block + log + continue with warning |
| Medium | Block + log + continue |

### Audit Log Entry

```json
{
  "timestamp": "2026-02-01T12:00:00Z",
  "level": "VIOLATION",
  "prohibition": "P-001",
  "operation": "git push --force origin main",
  "severity": "critical",
  "action": "blocked",
  "sessionId": "session-12345",
  "context": {
    "repository": "/workspace/my-project",
    "branch": "main"
  }
}
```
