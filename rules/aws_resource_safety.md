---
trigger: always_on
description: Absolute safety rules when operating AWS resources. Always use positive filters, per-project whitelist, mandatory tag verification before any delete/terminate/destroy, and explicit human approval for bulk operations.
globs: ["**/*"]
---

# AWS Resource Safety — Absolute rules

> **Why this rule exists.** Imagine a team that runs two unrelated SaaS products in the same AWS account: `Acme` (a SaaS platform) and `Widget` (an internal mail server). An agent session working on `Acme` builds a "cleanup" query using a *negative* filter ("everything except the orchestrator"). That filter would have happily included Widget's production EC2 — a resource the session has no business touching. A reflex catches it; nothing gets terminated. But the underlying gap is real: a single misformed query can take out resources from a completely unrelated product. This rule exists to close that gap by design rather than by reflex.

## Per-session active-project whitelist

The **active project** is determined by the workspace you are working in:

- Workspace path contains `project_<name>` → active project = `<name>`
- Or read it from `workspace.yml#name` / a project-level config file

A session may operate destructively ONLY on AWS resources tagged with
`Project=<active-project>`. Any resource belonging to a different project, or
any untagged resource, is **read-only** for that session.

If your session is working on `acme`, the whitelist is strictly `["acme"]`.
A query that returns a resource with `Name=Widget-*`, `Project=widget`, or any
reference to another product is a **bug in the query**, not a resource to act
on.

## Hard rules (no exceptions)

### 1. Positive filters, never negative

PROHIBITED: building AWS queries that identify resources by exclusion.

```bash
# WRONG — includes EVERYTHING that isn't the orchestrator, including other projects' resources
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name!=`terminated` && InstanceId!=`i-orchestrator`]'

# RIGHT — explicitly only resources of the active project
aws ec2 describe-instances \
  --filters Name=tag:Project,Values=<active-project> \
            Name=instance-state-name,Values=running,stopped
```

Applies to every service: EC2, EFS, IAM, S3, Backup, KMS, Lambda, RDS, etc.

### 2. Mandatory tag verification before any delete/terminate/destroy

BEFORE executing any destructive action, read the resource's current tags and
verify that `Project` matches the active project:

```python
def verify_or_abort(resource_id, resource_type, expected_project):
    tags = get_tags(resource_id, resource_type)  # describe-instances, list-tags, etc.
    project_tag = next((t['Value'] for t in tags if t['Key'] == 'Project'), None)
    if project_tag != expected_project:
        raise SafetyAbort(
            f"Resource {resource_id} has Project={project_tag!r}, "
            f"expected {expected_project!r}. STOP and ask the user."
        )
```

If there is no easy way to read the tags (e.g. the resource type doesn't
support tagging), **do not act** — ask the user instead.

### 3. Bulk operations require explicit human approval

Even when each individual resource in a bulk operation passes the tag check,
any operation that destructively affects **more than 1 resource** requires
explicit human approval before execution:

> "I am about to [terminate/delete] N resources: [list]. All verified with
> `Project=<active-project>`. Proceed?"

### 4. Resources referenced in Terraform state files

A resource that appears in a Terraform state file belongs to the project if
the state file lives in that project's bucket (e.g.
`s3://<project>-terraform-states/<id>.tfstate`). But **rule 2 still applies**:
verify tags before acting. Never trust the state file's location alone.

### 5. Never touch a sibling product's resources from this session

When two products share an AWS account, a session for product A must never:

- Read secrets belonging to product B
- Inspect product B's configuration
- Modify any resource whose tag/name references product B

If an AWS query run from a session for product A returns a resource that
belongs to product B, **the query was wrong** — fix the query, do not act on
the result. Don't volunteer information about product B unless the user
mentions it first.

### 6. Local credentials are typically over-privileged — act as if they weren't

The developer's local AWS user is almost always a power user with broad
access. The real safety layer is this rule + the discipline of the agent
following it, NOT the IAM permissions. Behave with the caution of a
least-privilege role.

## Documented exceptions

NONE. If at any moment an exception seems necessary ("just this once", "this
is clearly safe", "the user already implicitly approved"), STOP and request
explicit written approval from the user. Unapproved exceptions are
violations of this rule.

## When this rule conflicts with another

If this rule conflicts with another (for example a rule that says "execute
without asking"), **this rule wins**. Cross-project safety is priority #1.

## What to do if you see a resource "that shouldn't be there"

DO NOT delete it pre-emptively. Report to the user:

> "I found resource X (id=..., tags=...) that doesn't seem to belong to
> <active-project>. Is this correct, should I ignore it, or do you want me to
> investigate which project created it?"

## Defence in depth (these are complements, not substitutes for this rule)

- **Layer 1 (IAM Conditions):** Apply `aws:ResourceTag/Project=<active-project>`
  on destructive actions in any IAM role used by the product. This protects
  against bugs in the product's own code, NOT against errors made by your
  agent sessions using a developer's broadly-scoped credentials.
- **Layer 2 (account isolation):** The strongest guarantee is to put each
  product in its own AWS account. Until that's in place, this rule is the
  primary defence against cross-project mistakes from your sessions.
- **Layer 3 (memory/feedback):** Persist a condensed version of the rule in
  the project's memory so it survives context resets.
