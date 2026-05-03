---
trigger: always_on
description: Help messages, alert texts, and field help strings must describe the REAL process (trigger -> editable state -> final effect), not a simplified or inaccurate end-state summary.
globs: ["**/*"]
---

# Write Accurate Help Messages

> **NEVER describe behavior as fixed or final if it is actually editable or provisional.**
> Always describe the FULL process: what triggers it, what the user can change, and what the final effect is.

## The core rule

When writing any user-facing text (info banners, field `help=`, button tooltips,
translation strings), ask:

1. **Is this describing an end-state that the user can actually change?**
   -> If yes: describe it as a DEFAULT or STARTING POINT, not as a guaranteed outcome.

2. **Is there a multi-step process behind this field/button?**
   -> If yes: describe each step briefly (trigger -> editable window -> effect).

3. **What happens in the edge case (empty, zero, none)?**
   -> Always document the edge-case behavior explicitly.

## Anti-pattern (what caused this rule)

```
WRONG: describes the end-state as fixed, ignores editability

When the linked record is created, access will be granted only to the items
listed above. If the list is empty, no access will be created.
```

**Why it's wrong:** the list is pre-loaded but editable before the action
fires. "Granted only to the items listed above" implies it is locked, which
is false.

## Correct pattern

```
RIGHT: describes trigger -> editable state -> effect -> edge case

When the linked record is created from the source action, the items listed
above are pre-loaded by default into the new record.
You can review and edit them before triggering the sync.
The sync will grant access to whichever items remain in the list at that
moment.
If the list is empty, no items will be pre-loaded
(you can still add them manually on the record).
```

## Checklist before finalizing any help text

- [ ] Does the text describe a **process** (not just a result)?
- [ ] Is the text accurate even if the user edits the data before the action fires?
- [ ] Is the edge case (empty / none / zero) documented?
- [ ] Does the text avoid words like "will be granted only" / "will always" unless truly invariant?
- [ ] Is the description complete enough that a new user understands **what to do** and **what will happen**?

## When this rule applies

- Info/alert blocks in templates or views
- `help=` attributes on model fields
- Wizard descriptions, button confirm messages
- Any translation entry that describes a UI behavior
