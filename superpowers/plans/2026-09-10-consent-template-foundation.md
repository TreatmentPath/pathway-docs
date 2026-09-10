# Consent Template Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Implement FR1-FR3 by removing configurable consent invitation bodies, introducing practice-scoped template-management permission, and supporting superuser-managed global signing templates.

**Architecture:** `Documents.ConsentDocumentTemplate` becomes explicitly practice-owned or global. Existing feature-access machinery authorizes every template mutation, while normal authenticated staff retain read access to active selectable templates. A dedicated global Documents API powers a superuser page that reuses the existing signing-template editor rather than the unrelated `Notes.ConsentTemplate` detection UI.

**Tech stack:** Django 5.1, Django REST Framework, PostgreSQL, React 18, TypeScript, Vite, Vitest.

**Repositories:** Backend changes are committed in `TreatmentPathBackend`; frontend changes are committed in `perfect-pixel-playground-project`; this plan is committed separately in the `docs` repository.

---

## File responsibility map

### Backend

- `TreatmentPathBackend/TreatmentPath/Documents/models.py`: template ownership model and constraint.
- `TreatmentPathBackend/TreatmentPath/Documents/migrations/0012_consent_template_global_ownership.py`: nullable practice, `is_global`, ownership constraint.
- `TreatmentPathBackend/TreatmentPath/UserAuthentication/views/access_control.py`: `consent_templates_manage` definition and role defaults.
- `TreatmentPathBackend/TreatmentPath/Documents/permissions.py`: action-aware signing-template permission class.
- `TreatmentPathBackend/TreatmentPath/Documents/views/template_views.py`: practice/global queryset rules and global CRUD ViewSet.
- `TreatmentPathBackend/TreatmentPath/Documents/urls.py`: global signing-template routes.
- `TreatmentPathBackend/TreatmentPath/Documents/serializers.py`: global provenance fields and removal of `default_email_body` from the API.
- `TreatmentPathBackend/TreatmentPath/Documents/views/signing_views.py`: allow active practice or global templates when snapshotting a request.
- `TreatmentPathBackend/TreatmentPath/Documents/utils/notifications.py`: standard server-owned invitation body.
- `TreatmentPathBackend/TreatmentPath/Documents/tasks.py`: stop passing or consuming custom invitation body values.
- `TreatmentPathBackend/TreatmentPath/Documents/test_template_foundation.py`: FR1-FR3 API, permission, ownership, and sending regressions.
- `TreatmentPathBackend/TreatmentPath/UserAuthentication/tests/test_consent_template_permission.py`: permission declaration/defaults.

### Frontend

- `perfect-pixel-playground-project/src/types/signableDocuments.ts`: canonical template wire/UI types without body default and with global provenance.
- `perfect-pixel-playground-project/src/config/api.ts`: global template endpoints.
- `perfect-pixel-playground-project/src/pages/settings/components/consent/ConsentTemplateEditor.tsx`: remove body-default controls.
- `perfect-pixel-playground-project/src/pages/settings/components/consent/ConsentTemplatesView.tsx`: permission-aware management and reusable global mode.
- `perfect-pixel-playground-project/src/pages/settings/components/consent/ConsentTemplateRow.tsx`: Global badge and read-only actions.
- `perfect-pixel-playground-project/src/components/consent/SendConsentModal.tsx`: remove body prefill/customization and send field.
- `perfect-pixel-playground-project/src/pages/admin/GlobalConsentDocumentTemplates.tsx`: superuser signing-template management page.
- `perfect-pixel-playground-project/src/routes/admin.routes.tsx`: global signing-template route.
- `perfect-pixel-playground-project/src/components/admin/AdminSidebar.tsx`: distinct signing-template entry.
- Existing consent component tests: test removal, permissions, provenance, and global mode.

---

## Task 1: Declare the management permission

- [ ] Add failing backend tests in `UserAuthentication/tests/test_consent_template_permission.py`:

