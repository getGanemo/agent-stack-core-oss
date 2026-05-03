---
trigger: always_on
description: Git Required — locate git.exe before any git operation. Auto-install if missing. NEVER assume git is on PATH.
globs: ["**/*"]
---

# Git Required — Auto-Install Protocol

> **NEVER assume `git` is in the PATH. Always resolve the full path first.**

## Step 1 — Locate git executable

```powershell
$gitCandidates = @(
    "C:\Program Files\Git\cmd\git.exe",
    "C:\Program Files (x86)\Git\cmd\git.exe",
    "$env:LOCALAPPDATA\Programs\Git\cmd\git.exe"
)
$git = $gitCandidates | Where-Object { Test-Path $_ } | Select-Object -First 1
```

## Step 2 — If NOT found: install automatically

```powershell
winget install --id Git.Git -e --source winget --silent
# After install, re-run Step 1 to set $git
```

> Installation is automatic — do NOT ask the user for permission. Uses `auto_run_commands` policy.

## Step 3 — Always use full path in commands

```powershell
# CORRECT
$git = "C:\Program Files\Git\cmd\git.exe"
& $git clone ...
& $git -C $dir commit -m "..."

# WRONG — fails silently when git is not in PATH
git clone ...
```

## Confirmed path (post-install)

```
C:\Program Files\Git\cmd\git.exe
```

## When this rule applies

This rule activates before ANY of the following operations:
- `git clone` (download dep, sync module, deploy)
- `git commit` / `git push` (upload to GitHub)
- `git diff` (detect changed modules in `run_tests`)
- Any workflow that uses git (clone/commit/push/diff)
