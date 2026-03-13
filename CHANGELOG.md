# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- `/autoimprove:continue` command to resume interrupted sessions
- Session headers in log with planned count, focus, baseline, base commit, and status tracking
- Interrupted session detection in `/autoimprove:status` with resume hint

### Changed
- Improve-loop now writes and updates session headers in the log
- Log format extended with structured session metadata (backwards-compatible with legacy logs)

## [1.2.0] - 2026-03-13

### Added
- Progress indicators in the improvement loop — visible status lines at every step (iteration number, phase, scores)
- `.gitignore` to exclude generated autoimprove data from the plugin repo

### Fixed
- `hooks/sessionstart.sh` missing execute permission, which prevented the update check hook from running on some systems

### Changed
- README updated with progress indicator examples and session summary format
- Version reference in README auto-update example updated

### Removed
- Project-specific `.claude/autoimprove/config.md` from repo (users generate their own via `/autoimprove:setup`)

## [1.1.0] - 2026-03-12

### Added
- `/autoimprove:upgrade` command to check for and install the latest version
- SessionStart hook for daily update checks with stderr notification
- Upgrade instructions for users with stale installs (pre-upgrade-command)

## [1.0.0] - 2026-03-11

### Added
- Initial release
- `/autoimprove:setup` — auto-detect project stack and generate config
- `/autoimprove:improve` — autonomous improvement loop with git worktree isolation
- `/autoimprove:measure` — standalone score check
- `/autoimprove:status` — summary of all runs from log
- Support for 10+ languages (TypeScript, Python, Go, Rust, Ruby, Java, C#, PHP, Swift, Kotlin)
- Focused improvements via quoted string argument
- Safety rules: never modifies lock files, secrets, generated code, or deploys

[1.2.0]: https://github.com/benmarte/autoimprove/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/benmarte/autoimprove/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/benmarte/autoimprove/releases/tag/v1.0.0
