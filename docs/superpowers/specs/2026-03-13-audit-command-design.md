# Autoimprove Audit Command

## Problem

Users have no way to see what's wrong with their codebase before running the improve loop. They must blindly run iterations and hope the loop picks the right areas. There's also no way to prioritize which areas to fix first based on effort vs impact.

## Solution

Add a `/autoimprove:audit` command that scans the codebase, generates a prioritized deficiency report ranked by efficiency (points per iteration), and optionally transitions into focused improve loops — area by area — until the user stops or hits 100%.

## Command Interface

```
/autoimprove:audit
```

No arguments. The audit always scans everything and presents the full picture.

## Audit Report Format

The report is a hybrid: score breakdown at top, then a prioritized issue list.

```
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
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## How Deficiencies Are Counted

The audit runs the same measurement commands from `.claude/autoimprove/config.md` (via the measure skill), but also parses the raw output to count individual issues:

- **Type errors:** Parse compiler output (e.g., `tsc --noEmit 2>&1`), count error lines, group by file
- **Build:** Pass/fail — no granular issues. If build fails, it becomes top priority.
- **Tests:** Parse test runner output, count passing vs total, identify failing test names
- **Lint:** Parse linter output, count warnings/errors, group by rule

### Efficiency Calculation

For each area with a gap:
1. Count specific issues (e.g., 8 type errors)
2. Estimate iterations needed: `ceil(issue_count / issues_per_iteration)`
   - Type errors: ~2-4 per iteration (often clustered in files)
   - Lint warnings: ~2-3 per iteration
   - Tests: ~1 new test per iteration
   - These are initial heuristics — they can be refined over time based on actual session log data
3. Calculate efficiency: `point_gap / estimated_iterations`
4. Rank by efficiency (highest pts/iteration first)

Note: The score breakdown weights (type: 40, build: 20, tests: 30, lint: 10) are read dynamically from `.claude/autoimprove/config.md`, not hardcoded. If a metric is not applicable, its weight is redistributed as the measure skill already handles.

## Interactive Flow

After showing the report, prompt the user:

```
Would you like to start fixing? (Y/n)
```

**If no:** Leave the report. User can run `/autoimprove:improve` manually.

**If yes:**

```
Which area? (Enter number, or press Enter for #1 — Type safety, ~3 iterations)
>
```

### Auto-selection (press Enter)

Selects the area with the highest efficiency (pts/iteration). If the user picks a different area, that area's own estimate and focus are used instead.

### Invoking the improve loop

The audit skill invokes the improve-loop skill directly (not the improve command) with these parameters:

- **iterations:** The estimated count for the selected area
- **focus:** Generated focus string (e.g., "Fix type errors", "Add unit tests for untested functions")
- **session_mode:** `new` — each area is a separate session in the log
- **baseline_score:** The score just measured by the audit
- **start_iteration:** 1
- **total_iterations:** Same as iterations
- **experiment_offset:** Count of existing experiments in the log (to avoid branch collisions)

### Session semantics for multi-area runs

Each area the audit fixes is a **separate session** in the log. This is the simplest and most correct approach because:
- Each area has its own baseline, focus, and iteration count
- If the user stops mid-area, `/autoimprove:continue` can resume that specific area
- The log reads cleanly: one session per area, each with its own score progression

### After each area completes

Re-run measurement to get updated scores, then show:

```
━━━ Area Complete: Type safety ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Score: 61 → 77/100 (+16)
  Type safety:   40/40  ██████████  ✓ maxed

Remaining work:
  #  Area     Gap    Est. iterations  Efficiency
  1  Lint      3pts  1 iteration      3.0 pts/iter
  2  Tests    20pts  7 iterations     2.9 pts/iter

Continue to next area? (Y/n)
```

This repeats until all areas are maxed or the user stops.

## Setup Auto-runs Audit

After `/autoimprove:setup` generates the config, it automatically runs the audit. This gives first-time users a seamless experience: one command to go from zero to seeing their codebase report and optionally starting fixes.

- `/autoimprove:setup` → detects stack, writes config, then runs audit automatically
- `/autoimprove:audit` → standalone re-check anytime (does not re-run setup)
- If config already exists when running setup, ask: "Config already exists. Re-detect stack? (y/N)" to protect user customizations. If they decline, just run the audit with the existing config.

### Changes to setup command

The setup command (`commands/setup.md`) adds one line at the end: after generating the config, invoke the audit skill. The setup skill (`skills/detect-stack/SKILL.md`) is unchanged — it still just detects and writes the config.

## Removing the Measure Command

The `/autoimprove:measure` command is fully subsumed by audit. The user-facing command (`commands/measure.md`) is removed. The measure skill (`skills/measure/SKILL.md`) is kept as an internal utility used by both the audit skill and the improve-loop.

## Files to Add/Modify/Remove

| File | Change | Description |
|---|---|---|
| `commands/audit.md` | **Add** | New command entry point |
| `skills/audit/SKILL.md` | **Add** | Audit logic: measure, parse raw output, rank, report, interactive loop |
| `commands/measure.md` | **Remove** | Subsumed by audit |
| `commands/setup.md` | **Modify** | Add auto-run audit after config generation |
| `README.md` | **Modify** | Add audit command, remove measure command, update quick start and commands table |
| `CHANGELOG.md` | **Modify** | Add entry |

## Edge Cases

| Scenario | Behavior |
|---|---|
| Already at 100/100 | "Your codebase scores 100/100. Nothing to improve." |
| No config exists | "Run `/autoimprove:setup` first." |
| Build fails | Show build as top priority — other metrics may be unreliable with a broken build |
| Only one area has a gap | Skip selection prompt, just ask "Fix [area]? (~N iterations)" |
| User declines to start | Leaves the report — they can run `/autoimprove:improve` manually |
| Area doesn't improve after estimated iterations | Stop that area, show actual results, ask if user wants to continue with more iterations or move to next area |
| Interrupted session exists | Show a note at the top: "You have an interrupted session. Use `/autoimprove:continue` to resume, or run this audit for a fresh analysis." |
| Metric not applicable | Skip it (e.g., no linter configured). Redistribute weight as the measure skill already does. |
