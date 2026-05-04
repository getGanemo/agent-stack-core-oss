---
trigger: always_on
description: Resolve secrets via the per-product devvault catalog + per-machine vault config. Never hardcode credentials, API keys, tokens, or PEM files in any repo.
globs: ["**/*"]
---

# Use DevVault

Every Ganemo product has a **catalog of secrets it needs to operate**. The
catalog is versioned per-product. The actual secret values live ONLY on the
developer's machine, in `~/.devvault/`. Never write credentials, API keys,
tokens, OAuth client secrets, PEM files, or session cookies into any repo.

## Two-layer model

| Layer | File | Per | Versioned? |
|---|---|---|---|
| **Catalog** (logical name → relative path) | `<product>/agent-stack/devvault.yml` | product | yes |
| **Vault path** (where on disk) | `~/.devvault/.config.yml` | machine | NO |
| **Secret values** | files inside `~/.devvault/<vault_path>/<relative-path>` | machine | NO |

The catalog declares "this product needs an `aws` secret at `aws/atlas.yml`,
a `cloudflare` secret at `providers/cloudflare.yml`". The agent resolves each
logical name by joining `~/.devvault/.config.yml#vault_path` + the relative
path.

## How to read a secret

```python
import yaml
from pathlib import Path

# 1. Resolve where the vault lives on this machine.
config = yaml.safe_load((Path.home() / ".devvault" / ".config.yml").read_text())
vault_path = Path(config["vault_path"])

# 2. Resolve which catalog applies to this product.
#    In a workspace, look at <product>/agent-stack/devvault.yml.
#    From a stack repo, this is the local devvault.yml.
catalog = yaml.safe_load(Path("path/to/<product>/agent-stack/devvault.yml").read_text())

# 3. Read the secret you need by its logical name.
secret_path = vault_path / catalog["secrets"]["aws"]
aws_secrets = yaml.safe_load(secret_path.read_text())
```

`wsp doctor` verifies that `~/.devvault/.config.yml` exists and `vault_path`
resolves. If it fails, the agent must STOP and run the install procedure (see
the `install_wsp` workflow's analogue for devvault below) before any operation
that would consume secrets.

## Two-layer model + workspace overrides

The catalog/per-machine split above is the **base** model. As of `wsp` v0.10.0
there is also a **per-workspace override layer** for situations where one
workspace needs to point a logical secret somewhere different from the
catalog default (typical case: a test workspace pointing `cloudflare` to
`providers/cloudflare-staging.yml` instead of the prod file).

| Layer | File | Per | Versioned? |
|---|---|---|---|
| **Catalog** (logical name → relative path) | `<product>/agent-stack/devvault.yml` | product | yes |
| **Per-workspace override** (NEW) | `workspace.yml#devvault_overrides` | workspace | yes (in the workspace, not the stack) |
| **Vault path** (where on disk) | `~/.devvault/.config.yml` | machine | NO |
| **Secret values** | files inside `~/.devvault/<vault_path>/<relative-path>` | machine | NO |

`devvault_overrides` is a `{logical_name: alternative_path}` mapping. When
`wsp secrets check` runs from the workspace dir, the resolver wins-order is:

1. `workspace.yml#devvault_overrides[<name>]` — if set, used as the relative path.
2. `<product>/agent-stack/devvault.yml#secrets[<name>]` — catalog default.

Example:

```yaml
# workspace.yml
devvault_overrides:
  cloudflare: providers/cloudflare-staging.yml
  aws:        aws/atlas-dev.yml
```

This points the logical `cloudflare` and `aws` secrets to alternative files
inside the same `~/.devvault/<vault_path>/` tree, only for this workspace.
The catalog stays unchanged — other workspaces continue to resolve them to
the prod files.

### Materialization (`.stack/<product>/devvault.yml`)

`wsp bootstrap` materializes the catalog of every product the workspace
composes into `.stack/<product>/devvault.yml` as a **read-only mirror**. This
gives agents a stable local read path without needing the cache. `wsp doctor`
reports drift between the mirror and the canonical (the stack repo) and
suggests `wsp sync` to refresh.

Edit rules:

- **Canonical** lives in the stack repo (`<product>/agent-stack/devvault.yml`).
  All real edits happen there, then `wsp sync` propagates.
- **Mirror** (`.stack/<product>/devvault.yml`) is read-only. Treat it as
  derived state — the same way you treat `workspace.lock.yml`.

### Anti-pattern

**Editing `.stack/<product>/devvault.yml` to silence a missing-secret error
from `wsp secrets check`.** That mirror is regenerated on the next `wsp sync`
and your edit is gone. The two correct moves are:

- The secret really does live somewhere else on this developer's machine →
  add a `devvault_overrides` entry in `workspace.yml`.
- The secret is genuinely missing from the vault → add the file to
  `~/.devvault/<vault_path>/<relative-path>` (out-of-band, from your password
  manager). NEVER commit it to any repo.

## Bootstrapping a new machine

Once per machine, create:

```yaml
# ~/.devvault/.config.yml
vault_path: "/absolute/path/to/your/devvault"
```

The actual secret files (e.g. `aws/atlas.yml`, `providers/cloudflare.yml`)
are populated from your password manager / 1Password export / Bitwarden.
This is OUT OF SCOPE for AWaC and the CLI — it is the developer's
responsibility per machine.

## Anti-patterns

1. **Hardcoding a secret in any repo file**, even a placeholder like
   `API_KEY="changeme"`. Use `<placeholder>` in committed config files and
   resolve from devvault at runtime.
2. **Committing `~/.devvault/.config.yml` or any file inside `~/.devvault/`
   into a repo.** That directory is per-machine, never versioned.
3. **Putting `vault_path` inside `<product>/agent-stack/devvault.yml`**.
   The catalog is per-product (versioned, equal across devs). The path is
   per-machine. Mixing them is the most common mistake.
4. **Reading a secret value into a chat log or PR description.** The vault is
   the only place secret values may exist on the developer's machine.

## Cross-references

- Schema: `wsp schema devvault`
- CLI verification: `wsp doctor` (machine-level), `wsp secrets check` (per-product, validates each cataloged secret resolves to an existing file).
- Rule global complementaria: `aws_resource_safety.md` (qué cuentas existen).
