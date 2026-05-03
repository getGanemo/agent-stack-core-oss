---
trigger: always_on
description: Deploy a service ONLY from the workspace, never from a fresh git clone on the deploy target. Prevents workspace/prod divergence and stale-artifact incidents.
globs: ["**/*"]
---

# Rule: Deploy a service from the workspace ONLY

> **CRITICAL: The workspace is the source of truth for deploys, not GitHub.**

## The problem this prevents

Repeated production incidents have the same root cause: deploying from a
fresh `git clone` (or `git pull`) on the server instead of from the
workspace where the code was actually edited and tested. Symptoms include:

- Server picking up an older revision than the one the developer is looking at
- Build artefacts on the server (e.g. `user_data.sh`, env files) being a
  stale version that lacks recently-added dependencies
- Tests passing locally on the workspace's HEAD while production runs a
  different SHA

## Mandatory deploy sequence

1. **Edit** in the workspace
2. **Run local tests** (the project's `run_*_tests.*` script)
3. **Push files from the workspace -> the deploy target** (SCP via SSM, rsync,
   `kubectl cp`, whatever the project's transport is — but always FROM the
   workspace)
4. **Rebuild on the target** (`docker compose up -d --build` or equivalent)
5. **Verify health** (the project's health endpoint)
6. **Commit + push to GitHub only AFTER** the target is verified healthy

## Never do

- `git clone` the repo on the server and run from there
- `git pull` on the server to update code
- Push files from any source other than the workspace
- Deploy from a CI/CD pipeline that builds from GitHub unless the project has
  a verified parity guarantee

## After every deploy, verify parity

```bash
# Get server md5sums (example for an SSM-managed EC2)
aws ssm send-command ... 'find /opt/<service>/app -name "*.py" -exec md5sum {} +'

# Compare with local
find app -name "*.py" -exec md5sum {} + | sort -k2
```

If any file diverges, the deploy is incomplete. Push the missing files and
rebuild.
