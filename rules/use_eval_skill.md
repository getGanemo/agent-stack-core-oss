---
trigger: always_on
description: Activar /eval_skill para evaluar comportamiento de skills contra sus evals/evals.json. Se activa cuando el usuario pide evaluar, testear o auditar una skill.
globs: ["**/*"]
---

# Use Eval Skill — Activation Rule

> **Activate `/eval_skill` when the user asks to evaluate, test, or audit a skill's behavior.**

## Activation Conditions

### Explicit triggers (user asked directly)

Activate when the user uses any of these phrases:
- "evalúa la skill X" / "evaluate skill X"
- "evalúa el skill X" / "testa la skill X"
- "prueba si la skill X funciona"
- "corre los evals de X" / "run the evals for X"
- "¿está funcionando bien la skill X?"
- "verifica la skill X" / "check skill X"
- `/eval_skill` slash command (any argument)

### Implicit triggers (infer the need)

Activate when ALL of these are true:
1. A `SKILL.md` was just modified or created
2. The user asks "¿funciona?" / "¿está bien?" / "how does it look?"
3. Context implies skill quality is in question

### Periodic audit trigger

Activate when the user mentions:
- "auditoría de skills" / "skill audit"
- "revisar todas las skills" / "review all skills"
- "actualización del modelo" / "model update" + "skills"

## How to identify WHICH skill to evaluate

| User says | Skill to evaluate |
|---|---|
| Mentions a skill by name | That skill |
| Mentions a topic (e.g. "tests pipeline", "code audit", "native source inspection") | The matching skill in the active stacks (whose `description:` triggers on that topic) |
| Says "all" / "todas" | All skills that have `evals/evals.json` |
| Ambiguous | Ask: "Which skill do you want to evaluate?" |

## Mandatory protocol

1. **READ** `.agents/workflows/eval_skill.md` before executing — it defines the full inline protocol
2. **LOCATE** the skill: `.agents/skills/<skill-name>/SKILL.md` and `evals/evals.json`
3. **EXECUTE** the workflow steps defined in `eval_skill.md` (inline, no external API)
4. **REPORT** the score using the format defined in the workflow

## Skills with evals available

The composed workspace lists every skill that ships with `evals/evals.json` at
`.agents/skills/<skill-name>/evals/evals.json`. Run `/eval_skill all` to
exercise all of them, or `/eval_skill <name>` to target one.

## After evaluation

- Score ≥ 90%: report "Skill healthy ✓"
- Score 70–89%: report failing assertions + suggest SKILL.md improvements
- Score < 70%: open SKILL.md and propose specific rewrites
- Score 100% on a Capability Uplift skill: suggest testing the skill WITHOUT its SKILL.md in context to check if the model already handles it natively — if yes, recommend deprecating the skill
