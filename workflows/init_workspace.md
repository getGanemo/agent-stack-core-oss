---
description: >
  Crea un workspace AWaC end-to-end conduciendo el diálogo con el usuario:
  verifica entorno, lista templates disponibles, pregunta cuál elegir y bajo
  qué nombre, corre `wsp init` + `wsp bootstrap`, reporta el resultado. Ejecutar
  SIEMPRE que el usuario diga "creá un workspace", "setea workspace", "init
  AWaC", "iniciá un workspace", "create a workspace using AWaC", o equivalente.
  Pareja con la skill `use_wsp_cli` (cuándo invocar wsp en general) y con el
  workflow `install_wsp` (instalar el CLI si no está).
---

# Init workspace (autonomous setup)

Procedimiento autónomo para que un agente, partiendo de "creá el workspace
usando AWaC", deje al usuario con un workspace bootstrapeado y operativo
sin más fricción que **elegir template + nombre**.

// turbo-all

---

## Step 0 — Si el agente arrancó "a ciegas" (sin `.agents/`)

Si llegaste a un directorio sin `.agents/` ni `workspace.yml` y no tenés
contexto previo (no hay rules cargadas, no sabés qué stacks aplican), antes
de hacer cualquier otra cosa **leé la guía embebida del CLI**:

```bash
wsp guide init
```

`wsp guide <topic>` imprime el path al doc canónico para ese tema (en este
caso, este mismo workflow). Es el bootstrap inverso: cuando todavía no hay
`.agents/` cargados, le pedís al CLI que te diga dónde leer. Después seguí
con Step 1.

(Si `.agents/` ya está cargado en la sesión y este workflow ya está
disponible — saltar Step 0, ir directo a Step 1.)

---

## Step 1 — Verificar pre-requisitos

```bash
wsp --version 2>&1 || echo "WSP_MISSING"
gh auth status  >/dev/null 2>&1 || echo "GH_AUTH_MISSING"
```

| Estado | Acción |
|---|---|
| `wsp` no instalado | Invocar el workflow `install_wsp` (se incluye en core; lo trae cualquier workspace AWaC). Después volver a este Step 1. |
| `gh` no autenticado | `gh auth login --hostname github.com --git-protocol https --web`. Esperar al login. |
| Ambos OK | Avanzar a Step 2. |

Después: `wsp doctor`. Cualquier check `[FAIL]` debe resolverse antes de
continuar (registry inalcanzable, devvault config faltante, governance
mirror divergente). El error trae `remediation` que el agente debe seguir.

---

## Step 2 — Listar templates disponibles

```bash
wsp templates --json
```

Parsear el JSON. Mostrar al usuario una tabla numerada:

```
Templates disponibles:

  1) blank                    — Workspace mínimo. Editá a tu gusto.
  2) research-spike           — Investigación / spike, sin código de producto.
  3) branding                 — Workspace para crear archivo de marca.
  4) empresarial              — Proyecto empresarial generalista.
  5) tesis                    — Proyecto de tesis o investigación académica.
  6) odoo-module              — Desarrollo de un módulo Odoo standalone.
  7) odoo-modules-existing    — Trabajar sobre módulos Odoo ya existentes.
  8) acme-feature        — Feature en Acme (orchestrator + Odoo portal).
  9) widget-feature           — Feature en Widget (mail-api + 2 Odoo modules).
 10) atlas-feature           — Feature en Atlas.
 11) cobalt-feature          — Feature en Cobalt.
 12) delta-feature          — Feature en Prysm:ID.

¿Cuál usar? (número o id, default: blank)
```

Aceptar tanto el número (`1`–`12`) como el id literal (`acme-feature`).
Si el usuario teclea algo que no matchea: re-preguntar mostrando los ids.

`wsp templates --json` ahora trae también, por template:
`requires_confirmation` (bool — `true` para los `*-feature` de producto),
`composes_stacks` (lista — qué stacks compone), y `clones_repos` (lista —
qué product repos cloneará el bootstrap). El agente DEBE inspeccionar estos
campos antes de avanzar al Step 5 (ver el gate de confirmación allá).

---

## Step 3 — Pedir nombre del workspace

Pedir al usuario el nombre. Reglas:

- **kebab-case** (snake_case acepta como fallback). Pattern: `^[a-z][a-z0-9-]+$`.
- Tip al usuario: nombrarlo por la feature que va a trabajar
  (`acme-billing-rework`, no `mi-workspace-3`).
- Si el usuario dice "elegí vos" o algo similar: proponer 2-3 nombres derivados
  del template + un sufijo descriptivo y dejar elegir.

---

