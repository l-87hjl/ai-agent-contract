# ai-agent-contract

Formal specifications and guarantees for AI agent behavior.

## Purpose

This repository contains the formal contract that defines:
- What the agent is capable of doing
- What the agent is prohibited from doing
- Guarantees the agent must provide
- Validation rules for agent outputs

## Access Level

**Read-Only** - Agents can read this repository but cannot modify it.

## Structure

```
ai-agent-contract/
├── README.md                       # This file
├── LICENSE                         # BSL 1.1
├── AGENT_ENTRY.md                  # Agent's starting point
├── capabilities/
│   ├── ALLOWED_ACTIONS.md          # What agent CAN do (10 capabilities)
│   ├── PROHIBITED_ACTIONS.md       # What agent CANNOT do (12 prohibitions)
│   ├── REQUIRED_VALIDATIONS.md     # 11 validation requirements
│   ├── CAN_DO.md                   # Detailed capability reference
│   └── CANNOT_DO.md                # Detailed prohibition reference
├── guarantees/
│   ├── SAFETY_GUARANTEES.md        # Promises agent must keep
│   ├── AUDIT_REQUIREMENTS.md       # What must be logged
│   ├── ROLLBACK_PROCEDURES.md      # How to undo mistakes
│   ├── SAFETY_CONSTRAINTS.md       # Detailed constraints reference
│   ├── AUDIT_TRAIL.md              # Detailed audit reference
│   └── ROLLBACK_PROCEDURE.md       # Detailed rollback reference
├── validation/
│   ├── state-schema.json           # Schema for STATE.json
│   └── todo-schema.json            # Schema for TODO.json
├── integration/
│   ├── valid-session-example.md    # Complete valid session
│   ├── invalid-session-examples.md # Error handling examples
│   └── multi-step-workflow.md      # Multi-session task example
└── versioning/
    ├── CONTRACT_VERSION.json       # Version metadata
    └── COMPATIBILITY_MATRIX.md     # Version compatibility
```

## Quick Start for Agents

1. Read `AGENT_ENTRY.md` first
2. Review `capabilities/ALLOWED_ACTIONS.md` for what you CAN do
3. Review `capabilities/PROHIBITED_ACTIONS.md` for what you CANNOT do
4. Validate outputs against schemas in `validation/`

## Core Principle: Append-Only

**Contracts are append-only.** Once published:

1. Existing guarantees cannot be weakened
2. Existing prohibitions cannot be removed
3. New capabilities require new validation requirements
4. All changes require a new version number

## Related Repositories

| Repository | Purpose | Access |
|------------|---------|--------|
| [agent-boot](../agent-boot) | Boot contract and rules | Read-only |
| [agent-project-space](../agent-project-space) | Active workspace | Read/Write |
| [repo-bridge](../repo-bridge) | Cross-repo infrastructure | Inaccessible |

## Quick Reference

### Capabilities (10)
| ID | Operation | Rate Limit |
|----|-----------|------------|
| C-001 | Read Files | 100/min |
| C-002 | Write Files | 10/min |
| C-003 | Create Branches | 5/min |
| C-004 | Commit Changes | 10/min |
| C-005 | Commit to Main | 3/min |
| C-006 | Create PRs | 5/min |
| C-007 | Update PRs | 10/min |
| C-008 | Update STATE.json | 10/min |
| C-009 | Update TODO.json | 10/min |
| C-010 | Update CHANGELOG | 10/min |

### Critical Prohibitions
- P-001: Force Push (Critical)
- P-002: Delete Repositories (Critical)
- P-003: Modify Boot Repository (Critical)
- P-004: Bypass Validation (Critical)
- P-005: Commit Secrets (Critical)

### Safety Guarantees
- Max 1MB per file, 10MB absolute
- Max 50 files per commit
- Rate limits enforced
- All actions logged
- Rollback always possible

## Contract Version

**Current Version**: 1.0.0
**Effective Date**: 2026-02-01
**Compatible Boot Versions**: 1.0.x

## Validation

To verify agent compliance:

1. Check audit log exists at `workspace/agent/last-run.log`
2. Verify all operations match ALLOWED_ACTIONS
3. Verify no operations match PROHIBITED_ACTIONS
4. Verify all writes passed validations
5. Verify safety constraints respected

## License

Business Source License 1.1 - See [LICENSE](./LICENSE)

---

**This contract is binding.** Any agent claiming compliance must implement all requirements exactly as specified.
