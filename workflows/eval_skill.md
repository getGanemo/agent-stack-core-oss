---
description: Evaluate one or more Antigravity skills inline using their evals/evals.json test cases. Tests behavioral compliance, grades assertions, and reports a pass/fail score without any external API calls.
---

# Eval Skill

Run behavioral evaluation of a skill using its predefined test cases and assertions.

## When to use

- After modifying a `SKILL.md` to verify the change had the intended effect
- When a skill seems to be triggering incorrectly or giving wrong outputs
- Periodically (each Gemini model update) to check if skills are still needed
- When adding new evals to a skill's `evals/evals.json`

## Skill location

```
.agents/skills/<skill-name>/
├── SKILL.md          ← skill instructions
└── evals/
    └── evals.json    ← test cases with assertions
```

## Skills with evals available

The active stacks expose every skill that ships with `evals/evals.json` at
`.agents/skills/<skill-name>/evals/evals.json`. Examples in the universal
core: `verify_code`, `manage_project_state`, `use_wsp_cli`,
`create_repo_readme`, `create_deploy_spec`, `create_skill`. Topical stacks
add their own.

## Usage

Trigger with:

- `/eval_skill <skill-name>` — run all evals for that skill
- `/eval_skill all` — evaluate all skills with evals

## Protocol (execute inline — no external API needed)

// turbo-all

1. **Load skill context**:
   - Read `.agents/skills/<skill-name>/SKILL.md` completely
   - Read `.agents/skills/<skill-name>/evals/evals.json`
   - Confirm: N evals found, list their names

2. **For each eval** (sequential):
   - Read the `prompt` field
   - Mentally simulate: "If I received this prompt with the skill's instructions active, what would I respond?"
   - Generate a response **strictly following the skill's rules** (not general knowledge)
   - Evaluate each `assertion` against the response: PASS / FAIL
   - Record evidence quote (1-2 sentences max) for each assertion

3. **Grading rules**:
   - PASS only with CLEAR evidence the assertion is satisfied
   - Partial compliance = FAIL
   - `"weight": "critical"` assertions failing = entire eval FAIL
   - An eval PASSES only when ALL its assertions pass

4. **Report results**:
   ```
   ══════════════════════════════════════
     SKILL EVAL: <SKILL_NAME>
   ══════════════════════════════════════

   [PASS] Eval 1: <name>
     ✓ [a1] <assertion text>
          → "<evidence quote>"
     ✗ [a2] <assertion text>
          → "Response did not mention X"

   ...

   ══════════════════════════════════════
     SCORE: X/Y assertions  |  N/M evals passed
   ══════════════════════════════════════
   ```

5. **Post-eval actions**:
   - If score ≥ 90%: skill is healthy — no action needed
   - If score 70–89%: note which assertions failed — consider refining `SKILL.md`
   - If score < 70%: skill needs rework — open `SKILL.md` and address failing assertions
   - If score = 100% (on a skill that's "Capability Uplift"): consider deprecating — model may already handle it natively

## Adding new evals

Edit `.agents/skills/<skill-name>/evals/evals.json` following this schema:

```json
{
  "evals": [
    {
      "id": <number>,
      "name": "descriptive-name",
      "prompt": "Realistic user prompt",
      "assertions": [
        {
          "id": "a1",
          "text": "The response does/mentions X",
          "weight": "critical"
        }
      ]
    }
  ]
}
```

Good assertions are:
- Objectively verifiable (yes/no, not "is it good?")
- Testing a specific rule from the SKILL.md
- Realistic — the prompt should be something a real user would say
