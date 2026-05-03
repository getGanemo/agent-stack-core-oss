---
name: verify_code
description: >
  ACTIVATE before finalizing ANY Python code in an Odoo 19 module, especially after
  writing models, tests, or manifest. Audits for: (1) view_type=='tree' without 'list'
  in _get_view methods; (2) f-string or %-format in cr.execute (SQL injection);
  (3) chart_template_ref in setUpClass (removed Odoo 19); (4) manifest version not
  starting with '19.0.'; (5) uom_po_id field usage (removed Odoo 19); (6)
  currency_data['currency'] in tests (removed — use company_data['currency']);
  (7) detailed_type field usage (removed Odoo 19); (8) domain pipe count errors
  (N conditions need N-1 pipes for OR); (9) post-correction anti-pattern (create then
  write); (10) i18n PO files missing #: reference lines (only #. comments = install
  error in production — every msgid entry MUST have a #: model: or #: code: line).
  MARK AUDIT FAILED on any violation. Do NOT auto-fix complex issues — report
  file + line. Only auto-fix trivially obvious single-line changes.
---


# Verify Code (Static Analysis)

Run this skill to strictly audit the Python codebase (`*.py`) for compliance with project standards and Odoo version rules.

## 1. Odoo 19 Compatibility Checks
*   **View Types**:
    *   **Rule**: `tree` view type is deprecated in favor of `list` for list views, but `tree` is still used for hierarchy views.
    *   **Scan**: Search for `view_type == 'tree'` or `view_type in (..., 'tree', ...)` in `_get_view` methods and tests.
    *   **Audit Failure**: If code checks ONLY for `'tree'` without also accounting for `'list'`.
    *   **Fix**: Update conditions to `view_type in ('tree', 'list')`.

## 2. ORM Usage & Safety
*   **Safe Execution**:
    *   **Scan**: `cr.execute(...)` calls.
    *   **Audit Failure**: Use of f-strings or `%` formatting within the query string (SQL Injection risk).
    *   **Fix**: Use parameterized queries: `cr.execute("... WHERE id = %s", (id,))`.

*   **Domain Validity**:
    *   **Scan**: `search(...)` domains.
    *   **Audit Failure**: Incorrect number of `|` operators for the number of conditions.
    *   **Fix**: N conditions joined by OR need N-1 pipes.

## 3. Deployment & Environment
*   **Module Structure**:
    *   **Audit Failure**: Missing `__manifest__.py` or `__init__.py` in expected directories.
    *   **Audit Failure**: Manifest missing `license`, `maintainer`, or `icon` keys.
    *   **Audit Failure**: Duplicate keys in `__manifest__.py` (e.g., two `icon` entries).
    *   **Audit Failure**: `post_init_hook` using `[(5, 0, 0)]` (Delete All) on shared fields.
    *   **Audit Failure (XML Order)**: XML data files in `data` list inside `__manifest__.py` ordered incorrectly. Files defining parent menus or base views MUST precede files that reference them via `parent="..."` or `<xpath>`. (e.g., `menu_views.xml` must come before files adding submenus).
*   **Version & License (Odoo 19)**:
    *   **Audit Failure**: Manifest `version` not starting with `19.0.` (must be `19.0.X.X.X`).
    *   **Audit Failure**: Manifest `license` set to `Other proprietary` instead of `OPL-1` (required for Odoo 19 Apps Store).


## 4. Test Code Quality
*   **Scan**: `tests/*.py` files.
*   **Audit Failure**: Tests that rely on hardcoded IDs or deprecated methods.
*   **Audit Failure**: Tests checking `invisible` attribute on List/Tree view columns (Must check `column_invisible` for Odoo 19).
*   **Audit Failure**: `setUpClass` using `chart_template_ref` argument (REMOVED in Odoo 19). The base class `AccountTestInvoicingCommon` no longer accepts this parameter.
    *   **Fix**: Remove the argument: `super().setUpClass()` with no kwargs.
