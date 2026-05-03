---
name: auto_propagate
description: >
  ACTIVATE after any structural change: new skills, moved files, renamed
  configs, new conventions, changed devvault structure. Ensures all references
  are updated across the entire project without the user having to ask.
triggers:
  - new skill
  - renamed
  - moved
  - devvault
  - convention
  - structural change
---

# Rule: Auto-Propagate Structural Changes

After making ANY structural change (new skill, moved files, changed conventions, etc.),
**immediately** run the propagate_changes workflow from `.agents/workflows/propagate_changes.md`.

Do NOT wait for the user to ask "did you update X?". Propagation is your responsibility.

Check ALL of these:

1. `AGENTS.md` and `README.md`
2. All files in `.agents/rules/`, `.agents/skills/`, `.agents/workflows/`
3. Project and global memory (`MEMORY.md` indexes)
4. `devvault.yml` and any project-level deploy config (e.g. `<service>_deploy.yml`)
5. Global agent config (e.g. `~/.claude/CLAUDE.md`)
6. External docs (`docs_site/`) if applicable
