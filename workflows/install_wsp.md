---
description: >
  Installs (or upgrades) the `wsp` CLI on the user's machine without manual
  intervention. Resolves dependencies (Python, pipx, git, gh), downloads the
  wheel of the latest Release from the configured `workspace-cli` repo,
  installs it via pipx, and verifies with `wsp doctor`. Run this whenever a
  `wsp` command fails with "command not found", the user says "install wsp",
  "I don't have the CLI", "upgrade wsp", or when the agent detects that
  `wsp --version` does not respond.
---

# Install or upgrade `wsp`

Autonomous procedure to leave the `wsp` CLI operational on the user's machine. **Do not ask for confirmation between steps**: the only decision that requires input is whether the user wants a specific version (default: the latest published).

// turbo-all

---

## Step 1 — Detect current state

Run the checks in parallel and record what's missing:

```bash
wsp --version 2>&1 || echo "WSP_MISSING"
python --version 2>&1 || python3 --version 2>&1 || echo "PYTHON_MISSING"
pipx --version 2>&1 || echo "PIPX_MISSING"
gh auth status 2>&1 || echo "GH_AUTH_MISSING"
git --version 2>&1 || echo "GIT_MISSING"
```

Decide the path:

| State | Action |
|---|---|
| `wsp` already installed and up to date | Jump to Step 6 (verification). |
| `wsp` installed but out of date | Jump to Step 5 with `pipx install --force`. |
| `wsp` missing, dependencies OK | Step 4. |
| `pipx` missing | Step 2 first. |
| `python` missing | Step 2.5 — abort and tell the user to install Python ≥ 3.10 (not safe to install automatically without permission). |
| `gh` or `gh auth` missing | Step 3. |

---

## Step 2 — Install `pipx` if missing

`pipx` isolates each Python CLI in its own venv and is the idiomatic way to distribute Python tools.

| Platform | Command |
|---|---|
| Windows (winget) | `winget install pypa.pipx --accept-source-agreements --accept-package-agreements` |
| Windows (manual) | `python -m pip install --user pipx && python -m pipx ensurepath` |
| macOS | `brew install pipx && pipx ensurepath` |
| Linux (apt) | `sudo apt install -y pipx && pipx ensurepath` |
| Any OS with Python | `python -m pip install --user pipx && python -m pipx ensurepath` |

After `pipx ensurepath`, `~/.local/bin` (or its Windows equivalent) is on PATH, but **the current bash session doesn't see it yet**. Reopen the shell or export manually:

```bash
# For the current session (Git Bash / Linux / macOS)
export PATH="$HOME/.local/bin:$PATH"
# Native Windows
export PATH="$APPDATA/Python/Python311/Scripts:$PATH"
```

Verify: `pipx --version`. If it still fails, abort and tell the user to restart the terminal.

---

## Step 3 — Configure `gh` if missing

`wsp` clones private repos (the registry, each product's `agent-stack`, the product code repos themselves). Without `gh auth`, `git clone` fails with 403.

```bash
# Install
winget install --id GitHub.cli           # Windows
brew install gh                           # macOS
sudo apt install -y gh                    # Linux apt

# Login (interactive — the only human step in the whole recipe)
gh auth login --hostname github.com --git-protocol https --web
```

If the user already has `GH_TOKEN` exported, login isn't necessary; check `gh auth status` afterward.

**Don't proceed unless `gh auth status` ends with `Logged in to github.com`.** Without that, the next step fails.

---

## Step 4 — Download the wheel of the latest Release

Resolve the latest published tag and download the wheel. Use the
`workspace-cli` repo your team has forked (or the canonical AWaC `workspace-cli`
if you're consuming it directly):

```bash
WSP_REPO="${WSP_RELEASE_REPO:-<your-org>/workspace-cli}"
LATEST_TAG=$(gh release list --repo "$WSP_REPO" --limit 1 --json tagName --jq '.[0].tagName')
echo "Latest wsp release: $LATEST_TAG"

mkdir -p /tmp/wsp-install
gh release download "$LATEST_TAG" \
  --repo "$WSP_REPO" \
  --pattern '*.whl' \
  --dir /tmp/wsp-install \
  --clobber
```

If the user asked for a specific version, use that tag instead of `LATEST_TAG`.

If `gh release list` returns empty (no releases yet — very rare), fall back to dev install:

```bash
# Fallback: editable clone
git clone --quiet "https://github.com/$WSP_REPO" /tmp/wsp-source
pipx install -e /tmp/wsp-source --force
```

---

## Step 5 — Install (or replace) via `pipx`

```bash
WHEEL=$(ls /tmp/wsp-install/wsp-*.whl | head -1)
pipx install "$WHEEL" --force
```

`--force` covers both the upgrade and the reinstall case; if the installed version is exactly the same, `pipx` replaces the isolated venv without touching anything else.

---

## Step 6 — End-to-end verification

```bash
wsp --version
wsp doctor
```

Expected:

```
wsp, version 0.X.Y
[ok] git_present: ...
[ok] gh_present: ...
[ok] cache_dir_writable: ~/.wsp/cache
[ok] registry_reachable: <your-org>/agent-stack-core@main (N shortcuts, M templates)
[ok] governance_mirror: awac.yml <-> <your-governance-doc> aligned
```

Any `[FAIL]` check must be resolved before closing the workflow:

| Failure | Typical cause | Fix |
|---|---|---|
| `git_present` FAIL | `git` not on PATH | Install git: `winget install Git.Git` / `brew install git` / `apt install git`. |
| `gh_present` FAIL | `gh` not on PATH | Back to Step 3. |
| `cache_dir_writable` FAIL | Permissions on `~/.wsp/` | `chmod -R u+rw ~/.wsp/` (Unix) or take ownership (Windows). |
| `registry_reachable` FAIL with 403 | invalid `gh auth` | `gh auth refresh -s admin:org`. |
| `registry_reachable` FAIL repo not found | `WSP_REGISTRY_REPO` mis-set | `unset WSP_REGISTRY_REPO` or set it to your fork. |
| `governance_mirror` FAIL | `awac.yml` diverges from the governance doc | Not blocking for install; mention it and continue. Resolve via `wsp governance check` later. |

---

## Step 7 — Report to the user and clean up

Report one line with the final version (e.g. `wsp 0.4.0 installed and operational`). Remove the temp directory:

```bash
rm -rf /tmp/wsp-install /tmp/wsp-source
```

Done. The user can run `wsp init`, `wsp bootstrap`, `wsp scaffold-stack`, etc. immediately.

---

## Idempotency

This workflow is safe to run multiple times. Each step checks state before acting:

- Step 1 detects whether it's already installed.
- Steps 2/3 are no-ops if pipx/gh are already present.
- Step 4 overwrites the downloaded wheel.
- Step 5 with `--force` replaces the existing pipx venv.

Re-invoking the workflow with the CLI already up to date ends at Step 1 → Step 6 without touching anything.

---

## Cross-references

- CLI source: the `workspace-cli` repo (your fork or the canonical AWaC one).
- Releases: the GitHub Releases page of the same repo.
- Companion skill: `use_wsp_cli` (when to invoke `wsp` once it's installed).
- Spec: the public AWaC spec gist.
