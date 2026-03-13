# Audit Command Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `/autoimprove:audit` command that scans the codebase, shows a prioritized deficiency report with token estimates, and optionally starts focused improve loops area-by-area.

**Architecture:** A new audit skill handles measurement, raw output parsing, efficiency ranking, and the interactive fix loop. The setup command is updated to auto-run audit after generating config. The measure command is removed (skill kept as internal utility).

**Tech Stack:** Markdown-based Claude Code plugin (all `.md` files)

---

## Chunk 1: Audit Skill + Command

### Task 1: Create the audit skill

**Files:**
- Create: `skills/audit/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
mkdir -p skills/audit
```

- [ ] **Step 2: Write the audit skill**

Create `skills/audit/SKILL.md` with the full audit logic:

```markdown
---
description: Scan the codebase for deficiencies, generate a prioritized report ranked by efficiency (points per iteration), and optionally transition into focused improve loops area-by-area until the user stops or the score hits 100.
---

# Audit Skill

## Pre-flight

1. Check `.claude/autoimprove/config.md` exists. If not, stop: "Run /autoimprove:setup first."
2. Check git is available and working tree is clean.

## Step 1: Run Measurement Suite

Run the measure skill to get the composite score and per-metric breakdown. Capture both the scores AND the raw command output for each metric.

For each metric defined in the config:
- Run the command (e.g., `tsc --noEmit 2>&1`, `pnpm test 2>&1`, `pnpm lint 2>&1`)
- Record the score (using the measure skill's scoring logic)
- Also capture the raw output for deficiency counting

## Step 2: Count Individual Deficiencies

Parse the raw output from each command to count specific issues:

- **Type errors:** Count lines matching error patterns (e.g., `error TS` for TypeScript, `error:` for Rust). Group by file.
- **Build:** Pass/fail only — no granular count. If build fails, it becomes top priority.
- **Tests:** Count passing vs total from test runner output. Identify failing test names if any.
- **Lint:** Count warning/error lines from linter output. Group by rule if possible.

## Step 3: Calculate Efficiency

For each metric with a gap (score < max):

1. **Point gap** = max_weight - current_score
2. **Estimated iterations** = `ceil(issue_count / issues_per_iteration)` using these heuristics:
   - Type errors: ~3 per iteration (clustered in files)
   - Lint warnings: ~2.5 per iteration
   - Tests (new): ~1 per iteration
   - Build fix: ~1-2 iterations
   - These are initial estimates — actual results will vary
3. **Efficiency** = point_gap / estimated_iterations
4. **Estimated tokens** = estimated_iterations × 22000 (rough average per iteration)

Sort areas by efficiency (highest pts/iteration first).

## Step 4: Display Report

Print the audit report:

```
━━━ Codebase Audit ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Current Score: XX/100

  [Metric 1]:  XX/WW  [progress bar]  (gap pts to max) or ✓ maxed
  [Metric 2]:  XX/WW  [progress bar]  (gap pts to max) or ✓ maxed
  [Metric 3]:  XX/WW  [progress bar]  (gap pts to max) or ✓ maxed
  [Metric 4]:  XX/WW  [progress bar]  (gap pts to max) or ✓ maxed

━━━ Fastest Path to 100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  #  Area          Gap    Issues  Est. iterations  Efficiency
  1  [best area]   Xpts   N items  M iterations    X.X pts/iter ← best
  2  [next area]   Xpts   N items  M iterations    X.X pts/iter
  ...

  Total: ~N iterations to reach 100/100
  ⚡ Estimated token usage: ~XXXK tokens (rough estimate, actual usage varies)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

The progress bar uses filled blocks (█) and empty blocks (░), scaled to 10 characters per metric.

Read weights dynamically from `.claude/autoimprove/config.md`. If a metric is not applicable, skip it and redistribute weight as the measure skill already handles.

**If score is 100/100:** Print "Your codebase scores 100/100. Nothing to improve." and stop.

**If build fails:** Show build as #1 priority with a note: "⚠️ Build is failing — other metrics may be unreliable. Fix the build first."

**If only one area has a gap:** Skip the selection prompt. Just ask: "Fix [area]? (~N iterations, ~XXXK tokens) (Y/n)"

## Step 5: Interactive Fix Loop

**Check for interrupted session first:** If `.claude/autoimprove/log.md` exists and the last session has `IN_PROGRESS` status, show: "↩️ You have an interrupted session. Use `/autoimprove:continue` to resume, or continue with this audit for a fresh start."

Prompt the user:

```
Would you like to start fixing? (Y/n)
```

**If no:** Stop. User can run `/autoimprove:improve` manually.

**If yes:**

