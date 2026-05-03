---
name: create_agent_rule
description: USE before creating any new Agent Rule (.md in .agents/rules/). Ensures the rule (a) lives in the correct stack repo per the transversal-vs-product test, (b) is named with a product suffix when it could collide with a transversal rule, (c) has the strict frontmatter (trigger/description/globs) required by VS Code's Open Agent Manager, (d) is written with LF endings. NEVER create a rule directly in a workspace's .agents/rules/ — that gets blown away by `wsp sync`.
---

# Create Agent Rule

Use this skill whenever the user asks to create a new Rule for the agent. Rules live in `.agents/rules/` of every workspace, but they are **composed from stack repos** by `wsp bootstrap` / `wsp sync` — so the source of truth is a stack repo, not the workspace.

---

## 1. Decide which stack repo the rule lives in (do this FIRST)

Editing a rule directly inside a workspace's `.agents/rules/` is wrong — that directory is regenerated from the source stacks on every `wsp sync` and your edit will be blown away. The rule must live in a stack repo and reach workspaces via composition.

### Apply the transversal-vs-product test

From `agent-stack-core/awac.yml#org_scaffold.transversal_vs_product`:

1. **Could another product in your portfolio use this rule as-is?**
2. **Does the rule codify a team or technology convention rather than a product-specific one?**
3. **If two products needed it, would two copies be a mistake?**

Any "yes" → **transversal**. All "no" → **product-specific**.

### Map the answer to a stack repo

| Scope | Stack repo | Trigger to use this scope |
|---|---|---|
| **Universal** | `<your-org>/agent-stack-core/rules/<name>.md` | Rule is about agent behavior, AWaC, security guardrails. Examples already there: `anti_prompt_injection.md`, `aws_resource_safety.md`, `auto_propagate.md`, `capture_learnings.md`. |
| **Topical transversal** | `<your-org>/agent-stack-<topic>/rules/<name>.md` (topic ∈ aws, mcp, cloudflare, research, …) | Rule applies whenever this technology is used, across products. |
| **Cross-org by ecosystem** | the stack of that ecosystem org. **Example: any Odoo development rule goes in `<ecosystem-org>/agent-stack/rules/<name>.md`.** | Rule is about how to work with the ecosystem itself, regardless of which product consumes it. |
| **Product-specific** | `<product>/agent-stack/rules/<name>.md` | Rule encodes something true ONLY for this product. |

### Naming: when to add a product suffix

Add `_<product>` suffix to the filename when **either** is true:

- A transversal rule with the same base name already exists or could plausibly exist (e.g. `aws_resource_safety_<product>.md` extends and coexists with the universal `aws_resource_safety.md` from core — last-stack-wins composition would otherwise overwrite the universal one silently).
- The base name is generic enough that a reader seeing the file alone would not know which product it belongs to (`init_project_<product>.md` vs an ambiguous `init_project.md` in `<product>/agent-stack/rules/`).

Skip the suffix only when the file name *already* references a product-specific entity (`use_orchestrator_api.md`, `use_ssm_deploy_<product>.md` — though even there an explicit suffix is fine).

> **Rationale:** `composer.compose_agents` applies last-stack-wins on identical filenames. A product rule that accidentally shadows a universal rule with the same name will silently replace it in workspaces, with only a `collisions[]` entry in the bootstrap report. Suffixes prevent the silent shadowing.

### Default when unsure

Conservative default: **product-specific**. Promotion to transversal later is cheap; retraction after two products depend on a transversal rule is expensive.

---

## 2. Root Cause: Why Rules Fail to Appear

VS Code's **Open Agent Manager** detects rule files using a strict YAML parser.
Two conditions will prevent a rule from appearing in the Rules panel:

| Problem | Symptom | Root Cause |
|---|---|---|
| `CRLF` line endings | File exists but not listed | Windows editors save `\r\n` by default |
| Missing `trigger:` field | File exists but not listed | The panel requires this field in the frontmatter |