## Step 4 — Resolver y validar el directorio destino

Default: `<cwd>/<name>`. Permitir override por pregunta opcional:
"¿dónde lo creo? (default: `./<name>`)".

**Antes de crear**, chequear el destino:

```bash
TARGET="<resolved-path>"
test -e "$TARGET/workspace.yml" && echo "WORKSPACE_EXISTS"
test -d "$TARGET" && [ -n "$(ls -A "$TARGET" 2>/dev/null)" ] && echo "DIR_NOT_EMPTY"
```

| Condición | Acción |
|---|---|
| `WORKSPACE_EXISTS` | El directorio ya tiene un `workspace.yml`. Preguntar: "ya hay un workspace acá. ¿(a) usar el existente y solo correr bootstrap, (b) elegir otro path, (c) cancelar?" Default seguro: opción (b). NO sobrescribir sin ack explícito. |
| `DIR_NOT_EMPTY` (sin workspace.yml) | El directorio tiene contenido pero no es un workspace. Preguntar: "el directorio `<TARGET>` no está vacío. ¿(a) usar otro nombre / path, (b) cancelar?" NUNCA borrar el contenido sin ack explícito letra-por-letra del usuario. |
| Vacío o no existe | Avanzar. |

---

## Step 5 — `wsp init` (con gate de confirmación para templates de producto)

**Antes de invocar `wsp init`**, si el template elegido es de producto
(`requires_confirmation: true` en `wsp templates --json` — típicamente
cualquier `*-feature`), MOSTRAR al usuario el impacto y pedir ack
explícito. No alcanza con que el usuario haya elegido el template en Step 2;
los templates de producto cambian el flujo del producto entero (clonan
repos reales, embeben CLAUDE.md específico, conectan a `project_management`)
y la confirmación es por-deploy-equivalente.

Mostrar:

```
Template `<chosen_id>` es de producto. El init va a:

  - Componer stacks:    <composes_stacks>     (de wsp templates --json)
  - Clonar repos:       <clones_repos>
  - Embeber el flujo:   <product> (CLAUDE.md, project_management, etc.)
  - Path destino:       <TARGET>

¿Confirmás? (sí / no)
```

Si el usuario dice "no" / "cancelar" / cualquier cosa que no sea afirmativa
inequívoca → abortar y volver a Step 2 (re-elegir template). Si confirma:

```bash
wsp init "<name>" --template "<chosen_id>" --target "<TARGET>" --yes
```

(Para templates no-producto — `blank`, `research-spike`, `branding`,
`empresarial`, `tesis`, `odoo-module`, `odoo-modules-existing` — saltar el
gate y correr `wsp init` directamente sin `--yes`.)

Esto escribe `<TARGET>/workspace.yml` (substituyendo `<CHANGE-ME>` por
`<name>`) sin clonar nada. Si retorna error estructurado, leer
`code`/`category`/`cause`/`remediation` y reportar al usuario; no avanzar
a Step 6.

---

## Step 6 — `wsp bootstrap`

```bash
cd "<TARGET>"
wsp bootstrap --json
```

Esperado: el JSON incluye `stacks`, `repos`, `agent_files`, `file_count`,
`collisions`, `lock`. Mostrar al usuario un resumen de 4 líneas:

```
✓ Workspace bootstrapeado en <TARGET>
  Stacks:        N (lista corta)
  Product repos: M clonados
  .agents/:      F archivos compuestos (rules + skills + workflows)
  Lockfile:      <TARGET>/workspace.lock.yml
```

Si `bootstrap` falla:
- `WSP_004 repo_clone_failed` → casi siempre `gh auth` o repo inaccesible. Verificar `gh auth status` y permisos sobre la org del producto.
- `WSP_005 registry_fetch_failed` → registry inalcanzable. Verificar `WSP_REGISTRY_REPO`/`WSP_REGISTRY_BRANCH` y conectividad.
- Otros → seguir `remediation` del error estructurado.

---

## Step 6.5 — Verificar producto (solo si el template es product-specific)

Si el template elegido en Step 2 es `*-feature` (Acme / Widget /
Atlas / Cobalt / Prysm:ID), correr DOS chequeos antes de declarar
"workspace listo":

```bash
wsp secrets check <product> --json
wsp deploy <product> --plan --json
```

