# Prohibited Actions

Actions the agent must never take. These prohibitions are absolute.

---

## Prohibition Index

| ID | Prohibition | Severity |
|----|-------------|----------|
| P-001 | Force Push | Critical |
| P-002 | Delete Repositories | Critical |
| P-003 | Modify Boot Repository | Critical |
| P-004 | Bypass Validation | Critical |
| P-005 | Commit Secrets | Critical |
| P-006 | Write Large Files | High |
| P-007 | Execute Binaries | Critical |
| P-008 | Delete Protected Files | High |
| P-009 | Write to .git Directory | Critical |
| P-010 | Disable Audit Logging | Critical |
| P-011 | Modify Other Sessions | High |
| P-012 | Exceed Rate Limits | Medium |

---

## Git Safety (Critical)

### P-001: Force Push
- **Never** use `git push --force`, `-f`, or `+branch`
- **Instead**: Use `git revert` to undo changes safely
- Rationale: Force push destroys history and breaks audit trail

### P-009: Write to .git Directory
- **Never** write to `.git/`, `.git/hooks/`, `.git/config`
- **Instead**: Use git commands to modify repository state
- Rationale: Direct modification can corrupt repository

---

## Access Violations (Critical)

### P-002: Delete Repositories
- **Never** delete repositories (local or remote)
- **Never** run `rm -rf .git` or equivalent
- **Instead**: Archive or request manual deletion
- Rationale: Irreversible data loss

### P-003: Modify Boot Repository
- **Never** write to `agent-boot/` or any boot repository
- **Never** modify schemas, validation rules, or boot config
- **Instead**: Report issues through proper channels
- Rationale: Boot contains safety mechanisms

### P-007: Execute Binaries
- **Never** download and execute untrusted binaries
- **Never** run `curl | bash` or similar patterns
- **Instead**: Use package managers with verified sources
- Rationale: Critical security vulnerability

---

## Data Integrity

### P-004: Bypass Validation
- **Never** skip schema validation
- **Never** use `--no-verify` flags
- **Never** modify validation configuration
- **Instead**: Fix data to pass validation
- Rationale: Validations ensure safety

### P-010: Disable Audit Logging
- **Never** disable, delete, or modify log files
- **Never** redirect logs to /dev/null
- **Instead**: Always log all operations
- Rationale: Accountability requires complete audit trail

### P-011: Modify Other Sessions
- **Never** access other session's STATE.json
- **Never** modify other session's checkpoint data
- **Instead**: Work only within current session scope
- Rationale: Session isolation ensures reproducibility

---

## Security

### P-005: Commit Secrets
- **Never** commit files containing:
  - API keys (`API_KEY=`, `APIKEY=`)
  - AWS credentials (`AKIA...`, `AWS_SECRET_ACCESS_KEY`)
  - GitHub tokens (`ghp_`, `github_pat_`)
  - Private keys (`-----BEGIN.*PRIVATE KEY-----`)
  - Database URLs with credentials
  - Passwords (`password=`, `passwd=`)
- **Instead**: Use environment variables or secret managers
- Rationale: Secrets in git persist forever

**Detection Patterns:**
```
AKIA[A-Z0-9]{16}              # AWS Access Key
ghp_[a-zA-Z0-9]{36}           # GitHub PAT
-----BEGIN.*PRIVATE KEY-----   # Private Key
(password|passwd|pwd)\s*=      # Passwords
```

---

## Resource Limits

### P-006: Write Large Files
- **Never** write files > 10MB
- **Never** write binary files > 1MB
- **Instead**: Use Git LFS or external storage
- Rationale: Prevents repository bloat

### P-012: Exceed Rate Limits
- **Never** intentionally exceed rate limits
- **Never** batch operations to circumvent limits
- **Instead**: Plan operations within limits
- Rationale: Protects shared resources

---

## Destructive Actions (Without Confirmation)

### P-008: Delete Protected Files
- **Never** delete:
  - `AGENT_LINK.md`
  - `STATE.json`
  - `schemas/*`
  - `CHANGELOG.md`
- **Instead**: Modify contents, never delete
- Rationale: These files are essential for operation

**General Destructive Actions:**
- Deleting files without explicit approval
- Overwriting content without backup
- Bulk modifications without review
- Irreversible changes without warning

---

## Scope Violations

- Working outside assigned repositories
- Modifying unrelated files
- Taking actions not requested
- Changing configuration without approval
- Probing for accessible repositories
- Attempting to access repo-bridge infrastructure

---

## Self-Modification

- Attempting to change own rules
- Modifying boot or contract repositories
- Circumventing constraints
- Disabling safety checks

---

## Violation Handling

| Severity | Response |
|----------|----------|
| Critical | Block + abort session + alert |
| High | Block + log + continue with warning |
| Medium | Block + log + continue |

All violations are logged to audit trail:
```json
{
  "event": "PROHIBITION_BLOCK",
  "prohibition": "P-001",
  "severity": "critical",
  "action": "blocked"
}
```

---

## What To Do Instead

| Prohibited | Alternative |
|------------|-------------|
| Force push | `git revert <commit>` |
| Delete repo | Archive or request manual deletion |
| Modify boot | Report issue to maintainers |
| Skip validation | Fix data to pass validation |
| Commit secrets | Use environment variables |
| Large files | Use Git LFS |
| Execute binaries | Use package managers |
| Delete protected files | Modify contents only |
