---
name: create_deploy_spec
description: >
  USE when a product needs its `<product>/agent-stack/deploy.yml` written for
  the first time, or when adding a new deployable component to an existing
  one. Ensures the spec validates against schema `deploy/1` (run
  `wsp schema deploy` for the live source of truth) and that each component
  declares: name, target, repo, requires_human_approval, pre_steps,
  promote_after_pass when relevant. NEVER deploy a product without a deploy
  spec — improvising the procedure is forbidden by `use_deploy_spec` rule.
  Activate when: the user asks to deploy a product that has no deploy.yml,
  the product introduces a new deployable component, or refactoring a
  legacy ad-hoc deploy script into a declarative spec.
---

# Create a deploy spec

A product without `<product>/agent-stack/deploy.yml` cannot be deployed by
agents — `use_deploy_spec` rule forbids improvising. This skill walks an
agent through writing a valid spec from scratch (or extending an existing
one) so that future deploys are reproducible and ack-gated.

---

## 1. Decide which stack repo (always the product stack)

The spec is per-product. It always lives at:

```
<product>/agent-stack/deploy.yml
```

Even when the deploy procedure is "shared logic" (e.g. push Odoo modules to
Odoo.SH, then to a canonical branch) — the LOGIC lives in the topical
workflow inside the relevant ecosystem stack (e.g.
`<ecosystem-org>/agent-stack/workflows/deploy_to_odoo_sh.md`); only the
PARAMETERS live in the product spec (which project, which modules, which
branch, who acks).

If you find yourself wanting to put deploy logic in the spec, that's a sign
the topical workflow is missing something — extend the workflow instead.

---

## 2. Inventory the deployable components

For the product, list every artifact that gets deployed somewhere:
- API/services (`orchestrator`, `mail-api`, `api`, `platform`)
- Static sites (`web`, `docs`)
- Cross-org modules consumed by the product (e.g. shared add-ons in another org)
- Background workers, MCP servers, etc.

Each becomes one entry under `components:`. A component that doesn't have
a target type yet is NOT a component — it's research-stage code.

---

## 3. Pick the right `target` per component

| Component shape | Likely target |
|---|---|
| Python/Node service in containers, ECS Fargate | `aws_ecs` |
| Lambda function | `aws_lambda` |
| EC2 instance behind SSM (no SSH) | `aws_ec2_ssm` |
| Astro / Next.js / Vite static site on Cloudflare | `cloudflare_pages` |
| Cloudflare Worker | `cloudflare_workers` |
| Odoo addon module pushed to Odoo.SH then to canonical | `odoo_sh` |
| Docs site published via GH Pages | `github_pages` |
| Anything that doesn't fit and requires manual steps | `manual` (last resort; document the steps in pre_steps) |

If none of the targets match, propose extending the schema enum in the `wsp`
source (`wsp/schemas/deploy.schema.json`) and adding a new
`deploy_to_<target>.md` topical workflow.

---

## 4. Required fields per component

```yaml
- name: orchestrator             # Required. Stable id.
  target: aws_ec2_ssm            # Required. Picks the topical workflow.
  repo: <product>/orchestrator   # Recommended. Source repo.
  requires_human_approval: true  # Default true. Set false ONLY for low-risk
                                 # preview/staging deploys.
  pre_steps:
    - run_tests_local            # Workflow IDs from any active stack.
    - terraform_plan
  promote_after_pass: []         # Empty unless this is an Odoo-style flow.
```

For the Odoo case, `promote_after_pass` is non-trivial:

```yaml
- name: <product>_portal_module
  target: odoo_sh
  repo: <ecosystem-org>/<product>_portal
  requires_human_approval: true
  odoo_sh:
    project: <product>-portal
    branch: main
    module_scope: list
    modules: [<product>_portal]
  pre_steps:
    - run_odoo_tests_docker_wsl  # MANDATORY before Odoo.SH
  promote_after_pass:
    - target_repo: <ecosystem-org>/<product>_portal
      target_branch: 19.0
      require_pass_on: odoo_sh
```

---

## 5. `requires_human_approval` defaults

Default: `true` for everything that hits a real environment. Reasoning:
the agent SHOULD pause before each component because the user might want
to deploy components selectively (e.g. push the API but not the modules).

Acceptable to set `false`:
- Preview deploys to ephemeral environments (CF Pages preview branch).
- Internal-only admin deploys behind feature flags.

Never `false` for: production, anything customer-facing, anything that
mutates AWS resources, anything that mutates Odoo.SH.

---

## 6. Validation before commit

```bash
# Print the schema
wsp schema deploy

# Smoke-validate by running the deploy command in plan-only mode
wsp deploy <product> --plan
```

The CLI validates the spec at parse time and refuses to run if it doesn't
match `deploy/1`.

---

## 7. Step-by-step

1. **Decide host stack repo** — always `<product>/agent-stack/`.
2. **Inventory components** (§ 2).
3. **Pick `target` for each** (§ 3).
4. **Fill required fields** (§ 4).
5. **Decide approval defaults** (§ 5).
6. **Write the file** at `<product>/agent-stack/deploy.yml`.
7. **Validate**: `wsp deploy <product> --plan`.
8. **Commit + push** to the agent-stack repo. From this point, any
   workspace running `wsp sync` (or `wsp bootstrap`) sees the new spec.

---

## 8. Cross-references

- Schema: `wsp schema deploy`
- Rule: `use_deploy_spec.md` (in agent-stack-core/rules/)
- Workflow router: `deploy_product.md` (in agent-stack-core/workflows/)
- Topical examples: `deploy_to_<target>.md` in the relevant ecosystem stack (e.g. the Odoo ecosystem stack hosts `deploy_to_odoo_sh.md`).
- Related skills: `manage_project_state` (for the `00_Mapa_Repositorios.md` cross-references), `create_repo_readme`.
