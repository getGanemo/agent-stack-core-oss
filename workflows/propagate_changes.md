---
description: After any structural change (new skill, moved files, renamed configs, new conventions), propagate updates to ALL affected locations. Run this workflow automatically — never wait for the user to ask "did you update X?".
---

# Propagate Changes Workflow

When a structural change is made (new skill, moved files, changed conventions, renamed configs, new devvault entries, etc.), **automatically** update ALL affected locations in a single pass.

---

## Checklist — Run ALL steps, skip only if truly not applicable

### 1. Project-level files

| File | Check for |
|------|-----------|
| `AGENTS.md` | Structure description, skills table, deploy conventions, credential references |
| `README.md` | Folder structure, file descriptions, repo references, workflow diagram |
| `devvault.yml` | New secrets files, changed vault paths |
| Project deploy spec (e.g. `<service>_deploy.yml` or `<product>/agent-stack/deploy.yml`) | Repo, branch, modules list |

### 2. Agent system (`.agents/`)

| Location | Check for |
|----------|-----------|
| `.agents/rules/*.md` | References to old file paths, old conventions, missing triggers for new skills |
| `.agents/skills/*/SKILL.md` | References to old file paths, outdated instructions |
| `.agents/workflows/*.md` | References to old file paths, old credential sources |
| `.agents/workflows/resources/` | Hardcoded paths in scripts/templates |

**Search command:**
```bash
grep -rn "OLD_PATTERN" .agents/
```

### 3. Memory

| Location | Check for |
|----------|-----------|
| Project memory (`~/.claude/projects/.../memory/`) | Outdated entries, duplicates, stale references |
| Global memory (`~/.claude/memory/`) | Same — ensure global rules reflect current conventions |
| `MEMORY.md` indexes (both) | Duplicate entries, descriptions matching content |

### 4. DevVault (`~/.devvault/`)

| File | Check for |
|------|-----------|
| `projects.yml` | New/changed secret file paths |
| `README.md` | Structure description up to date |
| Provider/AWS/Odoo files | Moved credentials, new entries |

### 5. Global Claude config (`~/.claude/`)

| File | Check for |
|------|-----------|
| `CLAUDE.md` | Global instructions referencing old patterns |

### 6. External references

| Location | Check for |
|----------|-----------|
| `docs_site/` (Starlight) | API docs, changelog, sidebar entries |
| EC2 deployed code | docker-compose.yml, .env, running services |

---

## How to execute

1. **Identify what changed** — file paths, conventions, tools, credentials
2. **Build a grep pattern** for the old references:
   ```bash
   grep -rn "old_pattern" . .agents/ ~/.claude/memory/ ~/.claude/projects/*/memory/ ~/.devvault/
   ```
3. **Update each match** — don't batch, update one by one to avoid mistakes
4. **Verify zero remaining references:**
   ```bash
   grep -rn "old_pattern" . .agents/
   ```
5. **Report** what was updated (file list + what changed)

---

## CRITICAL: When to run this workflow

Run **automatically** (without user asking) after:
- Creating or renaming a skill
- Moving or renaming any config file
- Changing credential storage (devvault structure)
- Adding a new convention or rule
- Changing repo names, branch conventions, or org structure
- Adding new folders to the project structure

**Never** wait for the user to ask "did you update the README?" or "what about the rules?". Propagation is YOUR responsibility.