```python
from django.test import SimpleTestCase
from UserAuthentication.views.access_control import FEATURE_DEFINITIONS


class ConsentTemplatePermissionDefinitionTests(SimpleTestCase):
    def test_permission_is_practice_scoped_and_admin_enabled(self):
        definition = FEATURE_DEFINITIONS["consent_templates_manage"]
        self.assertEqual(definition["scopes"], ["practice"])
        self.assertTrue(definition["default_roles"]["practice_admin"])

    def test_permission_is_not_granted_to_other_default_roles(self):
        defaults = FEATURE_DEFINITIONS["consent_templates_manage"]["default_roles"]
        self.assertFalse(defaults["practice_dentist"])
        self.assertFalse(defaults["practice_staff"])
        self.assertFalse(defaults["practice_member"])
        self.assertFalse(defaults["individual"])
```

- [ ] Run the test and confirm RED with `KeyError: 'consent_templates_manage'`:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
source venv/bin/activate
cd TreatmentPath
python manage.py test --keepdb UserAuthentication.tests.test_consent_template_permission
```

- [ ] Add `consent_templates_manage` under the admin group in `FEATURE_DEFINITIONS` with label `Manage consent templates`, practice scope, and only `practice_admin=True`.

- [ ] Re-run the focused test and confirm GREEN.

- [ ] Commit only backend permission files:

```bash
git -C /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend add TreatmentPath/UserAuthentication/views/access_control.py TreatmentPath/UserAuthentication/tests/test_consent_template_permission.py
git -C /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend commit -m "feat: add consent template management permission"
```

## Task 2: Add explicit global template ownership

- [ ] Add failing model tests to `Documents/test_template_foundation.py` proving a practice template is valid, a global template is valid, and the two invalid ownership combinations fail database constraints.

```python
class ConsentTemplateOwnershipTests(TransactionTestCase):
    def test_global_template_has_no_practice(self):
        template = ConsentDocumentTemplate.objects.create(
            title="Global implant consent", body="Body", source_type="text",
            practice=None, is_global=True,
        )
        self.assertIsNone(template.practice_id)

    def test_practice_template_is_not_global(self):
        template = ConsentDocumentTemplate.objects.create(
            title="Local consent", body="Body", source_type="text",
            practice=self.practice, is_global=False,
        )
        self.assertEqual(template.practice_id, self.practice.id)
```

Use `self.assertRaises(IntegrityError)` inside separate `transaction.atomic()` blocks for `(practice=None, is_global=False)` and `(practice=<practice>, is_global=True)`.

- [ ] Run only `ConsentTemplateOwnershipTests` and confirm RED because `is_global` is absent and `practice` is non-nullable.

- [ ] Update the model:

```python
practice = models.ForeignKey(
    "UserAuthentication.Practice",
    on_delete=models.CASCADE,
    related_name="consent_document_templates",
    null=True,
    blank=True,
)
is_global = models.BooleanField(default=False, db_index=True)
```

Add a named `CheckConstraint` expressing `(is_global AND practice IS NULL) OR (NOT is_global AND practice IS NOT NULL)`.

- [ ] Generate migration `0012_consent_template_global_ownership.py` with `makemigrations Documents`. Inspect it to ensure it uses `AlterField`, `AddField`, and `AddConstraint`, with no raw SQL and no data rewrite.

- [ ] Run the focused ownership tests and confirm GREEN.

- [ ] Run `python manage.py makemigrations --check --dry-run` and confirm no uncommitted model state.

- [ ] Commit the model, migration, and tests in the backend repository.

## Task 3: Enforce mutation permission and global isolation

- [ ] Add failing API tests in `Documents/test_template_foundation.py` for these cases:

  - authenticated staff may list and retrieve active practice/global templates;
  - ungranted staff receive 403 for create, PUT, PATCH, archive, restore, and fields;
  - an individually granted staff user can perform all those actions;
  - `include_archived=true` receives 403 without management permission;
  - practice managers cannot mutate global templates or another practice's templates;
  - superusers can CRUD/archive/restore global templates through the global endpoint;
  - non-superusers receive 403 from every global endpoint.

- [ ] Run the new API tests and confirm RED: mutations currently succeed for any authenticated user and global rows are absent.

- [ ] Create `Documents/permissions.py` with an action-aware permission:

```python
class ConsentTemplateAccessPermission(BasePermission):
    management_actions = {
        "create", "update", "partial_update", "archive", "restore", "fields"
    }

    def has_permission(self, request, view):
        if not request.user or not request.user.is_authenticated:
            return False
        if getattr(view, "global_admin", False):
            return request.user.is_superuser
        if view.action == "list" and request.query_params.get("include_archived") == "true":
            return check_user_feature_access(
                request.user, "consent_templates_manage", request.user.current_practice
            )
        if view.action in self.management_actions:
            return check_user_feature_access(
                request.user, "consent_templates_manage", request.user.current_practice
            )
        return True