| Chequeo | Si falla |
|---|---|
| `wsp secrets check <product>` | Listar al usuario qué secrets faltan en el vault (cada uno con su `logical_name` + `expected_path`). Sugerir poblarlos desde su password manager o, si la falta es por path divergente, agregar un `devvault_overrides` en `workspace.yml` (ver `rules/use_devvault.md` § "Two-layer model + workspace overrides"). NO declarar el workspace "listo" hasta que el usuario decida cómo resolver. |
| `wsp deploy <product> --plan` | Revisar que el plan resuelve sin WSP_019 (`deploy_target_not_available`). Si el workspace tiene `deploy_overrides` y el override apunta a un target fuera de `targets_available`, reportar y pedir corregir antes de hand-off. |

Para templates no-producto (`blank`, `research-spike`, `branding`, etc.)
saltar este step entero — no hay catálogo de secrets ni deploy spec del
que validar.

---

## Step 7 — Hand-off al usuario

Cuando bootstrap pasa, mostrar el path y sugerir el próximo paso según el
template elegido:

| Template | Próximo paso típico |
|---|---|
| `*-feature` (Acme / Widget / Atlas / Cobalt / Prysm:ID) | `cd <TARGET> && code .` (o el editor preferido). El agente lee `CLAUDE.md` y arranca a trabajar la feature. |
| `odoo-module` / `odoo-modules-existing` | Verificar que `addons/` tiene los módulos esperados. Sugerir correr `run_tests` (workflow Odoo) tras la primera edición. |
| `research-spike` / `branding` / `empresarial` / `tesis` | Workspace para trabajo no-producto. Editar `workspace.yml` si querés agregar `aws`, `mcp` u otros stacks ad-hoc; correr `wsp sync` después. |
| `blank` | Workspace minimal. Editar `workspace.yml` para agregar stacks + repos según necesidad; correr `wsp bootstrap` cuando lo cambies. |

Imprimir 2 comandos de seguimiento útiles:

```
Para refrescar tras un cambio de stack upstream:    wsp sync
Para diff lockfile vs estado actual:                wsp status
```

---

## Step 8 — Registrar el outcome (si el producto tiene project_management)

Si el template elegido es de un producto con `<product>/project_management/`
clonado al workspace (acme/widget/atlas/cobalt/delta features),
escribir una entrada `progreso.md` en la fase actual del producto siguiendo
la skill `manage_project_state`:

```
[<fecha>] Workspace inicializado: <name>
- Template: <template_id>
- Path: <TARGET>
- Stacks: <list>
- Initiator: <usuario>
```

Esto cierra el loop con el sistema de roadmap del producto. Para templates
no-producto (blank, research, branding, etc.) saltar este paso.

---

## Anti-patterns

1. **Saltar Step 4** y pasar directo al `wsp init`. Si el directorio tiene
   contenido, `wsp init` falla con `WSP_006` y dejás un trabajo a medias.
2. **Asumir el template** porque el usuario dijo "para Acme" — siempre
   confirmar con el usuario antes de invocar `wsp init` (puede que quiera
   `odoo-module` para tocar un módulo Odoo de Acme en vez del template
   completo de feature).
3. **Borrar contenido del directorio sin ack explícito**. Step 4 es claro:
   nunca borrar sin que el usuario tipee su confirmación letra-por-letra.
4. **Saltar `wsp doctor` post-install**. Si gh auth está rota o el registry
   no resuelve, el bootstrap falla y vos perdés tiempo debuggeando lo que
   doctor te decía en 5 segundos.
5. **Reportar "workspace listo" sin haber verificado el bootstrap**.
   El criterio de éxito es que `wsp bootstrap` devolvió ok + lockfile escrito,
   no solo que `wsp init` pasó.
6. **Saltar el gate de confirmación del Step 5 para templates `*-feature`**.
   Aunque el usuario los haya elegido en Step 2, el impacto (clonar repos
   reales, embeber el flujo del producto) merece un ack explícito a último
   momento. `--yes` solo se pasa después de esa confirmación.
7. **Declarar "workspace listo" sin correr Step 6.5 cuando el template es
   product-specific**. Un workspace de producto sin secrets cargados o con
   un `deploy_overrides` mal configurado va a fallar en la primera tarea
   real; mejor descubrirlo ahora.

---

## Cross-references

- Skill: [`use_wsp_cli`](../skills/use_wsp_cli/SKILL.md) — cuándo invocar `wsp` en general.
- Workflow pareja: [`install_wsp.md`](install_wsp.md) — instalar el CLI si no está.
- Skill: [`manage_project_state`](../skills/manage_project_state/SKILL.md) — Step 8.
- CLI reference: <https://github.com/getGanemo/docs-agentic/blob/main/awac/05-cli-reference.md>.
- Templates registry: [`getGanemo/agent-stack-core/awac.yml#templates`](https://github.com/getGanemo/agent-stack-core/blob/main/awac.yml).
