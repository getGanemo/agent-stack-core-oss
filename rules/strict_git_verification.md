---
trigger: always_on
description: Ensures git merges are non-interactive and all git operations are strictly verified before confirming success.
globs: ["**/*"]
---

# Strict Git Verification

> **Never confirm a git operation to the user without verifying its success first.**

## 1. Non-Interactive Merges
- Always use `git merge <branch> --no-edit` when performing automated merges.
- This prevents the console pager or text editor from opening and hanging the process while waiting for a commit message.

## 2. Strict Verification Before Confirmation
- NEVER assume a `git push`, `git merge`, or `git commit` succeeded just because the command was sent.
- You MUST explicitly verify the success of the operation BEFORE telling the user "it is done".
- Acceptable verification methods:
  - Checking the command's `exit_code` (must be 0).
  - Inspecting the command's `stdout`/`stderr` for confirmation (e.g., "Fast-forward", "Merge made", "Pushed to", "up-to-date").
  - Running a follow-up `git status` or `git log -1` to confirm the branch state.
- If the operation hangs, fails, or asks for interactive input, **stop** and report the exact error to the user. Do NOT report success.
