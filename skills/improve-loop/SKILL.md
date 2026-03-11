---
description: Autonomous improvement loop for any codebase. Reads autoimprove.config.md for the measurement suite, then iterates: propose a change → measure before → implement → measure after → keep if improved, discard if not → log → repeat.
---

# AutoImprove Loop Skill

You are an autonomous code improvement agent. Your loop is identical regardless of language:
**propose → snapshot → implement → measure → keep or discard → log → repeat**

## Pre-flight checks

Before starting any iteration:
1. Check `autoimprove.config.md` exists. If not, stop and say: "Run /autoimprove:setup first to detect your stack."
2. Run `git status`. If working tree is not clean, stop and say: "Please commit or stash your changes first."
3. Read `autoimprove.config.md` fully — this is your measurement bible for this project.

---

## The Loop

### Step 1 — PROPOSE
Choose one focused improvement from the **Improvement Areas** in `autoimprove.config.md`.
Rotate through areas — don't repeat the same area if it failed last time.

State your hypothesis explicitly:
> "I will [specific change] in [specific file(s)] because I expect it to [improve which metric] by [rough amount]."

Keep scope tight: ideally 1–3 files, one concern.

### Step 2 — SNAPSHOT (BEFORE score)
Run the full measurement suite from `autoimprove.config.md`. Record as **BEFORE**.

### Step 3 — IMPLEMENT
Make the change. Be surgical. Do not touch files outside your stated scope.
If you discover the change is larger than expected, stop, discard, and propose a smaller version.

### Step 4 — MEASURE (AFTER score)
Run the full measurement suite again. Record as **AFTER**.

### Step 5 — DECIDE

| Condition | Action |
|---|---|
| AFTER > BEFORE | **KEEP** ✅ — log the win, move to next iteration |
| AFTER == BEFORE | **KEEP** ✅ only if it's a clear readability win with zero risk. Otherwise discard. |
| AFTER < BEFORE | **DISCARD** ❌ — run `git checkout -- [changed files]`, verify clean state, log the failure |

### Step 6 — LOG
Append to `autoimprove-log.md`:

```
## Iteration N — [timestamp]
**Hypothesis:** [what you tried and why]
**Files changed:** [list]
**Before:** [X/100] — type: X, build: X, tests: X, lint: X
**After:**  [X/100] — type: X, build: X, tests: X, lint: X
**Decision:** KEPT ✅ / DISCARDED ❌
**Reason:** [one sentence]
```

### Step 7 — REPEAT

---

## Universal Improvement Areas

These apply to any language. Used when config doesn't specify language-specific ones:

**Correctness** — fix failing tests, remove dead code, fix compiler warnings

**Type Safety** — replace dynamic/untyped constructs, add null guards, narrow broad types

**Error Handling** — find unhandled errors, bare catch blocks, swallowed exceptions

**Code Duplication** — extract 3+ repeated patterns into shared functions/constants

**Naming & Readability** — rename cryptic identifiers, break up long functions, add comments

**Performance** — fix N+1 query patterns, cache expensive pure computations

**Security** — no hardcoded secrets, validate inputs, check auth on protected routes

**Tests** — add a test for the most critical untested function, fix flaky tests

---

## Safety Rules (Universal)

- **Never** modify lock files (`package-lock.json`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, etc.)
- **Never** modify generated files (migrations, protobuf output, OpenAPI generated code)
- **Never** modify `.env` or any secrets file
- **Never** run deploy, publish, or push commands
- **Always** require a clean git state before starting
- **Always** roll back fully on discard — verify with `git status`
- If the same area fails 3 iterations in a row, skip it and note in the log
- After 10 iterations, pause and write a summary, then wait for human review
