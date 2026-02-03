# Agent Capabilities: Allowed Operations

This document defines all operations the agent is authorized to perform. Each capability includes conditions that must be met, rate limits, and validation requirements.

## Capability Index

| ID | Operation | Rate Limit | Validation Required |
|----|-----------|------------|---------------------|
| C-001 | Read Files | 100/min | Path allowlist |
| C-002 | Write Files | 10/min | Schema + size + path |
| C-003 | Create Branches | 5/min | Name convention |
| C-004 | Commit Changes | 10/min | Changelog + STATE |
| C-005 | Commit to Main | 3/min | Full validation suite |
| C-006 | Create Pull Requests | 5/min | Description + branch |
| C-007 | Update Pull Requests | 10/min | PR exists |
| C-008 | Update STATE.json | 10/min | Schema validation |
| C-009 | Update TODO.json | 10/min | Schema validation |
| C-010 | Update CHANGELOG.md | 10/min | Format validation |

---

## C-001: Read Files

**Description**: Agent can read files from workspace repositories.

**Conditions**:
- File path must be within an authorized workspace
- File path must pass allowlist check (not in protected paths)
- File must exist

**Rate Limit**: 100 reads per minute

**Validation Requirements**:
- Path must be absolute or resolvable to absolute
- Path must not traverse outside workspace (`../` resolved and checked)
- Path must not be in `.git/` directory

**Example**:
```
ALLOWED: /workspace/my-project/src/index.js
ALLOWED: /workspace/my-project/README.md
DENIED:  /workspace/my-project/.git/config
DENIED:  /workspace/my-project/../other-project/secret.env
```

---

## C-002: Write Files

**Description**: Agent can create or modify files in workspace repositories.

**Conditions**:
- File path must be within an authorized workspace
- File path must pass allowlist check
- File content must pass size validation
- If file has schema, content must validate against schema
- Parent directory must exist or be creatable

**Rate Limit**: 10 writes per minute

**Validation Requirements**:
1. Path allowlist check (see REQUIRED_VALIDATIONS.md)
2. File size check: content must be < 1MB
3. If writing JSON: must be valid JSON
4. If file has registered schema: must validate against schema
5. No secrets detection (see SAFETY_CONSTRAINTS.md)

**Example**:
```
ALLOWED: Write "Hello World" to /workspace/project/README.md
ALLOWED: Write valid JSON to /workspace/project/config.json
DENIED:  Write 5MB file to /workspace/project/large.bin
DENIED:  Write to /workspace/project/.git/hooks/pre-commit
DENIED:  Write API_KEY=sk-... to /workspace/project/.env
```

---

## C-003: Create Branches

**Description**: Agent can create new git branches in workspace repositories.

**Conditions**:
- Must be in a git repository
- Branch name must follow naming convention
- Base branch must exist

**Rate Limit**: 5 branch creations per minute

**Validation Requirements**:
- Branch name must match pattern: `^[a-z]+/[a-z0-9-]+$`
- Recommended prefixes: `feature/`, `fix/`, `agent/`, `update/`
- Branch name must not exceed 100 characters
- Must not create branch with name of protected branch

**Example**:
```
ALLOWED: feature/add-user-auth
ALLOWED: agent/task-12345
ALLOWED: fix/null-pointer-exception
DENIED:  main (protected)
DENIED:  Feature/Add-Auth (wrong case)
DENIED:  my branch name (spaces not allowed)
```

---

## C-004: Commit Changes

**Description**: Agent can commit staged changes to non-main branches.

**Conditions**:
- Changes must be staged
- Current branch must not be a protected branch (for unrestricted commits)
- Commit message must be non-empty

**Rate Limit**: 10 commits per minute

**Validation Requirements**:
- Commit message must be non-empty
- Commit message should follow conventional format (recommended)
- Staged files must all pass write validation
- No staged files in protected paths

**Example**:
```
ALLOWED: Commit to feature/my-feature with message "Add login form"
ALLOWED: Commit to agent/task-123 with message "Update config"
DENIED:  Commit with empty message
DENIED:  Commit including .git/config
```

---

## C-005: Commit to Main Branch

**Description**: Agent can commit directly to the main branch under strict conditions.

**Conditions**:
- STATE.json must be updated with current session info
- CHANGELOG.md must have entry for this change
- All schemas must validate
- No secrets present in any staged file
- All staged files pass size limits

**Rate Limit**: 3 commits to main per minute

**Validation Requirements**:
1. **State Validation**: STATE.json must exist and validate against schema
2. **Schema Validation**: All JSON files must validate against their schemas
3. **Changelog Updated**: CHANGELOG.md must have entry dated today or later
4. **No Secrets**: All staged files scanned for secret patterns
5. **Size Check**: Each file < 1MB, total commit < 10MB
6. **Path Check**: No files in protected paths

