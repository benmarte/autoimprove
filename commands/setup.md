---
description: Detect project stack and generate .claude/autoimprove/config.md
---

If `.claude/autoimprove/config.md` already exists, ask: "Config already exists. Re-detect stack? (y/N)" If the user declines, skip detection and go straight to the audit.

If config doesn't exist or user confirms re-detection:
Run the detect-stack skill to analyse this project and produce `.claude/autoimprove/config.md`.

After the config is ready (newly generated or already existing), automatically run the audit skill to show the codebase report.

The audit will show the score breakdown, prioritized deficiencies, and offer to start fixing interactively.
