---
description: >
  Crea un workspace AWaC end-to-end conduciendo el diálogo con el usuario:
  verifica entorno, lista templates disponibles, pregunta cuál elegir y bajo
  qué nombre, corre `wsp init` + `wsp bootstrap`, reporta el resultado. Ejecutar
  SIEMPRE que el usuario diga "creá un workspace", "setea workspace", "init
  AWaC", "iniciá un workspace", "create a workspace using AWaC", o equivalente.
  Pareja con la skill `use_wsp_cli` (cuándo invocar wsp en general) y con el
  workflow `install_wsp` (instalar el CLI si no está).
---

# Init workspace (autonomous setup)

Procedimiento autónomo para que un agente, partiendo de "creá el workspace
usando AWaC", deje al usuario con un workspace bootstrapeado y operativo
sin más fricción que **elegir template + nombre**.

// turbo-all

---

## Step 1 — Verificar pre-requisitos

```bash
wsp --version 2>&1 || echo "WSP_MISSING"
gh auth status  >/dev/null 2>&1 || echo "GH_AUTH_MISSING"
```

| Estado | Acción |
|---|---|
| `wsp` no instalado | Invocar el workflow `install_wsp` (se incluye en core; lo trae cualquier workspace AWaC). Después volver a este Step 1. |
| `gh` no autenticado | `gh auth login --hostname github.com --git-protocol https --web`. Esperar al login. |
| Ambos OK | Avanzar a Step 2. |

Después: `wsp doctor`. Cualquier check `[FAIL]` debe resolverse antes de
continuar (registry inalcanzable, devvault config faltante, governance
mirror divergente). El error trae `remediation` que el agente debe seguir.

---

## Step 2 — List available templates

```bash
wsp templates --json
```

Parse the JSON. Show the user a numbered table built from whatever templates
the registry currently exposes. The exact list depends on the fork — at a
minimum the public registry ships:

```
Available templates:

  1) blank             — Minimal workspace. Edit to taste.
  2) research-spike    — Investigation / spike, no product code.
  3) branding          — Workspace to author a brand file for a product.
  4) empresarial       — Generic enterprise / consulting project.
  5) tesis             — Academic / thesis project.
```

A typical fork extends this with one `<product>-feature` template per
product the team operates (e.g. `acme-feature`, `widget-feature`,
`atlas-feature`, etc.) plus tech-specific templates such as `odoo-module`
or `odoo-modules-existing`.

```
Which one? (number or id, default: blank)
```

Accept either the number or the literal id (`<product>-feature`). If the
user types something that doesn't match: ask again, showing the available
ids.

---

## Step 3 — Ask for the workspace name

Ask the user for the name. Rules:

- **kebab-case** (snake_case accepted as fallback). Pattern: `^[a-z][a-z0-9-]+$`.
- Tip to the user: name it after the feature being worked on
  (`<product>-billing-rework`, not `my-workspace-3`).
- If the user says "you choose": propose 2-3 names derived from the template
  plus a descriptive suffix and let them pick.

---

## Step 4 — Resolve and validate the target directory

Default: `<cwd>/<name>`. Allow override via an optional follow-up question:
"where should I create it? (default: `./<name>`)".

**Before creating**, check the target:

```bash
TARGET="<resolved-path>"
test -e "$TARGET/workspace.yml" && echo "WORKSPACE_EXISTS"
test -d "$TARGET" && [ -n "$(ls -A "$TARGET" 2>/dev/null)" ] && echo "DIR_NOT_EMPTY"
```

| Condition | Action |
|---|---|
| `WORKSPACE_EXISTS` | The directory already has a `workspace.yml`. Ask: "there's already a workspace here. (a) use the existing one and just run bootstrap, (b) pick another path, (c) cancel?" Safe default: option (b). NEVER overwrite without explicit ack. |
| `DIR_NOT_EMPTY` (no `workspace.yml`) | The directory has content but isn't a workspace. Ask: "the directory `<TARGET>` is not empty. (a) use another name / path, (b) cancel?" NEVER delete the content without explicit letter-by-letter ack from the user. |
| Empty or doesn't exist | Proceed. |

