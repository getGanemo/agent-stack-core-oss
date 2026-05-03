---
name: manage_project_state
description: >
  USE this skill any time you create, update, query, or audit a product's
  `project_management/` repo. It is the canonical source on how the macro
  state of a product is recorded across roadmap, phases, decisions, and
  chronological log. Activate when: (a) initializing a new product's
  `project_management` repo, (b) writing a phase progress entry, (c)
  recording an architectural decision, (d) updating the repository map after
  any structural change, (e) auditing whether a product's `project_management`
  is well-formed. Do NOT activate to write technical README content of
  individual code repos — that lives in each repo, not in `project_management`.
---

# Manage project state — `project_management` repo skill

Many product setups include a Category-A repo named `project_management`
(snake_case exception). This skill defines its **internal structure** and
**how to keep it healthy**. The repo's existence and obligation are defined
by your team's governance documentation. This skill defines what goes
inside.

## When to use

- **Initialization** of a new product → scaffold `00_Base/` + the first phase folder.
- **End of a working session** with a meaningful outcome → write a `progreso.md`
  entry for the current phase.
- **Architectural decision taken** → write a decision record in
  `00_Base/00_Decisiones_Arquitectura.md` (or split into per-decision files
  once the count exceeds ~10).
- **Repo created, renamed, archived, or transferred** → update
  `00_Base/00_Mapa_Repositorios.md`.
- **Auditing** an existing product's `project_management` → run the audit
  checklist (see "Audit checklist" below).

Skip this skill when:

- The change is internal to a code repo (then it belongs in that repo's README,
  CHANGELOG, or own docs).
- The work is ephemeral and has no decision or outcome worth preserving for
  future sessions.

## The skeleton

Every `<product>/project_management` repo MUST follow this filesystem layout:

```
project_management/
├── README.md                         # one-pager: product, current phase, links
├── 00_Base/                          # foundational, atemporal
│   ├── .project_definition.md        # product definition, vision, scope
│   ├── 00_Roadmap_<Product>.md       # phases, dependencies, target dates
│   ├── 00_Mapa_Repositorios.md       # all repos, URLs, purpose, stack
│   ├── 00_Arquitectura_Sistema.md    # architectural decisions affecting
│   │                                 # category C — what's monorepo, what's
│   │                                 # polyrepo, why
│   ├── 00_Decisiones_Arquitectura.md # ADRs (or split into per-decision
│   │                                 # files when count > 10)
│   ├── 00_Convenciones.md            # product-specific conventions that
│   │                                 # extend (not contradict) team governance
│   └── 00_Glosario.md                # product-specific terminology
├── Fase_<NN>_<NombreCorto>/          # one folder per phase in the roadmap
│   ├── progreso.md                   # CHRONOLOGICAL LOG, append-only
│   ├── plan.md                       # what this phase will deliver
│   ├── decisiones.md                 # phase-local decisions (smaller than
│   │                                 # the architectural ones in 00_Base)
│   └── <subdir or asset as needed>/
└── _archive/                         # phases closed and frozen — kept for
    ↳ reference, never edited again
```

Notes:

- **`00_Base/` is atemporal.** Nothing dated lives there — only living, current
  truth. If `.project_definition.md` changes, you change it in place; you don't
  add a "v2" copy.
- **Phase folders are temporal.** `progreso.md` is append-only chronologically.
  `plan.md` may be edited as the phase evolves; `decisiones.md` is append-only
  once a decision is recorded.
- **`_archive/` is frozen.** When a phase closes, move its folder there and
  treat it as immutable. Do not delete.

## File contracts

### `README.md` (root)

One-pager. Sections:

```markdown
# <Product> — project_management

> Macro state of the <Product> product. Internal-only. Public docs at <link to docs site>.

## Product in one line
<one-line product description>

## Current phase
- **<Fase NN_NombreCorto>** — see [Fase_<NN>_<NombreCorto>/progreso.md](./Fase_<NN>_<NombreCorto>/progreso.md)
- Target close: <YYYY-MM-DD>

## Shortcuts to canonical docs
- [Roadmap](./00_Base/00_Roadmap_<Product>.md)
- [Repository map](./00_Base/00_Mapa_Repositorios.md)
- [System architecture](./00_Base/00_Arquitectura_Sistema.md)
- [Architectural decisions](./00_Base/00_Decisiones_Arquitectura.md)
- [Product definition](./00_Base/.project_definition.md)
```

### `00_Base/.project_definition.md`

Atemporal. Captures:

- **Product**: name, brand, what it does in one paragraph.
- **Audience**: who pays, who uses, who decides.
- **Value proposition**: why this exists.
- **Scope**: in / out / future.
- **Business model**: how it makes money.
- **Constraints**: technical, legal, brand.

### `00_Base/00_Roadmap_<Product>.md`

Phases in order with dependencies and tentative dates. Each phase entry has:

