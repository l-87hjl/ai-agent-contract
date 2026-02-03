# Allowed Actions

Actions the agent is permitted to take, with conditions and rate limits.

---

## Capability Index

| ID | Operation | Rate Limit | Validation |
|----|-----------|------------|------------|
| C-001 | Read Files | 100/min | Path allowlist |
| C-002 | Write Files | 10/min | Schema + size + path |
| C-003 | Create Branches | 5/min | Name convention |
| C-004 | Commit Changes | 10/min | Changelog + STATE |
| C-005 | Commit to Main | 3/min | Full validation |
| C-006 | Create Pull Requests | 5/min | Description + branch |
| C-007 | Update Pull Requests | 10/min | PR exists |
| C-008 | Update STATE.json | 10/min | Schema validation |
| C-009 | Update TODO.json | 10/min | Schema validation |
| C-010 | Update CHANGELOG.md | 10/min | Format validation |

---

## File Operations

### In Workspace Repository (Read/Write)

- **Read any file** (C-001)
  - Path must be within authorized workspace
  - Cannot read from `.git/` directory
  - Rate limit: 100 reads/minute

- **Create new files** (C-002)
  - Must pass path allowlist check
  - Must be < 1MB (10MB absolute max)
  - Must pass secret detection scan

- **Modify existing files** (C-002)
  - Same validations as create
  - JSON files must validate against schema if registered

- **Delete files** (with restrictions)
  - Cannot delete protected files (AGENT_LINK.md, STATE.json)
  - Requires explicit approval for bulk deletions

- **Commit changes** (C-004)
  - Must have non-empty commit message
  - Staged files must pass all write validations

### In Boot/Contract Repositories (Read-Only)

- Read any file
- Write/modify operations prohibited (see P-003)

---

## State Management

- **Update STATE.json** (C-008)
  - Must validate against `validation/state-schema.json`
  - Required: `sessionId`, `status`, `lastUpdated`
  - Valid status: `idle`, `running`, `paused`, `completed`, `failed`

- **Update TODO.json** (C-009)
  - Must validate against `validation/todo-schema.json`
  - Valid transitions: `pending` → `in_progress` → `completed`
  - Cannot delete tasks, only change status

- **Append to CHANGELOG.md** (C-010)
  - Must follow Keep a Changelog format
  - Entry date must be today or later
  - Cannot modify or remove existing entries

- **Create files in workspace directories**
  - `inputs/` - Input data for tasks
  - `outputs/` - Results and generated content
  - `scratch/` - Temporary work files

---

## Git Operations

### Branches (C-003)

- Create branches with valid names
  - Pattern: `^[a-z]+/[a-z0-9-]+$`
  - Prefixes: `feature/`, `fix/`, `agent/`, `update/`
  - Max length: 100 characters
- Cannot create protected branch names (`main`, `master`)
- Rate limit: 5 branches/minute

### Commits to Feature Branches (C-004)

- Non-empty commit message required
- No files in protected paths
- Rate limit: 10 commits/minute

### Commits to Main Branch (C-005)

**Stricter requirements:**
```
CAN commit to main IF:
  - STATE.json updated AND valid
  - CHANGELOG.md has today's entry
  - All schemas validate
  - No secrets detected
  - Each file < 1MB
  - Total commit < 10MB
```
- Rate limit: 3 commits/minute

### Pull Requests (C-006, C-007)

- Create PRs with title and description
- Update PR title, description, labels
- Add reviewers (cannot remove)
- Rate limit: 5 creates/min, 10 updates/min

---

## Communication

- Ask clarifying questions
- Report progress on tasks
- Explain reasoning and decisions
- Request confirmation for destructive actions
- Admit uncertainty or lack of knowledge

---

## Task Management

- Work on assigned tasks
- Break large tasks into smaller steps
- Mark tasks as complete when done
- Add new tasks discovered during work
- Reprioritize with user approval

---

## Error Handling

- Detect and report errors
- Attempt recovery with documented steps
- Request help when stuck
- Document errors in STATE.json

---

## Validation Requirements

Before any write operation, these validations must pass:

| Validation | Check |
|------------|-------|
| V-001 | Path is in allowlist |
| V-002 | File size < 1MB |
| V-003 | JSON validates against schema |
| V-004 | No secrets detected |

Before commits to main, additionally:

| Validation | Check |
|------------|-------|
| V-005 | CHANGELOG.md updated |
| V-006 | STATE.json current |
| V-007 | No secrets in staged files |
| V-008 | Total size < 50MB |

See `REQUIRED_VALIDATIONS.md` for complete validation details.
