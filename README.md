# agent-stack-core

The universal foundation of **Agent Workspace as Code (AWaC)**. Every workspace loads this stack automatically.

This repo is the public reference. Adopters typically fork it, add their own product-stack shortcuts and templates to `awac.yml`, and point their `wsp` CLI at the fork via `WSP_REGISTRY_REPO`.

- **CLI**: the AWaC `wsp` CLI consumes this repo as the registry.
- **Spec**: see the public AWaC spec gist for the canonical schema description.

## Related canonical docs

- **Governance** (canonical, atemporal): your team's product-structure document. The `org_scaffold` block in this repo's `awac.yml` is the machine-readable mirror — changes to either must propagate to the other in the same PR. Run `wsp governance check` (or `wsp doctor`) before pushing changes to `awac.yml` to verify the mirror still aligns with the canonical doc.
- **Live audit per product**: your team's per-product audit log (analogous to `governance/product-structure.audit.md`).
- **CLI** (consumes this repo as the registry): the `wsp` command.

## What lives here

- **`awac.yml`** — the canonical registry: stack shortcuts, template catalogue, and org-introspection rules consumed by `wsp scaffold-stack`.
- **`rules/`** — universal rules: anti-prompt-injection, AWS resource safety (cross-project guardrails), git verification, capture-learnings, etc. These apply to every workspace.
- **`skills/`** — universal skills: `create_skill`, `create_agent_rule`, `create_repo_readme`, `create_deploy_spec`, `manage_project_state`, `use_wsp_cli`, `verify_code`.
- **`workflows/`** — universal workflows: `init_project`, `init_workspace`, `install_wsp`, `onboard_new_product`, `share_learnings`, `compact`, `propagate_changes`, etc.
- **`templates/`** — at minimum the `blank.yml` template; other templates live in their respective stacks but are catalogued here.
- **`hooks/`** — global hooks (Git/Claude Code) that ship with every workspace.

## How the registry works

When the CLI bootstraps a workspace, it always clones this repo first. It reads `awac.yml` to resolve shortcuts:

```yaml
# in workspace.yml
stacks: [core, aws, my-product]
```

The CLI looks up `aws` and `my-product` in `awac.yml/shortcuts` and resolves them to their full `<org>/<repo>` location. Anything containing a `/` is treated as a literal `<org>/<repo>` and skips the registry.

## Adding a new stack to the registry

1. Open a PR to your fork of this repo.
2. Add an entry under `awac.yml/shortcuts`.
3. (Optional) Add a template entry under `awac.yml/templates`.
4. Once merged, every new `wsp init` and `wsp sync` run resolves the new shortcut.

## Promoting an improvement back

If you edit a file that originated from this stack inside any workspace:

```bash
wsp promote .agents/rules/<file>.md
```

The CLI detects this stack as origin and opens a PR here. Standard team review applies.

## Maintainer

Issues and PRs welcome. Contact: `fernando@ganemo.co`.
