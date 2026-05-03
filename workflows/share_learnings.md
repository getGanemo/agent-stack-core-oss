---
description: Turn learnings from the current session into a publishable blog article. Generic content pipeline; the actual publication adapter is provided by your team's blog skill (e.g. `publish_blog_<platform>`). Always requires explicit user validation before publishing.
---

# Share Learnings

Transform learnings from the current session into a blog article. Never
publishes without user validation.

// turbo-all

## Phase 1 — Silent analysis

Extract from the session history:

**a) Technical learnings:** problems solved, architectural decisions, errors
fixed, pitfalls discovered.

**b) Business context:** which product domain was touched, which user
suffers the problem, is there a quantifiable before/after?

**c) Reusable material:** cleaned-up code blocks, comparison tables,
anonymised error excerpts.

**d) WHAT MUST NOT GO IN (security):**

- Credentials, API keys, tokens, passwords
- Internal IPs, EC2 instance IDs, AWS ARNs
- Project-specific paths, customer names, internal decisions
- Personal names of staff members

**e) MANDATORY GENERALISATION:** Every learning must be abstracted to a
general pattern.

| Specific (NO) | Generalised (YES) |
|---|---|
| "We configured `slowapi` in FastAPI" | "Every framework has rate limiting: FastAPI -> slowapi, Express -> express-rate-limit" |
| "We migrated secrets to AWS SSM" | "Use cloud-native secrets: AWS SSM, GCP Secret Manager, Azure Key Vault" |
| "Alice and Bob worked on X" | "The team" or generic first-person plural |

**Abstraction rules:**

1. If you can't generalise without losing the lesson, find an analogous
   example from another stack.
2. Alternative stacks enrich the SEO surface.
3. Speak from experience, not from implementation.

If the session has no substantial technical learnings → report that
honestly to the user.

---

## Phase 2 — Article proposal

Present the user with:

```
## Article proposal

**Central topic:** [one sentence]
**Angle:** Problem -> Solution
**Target audience:** who suffers this problem
**Tone:** deep-technical / mid-technical / accessible
**Estimated length:** short / medium / long

**Proposed structure:**
1. [Section 1]
2. [Section 2]

**Preliminary headline:** [headline following write_blog_copy formulas]

**Material from the session:**
- [code X]
- [decision Y]
- [error Z]
```

---

## Phase 3 — Product to position

Discuss with the user which product (if any) should be lightly positioned
in the article. Apply these placement rules regardless of product:

1. The product is mentioned **only once**, in the second-to-last paragraph.
2. Never urgency, never "this is the only solution".
3. Last paragraph = value, not CTA.

---

## Phase 4 — Confirm with the user (max 4 questions)

```
Before writing, I need to confirm:

1. Do you approve the proposed angle and audience?
2. Do you confirm positioning [product]?
3. Is there anything from the session that MUST NOT appear?
4. Any tweak to the preliminary headline?
```

---

## Phase 5 — Drafting and publication

**5a) Write the article:**
Activate the `write_blog_copy` skill (or your team's equivalent) and write
HTML directly (never markdown). Apply:

- 5 beats per section: Problem -> Empathy -> Explanation -> Solution -> Principle
- Second-person voice ("you")
- Headline using the skill's formulas
- Open with a scenario, not "in this article…"
- Close: checklist + soft CTA + value sentence

**5b) Save locally:**
Save the HTML in `project/blog/<slug>.html`.

**5c) Publish:**
Activate the project's `publish_blog_<platform>` skill (e.g. for Odoo, for
Ghost, for a custom CMS). It typically handles:

- Reading credentials from devvault
- Setting the correct author
- Escaping code blocks
- SEO: meta_title <= 60, meta_description <= 155
- Tag normalisation to avoid duplicates
- Draft -> verify -> publish

---

## Phase 6 — Final report

```
## Article published

**URL:** <published URL>
**Title:** [final title]
**Author:** [author]
**Tags:** [list]
**Length:** [N words]

**SEO applied:**
- Meta title (X/60 chars): [...]
- Meta description (X/155 chars): [...]

**Product positioned:** [Product] — once, in the second-to-last paragraph

**Checks passed:**
- Author correct
- Code blocks escaped
- SEO fields within limits
- No duplicate tags
```

---

## Skills referenced (mandatory)

| Phase | Skill | Purpose |
|---|---|---|
| 5a | `write_blog_copy` | Author content with empathic structure |
| 5c | `publish_blog_<platform>` | Upload to your blog platform with full SEO |

## Inviolable rules

1. **Never publish without user validation** (Phase 4).
2. **Never invent learnings** that did not happen.
3. **Never mention multiple products** in the same article.
4. **Never put the product in the introduction.** Only the second-to-last paragraph.
5. **Never end with a CTA.** Last sentence = value.
6. **Never expose confidential information.**
7. **Never use a specific example without generalising it.**
8. **Never write as an operations diary.** Use "the recommended pattern".
9. **Never skip Phase 6 (the report).**
