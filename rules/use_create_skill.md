---
trigger: always_on
description: Before creating any SKILL.md in .agents/skills/, READ the create_skill skill. Never create a skill without its evals/evals.json.
globs: ["**/*"]
---

# Use create_skill — Mandatory Protocol

> **BEFORE creating any new `.agents/skills/<name>/SKILL.md` file, you MUST read the skill first.**

## Activation Condition

This rule activates when any of the following are requested:

- Creating a new skill file
- Adding a skill to the project
- The `capture_learnings` rule decides a new skill is needed

## Mandatory Steps

1. **READ** → `.agents/skills/create_skill/SKILL.md` using `view_file`
2. **FOLLOW** the format defined in the skill (hyper-specific description, recipe structure)
3. **CREATE** the SKILL.md at `.agents/skills/<name>/SKILL.md`
4. **CREATE** the evals file at `.agents/skills/<name>/evals/evals.json` — **never skip this**
5. **VERIFY** the skill appears in VS Code Open Agent Manager after Reload Window

## Never Skip This Rule If

- You are about to write any file inside `.agents/skills/`
- The user asks to "add a skill", "create a skill", or "build a skill"
- `capture_learnings` determined a new skill should be created
