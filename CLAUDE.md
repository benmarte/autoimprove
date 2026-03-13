# Project Rules

## Release Checklist

Before cutting a new release, ALL of the following must be completed. Do not create the GitHub release until every item is done.

### 1. Changelog
- Create or update `CHANGELOG.md` with a new entry for the version
- Use [Keep a Changelog](https://keepachangelog.com/) format: Added, Changed, Fixed, Removed
- Every commit since the last release must be accounted for

### 2. Version Bump
- Update `VERSION` file to the new version number
- Follow semver: breaking changes = major, new features = minor, bug fixes = patch

### 3. README
- Update README.md to reflect any new features, changed behavior, or removed functionality
- Update any version numbers referenced in examples (e.g., update check example)
- Add new commands/options to the Commands table if applicable
- Update the "Plugin structure" tree if files were added/removed

### 4. Pre-release Verification
- `git status` must be clean (all changes committed)
- Review `git log --oneline v{LAST}..HEAD` to confirm nothing was missed
- Verify the plugin structure is valid (all skills have SKILL.md, hooks are executable)

### 5. Create Release
- Commit all of the above first
- Push to main
- Create GitHub release with `gh release create v{VERSION}`
- Release notes should summarize changes (not duplicate the full changelog)
- Include a link to the full changelog in the release notes

## Files to Never Modify During Automated Runs
- `LICENSE`
- `.claude-plugin/marketplace.json`
- `autoimprove.png`

## Commit Style
- Use imperative mood: "Add feature" not "Added feature"
- Keep subject line under 72 characters
- Include `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` when Claude writes the commit
