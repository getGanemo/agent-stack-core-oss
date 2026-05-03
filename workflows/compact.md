---
description: Condense the current session's knowledge and state into a hand-off file so a fresh conversation can pick up cleanly.
---

When the user invokes the `/compact` command, or explicitly asks you to
"compact the session", execute the following steps sequentially and
autonomously:

1. **Analyse the session history.** Evaluate architectural decisions,
   configurations achieved, errors resolved, and the exact point we have
   reached against any progress markers in `task.md`.

2. **Filter noise.** Skip tracebacks, old error logs, long explanations,
   and repetitive tutorials. Extract strictly: "what was done", "what
   valuable learnings emerged", and "what comes next".

3. **Cross-references.** Identify which architectural files in the
   workspace (e.g. `AGENTS.md`, `Project/00_Arquitectura_Sistema.md`,
   `task.md`) already contain the heavy documentation. Do NOT repeat or
   hallucinate information from them — list them as required reading.

4. **Draft the hand-off.** Using `write_to_file`, create or overwrite a
   file named `_CONTEXTO_COMPACTADO.md` at the project root. The file MUST
   follow this Markdown structure:

   - **Current project state** (e.g. "Phase 1 complete, frontend at 50%").
   - **Required initial reading**: paths to the foundational files the
     next agent must process before talking.
   - **Technical decisions / gotchas / quirks**: local decisions that are
     not yet documented but were agreed today (e.g. "we chose native
     PostgreSQL over MariaDB", or a specific bug in the user's
     environment).
   - **Next immediate task**: the exact goal to attack when the next
     agent boots.

5. **Hand-off prompt.** Send the user a final message announcing that
   compaction has finished. Show — in a code block — a "Boot prompt" the
   user can copy-paste as the first message of their next session.
