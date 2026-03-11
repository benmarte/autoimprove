---
description: Autonomous improvement loop for any codebase. Uses git worktrees to run every experiment in isolation — the main branch is never touched until a winning change is explicitly merged. Reads autoimprove.config.md for the measurement suite, then iterates: create worktree → propose → implement in worktree → measure → merge if improved, delete if not → log → repeat.
---

# AutoImprove Loop Skill

Every experiment runs in an isolated git worktree. The main codebase is **never modified** during experiments. Only winning changes get squash-merged back.

```
Main branch ──────────────────────────────────── (never touched mid-session)
                 │              │
           experiment-001  experiment-002
           (kept ✅ → merge) (discarded ❌ → deleted)
```

---

## Pre-flight checks

Before the first iteration:

1. Check `autoimprove.config.md` exists. If not, stop: "Run /autoimprove:setup first."
2. Check git is available: `git status`
3. Confirm main working tree is clean. If not, stop: "Please commit or stash changes before running autoimprove."
4. Record the base commit: `git rev-parse HEAD` — all experiments branch from here.
5. Run the worktree skill's **setup** step to create `.autoimprove-wt/` and update `.gitignore`.
6. Run the measure skill in the **main directory** to get the BASELINE score.
7. Report: "Baseline: XX/100. All experiments will run in isolated worktrees. Main branch is safe."

---

## The Loop

### Step 1 — CREATE WORKTREE

Use the worktree skill to create a new isolated branch and directory:

```bash
EXPERIMENT_ID=$(printf "%03d" $N)
git worktree add -b "autoimprove/experiment-$EXPERIMENT_ID" \
  ".autoimprove-wt/experiment-$EXPERIMENT_ID"
```

All work for this iteration happens inside `.autoimprove-wt/experiment-$EXPERIMENT_ID/`.
The main directory is not touched.

### Step 2 — PROPOSE

Choose one focused improvement from the **Improvement Areas** in `autoimprove.config.md`.
Rotate areas — don't repeat an area that failed last time.

State the hypothesis explicitly:
> "I will [specific change] in [file(s)] because I expect [metric] to improve by ~[X] points."

### Step 3 — SNAPSHOT (BEFORE score)

Measure from inside the worktree directory (same commands, different cwd):
```bash
cd .autoimprove-wt/experiment-$EXPERIMENT_ID
# run measurement suite from autoimprove.config.md
```
Record as **BEFORE**.

### Step 4 — IMPLEMENT

Make the change inside the worktree. The main directory is untouched.
Commit the change to the experiment branch:

```bash
cd .autoimprove-wt/experiment-$EXPERIMENT_ID
git add -A
git commit -m "experiment($EXPERIMENT_ID): $HYPOTHESIS_ONE_LINE"
```

### Step 5 — MEASURE (AFTER score)

Run the full measurement suite again from inside the worktree.
Record as **AFTER**.

### Step 6 — DECIDE

**If AFTER > BEFORE — KEEP ✅**

Squash-merge the experiment back to main:
```bash
cd [main project root]
git merge --squash "autoimprove/experiment-$EXPERIMENT_ID"
git commit -m "autoimprove($EXPERIMENT_ID): $HYPOTHESIS_ONE_LINE

Score: $BEFORE → $AFTER (+$DELTA pts)
Files changed: $FILES"

# Clean up
git worktree remove ".autoimprove-wt/experiment-$EXPERIMENT_ID"
git branch -D "autoimprove/experiment-$EXPERIMENT_ID"
```

**If AFTER == BEFORE — KEEP ✅ only for clear readability wins, DISCARD otherwise**

Same merge process as above if keeping, discard process if not.

**If AFTER < BEFORE — DISCARD ❌**

Main branch is already untouched. Just delete the worktree:
```bash
git worktree remove ".autoimprove-wt/experiment-$EXPERIMENT_ID" --force
git branch -D "autoimprove/experiment-$EXPERIMENT_ID"
```
No rollback needed — there was nothing to roll back.

### Step 7 — LOG

Append to `autoimprove-log.md` in the **main** directory:

```
## Iteration N — [timestamp]
**Hypothesis:** [what you tried and why]
**Branch:** autoimprove/experiment-NNN
**Files changed:** [list]
**Before:** [X/100] — type: X, build: X, tests: X, lint: X
**After:**  [X/100] — type: X, build: X, tests: X, lint: X
**Decision:** KEPT ✅ (merged to main) / DISCARDED ❌ (worktree deleted)
**Reason:** [one sentence]
```

### Step 8 — REPEAT from Step 1

---

## Session end — cleanup

After all iterations (or if the user stops early):

```bash
# Remove any remaining experiment worktrees
for wt in .autoimprove-wt/experiment-*; do
  git worktree remove "$wt" --force 2>/dev/null
done
git branch | grep "autoimprove/experiment" | xargs git branch -D 2>/dev/null
rm -rf .autoimprove-wt
```

Report: final score, iterations run, wins vs discards, list of merged commits.

---

## Universal Improvement Areas

Rotate through these (add language-specific ones from `autoimprove.config.md`):

- **Type safety** — fix type errors, replace `any`/`interface{}`/untyped constructs
- **Error handling** — unhandled promises, bare `catch {}`, swallowed errors
- **Dead code** — unused imports, variables, unreachable branches
- **Code duplication** — extract repeated logic (3+ occurrences) into shared utilities
- **Naming & readability** — cryptic names, functions over ~50 lines
- **Performance** — N+1 query patterns, missing memoization, unnecessary allocations
- **Security** — hardcoded secrets, missing input validation, unguarded auth routes
- **Tests** — add a test for the most critical untested function, fix flaky tests

---

## Safety Rules

- **Main branch is never modified** until a winning experiment is explicitly squash-merged
- **Never** modify lock files, generated files, migrations, `.env` — in any worktree
- **Never** run deploy, publish, or push commands
- If the same area fails 3 iterations in a row, skip it and note in the log
- After 10 iterations, pause, clean up worktrees, and wait for human review
- On any unexpected error: run the worktree skill's **cleanup** step, then stop and report
