---
trigger: always_on
description: Resolve secrets via the per-product devvault catalog + per-machine vault config. Never hardcode credentials, API keys, tokens, or PEM files in any repo.
globs: ["**/*"]
---

# Use DevVault

Every product has a **catalog of secrets it needs to operate**. The catalog
is versioned per-product. The actual secret values live ONLY on the
developer's machine, in `~/.devvault/`. Never write credentials, API keys,
tokens, OAuth client secrets, PEM files, or session cookies into any repo.

## Two-layer model

| Layer | File | Per | Versioned? |
|---|---|---|---|
| **Catalog** (logical name → relative path) | `<product>/agent-stack/devvault.yml` | product | yes |
| **Vault path** (where on disk) | `~/.devvault/.config.yml` | machine | NO |
| **Secret values** | files inside `~/.devvault/<vault_path>/<relative-path>` | machine | NO |

The catalog declares "this product needs an `aws` secret at `aws/<product>.yml`,
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
3. **Putting `vault_path` inside `<product>/agent-stack/devvault.yml`.**
   The catalog is per-product (versioned, equal across devs). The path is
   per-machine. Mixing them is the most common mistake.
4. **Reading a secret value into a chat log or PR description.** The vault is
   the only place secret values may exist on the developer's machine.

## Cross-references

- Schema: `wsp schema devvault`
- CLI verification: `wsp doctor` (machine-level), `wsp secrets check` (per-product, validates each cataloged secret resolves to an existing file).
- Companion universal rule: `aws_resource_safety.md` (which AWS accounts the project owns).
