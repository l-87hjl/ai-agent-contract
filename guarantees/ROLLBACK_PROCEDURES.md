# Rollback Procedures

How to undo agent changes if they're wrong.

---

## Decision Tree

```
Need to undo changes?
        │
        ▼
Changes pushed to remote?
    │           │
   NO          YES
    │           │
    ▼           ▼
Branch       Branch
shared?      shared?
  │   │        │   │
 NO  YES      NO  YES
  │   │        │   │
  ▼   ▼        ▼   ▼
reset revert  reset* revert
             (*+force,
              agent branch only)
```

---

## Strategy 1: Git Revert (Safe)

**When**: Shared branches, pushed changes, or main branch

**Why**: Creates new commit undoing changes. History preserved.

### Revert Single Commit
```bash
git log --oneline -5        # Find commit hash
git revert abc1234 --no-edit
git push
```

### Revert Multiple Commits
```bash
# Revert range (oldest to newest)
git revert --no-edit abc1234..def5678

# Or individually
git revert --no-edit abc1234
git revert --no-edit bcd2345
```

### Handle Conflicts
```bash
git revert abc1234
# If conflicts:
# 1. Edit files to resolve
# 2. git add <files>
# 3. git revert --continue
# Or: git revert --abort
```

---

## Strategy 2: Git Reset (Local Only)

**When**: Not pushed, personal branch, want clean history

**Warning**: Never use on shared branches or after pushing

### Reset to Commit
```bash
git log --oneline -5        # Find commit hash

# Keep changes unstaged
git reset abc1234

# Discard all changes
git reset --hard abc1234
```

### Reset Last N Commits
```bash
# Undo last commit, keep changes
git reset HEAD~1

# Undo last 3, discard changes
git reset --hard HEAD~3
```

### Reset Modes

| Mode | Command | Effect |
|------|---------|--------|
| Soft | `--soft` | Moves HEAD, changes staged |
| Mixed | (default) | Moves HEAD, changes unstaged |
| Hard | `--hard` | Moves HEAD, changes discarded |

---

## Strategy 3: Restore STATE.json

**When**: STATE.json corrupted or needs previous version

### From Git History
```bash
# Find previous versions
git log --oneline -- STATE.json

# Restore from specific commit
git checkout abc1234 -- STATE.json

# Commit the restoration
git add STATE.json
git commit -m "Restore STATE.json from abc1234"
```

### From Backup
```bash
# Check for backups
ls agent/backups/

# Restore
cp agent/backups/STATE.json.bak STATE.json
git add STATE.json
git commit -m "Restore STATE.json from backup"
```

---

## Common Scenarios

### Agent Made Wrong Code Changes

```bash
# If not pushed (personal branch)
git reset --hard HEAD~1

# If pushed or shared branch
git revert HEAD --no-edit
git push
```

### STATE.json Corrupted

```bash
# Find last good version
git log --oneline -5 -- STATE.json

# Restore it
git checkout <good-commit> -- STATE.json
git add STATE.json
git commit -m "Restore STATE.json"
```

### Agent Created Wrong Files

```bash
# If committed, revert
git revert <commit>

# If staged not committed
git reset HEAD <file>
rm <file>

# If only in working directory
rm <file>
```

### Undo Entire Session

```bash
# Find commit before session
git log --oneline --since="2026-02-01 09:00"

# Revert all session commits (shared branch)
git revert --no-edit <first>^..<last>

# Or reset (personal branch, not pushed)
git reset --hard <before-session>
```

---

## Quick Reference

| Situation | Command |
|-----------|---------|
| Undo last commit (keep changes) | `git reset HEAD~1` |
| Undo last commit (discard) | `git reset --hard HEAD~1` |
| Undo pushed commit | `git revert HEAD` |
| Restore single file | `git checkout <commit> -- <file>` |
| Undo merge | `git revert -m 1 <merge-commit>` |

---

## Rollback Script

```bash
#!/bin/bash
# rollback-agent.sh

case $1 in
  "revert-last")
    LAST=$(git log --oneline --grep="Agent:" -1 --format="%H")
    git revert "$LAST" --no-edit
    ;;
  "restore-state")
    GOOD=$(git log --oneline -- STATE.json | head -2 | tail -1 | cut -d' ' -f1)
    git checkout "$GOOD" -- STATE.json
    ;;
  "list-agent-commits")
    git log --oneline --grep="Agent:" -10
    ;;
  *)
    echo "Usage: rollback-agent.sh [revert-last|restore-state|list-agent-commits]"
    ;;
esac
```

---

## Verification After Rollback

- [ ] `git status` shows clean state
- [ ] `git log` shows expected history
- [ ] STATE.json validates: `cat STATE.json | jq .`
- [ ] Tests pass (if applicable)
- [ ] No unintended files affected

---

## When NOT to Rollback

- **Minor issues**: Fix forward instead
- **Partial problems**: Cherry-pick good changes
- **Learning opportunity**: Keep on branch for review
- **Already deployed**: Coordinate with deployment

---

## Emergency Recovery

If rollback fails:

1. Check audit log: `cat workspace/agent/last-run.log`
2. Use git reflog: `git reflog` (finds "lost" commits)
3. Contact repository administrator
4. Escalate if data loss occurs

Git reflog keeps commits for 90 days even after reset.