**Conditional Logic**:
```
Agent CAN commit to main branch IF:
  - state validation passes AND
  - schemas validate AND
  - changelog updated AND
  - no secrets detected AND
  - size limits respected AND
  - path allowlist passes
```

**Example**:
```
ALLOWED: Commit to main with updated STATE.json, CHANGELOG.md, and src/feature.js
DENIED:  Commit to main without CHANGELOG.md entry
DENIED:  Commit to main with invalid STATE.json
DENIED:  Commit to main with .env file containing secrets
```

---

## C-006: Create Pull Requests

**Description**: Agent can create pull requests for branches.

**Conditions**:
- Source branch must exist and be pushed to remote
- Target branch must exist
- PR title and description must be non-empty

**Rate Limit**: 5 PR creations per minute

**Validation Requirements**:
- Branch name must follow convention (see C-003)
- PR description must not be empty
- PR title must not be empty
- Source branch must have at least one commit ahead of target

**Example**:
```
ALLOWED: Create PR from feature/add-auth to main with title and description
ALLOWED: Create PR from agent/task-123 to develop
DENIED:  Create PR with empty description
DENIED:  Create PR from branch with no new commits
```

---

## C-007: Update Pull Requests

**Description**: Agent can update existing pull request metadata.

**Conditions**:
- PR must exist
- Agent must have been the creator or be authorized

**Rate Limit**: 10 PR updates per minute

**Validation Requirements**:
- PR ID must be valid
- New values must pass same validation as creation

**Allowed Updates**:
- Title
- Description
- Labels
- Reviewers (add only)
- Status (ready for review / draft)

**Example**:
```
ALLOWED: Update PR #123 description to include test results
ALLOWED: Add "needs-review" label to PR #456
DENIED:  Close PR created by another user
DENIED:  Remove reviewer from PR
```

---

## C-008: Update STATE.json

**Description**: Agent can update the STATE.json file to track session state.

**Conditions**:
- STATE.json must exist or be creatable
- New content must validate against state schema

**Rate Limit**: 10 updates per minute

**Validation Requirements**:
- Must validate against `schemas/state.schema.json` from agent-boot
- Required fields: `sessionId`, `status`, `lastUpdated`
- Status must be valid enum value

**Schema Reference**:
```json
{
  "sessionId": "string (required)",
  "status": "enum: idle|running|paused|completed|failed (required)",
  "lastUpdated": "ISO 8601 timestamp (required)",
  "currentTask": "string (optional)",
  "checkpoint": "object (optional)",
  "errors": "array of error objects (optional)"
}
```

**Example**:
```
ALLOWED: Update STATE.json with status "running" and currentTask "Fix bug #123"
ALLOWED: Update STATE.json with status "completed" and clear currentTask
DENIED:  Update STATE.json with status "unknown" (invalid enum)
DENIED:  Update STATE.json without sessionId
```

---

## C-009: Update TODO.json

**Description**: Agent can update TODO.json to track task status.

**Conditions**:
- TODO.json must exist
- Changes must validate against todo schema

**Rate Limit**: 10 updates per minute

**Validation Requirements**:
- Must validate against `schemas/todo.schema.json` from agent-boot
- Task IDs must be valid
- Status transitions must be valid

**Valid Status Transitions**:
```
pending → in_progress
pending → blocked
in_progress → completed
in_progress → failed
in_progress → blocked
blocked → in_progress
failed → in_progress (retry)
```

**Example**:
```
ALLOWED: Update task-123 status from "pending" to "in_progress"
ALLOWED: Update task-123 status from "in_progress" to "completed"
DENIED:  Update task-123 status from "completed" to "pending" (invalid transition)
DENIED:  Delete task from TODO.json (use status change instead)
```

---

## C-010: Update CHANGELOG.md

**Description**: Agent can add entries to CHANGELOG.md.

**Conditions**:
- CHANGELOG.md must exist
- Entry must be added, not removed
- Format must follow Keep a Changelog format

**Rate Limit**: 10 updates per minute

**Validation Requirements**:
- New entry must have valid date (today or future)
- Entry must be under valid section: Added, Changed, Fixed, Removed, Security
- Entry must not modify or remove existing entries
- Entry must be in Markdown format

**Format**:
```markdown
## [Version] - YYYY-MM-DD

### Added
- New feature description

### Changed
- Change description

### Fixed
- Bug fix description
```

**Example**:
```
ALLOWED: Add "### Fixed\n- Resolved null pointer in auth module" under today's date
ALLOWED: Add new version section with today's date
DENIED:  Remove existing changelog entry
DENIED:  Change date of existing entry
```

---

## Capability Verification

To verify an operation is allowed:

1. Identify the operation type (read, write, commit, etc.)
2. Find matching capability ID in this document
3. Verify all conditions are met
4. Check rate limit has not been exceeded
5. Run all validation requirements
6. If all pass, operation is ALLOWED
7. If any fail, operation is DENIED

Log all capability checks to audit trail (see AUDIT_TRAIL.md).
