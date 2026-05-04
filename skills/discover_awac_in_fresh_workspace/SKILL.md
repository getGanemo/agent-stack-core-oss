---
name: discover_awac_in_fresh_workspace
description: >
  USE this skill when an agent arrives in a directory that has NO `workspace.yml`
  AND NO `.agents/`, AND the user instruction asks to "set up a workspace",
  "configure this folder", "iniciá un agent workspace acá", "armá AWaC", "use
  wsp", or any equivalent that implies bootstrapping AWaC from zero. This is
  the canonical entry path for zero-context setup: the agent has no rules
  loaded yet, the folder is empty (or close to it), and the only signal is
  the user instruction. Activate to (a) verify `wsp` is installed, (b) read
  the embedded guide via `wsp guide discover`, (c) inventory templates with
  their semantic metadata (composes_stacks, clones_repos, requires_confirmation),
  (d) ask the user — never infer template from folder name, (e) gate product
  templates on explicit confirmation, (f) run `wsp init` + `wsp bootstrap`,
  (g) verify product secrets after bootstrap. Do NOT activate when there is
  already a `workspace.yml` (use `wsp sync` / `wsp status` instead) or when
  `.agents/` is loaded (use the `init_workspace` workflow directly).
---

# Discover AWaC in a fresh workspace

This skill is the **zero-context entry point** for an agent that lands in an
empty (or near-empty) directory with the user asking for a workspace, agent
rules, or AWaC. At this point the agent has no `.agents/` loaded, no
`workspace.yml` to read, and no project memory to lean on. Don't guess —
discover, ask, then act.

## When to use

Triggers (must match all three):

1. The cwd has **no `workspace.yml`**.
2. The cwd has **no `.agents/` directory**.
3. The user instruction mentions one or more of: "workspace", "agent
   workspace", "agent rules", "AWaC", "wsp", "configure this folder",
   "armá esto con AWaC", "iniciá un workspace", "set up a Ganemo workspace".

## When NOT to use (anti-triggers)

- The cwd already has a `workspace.yml` → workspace exists; use `wsp sync` /
  `wsp status` / the appropriate skill instead.
- The cwd has `.agents/` loaded with the `init_workspace` workflow available
  → run that workflow directly (it covers the same flow with stack-loaded
  context).
- The user is asking a conceptual question about AWaC ("what is wsp?") →
  link to the spec, do not bootstrap.
- The cwd is inside an existing product repo (you see `.git/` pointing to a
  product remote) → wrong place to create a workspace; warn the user.

## Intent table

| User intent | Step in this skill |
|---|---|
| "Configurá esto con AWaC" / "set up this folder as a Ganemo workspace" | run all steps in order |
| "Use wsp para armar el workspace acá" | run all steps in order |
| "Quiero empezar una feature de Acme acá" | run all steps; product-template gate at Step 5 is critical |
| "What templates can I use?" (no setup yet) | run Steps 1–3, then ask |
| "Sólo quiero leer las reglas de AWaC" | Step 2 only (`wsp guide discover`), then stop |

## Step 1 — Verify `wsp` is on PATH

```bash
wsp --version 2>&1 || python -m wsp --version 2>&1 || echo "WSP_MISSING"
```

| Resultado | Acción |
|---|---|
| Imprime una versión (e.g. `wsp 0.10.0`) | Avanzar a Step 2. |
| `WSP_MISSING` | Invocar el workflow autónomo de instalación: leer y seguir `workflows/install_wsp.md` (vive en este mismo stack `agent-stack-core`). NO pedirle al usuario que instale a mano salvo que el workflow falle y lo indique. Después volver a Step 1. |

## Step 2 — Read the embedded discovery guide

```bash
wsp guide discover
```

`wsp guide <topic>` imprime el path al doc canónico embebido para `<topic>`.
Para `discover` devuelve el path a este flujo de bootstrap (este mismo
SKILL.md, mirroreado en el CLI). Leer el archivo que indique antes de
invocar cualquier comando que mute estado. Es el escape-hatch para cuando
todavía no hay `.agents/` cargado y el agente necesita el procedimiento.

## Step 3 — Inventario de templates con metadata semántica

```bash
wsp templates --json
```

Cada entrada del JSON trae:

| Campo | Significado |
|---|---|
| `name` | id del template (ej. `acme-feature`). |
| `description` | una línea humana. |
| `requires_confirmation` | `true` si el template es product-specific (`*-feature`). Disparador del gate del Step 5. |
| `composes_stacks` | lista de stacks que se componen al hacer bootstrap. |
| `clones_repos` | lista de product repos que `wsp bootstrap` clonará a este workspace. |

Mostrar al usuario una tabla con AL MENOS `name`, `description`,
`requires_confirmation`, y un resumen de `clones_repos` (porque ese es el
impacto más visible — repos reales clonados a su disco).

## Step 4 — Pedirle al usuario qué template usar

**No inferir el template del nombre del folder.** El nombre del directorio es
una coincidencia: que se llame `acme-billing/` no significa que el
usuario quiere un workspace `acme-feature`; puede que sólo quiera
`odoo-module` para tocar uno de los módulos de Acme sin embeber todo
el flujo del producto.

