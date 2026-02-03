# Rollback Procedure

This document defines how to undo agent changes when they are incorrect, unwanted, or cause problems. All agent actions are designed to be reversible.

## Rollback Decision Tree

```
                    ┌─────────────────────┐
                    │  Need to undo       │
                    │  agent changes?     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │ Have changes been   │
                    │ pushed to remote?   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │ NO             │                │ YES
              ▼                │                ▼
    ┌─────────────────┐       │      ┌─────────────────┐
    │ Is branch       │       │      │ Is branch       │
    │ shared with     │       │      │ shared with     │
    │ others?         │       │      │ others?         │
    └────────┬────────┘       │      └────────┬────────┘
             │                │               │
     ┌───────┴───────┐       │       ┌───────┴───────┐
     │ NO       YES  │       │       │ NO       YES  │
     ▼           ▼   │       │       ▼           ▼
┌─────────┐ ┌───────┐│       │  ┌─────────┐ ┌───────┐
│ git     │ │ git   ││       │  │ git     │ │ git   │
│ reset   │ │revert ││       │  │ reset + │ │revert │
│ --hard  │ │       ││       │  │ force   │ │       │
└─────────┘ └───────┘│       │  │ push*   │ └───────┘
                     │       │  └─────────┘
                     │       │      │
                     │       │      │ *Only on agent
                     │       │      │  branches, never
                     │       │      │  on main
```

## Strategy 1: Git Revert (Safe, Shared Branches)

**When to Use**:
- Changes have been pushed to remote
- Branch is shared with other collaborators
- Need to maintain complete history
- Undoing changes on `main` branch

**How It Works**:
Git revert creates a new commit that undoes the changes from a previous commit. The original commit remains in history.

### Revert Single Commit

```bash
# Identify the commit to revert
git log --oneline -10

# Output example:
# a1b2c3d Agent: Add feature X
# e4f5g6h Previous commit
# ...

# Revert the agent commit
git revert a1b2c3d --no-edit

# This creates a new commit that undoes a1b2c3d
```

### Revert Multiple Commits

```bash
# Revert a range of commits (oldest to newest)
git revert --no-edit e4f5g6h..a1b2c3d

# Or revert specific commits individually
git revert --no-edit a1b2c3d
git revert --no-edit b2c3d4e
git revert --no-edit c3d4e5f
```

### Revert with Conflicts

```bash
# Start revert
git revert a1b2c3d

# If conflicts occur:
# 1. Edit conflicting files to resolve
# 2. Stage resolved files
git add <resolved-files>

# 3. Continue revert
git revert --continue

# Or abort if too complex
git revert --abort
```

### Push Reverted Changes

```bash
# After revert commits are created locally
git push origin <branch-name>
```

---

## Strategy 2: Git Reset (Local Only, Not Shared)

**When to Use**:
- Changes have NOT been pushed to remote
- Working on a personal/agent branch
- Want to completely remove commits from history
- Starting fresh on a task

**Warning**: Never use reset on shared branches or after pushing.

### Reset to Specific Commit

```bash
# View commit history
git log --oneline -10

# Reset to commit before agent changes (keeps files as unstaged changes)
git reset e4f5g6h

# Reset and discard all changes completely
git reset --hard e4f5g6h
```

### Reset Modes

| Mode | Command | Effect |
|------|---------|--------|
| Soft | `git reset --soft <commit>` | Moves HEAD, keeps changes staged |
| Mixed | `git reset <commit>` | Moves HEAD, keeps changes unstaged |
| Hard | `git reset --hard <commit>` | Moves HEAD, discards all changes |

### Reset Last N Commits

```bash
# Undo last commit, keep changes
git reset HEAD~1

# Undo last 3 commits, keep changes
git reset HEAD~3

# Undo last commit, discard changes
git reset --hard HEAD~1
```

---

## Strategy 3: Restore STATE.json from Previous Commit

**When to Use**:
- STATE.json is corrupted or incorrect
- Need to resume from a known good state
- Agent session ended unexpectedly

### Find Previous STATE.json

```bash
# View STATE.json history
git log --oneline -- STATE.json

# Output:
# a1b2c3d Agent: Update state to running
# x7y8z9w Agent: Initialize session
# p4q5r6s Previous session state
```

### Restore Specific Version

```bash
# Restore STATE.json from specific commit
git checkout p4q5r6s -- STATE.json

# Verify restoration
cat STATE.json

# Stage and commit the restoration
git add STATE.json
git commit -m "Restore STATE.json from commit p4q5r6s"
```

### Restore from Backup

If automatic backups exist:

```bash
# Check for backup files
ls -la agent/backups/

# Copy backup to STATE.json
cp agent/backups/STATE.json.bak STATE.json

# Verify and commit
git add STATE.json
git commit -m "Restore STATE.json from backup"
```

---

## Strategy 4: Branch-Based Recovery

**When to Use**:
- Major changes need to be undone
- Want to preserve agent work for reference
- Clean restart needed

### Create Recovery Branch

