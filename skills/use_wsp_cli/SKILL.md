---
name: use_wsp_cli
description: >
  USE this skill any time you are about to perform a workspace-level operation
  on a Ganemo AWaC workspace — creating one, refreshing it, inspecting drift,
  or seeding/refreshing a product's `agent-stack`. The `wsp` CLI
  (`getGanemo/workspace-cli`) is the canonical entry point for these operations
  and replaces hand-edited `.agents/`, hand-cloned product repos, and
  hand-curated CLAUDE.md/AGENTS.md headers. Activate when: (a) the user wants
  to start a new workspace, (b) `awac.yml` of a stack changed and you need to
  pull it, (c) the user asks "is my workspace clean / what changed", (d) you
  detect drift in `.agents/`, (e) the user wants to introspect a GitHub org and
  generate or refresh `<org>/agent-stack`, (f) the user wants to verify that
  the governance doc and `awac.yml#org_scaffold` are still in sync. Do NOT
  activate for ordinary code edits inside a workspace — those don't need wsp.
---

# Use the `wsp` CLI

`wsp` is the AWaC CLI. It lives at [`getGanemo/workspace-cli`](https://github.com/getGanemo/workspace-cli) and is installed via `pipx install -e <path>` (Python ≥ 3.10). Every Ganemo agent workspace was either created by `wsp` or is meant to be managed by it. Treat it as the **only** correct way to mutate workspace shape (stacks, product repos, `.agents/`, CLAUDE.md / AGENTS.md, lockfile).

## When to use this skill

| User intent | Command |
|---|---|
| "Onboardear un producto Ganemo nuevo" / "vamos a empezar `<producto>`" | invocar workflow [`onboard_new_product`](../workflows/onboard_new_product.md) — 9 pasos: scaffold-stack auto-register → Cat A repos → project_management init → devvault catalog → audit entry en docs-company → primer workspace local. |
| "Creá / setea / iniciá un workspace AWaC" (sin template específico) | invocar workflow [`init_workspace`](../workflows/init_workspace.md) — conduce el diálogo de elegir template + nombre + bootstrap. |
| "Start a new workspace for an Acme feature." (template ya conocido) | `wsp init <name> --template acme-feature && wsp bootstrap` |
| "¿está bien armado este producto?" / "audit de `<producto>`" | `wsp audit <producto>` — read-only check de Cat A + agent-stack assets + registry. |
| "I just edited a rule in `agent-stack-core`, refresh my open workspace." | `wsp sync` |
| "Are my product repos clean? Is anything drifted?" | `wsp status` |
| "Add `delta` as a stack shortcut." | edit `getGanemo/agent-stack-core/awac.yml` (then anyone running `wsp sync` picks it up) |
| "Create the agent-stack for a new product (lazy org)." | `wsp scaffold-stack <org>` |
| "Refresh `<org>/agent-stack`'s `awac.yml#repos` from GitHub." | `wsp scaffold-stack <org> --update` (opens a PR) |
| "Did anything diverge between `awac.yml#org_scaffold` and the governance doc?" | `wsp governance check` |
| "What's wrong with my environment? Why does bootstrap fail?" | `wsp doctor` |
| "Show me the JSON Schema of `workspace.yml` / `awac.yml` / `workspace.lock.yml`." | `wsp schema {workspace,awac,lock}` |
| "Give me a machine-readable catalog of `wsp` commands." | `wsp --agent-manifest` |
| "Estoy en un dir sin `.agents/` ni workspace.yml — ¿por dónde empiezo?" | `wsp guide <topic>` (e.g. `wsp guide discover`, `wsp guide init`) — imprime el path al doc embebido para `<topic>`. Es el escape-hatch cuando todavía no hay `.agents/` cargado. |
| "Creá un workspace pero quiero que me vaya guiando paso a paso." | `wsp init --interactive` — diálogo interactivo para elegir template + nombre + path. Equivalente al workflow `init_workspace` cuando el agente prefiere delegar el prompting al CLI. |
| "Mi `deploy.yml` está en `deploy/1`. Llevame a `deploy/2`." | `wsp migrate-deploy <product>` — migra el `deploy.yml` del producto en el cache de `deploy/1` → `deploy/2` y prepara un PR contra el stack repo (no edita el cache de forma persistente; el cache se refresca con `wsp sync`). Revisar el diff y mergear el PR a mano. |
| "Plan de deploy con mis overrides" / "raw stack defaults" | `wsp deploy <product> --plan` (con overrides — default) o `wsp deploy <product> --plan --no-overrides` (raw stack defaults, ignora `workspace.yml#deploy_overrides`). |
| "Verificar secrets con/sin overrides" | `wsp secrets check <product>` (con `devvault_overrides` aplicados — default) o `wsp secrets check <product> --no-overrides` (sólo catálogo). |

**Always prefer the CLI over manually editing**:
- `.agents/{rules,skills,workflows}/` content (`wsp sync` regenerates it).
- `workspace.yml` (re-run `wsp init` against a fresh template if the scaffold drifted, or edit *outside* the auto-generated regions).
- `workspace.lock.yml` (auto-managed by `wsp bootstrap` / `wsp sync`).
- The header of `CLAUDE.md` / `AGENTS.md` (auto-regenerated; only edit between `<!-- @awac:editable-start -->` and `<!-- @awac:editable-end -->`).
- `<org>/agent-stack/awac.yml#repos` (use `wsp scaffold-stack <org> --update`).

## When to skip this skill

- Pure code edits inside a product repo cloned in the workspace — that's normal git work, not workspace-level.
- Reading workspace files for context — no `wsp` involved.
- Running tests, linters, deploys, or anything that's not workspace shape mutation.
- One-off questions about how AWaC works conceptually — read the [spec](https://gist.github.com/GanemoCorp/fdc1042067615003e8aa4a8406198a61) instead of running commands.

## Decision flow when starting a task

1. **Is there a `workspace.yml` at the cwd?** If no → either we're outside a workspace, or the user wants to create one (use `wsp init`).
2. **Is there a `workspace.lock.yml`?** If no but `workspace.yml` exists → run `wsp bootstrap` first. Anything that needs the cache or the resolved stacks will fail otherwise.
3. **Is the user asking about state?** `wsp status` first, before assuming anything is in sync.
4. **Is the user asking to refresh stacks (rules / skills / workflows updated upstream)?** `wsp sync`. It does NOT touch product repos.
5. **Is the user asking to refresh the lockfile (e.g., they want the latest commit of every product repo too)?** `wsp bootstrap` (idempotent — safe to re-run).
6. **Is the user touching `awac.yml#org_scaffold` or `governance/product-structure.md`?** Run `wsp governance check` before pushing — same-PR rule means both must move together.

## Idiomatic invocations

```bash
# New workspace from a template
wsp init my-acme-feature --template acme-feature
cd my-acme-feature
wsp bootstrap

# Daily refresh (after pulling stack updates upstream)
wsp sync

# Read-only inspection
wsp status
wsp status --json | jq '.summary'

# Seed a product's agent-stack from scratch (creates the repo on GitHub)
wsp scaffold-stack <org>

# Refresh an existing agent-stack via PR (no direct main push)
wsp scaffold-stack <org> --update

# Dry-run scaffold (no push, just print the seed dir)
wsp scaffold-stack <org> --no-push

# Sanity-check: env + governance mirror
wsp doctor
wsp governance check
```

All commands accept `--json`. Errors carry a stable `code`/`category`/`cause`/`remediation` shape — `wsp --agent-manifest` documents the catalog for agent consumption.

## Configuration

- `WSP_REGISTRY_REPO` — defaults to `getGanemo/agent-stack-core`.
- `WSP_REGISTRY_BRANCH` — defaults to `main`.
- `WSP_CACHE_DIR` — defaults to `~/.wsp/cache`. Stacks and registry get cached here. Read-only from the user's perspective — `wsp` does `git reset --hard origin/<ref>` on every refresh, so DO NOT edit cached stacks by hand. Edit the upstream stack repo and run `wsp sync`.

## Anti-patterns

1. **Hand-editing `.agents/` and committing the change to the workspace.** That edit gets blown away by the next `wsp sync` or `wsp bootstrap`. If the change should propagate to other workspaces, edit the corresponding stack repo (`getGanemo/agent-stack-core` for universal, `getGanemo/agent-stack-<topic>` for transversal-by-topic, `<product>/agent-stack` for product-specific). If the change should ONLY apply here, write it inside the `<!-- @awac:editable-start -->` block of CLAUDE.md.
2. **Cloning product repos into the workspace by hand.** `wsp bootstrap` does it. The lockfile records the exact commits.
3. **Using `git pull` on product repos in the workspace to "update".** Use `wsp bootstrap` — it pulls FF-only and doesn't pisa local changes.
4. **Running `wsp scaffold-stack` against a product org WITHOUT `--update` if `<org>/agent-stack` already has content.** It refuses (by design). Use `--update` to open a PR.
5. **Skipping `wsp governance check` when changing either side of the mirror.** The check is now in `wsp doctor` too — you'll catch it accidentally even if you forget.
6. **Believing `wsp sync` is a substitute for `wsp bootstrap`.** sync does NOT clone product repos. If a workspace has never been bootstrapped (no lockfile), bootstrap first.
7. **Editing `workspace.lock.yml` by hand.** It's auto-generated. Edit `workspace.yml` and re-run `wsp bootstrap`.
8. **Inferring the workspace template from the folder name.** The folder name is coincidence; templates have semantic impact (they clone real product repos, embed product flows, gate deploys). A folder called `acme-billing/` does NOT imply the user wants the `acme-feature` template — they may want `odoo-module` instead. Always ASK the user (see the [`init_workspace`](../../workflows/init_workspace.md) workflow Step 2 and the [`discover_awac_in_fresh_workspace`](../discover_awac_in_fresh_workspace/SKILL.md) skill Step 4).
9. **Editing `.stack/<product>/{deploy,devvault}.yml` to silence an error.** Those files are read-only mirrors materialized by `wsp bootstrap` from the canonical in the stack repo. Your edit is gone on the next `wsp sync`. Edit the canonical, or use `workspace.yml#deploy_overrides` / `workspace.yml#devvault_overrides`.

## Cross-references

- Source / install: [`getGanemo/workspace-cli`](https://github.com/getGanemo/workspace-cli)
- Spec: <https://gist.github.com/GanemoCorp/fdc1042067615003e8aa4a8406198a61>
- Registry of stack shortcuts + templates + `org_scaffold`: [`getGanemo/agent-stack-core/awac.yml`](https://github.com/getGanemo/agent-stack-core/blob/main/awac.yml)
- Governance (canonical, atemporal): [`getGanemo/docs-company/governance/product-structure.md`](https://github.com/getGanemo/docs-company/blob/main/governance/product-structure.md)
- Live audit per product: [`getGanemo/docs-company/governance/product-structure.audit.md`](https://github.com/getGanemo/docs-company/blob/main/governance/product-structure.audit.md)

## Discoverability for agents

`wsp --agent-manifest` returns a JSON catalog of every command, its required args, options, and the JSON keys each emits. Agents that don't already know the CLI can self-discover from that.
