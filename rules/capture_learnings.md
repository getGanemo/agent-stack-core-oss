---
trigger: always_on
description: Automatic capture of confirmed learnings. When a solution is verified to work, document it immediately into the appropriate skill, rule, or guideline according to the decision tree.
globs: ["**/*"]
---

# Capture Learnings — Auto-Knowledge Rule

> **When a confirmed solution or successful pattern is identified during a session, capture it immediately into the appropriate artifact. Do NOT wait to be asked.**

## Activation conditions

This rule activates when ANY of the following is confirmed:

### 1. A fix resolved a test failure after >= 2 failed attempts
- Tests go from FAIL -> PASS after a code change
- The fix addressed a specific traceback or error pattern

### 2. The user explicitly confirms success
- User says: "perfecto", "funcionó", "listo", "eso era", "confirmed", "that worked"
- User approves a change that previously caused an error

### 3. Contextual inference (no explicit phrase needed)
Infer success from context when ALL of these signals are present:
- The previous iteration had a specific error (traceback, test failure, exception)
- A targeted code change was applied to address that error
- The conversation moves forward naturally (user asks a new question, requests next task, no follow-up complaint)
- A test runner exits with code 0 / the JSON summary shows `status: PASS`

### 4. A new platform incompatibility is discovered and resolved
- Any inheritance/override that required reading native source to get right
- Any removed field, method, or API discovered during development

### 5. A pattern is used successfully for the second time
- Same solution applied to a second module/repo = it's a repeatable pattern worth documenting

## Decision tree — Where to put the knowledge

```
What kind of learning is this?
|
|- An error to AVOID (gotcha, trap, removed API)
|   |- Platform-version specific  -> verify_code skill (new audit item)
|   |- ORM / domain / SQL trap    -> check_pitfalls skill (new gotcha)
|   '- General build pattern      -> your team's DEV_GUIDELINES
|
|- A HOW-TO / successful approach
|   |- About native platform source -> inspect_native skill
|   |- About the test pipeline      -> autodev skill
|   |- About views / templates      -> verify_views skill
|   '- General good practice        -> your team's DEV_GUIDELINES
|
|- A behavior I should ALWAYS follow automatically
|   '-> Create a NEW RULE in .agents/rules/
|         Use the create_agent_rule skill (LF, frontmatter)
|
|- A multi-step procedure repeated across modules
|   '-> Create a NEW SKILL in .agents/skills/<name>/SKILL.md
|         Use the create_skill skill (description, structure, mandatory evals)
|
'- A complete process invokable by slash command
   '-> Create a NEW WORKFLOW in .agents/workflows/<name>.md
```

## What to capture (format)

Every learning entry should contain:

```
### [Error or Pattern Name]

**Context**: When does this happen?
**Problem**: What was the symptom / error?
**Root Cause**: Why did it happen?
**Solution**: What fixed it?
**Prevention**: How to avoid it from the start?
```

## Mandatory protocol

### CRITICAL: Never document a failed solution as good

**Before writing anything**, determine confidence level:

| Confidence | Condition | Action |
|---|---|---|
| High | Tests pass (exit_code=0 + PASS) OR explicit user confirmation | Document immediately |
| High | Contextual signals ALL present (see section 3 above) + no ambiguity | Document immediately |
| Medium | Partial signals — no explicit test run but conversation moved forward | **ASK FIRST** |
| Low | Error appeared solved but no clear confirmation | **NEVER document** — ask first |

**When asking, use this exact format:**

> "Identified a potential learning: [1-line description of the fix]. Do you confirm this solution worked and should be documented?"

Only document AFTER receiving explicit user confirmation.

### Steps (only after confidence is confirmed)

1. **Identify the target artifact** using the decision tree
2. **Check for duplicates**: read the target file before writing — never create redundant entries
3. **Write the entry** following the format above
4. **If updating a skill** (verify_code, check_pitfalls, etc.): also add a corresponding `evals.json` assertion if the pattern is testable
5. **Report briefly**: "Captured learning: [1-line summary] -> added to [file]"

## Scope of "confirmed" — what does NOT qualify

- Attempts that failed (only capture what worked)
- Already documented gotchas (check first)
- Workarounds that bypassed the real issue (document separately as a known limitation)
- UI/styling choices (too specific, low reuse value)

## Priority order for documentation

| Priority | Artifact | When |
|---|---|---|
| Critical | `verify_code/SKILL.md` | Removed API, wrong view type, framework-version regression |
| Critical | `check_pitfalls/SKILL.md` | ORM trap, domain error, SQL pattern |
| High     | `autodev/SKILL.md`       | Test pipeline gotcha, container issue |
| High     | DEV_GUIDELINES           | Construction or architecture pattern |
| Normal   | `inspect_native/SKILL.md`| Native source reading pattern |
| Normal   | DEV_REVIEWS              | Audit checklist item |
