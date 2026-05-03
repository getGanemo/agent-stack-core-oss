---
description: Routes workspace initialisation to the correct topical workflow based on project type. Configures the devvault pointer and delegates to the matching `init_project_<type>.md`.
---

# Init Project (Router)

Routes workspace initialisation to a topical workflow based on the kind of
project the user wants to start.

This is a **router**: it doesn't do the real work, it picks the right
specialised workflow. The available choices depend on which stacks the
workspace composes; the table below shows a typical example.

// turbo-all

## Step 1 — Pick the project type

Present the user with the options exposed by the active stacks. A typical
fork might surface:

```
Project type:

  1) Odoo module development or research
  2) Product feature work — pick one of your active products
  3) Generic enterprise / consulting project
  T) Doctoral thesis

Select an option [1-3, T]: _
```

Save the selection in `PROJECT_TYPE`.

## Step 2 — Configure the devvault pointer (all options)

### 2.1 — Read the current `devvault.yml`

```powershell
$devvaultContent = Get-Content "devvault.yml" -Raw -ErrorAction SilentlyContinue
if ($devvaultContent -match 'vault_path:\s*"?([^"\r\n]+)"?') { $currentVaultPath = $matches[1].Trim() }
else { $currentVaultPath = "" }
```

### 2.2 — Confirm or update

Show the user the current `vault_path` and ask whether to keep it or replace
it with a new absolute path on this machine. Never commit the resulting
value into a versioned file — `vault_path` belongs to `~/.devvault/.config.yml`,
which is per-machine (see the `use_devvault` rule).

### 2.3 — Persist the project pointer

Write a minimal `devvault.yml` in the workspace pointing to the right
project name and listing the secrets the project needs. The exact secret
keys depend on the project; see `use_devvault.md` for the schema.

## Step 3 — Delegate to the topical workflow

| Type | Workflow to execute |
|------|---------------------|
| 1    | `.agents/workflows/init_project_odoo.md` |
| 2    | `.agents/workflows/init_project_<product>.md` (one per active product) |
| 3    | `.agents/workflows/init_project_empresarial.md` |
| T    | `.agents/workflows/init_project_tesis.md` |

**Action**: read and execute the workflow that matches the chosen
`PROJECT_TYPE`.

## Available topical workflows

Each topical workflow is independent and can also be invoked directly:

- `init_project_odoo` — Odoo modules, Odoo.SH tasks, dependencies
- `init_project_<product>` — repos, infrastructure, tests for a specific product
- `init_project_empresarial` — generic corporate client
- `init_project_tesis` — academic thesis structure
