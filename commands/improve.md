---
description: "Start the autonomous improvement loop using isolated git worktrees. Runs N iterations (default 5). Each experiment runs on its own branch — the main codebase is never touched until a winning change is merged. Pass a number and/or a quoted focus string, e.g. /autoimprove:improve 10 \"Fix all any types\""
arguments: optional number of iterations (default: 5), optional quoted focus string
---

Parse $ARGUMENTS:
- If a number is present, use it as the iteration count. Otherwise default to 5.
- If a quoted string is present (e.g. `"Replace fetch() with apiClient"`), use it as the **FOCUS** for this run.
- Both can be combined: `/autoimprove:improve 10 "Fix any types in convex/"` → 10 iterations, focused on that task.

Start the autoimprove loop for the parsed iteration count.

**Pre-flight:**
1. Check `autoimprove.config.md` exists. If not, stop: "Run /autoimprove:setup first."
2. Confirm git working tree is clean. If not, stop: "Please commit or stash changes first."
3. Run worktree setup (create `.autoimprove-wt/`, update `.gitignore`).
4. Run measure skill in the main directory — record as BASELINE.
5. Tell the user:
   > "Starting autoimprove — N iterations. Baseline: XX/100.
   > **Focus: [focus string]** (or "all improvement areas" if no focus given)
   > Each experiment runs in an isolated git worktree.
   > Your main branch will not be modified until a winning change is confirmed.
   > Logging all results to autoimprove-log.md."

**Run the improve-loop skill** for the requested number of iterations, passing the FOCUS string if one was provided.

**After all iterations — session summary:**
- Clean up all remaining worktrees (worktree skill cleanup step)
- Report: total iterations, wins vs discards, baseline → final score
- List all commits added to main (one per winning experiment)
- Show: `git log --oneline -N` where N = number of wins
- Remind user: "Review with `git log --oneline` and `git diff HEAD~N` before pushing."
