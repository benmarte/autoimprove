<div align="center">

# 🔁 autoimprove

### Autonomous codebase improvement loop for Claude Code

[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet?logo=anthropic)](https://code.claude.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Languages](https://img.shields.io/badge/languages-10%2B-blue)](#supported-languages)

*Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch) — but for any codebase, not just ML training loops.*

</div>

---

## What is this?

Karpathy's `autoresearch` lets an AI agent run ML experiments overnight: modify `train.py` → measure `val_bpb` → keep if better, discard if worse → repeat. You wake up to a log of experiments and a better model.

**autoimprove does the same thing for your codebase.**

Give Claude Code your project, run `/autoimprove:improve`, and let it iterate autonomously. It proposes a targeted change, scores your codebase before and after using your own tooling (TypeScript, `cargo clippy`, `pytest`, `golangci-lint` — whatever you already have), keeps the changes that improve the score, reverts the ones that don't, and logs everything. You wake up to a readable log of what worked, what didn't, and a cleaner codebase.

```
propose → measure BEFORE → implement → measure AFTER → keep ✅ or discard ❌ → log → repeat
```

---

## Quick start

```bash
# 1. Install the plugin (from your project root in Claude Code)
claude plugin install ./autoimprove-plugin

# 2. Auto-detect your stack and get a baseline score
/autoimprove:setup

# 3. Run the improvement loop (e.g. overnight)
/autoimprove:improve 20

# 4. Review in the morning
cat autoimprove-log.md
git diff
```

That's it. No config required upfront — `/autoimprove:setup` fingerprints your project and writes `autoimprove.config.md` automatically.

---

## How it works

### 1. Setup (once per project)

`/autoimprove:setup` scans your project root to detect:
- Language and framework
- Package manager (`npm`, `cargo`, `poetry`, `uv`, etc.)
- Test runner (`pytest`, `jest`, `go test`, `rspec`, etc.)
- Type checker (`tsc`, `mypy`, `pyright`, etc.)
- Linter (`eslint`, `ruff`, `golangci-lint`, `rubocop`, etc.)

It writes an `autoimprove.config.md` file in your project root — a plain Markdown config that maps your specific tools to a **0–100 composite quality score**. You can edit this file to customise the loop for your project.

### 2. The score

Every iteration, the loop measures your codebase on four axes:

| Metric | Weight | What it checks |
|---|---|---|
| **Type / compile errors** | 40 pts | `tsc --noEmit`, `cargo check`, `go build`, `mypy`, etc. |
| **Build success** | 20 pts | Does the project build without errors? |
| **Test pass rate** | 30 pts | `(passing / total) × 30` |
| **Lint errors** | 10 pts | `eslint`, `ruff`, `clippy`, `golangci-lint`, etc. |

If a metric doesn't apply (no tests yet, no linter configured), its weight is redistributed across the others.

### 3. The loop

Each iteration:

1. **Proposes** one bounded improvement with an explicit hypothesis — *"I will fix the three unhandled promise rejections in `api/invoices.ts` because I expect it to reduce TypeScript errors and improve the type score by ~8 points"*
2. **Measures** the current score (BEFORE)
3. **Implements** the change (surgical — 1–3 files at most)
4. **Measures** again (AFTER)
5. **Keeps** if AFTER ≥ BEFORE, **discards** with `git checkout` if AFTER < BEFORE
6. **Logs** the result to `autoimprove-log.md`

### 4. The log

After each iteration, `autoimprove-log.md` gets an entry like:

```
## Iteration 4 — 2026-03-11 02:14
**Hypothesis:** Replace 3 `any` types in convex/invoices.ts with proper TypeScript interfaces
**Files changed:** convex/invoices.ts
**Before:** 74/100 — type: 28, build: 20, tests: 18, lint: 8
**After:**  82/100 — type: 36, build: 20, tests: 18, lint: 8
**Decision:** KEPT ✅
**Reason:** Eliminated 2 TS errors by typing the invoice mutation arguments properly
```

---

## Commands

| Command | Description |
|---|---|
| `/autoimprove:setup` | Detect stack, generate `autoimprove.config.md`, show baseline score |
| `/autoimprove:improve [N]` | Run N iterations of the loop (default: 5) |
| `/autoimprove:measure` | Check current score without making any changes |
| `/autoimprove:status` | Show a summary of all runs from `autoimprove-log.md` |

---

## Supported languages

| Language | Type check | Build | Tests | Lint |
|---|---|---|---|---|
| **TypeScript / JavaScript** | `tsc --noEmit` | `npm/pnpm/yarn/bun build` | jest / vitest / mocha | eslint |
| **Next.js / Nuxt / Remix / Astro** | `tsc --noEmit` | framework build cmd | jest / vitest | eslint |
| **Python** | mypy / pyright | — | pytest | ruff / flake8 / pylint |
| **Go** | `go build ./...` | `go build` | `go test ./...` | golangci-lint / `go vet` |
| **Rust** | `cargo check` | `cargo build` | `cargo test` | `cargo clippy` |
| **Ruby** | sorbet (if configured) | — | rspec / minitest | rubocop |
| **Java / Kotlin** | `mvn compile` / `./gradlew build` | same | `mvn test` / `./gradlew test` | checkstyle / ktlint |
| **C# / .NET** | `dotnet build` | `dotnet build` | `dotnet test` | `dotnet format --verify-no-changes` |
| **PHP** | phpstan | — | phpunit | phpcs |
| **Swift** | `swift build` | `swift build` | `swift test` | swiftlint |
| **Any Makefile project** | `make check` / `make typecheck` | `make build` | `make test` | `make lint` |