> **CRITICAL:** Always write rule files with **LF** (`\n`) line endings — never CRLF.
> The `write_to_file` tool produces LF by default. **Always use it** to create or overwrite rule files. Never use a Windows text editor to create them.

---

## 3. Correct Rule File Format

Every rule file MUST have these three fields in the YAML frontmatter:

```markdown
---
trigger: always_on
description: <One-line summary of what this rule enforces.>
globs: ["**/*"]
---

# Rule Title

Rule content in Markdown...
```

### Field Reference

| Field | Required | Value | Notes |
|---|---|---|---|
| `trigger` | ✅ Yes | `always_on` | Without this, the panel ignores the file |
| `description` | ✅ Yes | String | Shown in the panel tooltip |
| `globs` | ✅ Yes | Array of glob patterns | `["**/*"]` for all files; be specific when possible |

### Common `globs` patterns

| Scope | Value |
|---|---|
| All files | `["**/*"]` |
| Python + XML | `["**/*.py", "**/*.xml"]` |
| Python only | `["**/*.py"]` |
| Python + XML + JS | `["**/*.py", "**/*.xml", "**/*.js"]` |
| Python + CSV | `["**/*.py", "**/*.xml", "**/*.csv"]` |

---

## 4. File Location

All rules MUST be authored in their **source stack repo** (decided in Section 1):

```
<stack-repo>/rules/<rule_name>.md
```

Inside any workspace, after `wsp bootstrap` or `wsp sync`, the same rule appears at `.agents/rules/<rule_name>.md` (composed from its source stack — DO NOT edit this composed copy; the next sync rewrites it).

Use **snake_case** for the filename. The filename becomes the rule's identifier.

---

## 5. Step-by-Step: Creating a New Rule

1. **Decide the host stack repo** — apply Section 1's transversal-vs-product test. Open it locally; clone if missing.
2. **Determine the rule name** — `snake_case`, descriptive, e.g. `use_mcp_tools`. Add `_<product>` suffix if needed (Section 1).
3. **Determine the glob scope** — be specific (avoid `**/*` unless the rule is truly universal).
4. **Write the file using `write_to_file`** with `Overwrite: false` at `<stack-repo>/rules/<name>.md` — this guarantees LF endings.
5. **Commit + push** to the stack repo. Any workspace running `wsp sync` (or `wsp bootstrap`) picks the rule up automatically — workspaces never receive rules pushed only to their own `.agents/rules/`.
6. **Verify** it appears in VS Code → Open Agent Manager → Rules panel after a *Reload Window* (`Ctrl+Shift+P` → Reload Window) in any workspace that consumes that stack.

### Template (copy-paste ready)

```markdown
---
trigger: always_on
description: <Short description of what this rule enforces.>
globs: ["**/*.py", "**/*.xml"]
---

# Rule Title

> **Summary directive in one line.**

## Section 1

- Bullet point...

## Section 2

- Bullet point...
```

---

## 6. Fixing Existing Rules with CRLF

If existing rule files are not appearing in the panel, they likely have CRLF endings. Fix them by **rewriting** each file with `write_to_file` (Overwrite: true), preserving the original content but adding `trigger: always_on` to the frontmatter.

**Do NOT** use PowerShell `(Get-Content) | Set-Content` — it may re-introduce CRLF.

---

## 7. Verification Checklist

After creating or fixing a rule, confirm:

- [ ] File is in `<stack-repo>/rules/` (NOT directly in a workspace's `.agents/rules/`)
- [ ] The chosen stack repo matches the transversal-vs-product test from Section 1
- [ ] Filename has `_<product>` suffix if it could collide with a transversal rule (Section 1)
- [ ] Frontmatter has `trigger: always_on`, `description:`, and `globs:`
- [ ] File was written with `write_to_file` tool (guarantees LF)
- [ ] Stack repo commit pushed to origin
- [ ] After `wsp sync` in a consuming workspace, rule appears in VS Code Open Agent Manager → Rules panel after Reload Window
