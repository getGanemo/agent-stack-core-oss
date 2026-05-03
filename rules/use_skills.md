---
trigger: always_on
description: Use specialized skills for development tasks rather than improvising. Isolation first; scaffold-then-edit; verify before finalizing.
globs: ["**/*"]
---

# Skill Usage Policy

> **CRITICAL MINDSET: ISOLATION FIRST**
> Before applying any skill to fix an "External Error", verify isolation.
> - If the error originates in a framework's own source (`<framework>/<core>/...`),
>   **assume your code caused it** via inheritance, hooks, or bad data.
> - **Do not blame the environment** until you have reproduced the issue without your module.

The exact set of available skills depends on which stacks the workspace
composes. The general policy below applies regardless of the technology.

## General policy

1. **New modules / packages**: NEVER hand-roll the skeleton. Use the project's
   `scaffold_<module>` skill so directory layout, manifest, and licence boilerplate
   stay consistent.
2. **Repeated structures** (spreadsheets, view types, mixins, etc.): use the
   matching `scaffold_*` skill if the active stacks provide one. If not, propose
   creating one via `create_skill` rather than copy-pasting.
3. **Testing**: ALWAYS generate tests using `scaffold_test` (when available). The
   skill encodes framework-version-correct setup patterns that are easy to get
   wrong by hand.
4. **Debugging**: When writing or reviewing logic, run `check_pitfalls` continuously
   to catch ORM traps, removed APIs, and version-specific regressions early.
5. **Review**: Before finalizing any task, run the relevant `verify_*` skill —
   typically `verify_code` for backend, `verify_views` for templates/markup, and
   `verify_security` for ACLs / access rules.
6. **Documentation**: Consult the project's `consult_*_docs` skill (NotebookLM-backed
   when available, deep research otherwise, raw URL only as last resort) when
   implementing anything where convention beats invention.

## Why this matters

Skills are **encoded institutional knowledge**. Bypassing them and "writing it
by hand" is how the same regression gets introduced for the third time.
