---
description: Analyse session learnings and update both DEV_GUIDELINES and DEV_REVIEWS to prevent future regressions.
---

# Update Knowledge Base (Guidelines & Reviews)

Run this workflow at the end of a session to capture knowledge. It covers
both **construction knowledge** (how to build) and **audit knowledge** (how
to check).

## 1. Session analysis

1. Identify critical errors solved (e.g. 500 errors, syntax, deployment).
2. Identify successful design patterns implemented.

## 2. Update construction guidelines (DEV_GUIDELINES)

*Focus: "how to write code correctly from the start."*

1. Review files in your team's DEV_GUIDELINES area (e.g.
   `docs/DEV_GUIDELINES/python_pitfalls.md`, `pwa_patterns.md`).
2. **Update:** add new cases (Context -> Problem -> Solution).
3. **Create:** if a new architecture topic emerges, create a new file.

## 3. Update audit checklists (DEV_REVIEWS)

*Focus: "how to verify code before deployment."*

1. Review files in your team's DEV_REVIEWS area (e.g. `REVIEW_SECURITY.md`,
   `REVIEW_VIEWS.md`).
2. **Action:** if a specific error was elusive, add a specific check to the
   relevant Review file so it's caught in future audits.
   *Example:* if a silent security bug was fixed, add "Check for X" in
   `REVIEW_SECURITY.md`.

## 4. Sync indexes

1. Update `DEV_GUIDELINES/README.md` if structural changes occurred.
2. Update `DEV_REVIEWS/README.md` (if it exists) or ensure the file list
   is current.

## 5. Report

1. Provide a summary distinguishing the two areas:
   - **Construction (Guidelines) changes:** [List]
   - **Audit (Reviews) changes:** [List]

## 6. Sync skills (chain reaction)

1. **Run `/improve_skills`:** immediately run the `improve_skills` workflow
   to ensure these new guidelines are propagated to the active `SKILL.md`
   files. This makes the agent's behaviour change instantly to reflect the
   new knowledge.
