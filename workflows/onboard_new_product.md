---
description: >
  Onboard a new product end-to-end on AWaC. Assumes the user has ALREADY
  created: the GitHub org, the domain, the AWS account, and a local empty
  directory. Drives the rest: agent-stack, the team's mandatory governance
  repos (Cat A), registration in the core registry, project_management
  scaffold, devvault catalog, an audit/log entry in the team's governance
  doc, and finally a working local workspace. Run this when the user says
  "onboard new product", "create a new product on AWaC", "let's start
  <name>", or similar.
---

# Onboard a new product (end-to-end)

Procedure to take a product from "I have the name + the org + the domain +
the AWS account" to "agent-stack alive, governance Category-A repos created,
project_management initialised, registry updated, first local workspace
bootstrapped". Roughly 9 steps, ~15-30 minutes wall-clock.

This workflow assumes a generic governance pattern with at least three
mandatory repos per product (`project_management`, `agent-stack`,
`infrastructure` — collectively "Category A"). Adopters with a different
governance pattern should adapt the steps to their own categories.

// turbo-all

---

## Step 0 — Gather required inputs

Ask the user for the inputs the product already has outside AWaC:

| Input | Example |
|---|---|
| Product display name | `Secrevo` |
| Slug (lowercase, no spaces) | `secrevo` |
| GitHub org (raw, with the casing GH uses) | `secrevo` (or `getsecrevo` per your team's org-naming convention if the bare slug is taken) |
| Primary domain | `secrevo.com` |
| AWS account ID (12 digits) | `123456789012` |
| Local workspace path (empty directory) | `~/dev/secrevo-bootstrap/` |

If any input is missing, stop and ask. DO NOT guess (especially the AWS
account ID — if it's wrong, the `infrastructure` repo description will be
wrong).

Confirm ALL inputs in one go before proceeding. Once confirmed, do not ask
again.

---

## Step 1 — Verify CLI prerequisites

```bash
wsp --version 2>&1 || echo "WSP_MISSING"
gh auth status >/dev/null 2>&1 || echo "GH_AUTH_MISSING"
gh api orgs/<ORG> >/dev/null 2>&1 || echo "ORG_NOT_VISIBLE"
```

| State | Action |
|---|---|
| `wsp` not installed | Invoke the [`install_wsp`](install_wsp.md) workflow first. |
| `gh` not authenticated | `gh auth login --web --hostname github.com`. |
| Org not visible | The user has to create the org in GitHub or grant access on the PAT/GitHub App. |
| All OK | Proceed. |

---

## Step 2 — Create `<org>/agent-stack` with auto-register in core

```bash
wsp scaffold-stack <ORG>
```

Because the org is empty:

- `wsp` will introspect it and see 0 repos.
- It will generate a seed with `Cat A: empty / initial scaffold; add Cat A when the product exists`.
- It will create `<ORG>/agent-stack` (private).
- It will **auto-register** `<product>: <ORG>/agent-stack` and a `<product>-feature` template in your fork of `agent-stack-core/awac.yml` (default since v0.8.0).

Verify the output: it must say `pushed: yes (main)` and `registry: ok — <product> registered in <your-org>/agent-stack-core`.

If `registry: skipped/no-op`, run the registry edit manually (rare — only happens if `--no-register` was used).

---

## Step 3 — Create the mandatory governance repos

Per the team's governance, each product should have at least
`project_management`, `agent-stack` (already created in Step 2), and
`infrastructure`. Create the remaining two.

### 3a — `project_management` (snake_case)

```bash
wsp scaffold-repo <ORG>/project_management --category A
```

- Creates the repo (private) with a governance-compliant description.
- Pushes a Cat A seed README.
- snake_case is the single naming exception; visibility is private.

### 3b — `infrastructure`

```bash
wsp scaffold-repo <ORG>/infrastructure --category A \
  --aws-account <AWS_ACCOUNT_ID> \
  --domain <DOMAIN>
```

- `--aws-account` and `--domain` are **mandatory** here (otherwise the
  description ends up with placeholders and the audit in Step 8 fails).
- Resulting description: `<Product> — Terraform (AWS account <ID> + DNS provider <domain>)`.

---

## Step 4 — Refresh the agent-stack's `awac.yml#repos` after Cat A

```bash
wsp scaffold-stack <ORG> --update --push-direct
```

Replaces the (empty until now) `repos:` block with the new list that includes
the 2 Cat A repos just created. **Direct push to `main`** (no PR) — the user
is the owner of the agent-stack just created seconds before in Step 2, and
the change is additive. Flag `--push-direct` (CLI v0.9.0+).

Wait for `pushed: yes (main)` in the output. No manual merge, no waits. If
the push fails on permissions (corner case), drop `--push-direct` to fall
back to standard PR mode.

---

## Step 5 — Initialise `<ORG>/project_management/00_Base/`

Clone the repo just created:

```bash
git clone https://github.com/<ORG>/project_management /tmp/<product>-pm
cd /tmp/<product>-pm
```

Apply the [`manage_project_state`](../skills/manage_project_state/SKILL.md)
skill to scaffold the initial content. Minimum:

```
project_management/
├── README.md                          # one-pager with current phase
├── 00_Base/
│   ├── .project_definition.md          # what the product is, audience, AWS account, domain
│   ├── 00_Roadmap_<Product>.md         # high-level roadmap + planned phases
│   ├── 00_Mapa_Repositorios.md         # table of product repos including any cross-org
│   ├── 00_Arquitectura_Sistema.md      # diagram + components
│   ├── 00_Decisiones_Arquitectura.md   # ADR index, initially empty
│   ├── 00_Convenciones.md
│   └── 00_Glosario.md
└── Fase_01_Fundamentos/
    ├── plan.md
    ├── progreso.md                     # APPEND-ONLY, first entry: "product onboarded YYYY-MM-DD"
    └── decisiones.md
```

Fill `.project_definition.md` with the inputs from Step 0 (name, slug, org,
domain, AWS account). Fill `00_Mapa_Repositorios.md` with the Cat A repos
just created.

Commit + push:

```bash
git add -A
git commit -m "Initial project_management scaffold for <Product>"
git push origin main
```

---

## Step 6 — Create the `devvault.yml` catalogue in the agent-stack

Clone `<ORG>/agent-stack`:

```bash
git clone https://github.com/<ORG>/agent-stack /tmp/<product>-stack
cd /tmp/<product>-stack
```

Create `devvault.yml` listing the typical secrets the product needs (per the
`use_devvault` rule):

```yaml
schema: devvault/1
product: <product>
description: >
  Secrets needed to operate <Product>. Resolves against the developer's
  local ~/.devvault/<vault_path>/ via the use_devvault rule.

secrets:
  aws_account: aws/account.yml             # Organizations management
  aws: aws/<product>.yml                   # Dedicated account <AWS_ACCOUNT_ID>
  cloudflare: providers/cloudflare.yml     # Section: <product>
  github_app: github/<product>-app.yml     # GitHub App credentials
  github_pem: github/<product>-app.pem
  # Add more as needed: stripe, openai, smtp, telegram, etc.
```

DO NOT put `vault_path` in there — the schema explicitly rejects it
(`vault_path` lives in `~/.devvault/.config.yml`, per-machine).

Commit + push (can go directly to main: it's additive and the repo is
small).

---

## Step 7 — Add the `<Product>` entry to the team's governance audit

Locate your team's governance audit document (the per-product compliance
log) and add an entry for the new product. If the audit document is hosted
in a knowledge brain or wiki, use the appropriate edit tool (e.g.
`mcp__blenau__edit_section`) rather than a raw git clone — that preserves
provenance and cross-link suggestions.

