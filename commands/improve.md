---
description: Start the autonomous improvement loop using isolated git worktrees
argument-hint: "[N] [\"focus string\"]"
---

Parse $ARGUMENTS:
- If a number is present, use it as the iteration count. Otherwise default to 5.
- If a quoted string is present (e.g. `"Replace fetch() with apiClient"`), use it as the **FOCUS** for this run.
- Both can be combined: `/autoimprove:improve 10 "Fix any types in convex/"` → 10 iterations, focused on that task.

Start the autoimprove loop for the parsed iteration count.

**Pre-flight:**
1. Check `.claude/autoimprove/config.md` exists. If not, stop: "Run /autoimprove:setup first."
2. Confirm git working tree is clean. If not, stop: "Please commit or stash changes first."
3. Run worktree setup (create `.claude/autoimprove/worktrees/`, update `.gitignore`).
4. Run measure skill in the main directory — record as BASELINE.
5. Print the pre-flight summary with progress indicators:
   ```
   ━━━ Pre-flight ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ✓ Config found
   ✓ Git working tree clean
   ✓ Worktree directory ready
   ✓ Baseline score: XX/100
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Starting N iterations. Focus: [focus or "all improvement areas"]
   Main branch is safe — experiments run in isolated worktrees.
   ```

**Run the improve-loop skill** for the requested number of iterations, passing the FOCUS string if one was provided. Pass `session_mode: new` so it creates a fresh session header in the log.

**After all iterations — session summary:**
- Clean up all remaining worktrees (worktree skill cleanup step)
- Report: total iterations, wins vs discards, baseline → final score
- List all commits added to main (one per winning experiment)
- Show: `git log --oneline -N` where N = number of wins
- Remind user: "Review with `git log --oneline` and `git diff HEAD~N` before pushing."