Don't see your stack? Edit `autoimprove.config.md` after setup to add your own commands.

---

## Customising autoimprove.config.md

After running `/autoimprove:setup`, edit the generated `autoimprove.config.md` to tailor the loop to your project:

```markdown
## Improvement Areas
- Check all Convex mutations have auth guards
- Replace fetch() calls with our internal apiClient wrapper
- Ensure every page component has a loading.tsx sibling

## Files to Never Modify
- convex/schema.ts
- src/generated/
- migrations/
- .env.local
```

You can also override any auto-detected command, change scoring weights, or add custom shell commands as additional metrics.

---

## What the loop improves

The loop rotates through these universal improvement areas (and adds language-specific ones based on your stack):

- **Type safety** — fix type errors, replace `any`/`interface{}`/untyped constructs
- **Error handling** — unhandled promises, bare `catch {}`, swallowed errors
- **Dead code** — unused imports, variables, unreachable branches
- **Code duplication** — extract repeated logic (3+ occurrences) into shared utilities
- **Naming & readability** — cryptic names, functions over ~50 lines
- **Performance** — N+1 query patterns, missing memoization, unnecessary allocations
- **Security** — hardcoded secrets, missing input validation, unguarded auth routes
- **Tests** — add a test for the most critical untested function, fix flaky tests

---

## Safety

The loop is designed to be safe to run unattended:

| Rule | Detail |
|---|---|
| 🔒 Never touches lock files | `package-lock.json`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, etc. |
| 🔒 Never touches generated files | Migrations, protobuf output, OpenAPI generated code |
| 🔒 Never touches secrets | `.env`, `.env.local`, any secrets file |
| 🔒 Never deploys or publishes | No `git push`, `npm publish`, `cargo publish`, etc. |
| 🔒 Requires clean git state | Won't start if `git status` shows uncommitted changes |
| 🔒 Full rollback on discard | `git checkout -- [files]` + verified clean state |
| 🔒 Pauses every 10 iterations | Writes summary and waits for human review |

You always review and push — the loop never commits or pushes on your behalf.

---

## Plugin structure

```
autoimprove-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── skills/
│   ├── detect-stack/
│   │   └── SKILL.md         # Fingerprints project, writes autoimprove.config.md
│   ├── improve-loop/
│   │   └── SKILL.md         # The core propose→measure→implement→decide loop
│   ├── measure/
│   │   └── SKILL.md         # Standalone score check
│   └── rollback/
│       └── SKILL.md         # Safe git rollback helper
└── commands/
    ├── setup.md             # /autoimprove:setup
    ├── improve.md           # /autoimprove:improve [N]
    ├── measure.md           # /autoimprove:measure
    └── status.md            # /autoimprove:status
```

---

## Example run

Here's what a real overnight session looks like. This is from a Next.js + Convex project starting at a score of 61/100:

```
## Iteration 1 — 23:04
**Hypothesis:** Replace 4 implicit `any` types in `convex/invoices.ts` with proper interfaces
**Files changed:** convex/invoices.ts
**Before:** 61/100 — type: 24, build: 20, tests: 10, lint: 7
**After:**  69/100 — type: 32, build: 20, tests: 10, lint: 7
**Decision:** KEPT ✅
**Reason:** Removed 4 TS7006 implicit-any errors by typing mutation arguments

## Iteration 5 — 23:37
**Hypothesis:** Move ExpenseList to a server component — it only reads data, no interactivity
**Files changed:** components/ExpenseList.tsx
**Before:** 71/100 — type: 32, build: 20, tests: 10, lint: 9
**After:**  68/100 — type: 26, build: 20, tests: 10, lint: 12
**Decision:** DISCARDED ❌
**Reason:** Removing "use client" broke useQuery hook — must stay client component. Rolled back.

## Iteration 8 — 00:02
**Hypothesis:** Add unit tests for calculateTaxEstimate() — most complex function, zero coverage
**Files changed:** lib/tax.test.ts (new)
**Before:** 78/100 — type: 36, build: 20, tests: 10, lint: 10
**After:**  84/100 — type: 36, build: 20, tests: 16, lint: 10
**Decision:** KEPT ✅
**Reason:** 2 new tests passing, covers basic and edge-case tax bracket logic

## Session Summary
Score: 61 → 84 (+23 pts) · Kept: 9 · Discarded: 1 · Duration: ~75 min
```

See [`autoimprove-log.example.md`](autoimprove-log.example.md) for the full 10-iteration session with summary table.

---

## Contributing

PRs welcome! Especially:
- New language profiles in `detect-stack/SKILL.md`
- Better improvement area prompts for specific frameworks
- Example `autoimprove.config.md` files for common stacks

---

## License

MIT