```bash
# Save current state to a recovery branch
git checkout -b recovery/agent-changes-2026-02-01

# Return to original branch
git checkout main

# Reset main to before agent changes
git reset --hard <commit-before-agent>
```

### Cherry-Pick Good Changes

If some agent changes were good:

```bash
# From clean branch, cherry-pick specific commits
git cherry-pick a1b2c3d  # Good commit 1
git cherry-pick d4e5f6g  # Good commit 2
# Skip bad commits
```

---

## Rollback Scenarios

### Scenario 1: Agent Made Wrong Code Changes

```bash
# 1. Identify the problematic commit
git log --oneline -5
# a1b2c3d Agent: Refactor auth module  <- Wrong changes
# e4f5g6h Previous good state

# 2. If on shared branch, use revert
git revert a1b2c3d --no-edit

# 3. If on personal branch and not pushed, use reset
git reset --hard e4f5g6h
```

### Scenario 2: STATE.json Corrupted

```bash
# 1. View STATE.json history
git log --oneline -5 -- STATE.json

# 2. Restore from last known good commit
git checkout <good-commit> -- STATE.json

# 3. Verify the content
cat STATE.json | jq .

# 4. Commit the fix
git add STATE.json
git commit -m "Restore STATE.json from <good-commit>"
```

### Scenario 3: Agent Created Wrong Files

```bash
# 1. If committed, revert the commit
git revert <commit-with-wrong-files>

# 2. If staged but not committed
git reset HEAD <wrong-file>
rm <wrong-file>

# 3. If only in working directory
rm <wrong-file>
```

### Scenario 4: Need to Undo Everything from a Session

```bash
# 1. Find the commit before the session started
git log --oneline --since="2026-02-01 09:00" --until="2026-02-01 17:00"

# 2. Find the parent of the first session commit
git log --oneline -1 <first-session-commit>^

# 3. Revert all session commits (if shared branch)
git revert --no-edit <first-session-commit>^..<last-session-commit>

# 4. Or reset (if personal branch, not pushed)
git reset --hard <commit-before-session>
```

---

## Automated Rollback Script

For convenience, this script automates common rollback operations:

```bash
#!/bin/bash
# rollback-agent.sh - Rollback agent changes

set -e

COMMAND=$1
TARGET=$2

case $COMMAND in
  "revert-last")
    # Revert the last agent commit
    LAST_AGENT_COMMIT=$(git log --oneline --grep="Agent:" -1 --format="%H")
    if [ -n "$LAST_AGENT_COMMIT" ]; then
      git revert "$LAST_AGENT_COMMIT" --no-edit
      echo "Reverted commit: $LAST_AGENT_COMMIT"
    else
      echo "No agent commits found"
      exit 1
    fi
    ;;

  "restore-state")
    # Restore STATE.json from specified commit or last good version
    if [ -n "$TARGET" ]; then
      git checkout "$TARGET" -- STATE.json
    else
      # Find last valid STATE.json
      GOOD_COMMIT=$(git log --oneline -- STATE.json | head -2 | tail -1 | cut -d' ' -f1)
      git checkout "$GOOD_COMMIT" -- STATE.json
    fi
    echo "STATE.json restored"
    ;;

  "reset-branch")
    # Reset branch to specified commit (dangerous, requires confirmation)
    if [ -z "$TARGET" ]; then
      echo "Usage: rollback-agent.sh reset-branch <commit>"
      exit 1
    fi
    read -p "This will discard all changes after $TARGET. Continue? [y/N] " confirm
    if [ "$confirm" = "y" ]; then
      git reset --hard "$TARGET"
      echo "Branch reset to $TARGET"
    fi
    ;;

  "list-agent-commits")
    # List recent agent commits
    git log --oneline --grep="Agent:" -10
    ;;

  *)
    echo "Usage: rollback-agent.sh <command> [target]"
    echo "Commands:"
    echo "  revert-last       Revert the last agent commit"
    echo "  restore-state     Restore STATE.json (optionally from <commit>)"
    echo "  reset-branch      Reset branch to <commit> (dangerous)"
    echo "  list-agent-commits  List recent agent commits"
    ;;
esac
```

---

## Rollback Verification Checklist

After any rollback, verify:

- [ ] `git status` shows clean working directory (or expected changes)
- [ ] `git log` shows expected commit history
- [ ] STATE.json is valid: `cat STATE.json | jq .`
- [ ] Application still works: run tests
- [ ] No unintended files were affected: `git diff --stat <before>..<after>`

---

## When NOT to Rollback

Consider alternatives to rollback when:

1. **Minor issues**: Fix forward instead of reverting
2. **Partial problems**: Cherry-pick or edit specific files
3. **Learning opportunity**: Keep changes on branch for review
4. **Already deployed**: May need coordinated rollback with deployment

---

## Emergency Contacts

If rollback procedures fail or situation is critical:

1. Check audit log at `workspace/agent/last-run.log`
2. Review git reflog: `git reflog` (local recovery of lost commits)
3. Contact repository administrator
4. Escalate to system maintainer if data loss occurs