Preguntar literalmente:

```
¿Qué template usamos? Opciones (de wsp templates --json):

  - blank                   (workspace mínimo)
  - research-spike          (investigación)
  - odoo-module             (un módulo Odoo standalone)
  - acme-feature       (feature en Acme: clona orchestrator + portal_odoo + ...)
  - widget-feature          (feature en Widget: clona mail-api + 2 módulos Odoo + ...)
  - <etc.>

(Tu respuesta determina qué se clona y qué flujo se embebe — no asumimos
nada por el nombre del directorio.)
```

Esperar respuesta explícita. Si el usuario dice "elegí vos" → proponer 2-3
opciones razonables CON su impacto y volver a preguntar.

## Step 5 — Confirmación reforzada para templates de producto

Si el template elegido tiene `requires_confirmation: true` (todo `*-feature`),
re-confirmar **antes** de invocar `wsp init`. Citá explícitamente:

```
Confirmemos: vas a iniciar un workspace `<chosen>` en `<cwd>`. Esto va a:

  - Componer estos stacks:    <composes_stacks>
  - Clonar estos repos:       <clones_repos>     (¡repos reales a tu disco!)
  - Embeber el flujo de:      <product>          (CLAUDE.md, project_management, etc.)
  - Habilitar deploys de:     <product>          (sujeto a sus gates de approval)

¿Confirmás? (sí / no)
```

Sólo después del "sí" inequívoco usar `--yes` en Step 6. Para templates
no-producto (`blank`, `research-spike`, `branding`, `empresarial`, `tesis`,
`odoo-module`, `odoo-modules-existing`) saltar este gate y correr `wsp init`
sin `--yes`.

## Step 6 — Init + bootstrap

```bash
# Para template no-producto:
wsp init "<name>" --template "<chosen>"

# Para template de producto (después de Step 5):
wsp init "<name>" --template "<chosen>" --yes

# Después, en cualquier caso:
cd "<name>"
wsp bootstrap --json
```

Esperar JSON con `stacks`, `repos`, `agent_files`, `file_count`, `lock`. Si
algo falla, leer `code`/`category`/`cause`/`remediation` del error
estructurado y reportar al usuario antes de tirar el siguiente paso.

## Step 7 — Verificar producto si el template fue product-specific

Si el template fue `*-feature`:

```bash
wsp secrets check <product> --json
```

Reportar al usuario el estado:

- **Todos los secrets resuelven** → workspace operativo. Reportar éxito y
  pasar el control.
- **Faltan secrets** → listar cada uno (`logical_name` + `expected_path`)
  y proponer las DOS resoluciones posibles: (a) poblar los archivos en el
  vault desde el password manager, (b) si la falta es por path divergente,
  agregar `devvault_overrides` en `workspace.yml` (ver `rules/use_devvault.md`).
  NO declarar el workspace "listo" hasta que el usuario decida cómo seguir.

Para templates no-producto este step se saltea entero.

## Anti-patterns

1. **Inferir el template del nombre del folder.** Es una coincidencia.
   Templates tienen impacto semántico (clonan repos, embeben flujos de
   producto). Siempre preguntar al usuario.
2. **Saltar el gate del Step 5 para `*-feature`.** "El usuario lo eligió en
   Step 4" no alcanza — el impacto (repos clonados, flujo embebido) merece
   un ack explícito a último momento.
3. **Pasar `--yes` antes de la confirmación humana del Step 5.** Ese flag
   bypassa la validación; usarlo sin ack es exactamente lo que el flag
   pretende NO permitir por accidente.
4. **Declarar "workspace listo" sin Step 7 cuando el template fue
   product-specific.** Un workspace de producto sin secrets va a fallar en
   la primera tarea real; mejor descubrirlo durante el setup.
5. **Editar `.agents/` o `.stack/<product>/*` a mano para arreglar algo del
   bootstrap.** Es estado derivado: se regenera con `wsp sync` y tu edit
   desaparece. Editar la fuente canónica (stack repo o `workspace.yml`).
6. **Correr `wsp init` cuando el cwd tiene contenido sin pedir confirmación.**
   El CLI te frenará con `WSP_006`, pero el orden correcto es: chequear
   destino primero, preguntar al usuario si querés override, recién después
   invocar `wsp init`.

## Cross-references

- Workflow: [`init_workspace`](../../workflows/init_workspace.md) — el flujo
  paso-a-paso completo (esta skill es el wrapper de descubrimiento que se
  activa cuando ese workflow todavía no está cargado).
- Workflow: [`install_wsp`](../../workflows/install_wsp.md) — instalación
  autónoma del CLI cuando Step 1 detecta `WSP_MISSING`.
- Skill: [`use_wsp_cli`](../use_wsp_cli/SKILL.md) — uso general del `wsp`
  CLI una vez que el workspace existe.
- Rule: [`use_devvault`](../../rules/use_devvault.md) — modelo de dos
  capas + `devvault_overrides` (Step 7).
- Rule: [`use_deploy_spec`](../../rules/use_deploy_spec.md) — modelo de dos
  capas + `deploy_overrides` + `targets_available`.