```
Which area? (Enter number, or press Enter for #1 — [area name], ~N iterations)
>
```

If the user picks a number, use that area's estimate. If they press Enter, use the most efficient area.

### Starting the improve loop

Count all `**Branch:** autoimprove/experiment-` entries in `.claude/autoimprove/log.md` (if it exists) to determine the experiment offset.

Invoke the improve-loop skill directly with:
- **iterations:** The estimated count for the selected area
- **focus:** Generated focus string for that area:
  - Type safety → "Fix type errors"
  - Build → "Fix build errors"
  - Tests → "Add unit tests for untested functions"
  - Lint → "Fix lint warnings"
- **session_mode:** `new`
- **baseline_score:** The score just measured
- **start_iteration:** 1
- **total_iterations:** Same as iterations
- **experiment_offset:** Count of existing experiments

### After area completes

Re-run the measurement suite (Step 1-3) to get updated scores. Then show:

```
━━━ Area Complete: [area name] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Score: XX → YY/100 (+delta)
  [area]:  WW/WW  ██████████  ✓ maxed

Remaining work:
  #  Area     Gap    Est. iterations  Efficiency
  1  [next]   Xpts   M iterations     X.X pts/iter
  ...

  Remaining: ~N iterations (~XXXK tokens)

Continue to next area? (Y/n)
```

**If yes:** Repeat the selection prompt (or auto-select if only one area left). Start a new improve-loop session for that area.

**If no:** Stop. Print final score.

**If all areas maxed:** "🎉 Codebase score: 100/100. All areas maxed!"

### If area doesn't fully improve

If the improve loop finishes its estimated iterations but the area isn't maxed yet, show actual results:

```
Area [name] partially improved: XX → YY (expected ZZ)
Run more iterations on this area? (Y/n)
```

If yes, run more iterations (re-estimate based on remaining gap). If no, move to the next area.
```

- [ ] **Step 3: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add skills/audit/SKILL.md
git commit -m "Add audit skill for codebase deficiency scanning

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 2: Create the audit command

**Files:**
- Create: `commands/audit.md`

- [ ] **Step 1: Write the audit command**

Create `commands/audit.md`:

```markdown
---
description: Scan your codebase for deficiencies and get a prioritized fix plan
---

Run the audit skill to scan the codebase and generate a prioritized deficiency report.

The audit will:
1. Measure your current score across all metrics
2. Count individual issues (type errors, failing tests, lint warnings)
3. Rank areas by efficiency (points gained per iteration)
4. Show estimated iterations and token usage to reach 100%
5. Optionally start fixing — area by area, interactively

No arguments needed. The audit always scans everything.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add commands/audit.md
git commit -m "Add /autoimprove:audit command

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Chunk 2: Setup Integration + Measure Removal

### Task 3: Update setup to auto-run audit

**Files:**
- Modify: `commands/setup.md`

- [ ] **Step 1: Replace measure call with audit call**

Replace the entire content of `commands/setup.md` with:

```markdown
---
description: Detect project stack and generate .claude/autoimprove/config.md
---

If `.claude/autoimprove/config.md` already exists, ask: "Config already exists. Re-detect stack? (y/N)" If the user declines, skip detection and go straight to the audit.

If config doesn't exist or user confirms re-detection:
Run the detect-stack skill to analyse this project and produce `.claude/autoimprove/config.md`.

After the config is ready (newly generated or already existing), automatically run the audit skill to show the codebase report.

The audit will show the score breakdown, prioritized deficiencies, and offer to start fixing interactively.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add commands/setup.md
git commit -m "Auto-run audit after setup, protect existing config

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 4: Remove measure command

**Files:**
- Remove: `commands/measure.md`

- [ ] **Step 1: Delete the measure command**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
rm commands/measure.md
```

Note: `skills/measure/SKILL.md` is kept — it's still used internally by the audit skill and improve-loop.

- [ ] **Step 2: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add -A commands/measure.md
git commit -m "Remove /autoimprove:measure command (subsumed by audit)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Chunk 3: README + Changelog

### Task 5: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update Quick Start section**

Find the Quick Start section. Replace the existing steps 2-4 with the new flow:

```markdown
# 2. Auto-detect your stack and see your codebase report
/autoimprove:setup

# 3. The audit shows what's wrong and offers to start fixing
# Or run the audit anytime for a fresh check
/autoimprove:audit

# 4. For unattended runs (e.g. overnight), use improve directly
/autoimprove:improve 20

# Or focus on a specific task
/autoimprove:improve 10 "Replace all any types with proper interfaces"

# 5. Review in the morning
cat .claude/autoimprove/log.md
git log --oneline
```

Update the description paragraph below it to mention that setup auto-runs the audit.

