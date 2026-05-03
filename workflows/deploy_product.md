---
description: >
  Universal deploy router. Reads `<product>/agent-stack/deploy.yml`, validates
  it against schema `deploy/1`, runs each component's `pre_steps`, requests
  human approval when required, delegates to the right `deploy_to_<target>.md`
  topical workflow, and only triggers `promote_after_pass` once the target
  acknowledges. ALWAYS invoke this workflow when the user asks to deploy a
  product — never invent the procedure.
---

# Deploy product (router)

This is the entry point for any deploy operation. It consumes the
per-product spec at `<product>/agent-stack/deploy.yml` and delegates the
actual deploy to per-target workflows. **Never deploy without running this
router.**

// turbo-all

---

## Step 1 — Resolve the spec

```bash
# In a workspace, the product stack is already cloned.
# Read the spec:
cat <product>/agent-stack/deploy.yml
```

If the file doesn't exist, STOP. Tell the user "this product has no deploy
spec" and invoke the `create_deploy_spec` skill to author one. Don't proceed
under any circumstance.

Validate against the schema:

```bash
wsp deploy <product> --plan
```

`--plan` parses the spec, prints what would happen, and exits 0 if valid.
Any validation error → STOP and fix the spec.

---

## Step 2 — Identify components to deploy

The user can ask:
- "deploy <product>" → all components in order.
- "deploy <product> orchestrator" → only that named component.
- "deploy <product> --component <component>" → same.

Default behavior when ambiguous: list components and ASK which one.

---

## Step 3 — For each component, run `pre_steps` in order

Each `pre_steps` entry is a workflow id. Resolve it from the active stacks
(check `.agents/workflows/` for files matching the id). Common ones:

| pre_step id | Topical stack | What it does |
|---|---|---|
| `run_tests_local` | varies (in `<product>/agent-stack/workflows/`) | Run the product's local test suite. |
| `run_odoo_tests_docker_wsl` | the Odoo ecosystem stack | Spin up a Docker Odoo + run module tests on Linux for Windows. MANDATORY before any Odoo.SH push. |
| `terraform_plan` | the AWS stack | Run `terraform plan` and show the diff. |
| `build_static` | varies | Compile static assets ahead of CF Pages push. |

Any pre_step that fails → ABORT this component. Report the failure, do not
proceed to the next component (unless the user explicitly says skip).

---

## Step 4 — Honor `requires_human_approval`

If the component sets `requires_human_approval: true` (default), STOP after
pre_steps and BEFORE delegating to the topical workflow. Print a structured
summary:

```
Ready to deploy:
  product:    <product>
  component:  <name>
  target:     <target>
  repo:       <repo>
  pre_steps:  PASS (<N> steps)
  target config:
    <yaml of the target sub-block>
Reply 'go' / 'subilo' / 'deploy' to proceed; anything else cancels.
```

Wait for the human ack. A generic "ok" earlier in the conversation does NOT
qualify — approval is per-component, per-deploy.

When `requires_human_approval: false`, skip this step but still print the
summary as a 1-line log entry.

---

## Step 5 — Delegate to the topical workflow

Resolve `deploy_to_<target>.md` from the active stacks:
- `odoo_sh` → the Odoo ecosystem stack's `deploy_to_odoo_sh.md`
- `aws_ecs` → the AWS stack's `deploy_to_aws_ecs.md`
- etc.

If the topical workflow doesn't exist in the active stacks, STOP. Tell the
user the stack is missing and either bootstrap with the right shortcut
(`wsp init … --template ...` should already include it) or extend the
relevant transversal stack with the new workflow.

Pass the component spec to the topical workflow. The topical workflow
returns either SUCCESS or FAILURE. Capture the result.

---

## Step 6 — `promote_after_pass`

If the topical workflow returned SUCCESS and the spec declares
`promote_after_pass`, execute each promotion:

```yaml
promote_after_pass:
  - target_repo: <ecosystem-org>/<product>_portal
    target_branch: 19.0
    require_pass_on: odoo_sh
```

`require_pass_on` filters the promotion to "only when target X acked
success". For Odoo, this prevents promoting to the canonical branch a
version that Odoo.SH already rejected.

Promotion = git push from the deploy artifact / branch into the canonical
repo branch. This is fast-forward only — refuse if the canonical branch
diverged.

Failure here → loud alert. The target deployed but the canonical repo
wasn't updated, so the next deploy will replay the same code.

---

## Step 7 — Honor `rollback_window_minutes`

If set (e.g. 30), monitor the target's health for that window post-deploy.
For aws_ecs: poll the service's task health. For odoo_sh: poll the build
status until "ready" + monitor the health endpoint of any module that
exposes one. For cloudflare_pages: poll the deployment status.

Any anomaly → surface it. Don't auto-rollback unless the topical workflow
explicitly supports it.

---

## Step 8 — Report

One-line outcome per component:

```
[OK]   orchestrator (aws_ec2_ssm) — deployed at 12:34:56, rollback window 30m
[FAIL] <product>_portal_module (odoo_sh) — pre_steps failed: run_odoo_tests_docker_wsl
[SKIP] admin (cloudflare_pages) — user declined approval
```

When done, write a `progreso.md` entry to the product's `project_management`
following the `manage_project_state` skill, recording the deploy outcome.

---

## Anti-patterns

1. **Skipping the router and calling a topical workflow directly.** That
   bypasses pre_steps, approval, and promotion. The router exists to
   sequence them.
2. **Treating "ok" earlier in the chat as approval for all components.**
   Each component asks separately.
3. **Promoting to canonical before the target acks.** That's the bug
   `require_pass_on` exists to prevent.
4. **Deploying when `wsp doctor` has FAIL checks.** Fix doctor first.

---

## Cross-references

- Rule: `use_deploy_spec.md` (companion).
- Skill: `create_deploy_spec` (when no spec exists).
- CLI: `wsp deploy <product> [--component <name>] [--plan]`.
- Schema: `wsp schema deploy`.