```

- [ ] Update the practice ViewSet queryset to use `Q(practice=practice, is_global=False) | Q(practice__isnull=True, is_global=True)`. Restrict mutation lookup to practice-owned rows even for users with management permission.

- [ ] Add `GlobalConsentDocumentTemplateViewSet` with `global_admin=True`; force `practice=None`, `is_global=True`, and `created_by=request.user` server-side. Ignore/override ownership fields from request payloads.

- [ ] Register `/documents/global-consent-templates/` detail, archive, restore, and fields routes.

- [ ] Add serializer fields `is_global` and a read-only `scope`/provenance label. Keep ownership fields read-only.

- [ ] Re-run the API tests and confirm GREEN, then run all `Documents.tests.ConsentTemplateFieldsEndpointTests` tests to catch cross-practice regressions.

- [ ] Commit backend permission, ViewSet, URL, serializer, and test changes.

## Task 4: Permit global templates during request snapshotting

- [ ] Add failing send tests proving an active global template can be sent by two practices, while another practice's local template and inactive global template are rejected.

- [ ] Run the tests and confirm RED from `_build_signing_request`'s exact `practice=practice` filter.

- [ ] Extract one named helper in `Documents/views/signing_views.py`:

```python
def selectable_consent_templates(practice, template_ids):
    return ConsentDocumentTemplate.objects.filter(
        Q(practice=practice, is_global=False)
        | Q(practice__isnull=True, is_global=True),
        id__in=template_ids,
        is_active=True,
    )