- [ ] **Step 2: Update Commands table**

Replace the commands table with the updated list. Remove the `measure` row, add the `audit` row:

```markdown
| Command | Description |
|---|---|
| `/autoimprove:setup` | Detect stack, generate config, and run initial audit |
| `/autoimprove:audit` | Scan codebase for deficiencies and get a prioritized fix plan |
| `/autoimprove:improve [N] ["focus"]` | Run N iterations of the loop (default: 5), optionally focused on a specific task |
| `/autoimprove:continue [N] ["focus"]` | Resume an interrupted session — inherits remaining iterations and focus from the log |
| `/autoimprove:status` | Show a summary of all runs from `.claude/autoimprove/log.md` |
| `/autoimprove:upgrade` | Check for and install the latest version |
```

- [ ] **Step 3: Add Audit section**

Add a new section after "How it works" → "2. Isolated experiments via git worktrees" and before "3. The score". Title it "### 2.5 The audit" (or renumber existing sections). Content:

```markdown
### The audit

Before diving into fixes, `/autoimprove:audit` scans your codebase and shows exactly what needs work:

``​`
━━━ Codebase Audit ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Current Score: 61/100

  Type safety:   24/40  ██████░░░░  (16 pts to max)
  Build:         20/20  ██████████  ✓ maxed
  Tests:         10/30  ███░░░░░░░  (20 pts to max)
  Lint:           7/10  ███████░░░  (3 pts to max)

━━━ Fastest Path to 100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  #  Area          Gap    Issues  Est. iterations  Efficiency
  1  Type safety   16pts  8 errors   3 iterations   5.3 pts/iter ← best
  2  Lint           3pts  2 warnings 1 iteration    3.0 pts/iter
  3  Tests         20pts  0/4 covered 7 iterations  2.9 pts/iter

  Total: ~11 iterations to reach 100/100
  ⚡ Estimated token usage: ~250K tokens (rough estimate, actual usage varies)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
``​`

The audit ranks areas by **efficiency** — points gained per iteration — so you fix the highest-impact issues first. It then offers to start fixing interactively, area by area, or you can run `/autoimprove:improve` directly.

Setup auto-runs the audit after generating your config, so first-time users see this report immediately.
```

- [ ] **Step 4: Update the plugin structure tree**

Find the plugin structure section. Add the audit skill directory and audit command. Remove `measure.md` from the commands list:

```
├── skills/
│   ├── audit/
│   │   └── SKILL.md         # Codebase deficiency scan, prioritized report, interactive fix loop
│   ├── detect-stack/
│   │   └── SKILL.md
│   ├── worktree/
│   │   └── SKILL.md
│   ├── improve-loop/
│   │   └── SKILL.md
│   ├── measure/
│   │   └── SKILL.md         # Internal scoring utility (used by audit and improve-loop)
│   └── rollback/
│       └── SKILL.md
└── commands/
    ├── audit.md              # /autoimprove:audit
    ├── continue.md           # /autoimprove:continue [N] ["focus"]
    ├── setup.md              # /autoimprove:setup
    ├── improve.md            # /autoimprove:improve [N] ["focus"]
    ├── status.md             # /autoimprove:status
    └── upgrade.md            # /autoimprove:upgrade
```

- [ ] **Step 5: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add README.md
git commit -m "Document /autoimprove:audit, update quick start, remove measure from docs

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

### Task 6: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Replace the entire Unreleased section**

Find the `## [Unreleased]` section and **replace it entirely** (it currently has session-continue entries from a prior change). The new section combines both the continue and audit features:

```markdown
## [Unreleased]

### Added
- `/autoimprove:audit` command — scan codebase for deficiencies with prioritized fix plan
- Audit report with score breakdown, progress bars, efficiency ranking, and token usage estimates
- Interactive fix loop — audit offers to start fixing area-by-area after showing the report
- Setup auto-runs audit after generating config for seamless first-time experience
- `/autoimprove:continue` command to resume interrupted sessions
- Session headers in log with planned count, focus, baseline, base commit, and status tracking
- Interrupted session detection in `/autoimprove:status` with resume hint

### Changed
- Improve-loop now writes and updates session headers in the log
- Log format extended with structured session metadata (backwards-compatible with legacy logs)
- Setup command now protects existing config (asks before re-detecting stack)

### Removed
- `/autoimprove:measure` command (subsumed by `/autoimprove:audit` — measure skill kept as internal utility)
```

- [ ] **Step 2: Commit**

```bash
cd /Users/benmarte/Documents/github/claude-plugins/autoimprove
git add CHANGELOG.md
git commit -m "Add audit feature to changelog

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```
