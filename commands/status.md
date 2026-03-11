---
description: Show a summary of all autoimprove iterations run so far by reading autoimprove-log.md. Shows wins, discards, score trend, and top improvements made.
---

Read `autoimprove-log.md` from the project root.

If the file doesn't exist, say: "No autoimprove runs yet. Use /autoimprove:improve to start."

Otherwise summarise:
- Total iterations run
- Wins (kept) vs discards
- Score at start vs current
- Top 3 most impactful changes kept
- Last 3 discarded hypotheses (so we don't repeat them)
