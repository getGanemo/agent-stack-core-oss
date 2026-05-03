---
trigger: always_on
description: Before deploying any component of a product, read its <product>/agent-stack/deploy.yml spec. Run pre_steps, honor requires_human_approval, and execute promote_after_pass only when the primary target acks. Never invent a deploy procedure.
globs: ["**/*"]
---

# Use the deploy spec

Each product declares its deploy procedure in
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
   specific transversal stack (e.g. `deploy_to_odoo_sh.md` in the Odoo
   ecosystem stack, `deploy_to_aws_ecs.md` in the AWS stack, etc.).
6. **Wait for the target ack** before any `promote_after_pass`. Never push
   to the canonical repo branch ahead of the target's success signal.
7. **Honor `rollback_window_minutes`** when set: monitor the deployed
   service for that many minutes after success, surface anomalies.
8. **Report**: 1-line outcome per component (success / aborted / failed).

## Anti-patterns

1. **Deploying without reading `deploy.yml` first.** Even when the user says
   "deploy <product>" and you've done it before — re-read the spec, things
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
- Topical workflows live in the relevant ecosystem stack (e.g. `deploy_to_odoo_sh.md` in the Odoo stack, `deploy_to_aws_ecs.md` in the AWS stack, `deploy_to_cloudflare_pages.md` in the Cloudflare stack).
