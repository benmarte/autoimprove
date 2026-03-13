---
description: Resume an interrupted autoimprove session from where it left off
argument-hint: "[N] [\"focus string\"]"
---

Resume an interrupted autoimprove session by reading the log.

Parse $ARGUMENTS:
- If a number is present, use it as the iteration count override. Otherwise use the remaining count from the log.
- If a quoted string is present, use it as the **FOCUS** override. Otherwise inherit the focus from the interrupted session.

## Step 1: Read the log

Read `.claude/autoimprove/log.md` from the project root.

If the file doesn't exist, stop: "No previous session found. Use `/autoimprove:improve` to start one."

## Step 2: Find the last session

Find the last `## Session —` header in the log. If none found (legacy log format), stop: "No resumable session found. The log uses an older format without session headers. Use `/autoimprove:improve` to start a new session with the updated format."

## Step 3: Check session status

Parse the `**Status:**` line from the last session header.

- If `COMPLETED` → stop: "Last session completed successfully. Use `/autoimprove:improve` to start a new one."
- If `IN_PROGRESS` or any other non-COMPLETED value → this session was interrupted. Proceed.

## Step 4: Parse session metadata

Extract from the session header:
- `Planned`: total planned iterations (number after "Planned:")
- `Focus`: the focus string (text between quotes after "Focus:", or "all improvement areas")
- `Base commit`: the commit SHA (text after "Base commit:")
- `Baseline`: the baseline score (number before "/100" after "Baseline:")

## Step 5: Count completed iterations

Count the number of `## Iteration` or `### Iteration` headers that appear AFTER the last `## Session —` header. This is the completed count.

## Step 6: Calculate remaining iterations

- `remaining = Planned - completed`
- If the user provided N as an argument, use N instead of `remaining`
- `new_total = completed + remaining` (or `completed + N` if overridden)
- `start_at = completed + 1`

If remaining ≤ 0 and no N override: stop: "All planned iterations appear to have completed. Use `/autoimprove:improve` to start a new session."

## Step 7: Determine focus

- If the user provided a focus string, use it
- Otherwise use the focus from the session header
- If the session had "all improvement areas", pass no focus

## Step 8: Count existing experiments

Count all `**Branch:** autoimprove/experiment-` entries in the entire log to determine the experiment offset. This prevents branch name collisions with previous sessions.

## Step 9: Detect codebase changes

Compare `git rev-parse HEAD` with the base commit from the session header.

If they differ:
```
⚠️ Codebase has changed since last session (abc1234 → def5678). Re-measuring baseline.
```
Run the measure skill to get a fresh baseline. Use this new baseline instead of the logged one.

If they match: use the logged baseline score and skip re-measurement.

## Step 10: Clean up orphan worktrees

Run the rollback skill's cleanup step to remove any leftover experiment worktrees from the crashed session.

## Step 11: Print resume summary

```
━━━ Resuming Session ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
↩️  Continuing from iteration start_at/new_total
🎯 Focus: "focus string"
📊 Baseline: XX/100
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 12: Run the improve-loop skill

Pass these parameters to the improve-loop skill:
- **Iteration count:** `remaining` (or N if overridden)
- **Focus:** the determined focus string
- **session_mode:** `continue`
- **start_iteration:** `start_at`
- **total_iterations:** `new_total`
- **baseline_score:** the baseline (logged or freshly measured)
- **experiment_offset:** count of existing experiments in log

The improve-loop will handle the rest: create worktrees, iterate, log, and update the session status to COMPLETED when done.
