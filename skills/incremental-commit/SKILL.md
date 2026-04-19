---
name: incremental-commit
description: This skill should be used when creating clean, sequential commits from staged changes without pushing. It uses an auto-squash workflow where fixup commits are tagged as work progresses, then collapsed into logical commits at the end. Trigger phrases: "incremental commit, clean history".
---

# Incremental Commit

## Overview

Creates a clean, sequential commit history by checking staged changes, grouping them into logical commits, and using an auto-squash workflow. Changes are committed incrementally without pushing, resulting in a clean history ready for review or later push.

## Workflow

### Step 1: Check Staged Changes

Before committing, inspect what's staged:

```bash
git diff --cached --stat
git diff --cached
```

Review the staged changes to understand what logical groups (commits) they form.

### Step 2: Create Initial Commits

For each logical group of changes, create a focused commit:

```bash
git add <files-for-first-commit>
git commit -m "description of first logical change"
```

### Step 3: Tag Fixups with Fixup

As you continue working and discover issues with previous commits, use fixup to tag corrections:

```bash
git commit --fixup=<sha-of-commit-to-fix> -m "fixup: correction detail"
```

The `fixup:` prefix signals that this commit should be squashed into its target.

### Step 4: Finalize with Auto-Squash

When all work is staged and committed, collapse fixups into their targets:

```bash
hunk rebase autosquash --onto <base-branch>
```

This squashes all `fixup:` prefixed commits into their targets, producing a clean sequential history.

### Step 5: Verify (Do Not Push)

Review the result:

```bash
hunk rebase list --onto <base-branch>
git log --oneline -10
```

Confirm the history looks clean and sequential before pushing separately.

## Key Principles

1. **No pushing** - This skill only handles staging, committing, and history cleanup. Push is a separate step under user control.
2. **Tag fixups immediately** - When you notice a fix is needed for a previous commit, use `--fixup` right away rather than waiting.
3. **Logical grouping** - Each initial commit should represent one logical change. Small, focused commits squash together cleanly.
4. **Auto-squash at the end** - Don't manually rebase; let the autosquash workflow handle consolidation.

## Decision Tree

```
Start: Staged changes exist?
├── No → Stage changes first with `git add`
└── Yes → Group changes into logical commits
         ├── Single logical group → Simple commit
         └── Multiple groups → Sequential commits per group
                              → Tag fixups with --fixup as discovered
                              → Run autosquash at end
```
