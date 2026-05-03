---
name: create_repo_readme
description: >
  USE before creating a new product repo on GitHub, or when auditing/improving
  the README of an existing one. Each repo MUST have a README answering: what
  does this repo do, what stack, who consumes it, how do I run it locally,
  where is the deploy. Different repo categories have different
  required-section checklists (encoded in the wsp CLI). NEVER hand-author a
  product repo README without running this skill — it ensures the per-category
  convention is met. Activate when: (a) creating a new product repo, (b) the
  user mentions a repo without README or with a stub, (c) reviewing a product
  for governance compliance and noting missing/poor READMEs.
---

# Create / audit product repo README

A repo without a useful `README.md` is **governance debt** (your team's
"description conventions" extends to the README itself per AWaC v1). This
skill defines the convention and points at the canonical CLI command that
enforces it.

---

## 1. Scope: which READMEs this skill covers

| Type | Covered by this skill? |
|---|---|
| Stack repo READMEs (e.g. `<org>/agent-stack`, `agent-stack-core`) | NO — those are AWaC-owned and `wsp scaffold-stack` generates/updates them. |
| Product repo READMEs (governance Categories A/B/C/D — `infrastructure/`, `web/`, `docs/`, `<service>/`, `platform/`, `api/`, `mcp-server/`, `*-starter/`, etc.) | Yes — this skill. |
| Cross-org ecosystem modules (e.g. shared add-ons consumed by multiple products) | Yes — this skill (briefer template). |
| Sub-folder READMEs inside a product repo (e.g. `<service>/api/v2/README.md`) | NOT covered — left to the team's discretion. |

If you need a stack repo README, use `wsp scaffold-stack <org>` instead — it
already generates the right thing.

---

## 2. The convention by category

Each category has a **required-sections checklist** that the CLI enforces.
The exact list is encoded in the `wsp` source (`wsp/scaffold_repo_action.py`)
— this skill states the intent.

### Category A — governance and operations (`project_management`, `agent-stack`, `infrastructure`)

Required: **Purpose · Structure · Usage · Cross-references**. Cross-references
MUST link to your team's governance doc and the product's `project_management`.
For `infrastructure/` add a "Deployment / Apply" section with the AWS account
ID and Terraform workflow. For `agent-stack/` mention that it is composed by
`wsp`.

### Category B — public surface (`web`, `docs`, `docs-users`, `docs-developers`)

Required: **Purpose · Public URL · Stack · Local development · Cross-references**.
Recommended: Build / Deployment. Public URL must be the actual `<product>.com`
or `docs.<product>.com`.

### Category C — product code (`<service>`, `platform`, `api`, `admin`, …)

Required: **Purpose · Stack · Architecture role · Primary consumers · Local
development · Tests · Deployment · Cross-references**. "Primary consumers" is
the Category-C-specific section: per governance, a Cat C repo's description
must declare its consumer (especially for backend-only repos consumed by a
companion frontend or third-party app). Treat this section as non-negotiable.

### Category D — optional components (`mcp-server`, `terraform-modules`, `*-starter`, `*-build`)

Required: **Purpose · Stack · Usage · Cross-references**. For starters
(public, the only exception to the private rule) "Usage" doubles as the
public "Quick start" — write it as if a developer outside your team were
reading it.

### Cross-org modules (e.g. `<ecosystem-org>/<module>`)

Required: **Producer product · Manifest · Cross-references**. Briefer than
the other categories because the module's manifest already declares name,
version, dependencies, licence. Cross-references must include the producer
product's `<product>/agent-stack/awac.yml#repos` so traceability between the
producer product and the cross-org module is anchored from one place.

---

## 3. Decision flow

When the user says "create the `<product>/<repo>` repo" or "audit the
READMEs of these repos":

```
                ┌────────────────────────┐
                │ Repo exists on GitHub? │
                └──────┬──────────┬──────┘
                       │ no       │ yes
                       ▼          ▼
        ┌────────────────────┐   ┌──────────────────────────────┐
        │ wsp scaffold-repo  │   │ wsp scaffold-repo --update   │
        │ <full> --category X│   │ <full> --category X          │
        │                    │   │                              │
        │ Creates the repo,  │   │ Audits live README. If FAIL, │
        │ pushes seed README │   │ opens PR appending missing   │
        │ to main.           │   │ required sections preserving │
        │                    │   │ existing content.            │
        └────────────────────┘   └──────────────────────────────┘
```

### Manual fallback (no CLI available)

If `wsp` isn't installed yet (run the `install_wsp` workflow first), or if
the user explicitly wants to write the README by hand:

1. Decide the category. Apply your team's "repository categories" governance.
2. Use the matching template from § 2 above as a checklist.
3. Open a PR; reviewer checks that all required sections exist and aren't empty.

---

## 4. Anti-patterns

1. **Empty / one-line README**: `# <service>` and nothing else. Audit
   marks `too_short` (< 200 chars) and fails. Fix it.
2. **Generic README copy-pasted from another repo**: kills the
   "primary consumers" + "architecture role" specificity. Each repo's role
   is unique enough that copy-paste is wrong.
3. **README pointing at the product's `project_management` for everything**:
   the README must stand alone enough that a contributor opening only this
   repo can run + test + deploy. `project_management` is the product-wide
   bird's-eye view — not a substitute for repo-level docs.
4. **Sub-folder READMEs duplicating the repo-level README's content**:
   sub-folders should answer "what's in THIS folder", not repeat the repo
   summary.
5. **Hand-written README skipping the convention because "this repo is
   different"**: the convention has its categories precisely because repos
   differ. If yours doesn't fit any category, that's a governance
   conversation, not a README convention escape.

---

## 5. Cross-references

- Governance: `<your-governance-doc>` (your team's product-structure document)
- CLI command source: `wsp/scaffold_repo_action.py` (templates + audit checklist live here — single source of truth)
- Companion skill: `use_wsp_cli` (when to use `wsp` in general)
- Companion workflow: `install_wsp` (provision the CLI on a fresh machine)

---

## 6. Verification checklist

Before considering a repo's README done:

- [ ] Category decided.
- [ ] `wsp scaffold-repo <full> --category <X>` ran (either creating or `--update` audit) and reported **PASS**.
- [ ] All `<TODO: …>` placeholders the CLI inserted have been replaced with real content.
- [ ] Cross-references link to the live governance + product `project_management`.
- [ ] For Cat C: the "Primary consumers" section is filled in concretely.
- [ ] For Cat D starters: the "Usage / Quick start" is friendly to an external developer.
- [ ] For cross-org modules: cross-reference to the producer's `agent-stack/awac.yml#repos` is present.
