---
description: Audit and update Agent Skills based on the latest DEV_GUIDELINES and project learnings.
---

# Continuous Skill Improvement Workflow

Run this workflow periodically (e.g. monthly or after major documentation
updates) to ensure the agent's skills remain in sync with the team's
evolving best practices.

## 1. Knowledge base audit

1. Read the current contents of your team's DEV_GUIDELINES and DEV_REVIEWS
   areas.
2. Note any **new files** or **significant changes** in either area since
   the last audit.
3. *Self-correction:* if you find a new "Anti-Pattern" or "Security Check"
   documented, verify whether the corresponding skill (`check_pitfalls` or
   `verify_*`) covers it.

## 2. Skill gap analysis

1. List all current skills in `.agents/skills/`.
2. **Mapping check:**
   - Does every "Pattern" file have a corresponding "Scaffold" skill?
   - Does every "Review" file have a corresponding "Verify" skill?
3. **Gap identification:**
   - *Example:* if `DEV_GUIDELINES/graphql_api.md` exists but no
     `scaffold_graphql` skill exists -> **gap found**.

## 3. Action

1. **Update existing:** if a guideline changed (e.g. "use the new framework
   syntax"), update the relevant `SKILL.md`.
2. **Create new:** if a gap was found, use the `create_skill` skill to
   create a new one — ensures hyper-specific description, recipe structure,
   and `evals/evals.json` from the start.
3. **Archive:** if a guideline was deleted or deprecated, move the
   corresponding skill to an `archived/` folder or delete it.

## 4. Chain reaction (rules & indexes)

1. **Update rules:**
   - Review `.agents/rules/use_skills.md`. Does it mandate the use of the
     new/updated skill?
   - Review other rule files. Do they need to verify compliance with the
     new standard?
2. **Sync indexes:**
   - Update `DEV_GUIDELINES/README.md`: ensure any new guideline file is
     listed in the index.
   - Update root `README.md`: ensure high-level architecture changes are
     reflected.

## 5. Report

1. Generate a brief summary of actions taken:
   - Skills updated: [List]
   - Skills created: [List]
   - Gaps identified: [List]