*   **Audit Failure**: Product creation using `uom_po_id` field (REMOVED in Odoo 19, unified into `uom_id`).
    *   **Fix**: Remove `uom_po_id` from `product.product` / `product.template` creation dicts.
*   **Audit Failure**: Using `self.currency_data['currency']` (REMOVED in Odoo 19 test base classes).
    *   **Fix**: Use `self.company_data['currency']` instead. If you need a specific currency like PEN, activate it explicitly.

## 5. Core Architecture (Multi-Company Caching)
*   **Scan**: `models/*.py` files.
*   **Audit Failure (Mixin Missing)**: If `_tags_invisible_per_country` is used, the model MUST inherit `l10n.country.filter.mixin`.
*   **Audit Failure (Bad Practice)**: Manually patching `_get_view_cache_key` on ANY model (especially `base`) is strictly forbidden.
*   **Fix**: Use the Mixin strategy documented in `core_architecture.md`.

## 6. Accounting & Financial Logic
*   **Scan**: `models/*.py` involved in Account Moves/Payments.
*   **Audit Failure**: Loop updating `line_ids` (Debit/Credit/Balance) without `with env.context(check_move_validity=False):`.
*   **Audit Failure**: Currency conversion using logic `1.0 / rate` (suggests misunderstanding of Indirect Rate).
*   **Audit Failure (Anti-Pattern)**: "Post-Correction" logic. creating a record and immediately using `write()` to fix values (e.g. `move = create(...); move.line_ids.write(...)`). Must use Pre-Calculation.

## 7. Product Compatibility
*   **Scan**: `models/*.py` and `tests/*.py`.
*   **Audit Failure**: Usage of `detailed_type` without strict version check (Removed in Odoo 19).
*   **Audit Failure**: Usage of `type='product'` (Invalid value).
*   **Fix**: Use `type='consu'` with `is_storable=True` for Storable Products.

## 8. User Groups Compatibility
*   **Scan**: `models/*.py` and `tests/*.py`.
*   **Audit Failure**: Usage of `.groups_id` (likely meant `group_ids`).
*   **Fix**: Correct to `group_ids` for `res.users`.


## 9. i18n / PO File Format (Producción Risk)
*   **Scan**: `i18n/*.po` files in any module that has them.
*   **Audit Failure**: Any `msgid` entry that has ONLY `#.` comment lines and NO `#:` reference line.
    *   `#.` lines are informational comments — Odoo IGNORES them for translation loading.
    *   `#:` lines are the actual indexed references that tell Odoo where to apply each `msgstr`.
*   **Correct pattern** (every entry needs a `#:` line):
    ```po
    #. module: my_module
    #: model:ir.model.fields,field_description:my_module.field_model__fieldname
    msgid "My Label"
    msgstr "Mi Etiqueta"
    ```
*   **`#:` reference types by source:**
    - Field `string=` → `model:ir.model.fields,field_description:mod.field_model__fname`
    - Field `help=` → `model:ir.model.fields,help:mod.field_model__fname`
    - Selection label → `model:ir.model.fields.selection,name:mod.selection__model__field__val`
    - `ir.ui.menu` name → `model:ir.ui.menu,name:mod.xml_id`
    - `ir.actions.act_window` name/help → `model:ir.actions.act_window,name:mod.xml_id`
    - Python `_("...")` → `code:models/file.py:0`
    - XML `string=` attribute → `model_terms:ir.ui.view,arch_db:mod.view_xml_id`