Suggested initial entry:

```markdown
### <Product>

- **Status**: onboarded YYYY-MM-DD, foundations phase.
- **Org**: `<ORG>`.
- **Domain**: `<DOMAIN>`.
- **AWS account**: `<AWS_ACCOUNT_ID>`.
- **Cat A**: project_management (snake), agent-stack, infrastructure — all present.
- **Cat B/C/D/E**: pending depending on the product's architecture.
- **Pending**: define the architecture → create Cat C repos → declare any
  cross-org modules if applicable.
```

---

## Step 8 — Audit the product

```bash
wsp audit <product>
```

Expected: `summary: ok=N  warn=M  fail=0  (PASS)`.

If there is a `FAIL`:

- `cat_a/<repo>_exists FAIL` → a Cat A repo is missing. Go back to Step 3.
- `registry/shortcut FAIL` or `registry/template FAIL` → auto-register did
  not run. Re-run `wsp scaffold-stack <ORG>` (it's a no-op for the repo but
  re-attempts the registry edit).
- `agent_stack/awac_yml_repos WARN: empty` → `awac.yml#repos` is empty. Go
  back to Step 4 (`wsp scaffold-stack <ORG> --update --push-direct`).

If there are `WARN`s you can't resolve today (e.g. `deploy.yml missing`
because the product has nothing deployable yet), that's fine — `WARN` does
not block.

