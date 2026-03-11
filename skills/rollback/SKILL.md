---
description: Safely discard the current uncommitted changes from the last autoimprove iteration and restore the codebase to the last clean git state.
---

# Rollback Skill

Safely undo the last autoimprove change.

## Steps

1. Show what will be discarded:
```bash
git diff --stat
git status
```

2. Confirm with the user before proceeding (list files that will be reset).

3. Restore:
```bash
git checkout -- .
git clean -fd   # only if new files were created
```

4. Verify clean state:
```bash
git status
```

5. Report: "Rolled back successfully. Working tree is clean."

## Safety rules
- Never run `git reset --hard` — only `git checkout -- .`
- Never touch committed history
- If in doubt, show the diff and ask the user to confirm