*   **Fix**: Derive the correct `#:` from the patterns above. NEVER ship a PO file where entries only have `#.` comment lines.
*   **Audit Failure (Silent Translation Drop)**: `#, python-format` flag on an entry whose `msgid` has NO `%s` or `%(x)s` placeholders.
    *   Odoo validates format strings at install time — mismatched flags cause the **entire msgstr to be silently discarded**. The message will appear in English in production even though the PO entry looks correct.
    *   **Scan pattern**: grep for `#, python-format` and verify the NEXT `msgid` line contains `%s`, `%(`, or `%d`. If not → remove the flag.
    *   **Correct**: Use `#, python-format` ONLY when the msgid contains at least one Python format placeholder.
    *   **Wrong** (causes silent drop):
        ```po
        #: code:models/project_task.py:34
        #, python-format
        msgid "You must start the timer before adding products."
        msgstr "Debe iniciar el temporizador antes de agregar productos."
        ```
    *   **Correct** (no flag when no placeholders):
        ```po
        #: code:models/project_task.py:34
        msgid "You must start the timer before adding products."
        msgstr "Debe iniciar el temporizador antes de agregar productos."
        ```

*   **Audit Failure (Missing odoo-python Marker — CRITICAL)**: `code:` PO entry missing `#. odoo-python` comment.
    *   Root cause: `translate.py:1855` filters entries by `PYTHON_TRANSLATION_COMMENT in row['comments']` where `PYTHON_TRANSLATION_COMMENT = 'odoo-python'`. Without it, translation is **silently discarded**.
    *   Root cause 2: `translate.py:853` `re.match(r"(module[s]?): (\w+)", entry.comment)` requires `#. module: <name>` in the same entry. Without it → **`AttributeError: 'NoneType'.groups()`** (server crash).
    *   **Both comment lines are REQUIRED for `code:` entries:**
        ```po
        #. module: my_module     ← translate.py:853 regex (extract module name — REQUIRED)
        #. odoo-python           ← translate.py:1855 filter (Python translation — REQUIRED)
        #: code:addons/my_module/models/project_task.py:0
        msgid "Error message"
        msgstr "Mensaje de error"
        ```
    *   **Scan pattern**: for every `#: code:` line, check the PRECEDING `#.` lines. There MUST be BOTH `module: <name>` and `odoo-python`.
    *   **Wrong 1** (silent translation loss — no odoo-python):
        ```po
        #. module: my_module
        #: code:addons/my_module/models/project_task.py:0
        msgid "Error"
        msgstr "Error"
        ```
    *   **Wrong 2** (server crash — no module:):
        ```po
        #. odoo-python
        #: code:addons/my_module/models/project_task.py:0
        msgid "Error"
        msgstr "Error"
        ```
    *   Verified empirically in `addons_nativos/industry_fsm_sale/i18n/ar.po` — native Odoo uses both.


## 10. json.dumps Safety in Model Methods Called During Tests
*   **Scan**: `json.dumps(` in `models/*.py` where the input comes from `resp.json()`.
*   **Audit Failure**: `json.dumps(resp.json().get('key', []))` without type validation. During tests, `resp.json()` may return a Mock object (not a dict), causing `TypeError: Object of type Mock is not JSON serializable`.
*   **Fix**: Validate types before serializing:
    ```python
    data = resp.json()
    if not isinstance(data, dict):
        return
    tokens = data.get('key', [])
    rec.field = json.dumps(tokens if isinstance(tokens, list) else [])
    ```

## 11. Odoo 19 Deprecated Class Attributes (`_sql_constraints`)
*   **Scan**: `_sql_constraints = [` in `models/*.py`.
*   **Audit Failure**: ANY occurrence of `_sql_constraints` as a class attribute in a model. No exceptions — the legacy API is deprecated in Odoo 19.
*   **Why this matters (not cosmetic)**: During module load, the registry emits `WARNING odoo.registry: Model attribute '_sql_constraints' is no longer supported, please define model.Constraint on the model.` This warning is written to `ir.logging` on every Odoo.SH build instance and to `odoo.log`. The Odoo.SH build health detector counts `ir.logging` entries above INFO — a single custom module with `_sql_constraints` is enough to make the build appear as `warning` or `error` in the builds page even when all tests pass. It will also stop working entirely in a future Odoo release.
*   **Fix**: Migrate each `(name, sql, message)` tuple to a separate class attribute `_<name> = models.Constraint(sql, message)`. The attribute name MUST start with `_` (ORM descriptor). No new imports required — `models.Constraint` lives in the already-imported `odoo.models`.
    ```python
    # ❌ WRONG (Odoo ≤18, deprecated in 19)
    class SaasTenant(models.Model):
        _name = 'saas.tenant'

        _sql_constraints = [
            ('slug_unique', 'UNIQUE(slug)', 'Tenant slug must be unique.'),
        ]

    # ✅ CORRECT (Odoo 19+)
    class SaasTenant(models.Model):
        _name = 'saas.tenant'

        _slug_unique = models.Constraint(
            'UNIQUE(slug)',
            'Tenant slug must be unique.',
        )
    ```
