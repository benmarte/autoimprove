# Autoimprove Session Continue

## Problem

When a user cancels an autoimprove session mid-run (e.g., Ctrl+C, context limit, crash), the remaining iterations are lost. The user must manually remember how many iterations were left and what focus was active, then start a fresh session. The log records what happened but provides no mechanism to resume.

## Solution

Add a `/autoimprove:continue` command that reads the session log to determine where to pick up, inheriting planned count and focus from the interrupted session.

## Command Interface

```
/autoimprove:continue [N] ["focus"]
```

- **No arguments:** Resume with remaining iterations and original focus
- **N only:** Override remaining count (run N more iterations)
- **"focus" only:** Override focus string, use remaining count
- **Both:** Override both

## Log Format Changes

### Session Header (new)

Each session now begins with a structured header block:

```markdown
## Session — 2026-03-13T14:30:00Z
**Planned:** 10 iterations
**Focus:** "Fix any types"
**Baseline:** 72/100
**Base commit:** abc1234
**Status:** IN_PROGRESS (0/10 completed)
```

### Status Field Values

| Value | Meaning |
|---|---|
| `IN_PROGRESS (N/M completed)` | Session is actively running |
| `COMPLETED (M/M)` | All planned iterations finished |
| `INTERRUPTED (N/M completed)` | Session did not reach planned count |

### Status Lifecycle

1. Session starts → `IN_PROGRESS (0/M completed)`
2. After each iteration → update to `IN_PROGRESS (N/M completed)`
3. All iterations done → `COMPLETED (M/M)`
4. Any `IN_PROGRESS` status found when `continue` or `status` is invoked → treated as `INTERRUPTED`

There is no process detection. Since each Claude Code session is independent, any `IN_PROGRESS` status encountered at command invocation time is definitionally interrupted — the original session is gone.

### Legacy Log Compatibility

Logs written before this feature will not have session headers. When parsing the log:

- If no session header (`## Session —`) is found → treat the entire log as a completed legacy session
- `continue` on a legacy log → "No resumable session found. Use `/autoimprove:improve` to start a new session with the new format."
- New sessions always write the new header format
- Legacy iteration entries (without a session header) are preserved as-is; no migration is needed

### Status Update Mechanism

The status line is updated in-place after each iteration via string replacement:

1. Read `.claude/autoimprove/log.md`
2. Find the last `**Status:**` line in the last session header
3. Replace it with the updated status (e.g., `IN_PROGRESS (3/10 completed)`)
4. Write the file back

If the status line cannot be found (file corruption, manual edits), append a new status line to the session header block and log a warning. This is a best-effort update — the iteration records themselves are the source of truth for what completed.

## Continue Command Flow

1. **Read log** — Parse `.claude/autoimprove/log.md`, find the last session header
2. **Check status:**
   - If `COMPLETED` → "No interrupted session found. Use `/autoimprove:improve` to start a new one."
   - If no log exists → "No previous session found."
   - If `IN_PROGRESS` or `INTERRUPTED` → proceed
3. **Parse session metadata** — planned count, completed count, focus, base commit
4. **Calculate iterations:**
   - Default: `remaining = planned - completed`
   - If user provides N: use N instead
5. **Inherit focus** — Use logged focus unless user provides override
6. **Detect codebase changes:**
   - Compare current `HEAD` to logged base commit
   - If different: warn `"Warning: Codebase has changed since last session (abc1234 → def5678). Re-measuring baseline."` and take fresh baseline measurement
   - If same: use logged baseline, skip re-measurement
7. **Clean up orphan worktrees** — Run existing rollback/cleanup logic before resuming
8. **Resume loop** — Call improve-loop starting at iteration `completed + 1`, numbered out of `completed + remaining`
9. **Update session header** — On completion, update status to `COMPLETED`

## Improve Loop Skill Changes

The improve-loop skill needs to accept optional starting parameters:

### New Parameters

| Parameter | Default | Description |
|---|---|---|
| `start_iteration` | 1 | Which iteration number to start at |
| `total_iterations` | N (from args) | Total iteration count for display |
| `session_mode` | `new` | `new` creates session header; `continue` appends to existing |
| `baseline_score` | (measured) | Skip baseline measurement if provided |

### Session Header Management

- `improve` command → `session_mode: new`, creates session header
- `continue` command → `session_mode: continue`, appends to existing session block
- Both update the status field after each iteration

### Status Update Logic

After each iteration completes:
```
**Status:** IN_PROGRESS (N/M completed)
```

This means if the process dies between iterations, the log accurately reflects the last completed iteration.

## Status Command Changes

When displaying log summary, if the last session is interrupted:

```
━━━ Last Session: INTERRUPTED ━━━━━━━━━━━━
📊 Score: 72 → 81 (+9)
🔁 Completed: 4/10 iterations (6 remaining)
🎯 Focus: "Fix any types"
💡 Use /autoimprove:continue to resume
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Files to Add/Modify

| File | Change | Description |
|---|---|---|
| `commands/continue.md` | **Add** | New command entry point |
| `commands/improve.md` | **Modify** | Pass `session_mode: new` to improve-loop |
| `skills/improve-loop/SKILL.md` | **Modify** | Add session header format, accept start params, update status |
| `commands/status.md` | **Modify** | Show interrupted session hint |
| `README.md` | **Modify** | Document new command and session continuation |
| `CHANGELOG.md` | **Modify** | Add entry for new feature |

## Edge Cases

| Scenario | Behavior |
|---|---|
| No log exists | "No previous session found." |
| Last session completed | "No interrupted session found. Use `/autoimprove:improve`." |
| Orphan worktrees from crash | Clean up before resuming |
| User overrides with more iterations | Allowed — extends the session total |
| User overrides with fewer iterations | Allowed — runs fewer than remaining |
| HEAD moved since last session | Warn and re-measure baseline |
| Multiple interrupted sessions | Only the most recent is resumable |
| Continue after continue | Works — updates `Planned` field to new total (`completed + N`) and continues |
| Experiment branch numbering | Continues from last number (count iteration entries in log) to avoid collisions |

## Iteration Numbering

Iterations continue from where they left off. If original was 10 iterations and 4 completed:

```
━━━ Iteration 5/10 ━━━━━━━━━━━━━━━━━━━━
```

If user overrides with `/autoimprove:continue 3`:

```
━━━ Iteration 5/7 ━━━━━━━━━━━━━━━━━━━━━
```

(4 completed + 3 new = 7 total)

The session header's `Planned` field is updated to reflect the new total (7 in this case), keeping header and display in sync.