---

## Step 9 — Create the first local workspace

```bash
cd <USER_LOCAL_DIR>   # the empty directory from Step 0
wsp init <product>-bootstrap --template <product>-feature
cd <product>-bootstrap
wsp bootstrap
```

The `<product>-feature` template already exists (created by scaffold-stack
in Step 2 and registered in the registry in the same step).

Expected bootstrap output:

- 1+ stacks resolved (at least `core`).
- 2 repos cloned (`project_management` + `infrastructure`).
- `.agents/` composed.
- `workspace.lock.yml` written.

Hand-off to the user:

```
Product <Product> onboarded.
  Org:                    <ORG>
  Cat A repos:            project_management + agent-stack + infrastructure
  Registry:               <product> shortcut + <product>-feature template
  Local workspace:        <USER_LOCAL_DIR>/<product>-bootstrap
  Audit:                  PASS (N ok, M warn, 0 fail)

Next steps (when you're ready):
  1. Define the product's architecture → create Cat C repos.
  2. If it has cross-org modules → declare them and add them to awac.yml#repos.
  3. Configure DNS / AWS Organizations for the new account.
  4. Add deploy.yml in <ORG>/agent-stack when there's something deployable.
  5. Load the secret values into ~/.devvault/<declared paths> from your
     password manager. Then run `wsp secrets check <product>` to verify.
```

---

## Anti-patterns

1. **Skipping Step 0** and inferring inputs. If the AWS account ID is wrong
   in the `infrastructure.description`, `wsp audit` catches it but the repo
   is left with an out-of-date description visible on GitHub.
2. **Skipping Step 4**. If `--update` is not run after creating Cat A,
   `awac.yml#repos` of the agent-stack stays empty and no bootstrapped
   workspace will clone anything useful from the product.
3. **Creating Cat A via gh CLI directly** (without going through
   `wsp scaffold-repo`). You lose the governance-compliant description, the
   seed README, and the audit pass.
4. **Editing the team's governance audit document directly via raw git** in
   Step 7 when it's hosted in a knowledge brain. Use the brain's edit tool
   to preserve provenance and cross-link suggestions.
5. **Loading secret values via the agent.** The agent NEVER touches
   `~/.devvault/`. That is the human's responsibility from the password
   manager.

---

## Cross-references

- Companion workflow: [`init_workspace.md`](init_workspace.md) — Step 9 is
  basically `init_workspace` with a known template.
- Companion workflow: [`install_wsp.md`](install_wsp.md) — Step 1 fallback.
- Skill: [`manage_project_state`](../skills/manage_project_state/SKILL.md) — Step 5.
- Skill: [`use_wsp_cli`](../skills/use_wsp_cli/SKILL.md) — when to invoke wsp.
- Rule: [`use_devvault`](../rules/use_devvault.md) — Step 6.
- Governance: `<your-governance-doc>` (your team's product-structure document).
