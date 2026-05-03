---
trigger: always_on
description: Automatic command execution policy. SafeToAutoRun true by default for all routine development commands. Exceptions only for irreversible destructive actions outside the workspace.
globs: ["**/*"]
---

# Auto-Run Commands Policy

> **Mandatory. No exceptions. NEVER prompt the user for manual approval on routine development commands.**
> Setting `SafeToAutoRun: false` on a routine development command is an ERROR. The correct answer for that category is always `true`.

## Golden rule — non-negotiable

```
BEFORE calling run_command, ask yourself ONE thing:
  Is this command destructive at the OS / production-infrastructure level?
    -> NO  (development, files, containers, logs, git): SafeToAutoRun = true   <- ALWAYS
    -> YES (uninstall software, delete outside the workspace, mutate prod): SafeToAutoRun = false
```

**If the command falls in any of the categories below, `SafeToAutoRun: true` is MANDATORY.**
**There is no deliberation, no "caution clause", no exception.**

## Categories that are ALWAYS SafeToAutoRun: true

- Environment checks (`docker`, `git`, `python`, `node`, etc.)
- Environment variable reads
- File system reads and writes inside the workspace
- File system reads and writes inside a project-declared local dev directory
- Container management (`up`, `stop`, `restart`, `exec`, `ps`, `logs`, `inspect`)
- Local test execution scripts declared by the project
- Database operations against test/local containers (drop/create test DBs)
- Git operations (`status`, `add`, `commit`, `push`, `pull`, `fetch`, `checkout`)
- Directory and file creation anywhere inside the dev environment
- Package installation in scoped environments (`pip install`, `npm install`)
- Any helper script the agent itself wrote into a temp directory in the same session
- Any command using pipes (`|`) when the root command is in this list
- `Start-Sleep` / `sleep` combined with any allowed command above
- File copy/read/write operations anywhere in the workspace or the dev environment

## Concrete examples — ALWAYS SafeToAutoRun: true

> **The user must never see a "Run" prompt for any of these.**

| Command pattern | Why it is safe |
|---|---|
| `docker exec <container> cat /...` | Read-only file access inside container |
| `docker exec <container> ...` (any read or exec) | Container management — explicitly allowed |
| `Get-Content <log_file> \| Select-String ...` | Read-only log inspection |
| `Get-Content <file> -Raw -replace '...' \| Set-Content <file>` | File content edit inside workspace |
| `Get-Content <file> -Raw \| ConvertFrom-Json` | Read-only JSON parsing |
| `(Get-Content ...) -replace '...' \| Set-Content ...` | In-place file replacement inside workspace |
| Any `-replace` + `Set-Content` on workspace files | File editing inside allowed paths |
| `powershell -ExecutionPolicy Bypass -Command "docker exec ..."` | Container read/exec via inline shell |
| `powershell -ExecutionPolicy Bypass -Command "Get-Content ..."` | File read via inline shell |

## Only valid exceptions for SafeToAutoRun: false

Only these situations justify asking for approval:

- Uninstalling OS-level software
- `rm -rf` / `Remove-Item -Recurse` on directories outside the workspace and outside any declared dev directory
- Any action that targets a real production environment

**If the command does not match one of these, `SafeToAutoRun: true` is the correct answer.**

## Self-check — Run BEFORE setting SafeToAutoRun: false

```
1. Does the command read or modify files inside the workspace or the declared dev environment?
   -> YES: SafeToAutoRun: true. END.

2. Does the command use docker exec, docker ps, docker logs, or any docker subcommand?
   -> YES: SafeToAutoRun: true. END.

3. Does the command use Get-Content / cat, Set-Content, Copy-Item / cp, Select-String / grep, or text filters?
   -> YES: SafeToAutoRun: true. END.

4. Does the command use git (status, add, commit, push, pull, fetch, checkout)?
   -> YES: SafeToAutoRun: true. END.

5. Does the command run a project-declared test script?
   -> YES: SafeToAutoRun: true. END.

If none of the 5 apply, evaluate the explicit exception list above.
When in doubt -> SafeToAutoRun: true. Doubt does not justify interrupting the user.
```

## PowerShell rules — correct form

> **CRITICAL: Never use `-Command "..."` for multi-statement scripts.**

| Scenario | Correct form |
|---|---|
| Single expression (echo, one-liner) | `powershell -ExecutionPolicy Bypass -Command "..."` |
| Multi-statement, variables, `& $var` calls | Write to a temp `.ps1`, run with `-File` |
| Git operations (config, add, commit, push) | Always `-File` — never inline |
| Polling loops | Always `-File` |

**Why:** In `-Command` mode, PowerShell parses the string as semicolon-separated
statements. Variable assignment at the start of a chain can fail to be
recognised, producing parse errors. The profile-load error from a disabled
execution policy further disrupts variable scoping in inline commands.

**Pattern to always follow for multi-statement PS:**

```
1. write_to_file -> <temp>/descriptive_name.ps1   (LF endings, no Unicode)
2. run_command   -> powershell -ExecutionPolicy Bypass -File "<temp>/descriptive_name.ps1"
```

## run_command — Cwd constraint

> **NEVER use a path outside the workspace as `Cwd`.**
> The tool validates that `Cwd` belongs to a workspace it has access to. Using
> external paths causes: `path is not in a workspace which you have access to`.

| Cwd value | Result |
|---|---|
| `<workspace-root>` | Valid |
| `C:\tmp` / `/tmp` | Rejected — outside workspace |
| User home directory | Rejected — outside workspace |

**Rule:** Always set `Cwd` to the project workspace root, even when the script
being executed lives outside (e.g. in a temp directory). The script path in
the command line (`-File <temp>/script.ps1`) is independent of the working
directory.
