# AI Agent Contract Specification

This repository defines the **formal contract** between an AI agent and users/systems it interacts with. It specifies what the agent can do, what it cannot do, what guarantees it provides, and how to verify compliance.

## Purpose

The AI Agent Contract serves as the "legal agreement" that governs agent behavior. It provides:

- **Explicit Capabilities**: What operations the agent is authorized to perform
- **Clear Prohibitions**: What the agent must never do under any circumstances
- **Safety Guarantees**: Constraints that protect users and systems
- **Audit Requirements**: How agent actions are logged and verified
- **Rollback Procedures**: How to undo agent changes if needed

## Core Principle: Append-Only

**Contracts are append-only.** Once a contract version is published and in use:

1. Existing guarantees cannot be weakened
2. Existing prohibitions cannot be removed
3. New capabilities can only be added with new validation requirements
4. All changes require a new version number

This ensures that systems relying on contract guarantees remain protected even as the contract evolves.

## Repository Structure

```
ai-agent-contract/
├── README.md                          # This file
├── capabilities/
│   ├── CAN_DO.md                      # Allowed operations with conditions
│   ├── CANNOT_DO.md                   # Explicit prohibitions
│   └── REQUIRED_VALIDATIONS.md        # Validation requirements
├── guarantees/
│   ├── ROLLBACK_PROCEDURE.md          # How to undo agent changes
│   ├── AUDIT_TRAIL.md                 # Logging requirements
│   └── SAFETY_CONSTRAINTS.md          # Rate limits, size limits, protections
├── integration/
│   ├── valid-session-example.md       # Complete valid session walkthrough
│   ├── invalid-session-examples.md    # Error handling examples
│   └── multi-step-workflow.md         # Multi-session task example
└── versioning/
    ├── CONTRACT_VERSION.json          # Current version metadata
    └── COMPATIBILITY_MATRIX.md        # Version compatibility information
```

## Related Repositories

| Repository | Purpose | Relationship |
|------------|---------|--------------|
| [agent-boot](../agent-boot) | Implementation details, schemas, boot sequence | Implements this contract's requirements |
| Workspaces | User project repositories | Follow guarantees defined here |
| repo-bridge | Cross-repo communication | Enforces constraints defined here |

## Quick Reference

### What the Agent CAN Do

- Read files from workspace repositories
- Write files (with validation)
- Create branches and commits
- Create and update pull requests
- Update STATE.json and TODO.json

See [capabilities/CAN_DO.md](capabilities/CAN_DO.md) for full details.

### What the Agent CANNOT Do

- Force push to any branch
- Delete repositories
- Modify the boot repository
- Bypass validation requirements
- Commit secrets or large binary files

See [capabilities/CANNOT_DO.md](capabilities/CANNOT_DO.md) for full details.

### Safety Guarantees

- Maximum 1MB per file
- Maximum 50 files per commit
- Rate limited to 10 writes/minute
- All actions logged to audit trail
- Rollback always possible

See [guarantees/](guarantees/) for full details.

## Contract Version

**Current Version**: 1.0.0
**Effective Date**: 2026-02-01
**Compatible Boot Versions**: 1.0.x

See [versioning/CONTRACT_VERSION.json](versioning/CONTRACT_VERSION.json) for version history.

## Using This Contract

### For Agent Implementations

1. Parse `CONTRACT_VERSION.json` to verify compatibility
2. Load all capability files to build allowed operations list
3. Load all prohibition files to build blocked operations list
4. Implement validation requirements before any write operation
5. Follow audit trail format for all logging
6. Implement rollback procedures for error recovery

### For System Integrators

1. Verify contract version matches expected version
2. Configure repo-bridge to enforce prohibitions
3. Set up audit log collection from workspace paths
4. Implement monitoring for safety constraint violations
5. Test rollback procedures before production use

### For Users

1. Review capabilities to understand what the agent will do
2. Review prohibitions to understand safety boundaries
3. Check audit logs after agent sessions
4. Use rollback procedures if agent makes unwanted changes

## Verification

To verify an agent session was compliant:

1. Check audit log exists at `workspace/agent/last-run.log`
2. Verify all logged operations are in CAN_DO list
3. Verify no logged operations are in CANNOT_DO list
4. Verify all writes passed required validations
5. Verify safety constraints were respected

## Contributing

This contract is versioned and changes require:

1. Increment version in `CONTRACT_VERSION.json`
2. Add changelog entry with date and description
3. Update `COMPATIBILITY_MATRIX.md` if boot compatibility changes
4. All changes must maintain append-only principle

---

**This contract is binding.** Any agent claiming compliance with this contract version must implement all requirements exactly as specified.