*   **Verification after fix**: `grep -r _sql_constraints addons/<module>/` must return empty. The warning disappears from `ir.logging` on the next deploy.
*   **Cross-ref (development)**: `.agents/skills/check_pitfalls/SKILL.md` §25 has the full pitfall context for devs writing new models. This verify-layer check is the safety net in case it was missed during development.

## 12. Test Logger Hygiene (`mute_logger` on Error-Path Tests)
*   **Scan**: `tests/*.py` files for tests that mock exceptions — grep for `side_effect=Exception`, `side_effect=ConnectionError`, `side_effect=Timeout`, or any `side_effect=<ExceptionClass>` pattern inside `patch(...)` or `Mock(...)`.
*   **Audit Warning** (NOT automatic failure — heuristic, false positives possible): For each match, check if the containing test function has a `@mute_logger('odoo.addons.<module>...')` decorator AND whether the code path being tested has a `_logger.warning/error(...)` call. If both are true but the decorator is missing, flag as WARNING for review.
*   **Why this matters**: Tests that mock exceptions trigger error-handling code paths in production code. Those handlers legitimately emit `_logger.warning/error(...)` — behavior we WANT in production for observability. During `--test-enable` runs, however, those warnings are written to `ir.logging` on the build instance. On Odoo.SH, the build health detector counts any entry above INFO, so two unmuted error-path tests are enough to mark a build as `warning` even with `0 failed / 0 error of N tests`.
*   **Fix** (when the warning is legitimately expected during the test):
    ```python
    from odoo.tools import mute_logger

    @mute_logger('odoo.addons.saas_orchestrator.models.saas_instance')
    def test_cron_sync_handles_api_error(self):
        with patch(MOCK_PATH, side_effect=Exception("Connection refused")):
            # The cron calls _logger.warning(...) when catching — that's the
            # desired production behavior, but during this test it would
            # pollute ir.logging on the build instance.
            self.env['saas.instance']._cron_sync_instance_status()
        self.assertEqual(self.instance.state, 'running')
    ```
*   **False positives to accept** (do NOT flag as failure — leave as review item):
    - Test uses `self.assertLogs(...)` to verify the log message explicitly — in that case muting would break the assertion. Leave it alone.
    - The code path under test does NOT call `_logger.warning/error` — nothing to mute.
    - The exception propagates out of the handler (test verifies `assertRaises`) — there's no warning being emitted.
*   **Decorator scope rules** (see `scaffold_test §8` for full reasoning):
    1. One decorator per test, NEVER at class level (would silence unrelated tests).
    2. Logger name must match exactly: `odoo.addons.<module>.<submodule>.<file>` — same as `__name__` of the file containing `_logger`.
    3. Production is unaffected — `tests/` is never imported in runtime without `--test-enable`.
*   **Cross-ref (development)**: `.agents/skills/scaffold_test/SKILL.md` §8 has the development-side rule with all three conditions for applying mute_logger. This verify-layer check is the safety net when that skill was not consulted.

## Execution Guide
1.  Use `grep_search` to find instances of violations.
2.  If violations are found, MARK THE AUDIT AS FAILED until resolved.
3.  Do not auto-fix unless completely trivial; prefer reporting the specific file and line.
4.  For heuristic checks (like §12), report as WARNING with source context — let the user decide whether each instance is a false positive.

