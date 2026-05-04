---
trigger: always_on
description: Before deploying any component of a Ganemo product, read its <product>/agent-stack/deploy.yml spec. Run pre_steps, honor requires_human_approval, and execute promote_after_pass only when the primary target acks. Never invent a deploy procedure.
globs: ["**/*"]
---

# Use the deploy spec

Each Ganemo product declares its deploy procedure in
`<product>/agent-stack/deploy.yml` (schema `deploy/1` — see
`wsp schema deploy`). Agents MUST consume that spec; never invent a deploy
flow ad hoc.

## What the spec declares

For each deployable component:

| Field | Meaning |
|---|---|
| `name` | Stable component id used in CLI/logs. |
| `target` | One of `odoo_sh`, `aws_ecs`, `aws_lambda`, `aws_ec2_ssm`, `cloudflare_pages`, `cloudflare_workers`, `github_pages`, `manual`. Determines which topical workflow runs. |
| `repo` | Source repo (org/name) for this component, when applicable. |
| `requires_human_approval` | Default `true`. The agent PAUSES before pushing this component until the human types an explicit ack ("subilo", "go", "deployá"). |
| `pre_steps` | Workflow IDs that MUST PASS before deploy. Failures abort. |
| `promote_after_pass` | Push to additional repos/branches AFTER the target acks success. Used for the Odoo flow (push to Odoo.SH first, then to canonical repo branch). |
| `<target>:` sub-block | Target-specific config (project, branch, modules, cluster, etc.). |

## What the agent does at deploy time

1. **Resolve the spec**: read `<product>/agent-stack/deploy.yml`. If the
   product has no spec, STOP and run the `create_deploy_spec` skill first.
2. **Pick the components to deploy**: a single `name`, multiple, or "all".
   Default behavior when ambiguous: ask the user.
3. **For each component, run `pre_steps` in order**. Any failure → abort and
   report. Don't skip pre_steps under any circumstance.
4. **If `requires_human_approval: true`**: print the deploy plan and STOP.
   Only proceed when the user explicitly approves THIS component (a generic
   "ok" earlier in the conversation does NOT cover later components).
5. **Delegate the actual deploy to the topical workflow** for the target
   (`deploy_to_<target>.md`). Each topical workflow lives in the most
   specific transversal stack (`deploy_to_odoo_sh.md` →
   `erp-partners/agent-stack/`, `deploy_to_aws_ecs.md` →
   `getGanemo/agent-stack-aws/`, etc.).
6. **Wait for the target ack** before any `promote_after_pass`. Never push
   to the canonical repo branch ahead of the target's success signal.
7. **Honor `rollback_window_minutes`** when set: monitor the deployed
   service for that many minutes after success, surface anomalies.
8. **Report**: 1-line outcome per component (success / aborted / failed).

## Two-layer model: stack defaults + workspace overrides

As of `wsp` v0.10.0 (`deploy/2`), every field in a component of `deploy.yml`
is a **stack default**. A workspace can override per-component via
`workspace.yml#deploy_overrides`. This lets one workspace deploy to a staging
project, another to a Lambda variant of the same component, without forking
the stack.

| Layer | File | Per | Versioned? |
|---|---|---|---|
| **Stack defaults** | `<product>/agent-stack/deploy.yml` | product | yes |
| **Workspace overrides** (NEW) | `workspace.yml#deploy_overrides` | workspace | yes (in the workspace) |
| **Materialized mirror** (read-only) | `.stack/<product>/deploy.yml` | workspace cache | derived |

Override schema, by component name:

```yaml
# workspace.yml
deploy_overrides:
  mail_api:
    target: aws_lambda            # swap target away from stack default aws_ecs
    aws_lambda:
      function_name: mail-api-staging
  odoo_sh:
    odoo_sh:
      project: acme-staging  # override project for this workspace only
  reporting_worker:
    skip: true                    # exclude from this workspace's deploys
```

Resolution rules:

- The override is **shallow-merged per component** — fields you don't restate
  fall through to the stack default.
- `skip: true` on a component excludes it entirely from `wsp deploy <product>`
  for this workspace (useful for workspaces that don't touch a particular
  component).
- `target` swaps trigger validation against the new field below.

### `targets_available` (deploy/2 new field)

A component MAY declare `targets_available: [<list of target ids>]` in its
stack default. This is the closed set a workspace's `deploy_overrides` may
swap `target` to. If a workspace tries to override `target` to a value not
in the list, the CLI raises **WSP_019** (`deploy_target_not_available`) and
refuses to plan.

Example:

```yaml
# stack default (deploy/2)
- name: mail_api
  target: aws_ecs
  targets_available: [aws_ecs, aws_lambda]
  aws_ecs: { cluster: widget-prod, service: mail-api }
  aws_lambda: { function_name: mail-api }
```

A workspace may now override to `aws_lambda` but not to, say,
`cloudflare_workers` — that path is closed by the stack author.

### Materialization (`.stack/<product>/deploy.yml`)

`wsp bootstrap` materializes the stack `deploy.yml` into
`.stack/<product>/deploy.yml` as a **read-only mirror**. Treat it like the
devvault mirror: derived state, regenerated by `wsp sync`. Don't edit it to
silence anything — edit the canonical in the stack repo, or add a
`deploy_overrides` entry in `workspace.yml`.

### CLI: seeing raw stack defaults

`wsp deploy <product>` applies workspace overrides by default. To see the
raw stack defaults (e.g., to compare against the override, or to debug a
WSP_019), use:

```bash
wsp deploy <product> --no-overrides --plan
```

Same flag exists on `wsp secrets check`.

## Anti-patterns

1. **Deploying without reading `deploy.yml` first.** Even when the user says
   "deployá acme" and you've done it before — re-read the spec, things
   change.
2. **Skipping `pre_steps`** because they take long. They exist precisely
   because the alternative (deploying broken code) is worse.
3. **Skipping `requires_human_approval`** even when "the user is logged in"
   or "this is the same flow as last time". Approval is per-deploy, per-component.
4. **Running `promote_after_pass` before the target acks.** That is the
   exact failure mode the field exists to prevent — pushing to the canonical
   branch a version that the deploy target already rejected.
5. **Inventing a deploy procedure** because the user wants something done
   right now. If the spec is missing, the right move is `create_deploy_spec`,
   not improvisation.

## Cross-references

- Schema: `wsp schema deploy`
- Skill: `create_deploy_spec` (when a product has no spec yet).
- CLI command: `wsp deploy <product> [--component <name>]` runs the router.
- Topical workflows: `deploy_to_odoo_sh.md` (in erp-partners), `deploy_to_aws_ecs.md` (in agent-stack-aws), `deploy_to_cloudflare_pages.md` (in agent-stack-cloudflare).
