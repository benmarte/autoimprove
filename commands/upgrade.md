---
description: Check for and install the latest version of autoimprove
---

Upgrade autoimprove to the latest version. Follow these steps:

## Step 1: Check current version

Read the `VERSION` file in the autoimprove plugin root to get the current version. The plugin root is the directory where this command file lives (go up one level from `commands/`).

## Step 2: Check latest version

Run this command to get the latest release version from GitHub:

```bash
gh api repos/benmarte/autoimprove/releases/latest --jq '.tag_name'
```

## Step 3: Compare and upgrade

If the versions match, report: "autoimprove is up to date (v{version})."

If a newer version is available, run the upgrade:

```bash
/plugin install autoimprove@autoimprove
```

Then report:
- "autoimprove upgraded from v{old} to v{new}."
- "Restart Claude Code to use the new version."

## Step 4: Show changelog

After upgrading, fetch and display the release notes for the new version:

```bash
gh api repos/benmarte/autoimprove/releases/latest --jq '.body'
```
