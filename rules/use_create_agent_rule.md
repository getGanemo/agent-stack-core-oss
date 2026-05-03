---
trigger: always_on
description: Before creating any file in .agents/rules/, READ the create_agent_rule skill. Never create rules without following the LF + frontmatter format.
globs: ["**/*"]
---

# Create Agent Rule — Mandatory Protocol

> **BEFORE creating any new `.agents/rules/*.md` file, you MUST read the skill first.**

## Activation Condition

This rule activates when any of the following are requested:

- Creating a new Rule for the agent
- Modifying an existing rule file
- Fixing rules that are not appearing in the Open Agent Manager panel

## Mandatory Steps

1. **READ** → `.agents/skills/create_agent_rule/SKILL.md` using `view_file`
2. **FOLLOW** the format defined in the skill (frontmatter, LF endings, `trigger: always_on`)
3. **CREATE** the file using the `write_to_file` tool — never a text editor or shell redirect
4. **VERIFY** the rule appears in VS Code → Open Agent Manager → Rules panel

## Never Skip This Rule If

- You are about to write any file inside `.agents/rules/`
- The user asks to "add a rule", "create a rule", or "update a rule"
- You are fixing rules that are missing from the panel
