---
description: Show summary of all autoimprove iterations from the log
---

Read `.claude/autoimprove/log.md` from the project root.

If the file doesn't exist, say: "No autoimprove runs yet. Use /autoimprove:improve to start."

Otherwise:

### Check for interrupted session

Find the last `## Session —` header. If found, check its `**Status:**` line:

- If `IN_PROGRESS` (interrupted session), show:
  ```
  ━━━ Last Session: INTERRUPTED ━━━━━━━━━━━━
  📊 Score: [baseline] → [last score] (+delta)
  🔁 Completed: N/M iterations (remaining left)
  🎯 Focus: "focus string"
  💡 Use /autoimprove:continue to resume
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ```

- If `COMPLETED` or no session header (legacy format), show the standard summary.

### Standard summary

Summarise:
- Total iterations run
- Wins (kept) vs discards
- Score at start vs current
- Top 3 most impactful changes kept
- Last 3 discarded hypotheses (so we don't repeat them)
