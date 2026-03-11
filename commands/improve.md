---
description: Start the autonomous improvement loop. Runs N iterations (default 5) of propose → implement → measure → keep/discard. Pass a number to control iterations, e.g. /autoimprove:improve 10
arguments: optional number of iterations (default: 5)
---

Start the autoimprove loop for $ARGUMENTS iterations (use 5 if no number given).

Before starting:
1. Check `autoimprove.config.md` exists. If not, stop: "Run /autoimprove:setup first."
2. Confirm git working tree is clean (`git status`). If not, stop: "Please commit or stash changes first."
3. Run the measure skill and record the BASELINE score.
4. Tell the user: "Starting autoimprove — N iterations. Baseline: XX/100. Logging to autoimprove-log.md."

Then run the improve-loop skill for the requested number of iterations.

After all iterations complete:
- Report: total wins vs discards, score change (baseline → final)
- List all files changed and kept
- Remind user: "Review autoimprove-log.md and run `git diff` before committing."