---

## Step 5 — `wsp init`

```bash
wsp init "<name>" --template "<chosen_id>" --target "<TARGET>"
```

This writes `<TARGET>/workspace.yml` (substituting `<CHANGE-ME>` with
`<name>`) without cloning anything. If it returns a structured error, read
`code`/`category`/`cause`/`remediation` and report to the user; don't
proceed to Step 6.

---

## Step 6 — `wsp bootstrap`

```bash
cd "<TARGET>"
wsp bootstrap --json
```

Expected: the JSON includes `stacks`, `repos`, `agent_files`, `file_count`,
`collisions`, `lock`. Show the user a 4-line summary:

```
Workspace bootstrapped at <TARGET>
  Stacks:        N (short list)
  Product repos: M cloned
  .agents/:      F files composed (rules + skills + workflows)
  Lockfile:      <TARGET>/workspace.lock.yml
```

If `bootstrap` fails:

- `WSP_004 repo_clone_failed` → almost always `gh auth` or an inaccessible repo. Verify `gh auth status` and permissions on the product org.
- `WSP_005 registry_fetch_failed` → registry unreachable. Verify `WSP_REGISTRY_REPO`/`WSP_REGISTRY_BRANCH` and network.
- Other → follow `remediation` from the structured error.

---

## Step 7 — Hand-off to the user

When bootstrap passes, show the path and suggest the next step based on the
chosen template:

| Template | Typical next step |
|---|---|
| `<product>-feature` | `cd <TARGET> && code .` (or your preferred editor). The agent reads `CLAUDE.md` and starts working on the feature. |
| `odoo-module` / `odoo-modules-existing` | Verify that `addons/` has the expected modules. Suggest running `run_tests` (Odoo workflow) after the first edit. |
| `research-spike` / `branding` / `empresarial` / `tesis` | Non-product workspace. Edit `workspace.yml` if you want to add `aws`, `mcp` or other stacks ad-hoc; then run `wsp sync`. |
| `blank` | Minimal workspace. Edit `workspace.yml` to add stacks + repos as needed; run `wsp bootstrap` whenever you change it. |

Print 2 useful follow-up commands:

```
To refresh after a stack changed upstream:    wsp sync
To diff the lockfile vs the current state:    wsp status
```

---

## Step 8 — Record the outcome (if the product has `project_management`)

If the chosen template belongs to a product whose `project_management/` repo
was cloned into the workspace (any `<product>-feature` template), write a
`progreso.md` entry in the product's current phase following the
`manage_project_state` skill:

```
[<date>] Workspace initialised: <name>
- Template: <template_id>
- Path: <TARGET>
- Stacks: <list>
- Initiator: <user>
```

This closes the loop with the product's roadmap. For non-product templates
(blank, research, branding, etc.) skip this step.

---

## Anti-patterns

1. **Skipping Step 4** and going straight to `wsp init`. If the directory
   has content, `wsp init` fails with `WSP_006` and you leave a half-done
   job behind.
2. **Assuming the template** because the user said "for `<product>`" —
   always confirm with the user before invoking `wsp init` (they might want
   `odoo-module` to touch a single Odoo module of that product instead of
   the full feature template).
3. **Deleting directory contents without explicit ack**. Step 4 is clear:
   never delete without the user typing their letter-by-letter confirmation.
4. **Skipping `wsp doctor` post-install**. If `gh auth` is broken or the
   registry doesn't resolve, bootstrap fails and you waste time debugging
   what doctor would have told you in 5 seconds.
5. **Reporting "workspace ready" without having verified bootstrap.** The
   success criterion is that `wsp bootstrap` returned ok + lockfile was
   written, not just that `wsp init` passed.

---

## Cross-references

- Skill: [`use_wsp_cli`](../skills/use_wsp_cli/SKILL.md) — when to invoke `wsp` in general.
- Companion workflow: [`install_wsp.md`](install_wsp.md) — install the CLI if it's missing.
- Skill: [`manage_project_state`](../skills/manage_project_state/SKILL.md) — Step 8.
- Templates registry: `agent-stack-core/awac.yml#templates` (in your fork).