- **Name**: `Fase_<NN>_<NombreCorto>`.
- **Summary**: 1–3 lines.
- **Deliverables**: bulleted concrete outcomes.
- **Dependencies**: which previous phases must close first.
- **State**: `pending | in_progress | closed`.
- **Dates**: start / target close (real close goes in the phase's own folder).

### `00_Base/00_Mapa_Repositorios.md`

**This file is the per-product mirror of the team's governance.** It lists
every repo of this product (any category) plus every cross-org repo. For each:

| Repo | URL | Category | Stack | Branch default | Visibility | Description |
|---|---|---|---|---|---|---|

If the product has cross-org modules (e.g. shared modules in another org),
list them here so cross-references between this product and the shared org
are traceable from one place.

### `00_Base/00_Arquitectura_Sistema.md`

Architecture decisions that shape the product code repos: monorepo vs
polyrepo, layer separation vs audience separation, primary stack, why. This
is the file the governance "applying to a new product" step requires before
creating product-code repos.

### `00_Base/00_Decisiones_Arquitectura.md`

ADR-lite log. Each decision:

```markdown
## ADR <NNN>: <title>

- **Date:** YYYY-MM-DD
- **State:** proposed | accepted | rejected | superseded by ADR <NNN>
- **Context:** what forced the decision
- **Decision:** what we decided
- **Consequences:** what it implies, what it forecloses
- **Alternatives considered:** brief
```

When count exceeds ~10, split into per-decision files in
`00_Base/decisiones/<NNNN>-<slug>.md` and keep the index in `00_Decisiones_Arquitectura.md`.

### `Fase_<NN>_<NombreCorto>/progreso.md`

**Append-only chronological log.** Format:

```markdown
# Phase <NN>: <NombreCorto> — progress

## YYYY-MM-DD — <short title of entry>
<what happened, what was decided, what was tried, links to commits/PRs>

## YYYY-MM-DD — ...
```

Entries are dated, ordered by date ascending. Latest entry on the bottom.
Never edit a past entry; correct in a new dated entry that supersedes.

### `Fase_<NN>_<NombreCorto>/plan.md`

What this phase will deliver. Editable as the phase evolves. Three sections:

- **Objectives** — concrete, verifiable.
- **Deliverables** — checklist; checked when delivered.
- **Risks / dependencies** — what could derail it, what it depends on.

### `Fase_<NN>_<NombreCorto>/decisiones.md`

Decisions whose scope is the phase, not the product architecture. Smaller
than the ADRs in `00_Base/`. Same template, lighter expectation.

## Initializing a new product's `project_management`

When the team's governance "create a new product" step creates the
`project_management` repo, scaffold this minimum:

```
project_management/
├── README.md                                    # filled with the placeholder
├── 00_Base/
│   ├── .project_definition.md                   # filled, even if rough
│   ├── 00_Roadmap_<Product>.md                  # at least Fase_01 listed
│   ├── 00_Mapa_Repositorios.md                  # with the Category A repos
│   ├── 00_Arquitectura_Sistema.md               # placeholder if not yet decided
│   ├── 00_Decisiones_Arquitectura.md            # empty index
│   ├── 00_Convenciones.md                       # placeholder
│   └── 00_Glosario.md                           # placeholder
└── Fase_01_<NombreCorto>/
    ├── plan.md
    ├── progreso.md                              # one entry: "phase started"
    └── decisiones.md                            # empty
```

Commit message: `Initial scaffold of project_management for <product>`.

## Recording session outcomes

After any meaningful working session on the product, append a `progreso.md`
entry under the current phase. Minimum useful entry:

```markdown
## YYYY-MM-DD — <one-line summary>

- What was done: <bullets>
- Outcomes: <commits, PRs, decisions>
- Pending: <bullets>
- Next: <one-liner>
```

If the session produced an architectural decision, **also** add the ADR in
`00_Decisiones_Arquitectura.md`. If it produced a structural repo change
(create/rename/archive/transfer), **also** update `00_Mapa_Repositorios.md`.

Do NOT batch a week of work into one entry. Frequent small entries are
healthier than infrequent large ones.

## Closing a phase

When a phase's `Deliverables` checklist is complete:

1. Append a final `progreso.md` entry with: `## YYYY-MM-DD — Phase NN closed`,
   summary of outcomes, and the link to the next phase.
2. Update `00_Roadmap_<Product>.md`: set `State: closed`, real close date.
3. Move the phase folder to `_archive/Fase_<NN>_<NombreCorto>/` (one git mv).
4. Open the next phase folder.

Once archived, the folder is frozen — never edited again.

## Audit checklist

Use when auditing an existing `project_management` for compliance with this
skill:

- [ ] `README.md` exists, points to current phase.
- [ ] `00_Base/` has the 7 expected files (or their split equivalents).
- [ ] `.project_definition.md` is non-empty and current.
- [ ] `00_Mapa_Repositorios.md` lists every repo of the product, including
      any cross-org repos.
- [ ] At least one phase folder exists.
- [ ] Each phase folder has `plan.md`, `progreso.md`, `decisiones.md`.
- [ ] `progreso.md` is chronological and ends with a recent entry (≤ 30 days
      since last meaningful work, or marks the phase closed).
- [ ] `_archive/` exists and only contains closed phases.
- [ ] No technical README content of code repos lives here (those belong in
      the code repos themselves).

Audit output goes into the next `progreso.md` entry as a date-stamped section
listing gaps and follow-ups.

## Anti-patterns

- Writing technical README content here instead of in the code repo.
- Editing past `progreso.md` entries instead of appending new dated ones.
- Creating phase folders in advance of the roadmap (only the **current** phase
  needs a folder; future phases live in the roadmap).
- Skipping `00_Mapa_Repositorios.md` updates when a repo is created/renamed
  (this is the file your team's cross-org traceability relies on).
- Letting `00_Base/.project_definition.md` go stale when scope changes.
- Mixing cross-cutting team policies into here that should live in your team's
  central governance documentation rather than per-product.

## Cross-references

- Governance: `<your-governance-doc>` (your team's product-structure document).
- AWaC CLI: the workspace CLI consumes `agent-stack/awac.yml#repos`, which is
  in turn driven by what's listed in `00_Mapa_Repositorios.md`.
