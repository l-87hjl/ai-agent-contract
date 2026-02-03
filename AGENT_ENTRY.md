# Agent Entry Point - Contract Repository

## Repository Type
**Contract (Read-Only)**

## Purpose
This repository contains formal specifications defining your capabilities and constraints. It serves as the "legal agreement" governing agent behavior.

---

## Required Reading

| File | Purpose | Priority |
|------|---------|----------|
| `capabilities/ALLOWED_ACTIONS.md` | What you CAN do (10 capabilities) | Required |
| `capabilities/PROHIBITED_ACTIONS.md` | What you CANNOT do (12 prohibitions) | Required |
| `capabilities/REQUIRED_VALIDATIONS.md` | Validation requirements (11 checks) | Required |
| `guarantees/SAFETY_GUARANTEES.md` | Safety constraints you must respect | Required |
| `guarantees/AUDIT_REQUIREMENTS.md` | Logging requirements | Required |
| `guarantees/ROLLBACK_PROCEDURES.md` | How to undo mistakes | Reference |

---

## Quick Reference

### Capability IDs
| ID | Operation | Rate Limit |
|----|-----------|------------|
| C-001 | Read Files | 100/min |
| C-002 | Write Files | 10/min |
| C-003 | Create Branches | 5/min |
| C-004 | Commit Changes | 10/min |
| C-005 | Commit to Main | 3/min |
| C-006 | Create Pull Requests | 5/min |
| C-007 | Update Pull Requests | 10/min |
| C-008 | Update STATE.json | 10/min |
| C-009 | Update TODO.json | 10/min |
| C-010 | Update CHANGELOG.md | 10/min |

### Critical Prohibitions
| ID | Prohibition | Severity |
|----|-------------|----------|
| P-001 | Force Push | Critical |
| P-002 | Delete Repositories | Critical |
| P-003 | Modify Boot Repository | Critical |
| P-004 | Bypass Validation | Critical |
| P-005 | Commit Secrets | Critical |

---

## Validation

Before writing any file to the workspace, validate against:
- `validation/state-schema.json` for STATE.json
- `validation/todo-schema.json` for TODO.json

All writes require:
1. Path allowlist check (V-001)
2. File size check (V-002)
3. JSON schema validation if applicable (V-003)
4. Secret detection scan (V-004)

---

## Prohibited Actions

- DO NOT modify any files in this repository
- DO NOT modify files in the boot repository
- DO NOT ignore specifications defined here
- DO NOT skip validation steps
- DO NOT commit secrets or credentials

---

## Contract Version

**Current Version**: 1.0.0
**Effective Date**: 2026-02-01
**Compatible Boot Versions**: 1.0.x

See `versioning/CONTRACT_VERSION.json` for full version details.

---

## Related Repositories

| Repository | Access | Purpose |
|------------|--------|---------|
| `agent-boot` | Read-Only | Rules, protocols, and schemas |
| `agent-project-space` | Read/Write | Your active workspace |
| `repo-bridge` | Inaccessible | Infrastructure (enforces constraints) |

---

## Integration Examples

For complete session examples, see:
- `integration/valid-session-example.md` - Successful session walkthrough
- `integration/invalid-session-examples.md` - Error handling scenarios
- `integration/multi-step-workflow.md` - Multi-session task example
