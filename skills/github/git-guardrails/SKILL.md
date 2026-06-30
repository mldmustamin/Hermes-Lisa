---
name: git-guardrails
description: "Safe git push workflow: pre-push inspection, headless token auth, post-push verification."
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [Git, GitHub, Safety, Workflow, Authentication]
    related_skills: [github-auth, github-pr-workflow]
---

# Git Guardrails

Safe git push workflow for agent/headless environments. Extends `github-auth` and `github-pr-workflow` with real-world gotchas.

## Trigger

When you are about to push to GitHub — whether bare pushes or as part of a PR flow.

## Pre-Push: Always Inspect

**Never** blindly push. The user will call you out. Always inspect first:

```bash
# What commits are queued?
git log origin/main..HEAD --oneline

# What files changed, how big?
git diff --stat origin/main..HEAD

# Quick: ahead count
git status -sb
```

If the diff is massive (100+ files, 10K+ lines), tell the user what's in it before pushing. They may want to filter.

## Push: Inline Token (Headless)

Agent environments have no TTY. `git push` with a stored remote URL will fail with:
```
fatal: could not read Username for 'https://github.com': No such device or address
```

**Always push with token inline:**
```bash
git push https://USER:TOKEN@github.com/OWNER/REPO.git main
```

Do NOT use `git remote set-url` with token — it leaves credentials in config. Inline URL is one-shot.

## Post-Push: Verify

After push, the local `origin/main` ref is often stale. `git status` may still show "ahead N" even though the push succeeded. Always:

```bash
git fetch origin && git status -sb
# Should show "## main...origin/main" with no ahead/behind
```

If still ahead after fetch, the push didn't land — retry.

## Race Condition

Background pushes can race:
```
! [remote rejected] main -> main (cannot lock ref: is at <new_sha> but expected <old_sha>)
```
An earlier push already landed. Ignore — just `git fetch origin` to confirm.

## Large Push Timeouts

Pushes with 100+ files or 10K+ lines may time out in foreground (60-120s). Background them with `notify_on_complete=true` and `timeout=600`. The push lands but local `origin/main` ref stays stale — always `git fetch origin` after.

## Full Safe Push Sequence

```bash
# 1. Inspect
git log origin/main..HEAD --oneline
git diff --stat origin/main..HEAD

# 2. Push with inline token
git push https://USER:TOKEN@github.com/OWNER/REPO.git main

# 3. Verify
git fetch origin && git status -sb
```