```

Use it in `_build_signing_request`. Preserve requested template order explicitly rather than depending on database ordering, and keep snapshot behavior unchanged.

- [ ] Run the focused send tests and existing context-scope/placeholder tests; confirm GREEN.

- [ ] Commit backend selection helper and tests.

## Task 5: Remove configurable invitation body from the backend contract

- [ ] Add failing tests proving:

  - detail serialization omits `default_email_body`;
  - create/update ignore or reject `default_email_body` rather than persisting new content;
  - send serialization omits/rejects `custom_email_body`;
  - `build_consent_email_body` produces the standard invitation even when the legacy DB column contains text;
  - template tasks no longer pass custom body values.

- [ ] Run the tests and confirm RED because the serializer and renderer still consume both fields.

- [ ] Remove `default_email_body` from Documents serializers and `custom_email_body` from send serializers/task signatures. Leave the database column in place but mark the model comment as deprecated/inert.

- [ ] Replace the renderer's body selection with a fixed server-owned plain-text invitation using `recipient_display_name`, `practice.name`, and `short_url`. Retain optional subject override and template default subject.

- [ ] Update all task and `_dispatch` call sites so no body argument is accepted or forwarded.

- [ ] Run focused notification/task/send tests plus `Documents` tests and confirm GREEN.

- [ ] Commit the backend FR1 change.

## Task 6: Update canonical frontend types and send flow

- [ ] Add or update frontend tests that assert:

  - the template editor has no `Default Email Body` control;
  - selecting a template never renders a custom message-body textarea;
  - send payloads contain no `custom_email_body`;
  - transformed templates expose `isGlobal` and `scopeLabel` from the API.

- [ ] Run the focused tests and confirm RED against existing controls/payloads.

- [ ] Remove `default_email_body`/`defaultEmailBody` from `ApiConsentTemplate`, `ConsentTemplate`, payload types, transforms, editor state, editor JSX, FormData, and JSON bodies.

- [ ] Remove `customBody`, template-body prefill, the message-body textarea, and `custom_email_body` from `SendConsentModal.tsx`. Keep optional custom subject behavior.

- [ ] Add canonical `is_global` to the wire type and `isGlobal`/`scopeLabel` to the UI type without `as any` casts.

- [ ] Run focused tests, `npm run typecheck`, and `npm run build`; confirm the touched files introduce no errors.

- [ ] Commit frontend type/editor/send changes.

## Task 7: Gate practice template management in the interface

- [ ] Extend `ConsentTemplatesView.test.tsx` and `ConsentTemplateRow.test.tsx` with stable `useFeatureAccessContext` mocks proving:

  - users without `consent_templates_manage` can preview active templates but see no Add/Edit/Archive/Restore actions;
  - granted users see management actions for practice templates;
  - global templates always show a `Global` badge and never show mutation actions in practice mode.

- [ ] Run tests and confirm RED because actions are unconditional and no global badge exists.

- [ ] Read `hasFeature("consent_templates_manage")` once in `ConsentTemplatesView` and pass explicit `canManage`/`readOnly` props to rows. Do not infer authorization from role names.

- [ ] Ensure external `createRequest` signals are ignored when the user lacks permission, covering both Settings and Create-area hosts of the shared component.

- [ ] Render a visible Global badge based on canonical `isGlobal`, and suppress global row mutation actions in practice mode.

- [ ] Run the focused tests and confirm GREEN.

- [ ] Commit frontend permission/provenance UI changes.

## Task 8: Add the global signing-template admin page

- [ ] Add failing tests for a global-mode `ConsentTemplatesView` or `GlobalConsentDocumentTemplates` page proving it calls the global endpoint, allows add/edit/archive/restore, sends no practice ownership fields, and labels the page `Global consent signing templates`.

- [ ] Run the tests and confirm RED because the page and endpoints do not exist.

- [ ] Parameterize `ConsentTemplatesView` with an explicit mode/API adapter rather than duplicating its CRUD logic. Practice mode uses practice endpoints and feature permission; global mode uses global endpoints and trusts the admin route guard plus server superuser enforcement.

- [ ] Add `GlobalConsentDocumentTemplates.tsx`, route `/admin/consent-signing-templates`, and an AdminSidebar entry titled `Consent Signing Templates`. Keep the existing `/admin/consent-templates` entry titled `Consent Detection Templates` so the two domains cannot be confused.

- [ ] Run focused admin/template tests and confirm GREEN.

- [ ] Run `npm run typecheck` and `npm run build`.

- [ ] Commit frontend global-admin page changes.

## Task 9: Slice-level verification

- [ ] Run backend formatting/pre-commit on every touched Python and migration file.

- [ ] Run backend suites:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
source venv/bin/activate
cd TreatmentPath
python manage.py test --keepdb Documents UserAuthentication.tests.test_consent_template_permission
python manage.py makemigrations --check --dry-run
```

- [ ] Run frontend suites:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npm run test:run -- src/pages/settings/components/consent src/components/consent src/pages/admin
npm run typecheck
npm run build
```

- [ ] Search touched code for forbidden type bypasses and legacy body usage:

```bash
rg -n "as any|default_email_body|defaultEmailBody|custom_email_body|customBody" \
  /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/Documents \
  /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project/src/types/signableDocuments.ts \
  /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project/src/components/consent \
  /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project/src/pages/settings/components/consent
```

Expected: only the deliberately retained inert Django model/database column may match; no frontend or runtime email-path matches.

- [ ] Verify representative values end-to-end through serializer/API tests: one practice template and one global template are visible to Practice A; Practice B sees only the global template; legacy body text never appears in generated invitation content.

- [ ] Inspect `git diff --check` and repository statuses. Confirm only intended files changed and no scratch artifacts remain.

- [ ] Complete the careful-coding evidence checklist in the handoff, including every red and green test command.

- [ ] Perform the self-improvement check. This slice has no user correction or novel skill issue at plan time, so no skill-curator dispatch is expected unless implementation reveals one.
