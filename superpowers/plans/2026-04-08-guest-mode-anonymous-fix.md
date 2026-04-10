# Guest Mode Anonymous Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix guest mode so it logs in as a truly anonymous identity (never exposing the superuser's real name), and ensure all `created_by` fields across the codebase render "System Admin" when the creator is a superuser.

**Architecture:** A single shared utility function `get_display_name_for_user` is added to a common backend utils module. All serializers that render `created_by` names are updated to call it. This function returns `"System Admin"` when the user is a superuser, and the real name otherwise. On the frontend, the `GuestAccess.tsx` page is updated to build a neutral guest identity from the token response rather than using the superuser's personal details.

**Tech Stack:** Django REST Framework (Python), React/TypeScript, simplejwt for JWT tokens.

---

## File Map

### Backend — files to modify

| File | What changes |
|---|---|
| `TreatmentPathBackend/TreatmentPath/TreatmentPath/utils.py` | **CREATE** — shared `get_display_name_for_user()` helper |
| `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers.py` | Update 7 `get_created_by` methods to use shared helper |
| `TreatmentPathBackend/TreatmentPath/activityLog/serializers.py` | Update 3 `get_created_by_name` methods |
| `TreatmentPathBackend/TreatmentPath/messaging/serializers.py` | Update 4 `get_created_by_name` methods |
| `TreatmentPathBackend/TreatmentPath/Labs/serializers.py` | Update 1 `get_created_by_name` method |
| `TreatmentPathBackend/TreatmentPath/Assets/serializers.py` | Update 3 `get_created_by_name` methods |
| `TreatmentPathBackend/TreatmentPath/compliance/serializers.py` | Update `get_user_display_name` helper (6 methods call it) |
| `TreatmentPathBackend/TreatmentPath/HR/serializers.py` | Update `get_user_full_name` helper (Appointments uses it too) |
| `TreatmentPathBackend/TreatmentPath/Appointments/serializers.py` | Update local `get_user_full_name` helper |
| `TreatmentPathBackend/TreatmentPath/Stock/serializers.py` | Update `get_created_by_name` method |

### Frontend — files to modify

| File | What changes |
|---|---|
| `perfect-pixel-playground-project/src/pages/GuestAccess.tsx` | Build neutral guest user identity — no superuser name/email exposed |

---

## Task 1: Create shared backend utility

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/TreatmentPath/utils.py`
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_utils.py`

The `TreatmentPath/TreatmentPath/` directory is the Django project root (same level as `settings.py`). Check it exists:

```bash
ls TreatmentPathBackend/TreatmentPath/TreatmentPath/
```

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_utils.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock
from TreatmentPath.utils import get_display_name_for_user


class GetDisplayNameForUserTests(TestCase):

    def _make_user(self, user_type, first="Jane", last="Smith", email="jane@example.com"):
        user = MagicMock()
        user.user_type = user_type
        user.first_name = first
        user.last_name = last
        user.email = email
        return user

    def test_superuser_returns_system_admin(self):
        user = self._make_user("superuser")
        self.assertEqual(get_display_name_for_user(user), "System Admin")

    def test_practice_admin_returns_full_name(self):
        user = self._make_user("admin")
        self.assertEqual(get_display_name_for_user(user), "Jane Smith")

    def test_dentist_returns_full_name(self):
        user = self._make_user("dentist")
        self.assertEqual(get_display_name_for_user(user), "Jane Smith")

    def test_staff_returns_full_name(self):
        user = self._make_user("staff")
        self.assertEqual(get_display_name_for_user(user), "Jane Smith")

    def test_no_name_falls_back_to_email(self):
        user = self._make_user("admin", first="", last="")
        self.assertEqual(get_display_name_for_user(user), "jane@example.com")

    def test_none_user_returns_none(self):
        self.assertIsNone(get_display_name_for_user(None))
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test TreatmentPath.tests.test_utils -v 2
```

Expected: `ImportError` or `ModuleNotFoundError` — `TreatmentPath.utils` doesn't exist yet.

- [ ] **Step 3: Create the utility module**

Create `TreatmentPathBackend/TreatmentPath/TreatmentPath/utils.py`:

```python
def get_display_name_for_user(user):
    """
    Return the display name for a user.
    Superusers are shown as 'System Admin' to preserve anonymity in guest sessions.
    """
    if not user:
        return None
    if getattr(user, "user_type", None) == "superuser":
        return "System Admin"
    first = getattr(user, "first_name", "") or ""
    last = getattr(user, "last_name", "") or ""
    full_name = f"{first} {last}".strip()
    return full_name or getattr(user, "email", None)
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPath.tests.test_utils -v 2
```

Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/TreatmentPath/utils.py \
        TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_utils.py
git commit -m "feat: add get_display_name_for_user utility — superusers show as System Admin"
```

---

## Task 2: Update TreatmentPlan serializers (7 locations)

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers.py`

All 7 `get_created_by` / inline name renders in this file. Add the import at the top of the file, then update each method.

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_created_by_serializers.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock, patch


def _make_superuser():
    u = MagicMock()
    u.user_type = "superuser"
    u.first_name = "Super"
    u.last_name = "Admin"
    u.email = "super@system.com"
    return u


def _make_practice_user():
    u = MagicMock()
    u.user_type = "admin"
    u.first_name = "Alice"
    u.last_name = "Jones"
    u.email = "alice@practice.com"
    return u


class PatientSerializerCreatedByTest(TestCase):
    def test_superuser_created_by_shows_system_admin(self):
        from TreatmentPlan.serializers import PatientSerializer
        patient = MagicMock()
        patient.created_by = _make_superuser()
        s = PatientSerializer()
        self.assertEqual(s.get_created_by(patient), "System Admin")

    def test_practice_user_created_by_shows_name(self):
        from TreatmentPlan.serializers import PatientSerializer
        patient = MagicMock()
        patient.created_by = _make_practice_user()
        s = PatientSerializer()
        self.assertEqual(s.get_created_by(patient), "Alice Jones")

    def test_no_created_by_returns_none(self):
        from TreatmentPlan.serializers import PatientSerializer
        patient = MagicMock()
        patient.created_by = None
        s = PatientSerializer()
        self.assertIsNone(s.get_created_by(patient))
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_created_by_serializers -v 2
```

Expected: Tests for superuser FAIL — currently returns `"Super Admin"` not `"System Admin"`.

- [ ] **Step 3: Add import to TreatmentPlan/serializers.py**

At the top of `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers.py`, find the existing imports block and add:

```python
from TreatmentPath.utils import get_display_name_for_user
```

- [ ] **Step 4: Update get_created_by (patient) — line ~382**

Find:
```python
    def get_created_by(self, patient):
        """Return the full name of the user who created this patient"""
        if patient.created_by:
            return f"{patient.created_by.first_name} {patient.created_by.last_name}"
        return None
```

Replace with:
```python
    def get_created_by(self, patient):
        """Return the display name of the user who created this patient"""
        return get_display_name_for_user(patient.created_by)
```

- [ ] **Step 5: Update to_representation inline render (treatment plan) — line ~1674**

Find:
```python
            "created_by": (
                f"{instance.created_by.first_name} {instance.created_by.last_name}"
                if instance.created_by
                else None
            ),
```

Replace with:
```python
            "created_by": get_display_name_for_user(instance.created_by),
```

- [ ] **Step 6: Update get_created_by (intake) — line ~2350**

Find:
```python
    def get_created_by(self, intake):
        """Return the full name of the user who created this intake"""
        if intake.created_by:
            return f"{intake.created_by.first_name} {intake.created_by.last_name}"
        return None
```

Replace with:
```python
    def get_created_by(self, intake):
        """Return the display name of the user who created this intake"""
        return get_display_name_for_user(intake.created_by)
```

- [ ] **Step 7: Update get_created_by (nurture) — line ~2855**

Find:
```python
    def get_created_by(self, nurture):
        """Return the full name of the user who created this nurture"""
        if nurture.created_by:
            return f"{nurture.created_by.first_name} {nurture.created_by.last_name}"
        return None
```

Replace with:
```python
    def get_created_by(self, nurture):
        """Return the display name of the user who created this nurture"""
        return get_display_name_for_user(nurture.created_by)
```

- [ ] **Step 8: Update get_created_by (ArchiveSerializer) — line ~3029**

Find:
```python
    def get_created_by(self, obj):
        """Get full name of user who created the archive"""
        if obj.created_by:
            return f"{obj.created_by.first_name} {obj.created_by.last_name}"
        return None


# Archive List Serializer
```

Replace with:
```python
    def get_created_by(self, obj):
        """Get display name of user who created the archive"""
        return get_display_name_for_user(obj.created_by)


# Archive List Serializer
```

- [ ] **Step 9: Update get_created_by (ArchiveListSerializer) — line ~3054**

Find:
```python
    def get_created_by(self, obj):
        """Get full name of user who created the archive"""
        if obj.created_by:
            return f"{obj.created_by.first_name} {obj.created_by.last_name}"
        return None
```

Replace with:
```python
    def get_created_by(self, obj):
        """Get display name of user who created the archive"""
        return get_display_name_for_user(obj.created_by)
```

- [ ] **Step 10: Update get_created_by (NoteHistorySerializer) — line ~3114**

Find:
```python
    def get_created_by(self, obj):
        if obj.created_by:
            return (
                f"{obj.created_by.first_name} {obj.created_by.last_name}".strip()
                or obj.created_by.email
            )
        return "Unknown"
```

Replace with:
```python
    def get_created_by(self, obj):
        return get_display_name_for_user(obj.created_by) or "Unknown"
```

- [ ] **Step 11: Run tests to verify they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_created_by_serializers -v 2
```

Expected: All 3 tests PASS.

- [ ] **Step 12: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers.py \
        TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_created_by_serializers.py
git commit -m "feat: TreatmentPlan serializers use get_display_name_for_user for created_by"
```

---

## Task 3: Update activityLog serializers (3 locations)

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/activityLog/serializers.py`

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/activityLog/tests/test_created_by_serializers.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock


def _make_superuser():
    u = MagicMock()
    u.user_type = "superuser"
    u.first_name = "Super"
    u.last_name = "Admin"
    u.email = "super@system.com"
    return u


class ActivityLogSerializerCreatedByTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from activityLog.serializers import ActivityLogSerializer
        obj = MagicMock()
        obj.created_by = _make_superuser()
        obj.is_system_generated = False
        s = ActivityLogSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System Admin")

    def test_none_non_system_shows_unknown(self):
        from activityLog.serializers import ActivityLogSerializer
        obj = MagicMock()
        obj.created_by = None
        obj.is_system_generated = False
        s = ActivityLogSerializer()
        self.assertEqual(s.get_created_by_name(obj), "Unknown")

    def test_none_system_generated_shows_system(self):
        from activityLog.serializers import ActivityLogSerializer
        obj = MagicMock()
        obj.created_by = None
        obj.is_system_generated = True
        s = ActivityLogSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System")
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test activityLog.tests.test_created_by_serializers -v 2
```

Expected: `test_superuser_shows_system_admin` FAILS — returns `"Super Admin"`.

- [ ] **Step 3: Add import and update all 3 methods**

At the top of `TreatmentPathBackend/TreatmentPath/activityLog/serializers.py`, add:

```python
from TreatmentPath.utils import get_display_name_for_user
```

Then update all three `get_created_by_name` methods (at lines ~55, ~142, ~235). Each currently looks like:

```python
    def get_created_by_name(self, obj):
        """Return the full name of the user who created this activity"""
        if obj.created_by:
            return f"{obj.created_by.first_name} {obj.created_by.last_name}".strip()
        return "System" if obj.is_system_generated else "Unknown"
```

Replace all three with:

```python
    def get_created_by_name(self, obj):
        """Return the display name of the user who created this activity"""
        if obj.created_by:
            return get_display_name_for_user(obj.created_by) or "Unknown"
        return "System" if obj.is_system_generated else "Unknown"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test activityLog.tests.test_created_by_serializers -v 2
```

Expected: All 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/activityLog/serializers.py \
        TreatmentPathBackend/TreatmentPath/activityLog/tests/test_created_by_serializers.py
git commit -m "feat: activityLog serializers use get_display_name_for_user for created_by"
```

---

## Task 4: Update messaging serializers (4 locations)

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/messaging/serializers.py`

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/messaging/tests/test_created_by_serializers.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock


def _make_superuser():
    u = MagicMock()
    u.user_type = "superuser"
    u.first_name = "Super"
    u.last_name = "Admin"
    u.email = "super@system.com"
    return u


def _make_staff():
    u = MagicMock()
    u.user_type = "staff"
    u.first_name = "Bob"
    u.last_name = "Lee"
    u.email = "bob@practice.com"
    return u


class MessagingCreatedByTest(TestCase):
    def test_template_superuser_shows_system_admin(self):
        from messaging.serializers import MessageTemplateSerializer
        obj = MagicMock()
        obj.created_by = _make_superuser()
        s = MessageTemplateSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System Admin")

    def test_template_staff_shows_name(self):
        from messaging.serializers import MessageTemplateSerializer
        obj = MagicMock()
        obj.created_by = _make_staff()
        s = MessageTemplateSerializer()
        self.assertEqual(s.get_created_by_name(obj), "Bob Lee")

    def test_template_none_returns_none(self):
        from messaging.serializers import MessageTemplateSerializer
        obj = MagicMock()
        obj.created_by = None
        s = MessageTemplateSerializer()
        self.assertIsNone(s.get_created_by_name(obj))
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test messaging.tests.test_created_by_serializers -v 2
```

Expected: `test_template_superuser_shows_system_admin` FAILS.

- [ ] **Step 3: Add import and update all 4 methods**

At the top of `TreatmentPathBackend/TreatmentPath/messaging/serializers.py`, add:

```python
from TreatmentPath.utils import get_display_name_for_user
```

Find and replace all 4 `get_created_by_name` methods. They currently look like one of these two patterns:

```python
# Pattern A (lines ~1246, ~1290, ~1398):
    def get_created_by_name(self, obj):
        if obj.created_by:
            return f"{obj.created_by.first_name} {obj.created_by.last_name}"
        return None

# Pattern B (call log, line ~1646):
    def get_created_by_name(self, obj):
        """Get the name of the user who created this call log"""
        if obj.created_by:
            return (
                f"{obj.created_by.first_name} {obj.created_by.last_name}".strip()
                or obj.created_by.email
            )
        return None
```

Replace ALL four with:

```python
    def get_created_by_name(self, obj):
        return get_display_name_for_user(obj.created_by)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test messaging.tests.test_created_by_serializers -v 2
```

Expected: All 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/messaging/serializers.py \
        TreatmentPathBackend/TreatmentPath/messaging/tests/test_created_by_serializers.py
git commit -m "feat: messaging serializers use get_display_name_for_user for created_by"
```

---

## Task 5: Update Labs, Assets, Stock serializers

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Labs/serializers.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Assets/serializers.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Stock/serializers.py`

These all follow the same inline pattern (`first_name`/`last_name`/`email` f-strings). No separate test file needed for each — one combined test covers all.

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_labs_assets_stock_created_by.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock


def _make_superuser():
    u = MagicMock()
    u.user_type = "superuser"
    u.first_name = "Super"
    u.last_name = "Admin"
    u.email = "super@system.com"
    return u


class LabsCreatedByTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from Labs.serializers import LabCaseSerializer
        obj = MagicMock()
        obj.created_by = _make_superuser()
        s = LabCaseSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System Admin")


class AssetsCreatedByTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from Assets.serializers import AssetSerializer
        obj = MagicMock()
        obj.created_by = _make_superuser()
        s = AssetSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System Admin")


class StockCreatedByTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from Stock.serializers import StockItemSerializer
        obj = MagicMock()
        obj.created_by = _make_superuser()
        s = StockItemSerializer()
        self.assertEqual(s.get_created_by_name(obj), "System Admin")
```

> **Note:** The exact serializer class names (`LabCaseSerializer`, `AssetSerializer`, `StockItemSerializer`) may differ. Run `grep "class.*Serializer" Labs/serializers.py | head -5` to confirm the correct class names and adjust the test imports accordingly before running.

- [ ] **Step 2: Confirm serializer class names**

```bash
cd TreatmentPathBackend/TreatmentPath
grep "class.*Serializer" Labs/serializers.py | head -5
grep "class.*Serializer" Assets/serializers.py | head -5
grep "class.*Serializer" Stock/serializers.py | head -5
```

Update the test file with the correct class names from the output.

- [ ] **Step 3: Run test to verify it fails**

```bash
python manage.py test TreatmentPath.tests.test_labs_assets_stock_created_by -v 2
```

Expected: All 3 FAIL with wrong name returned.

- [ ] **Step 4: Update Labs/serializers.py**

Add at top:
```python
from TreatmentPath.utils import get_display_name_for_user
```

Find `get_created_by_name` (line ~436):
```python
    def get_created_by_name(self, obj):
        if obj.created_by:
            first = obj.created_by.first_name or ''
            last = obj.created_by.last_name or ''
            full_name = f"{first} {last}".strip()
            return full_name or obj.created_by.email
        return None
```

Replace with:
```python
    def get_created_by_name(self, obj):
        return get_display_name_for_user(obj.created_by)
```

- [ ] **Step 5: Update Assets/serializers.py**

Add at top:
```python
from TreatmentPath.utils import get_display_name_for_user
```

Find all 3 `get_created_by_name` methods (lines ~217, ~263, ~467) — all have the same pattern:
```python
    def get_created_by_name(self, obj):
        if obj.created_by:
            first = obj.created_by.first_name or ''
            last = obj.created_by.last_name or ''
            full_name = f"{first} {last}".strip()
            return full_name or obj.created_by.email
        return None
```

Replace all three with:
```python
    def get_created_by_name(self, obj):
        return get_display_name_for_user(obj.created_by)
```

- [ ] **Step 6: Update Stock/serializers.py**

Add at top:
```python
from TreatmentPath.utils import get_display_name_for_user
```

Find `get_created_by_name` (line ~315) and replace the inline f-string with:
```python
    def get_created_by_name(self, obj):
        return get_display_name_for_user(obj.created_by)
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPath.tests.test_labs_assets_stock_created_by -v 2
```

Expected: All 3 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/Labs/serializers.py \
        TreatmentPathBackend/TreatmentPath/Assets/serializers.py \
        TreatmentPathBackend/TreatmentPath/Stock/serializers.py \
        TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_labs_assets_stock_created_by.py
git commit -m "feat: Labs/Assets/Stock serializers use get_display_name_for_user for created_by"
```

---

## Task 6: Update compliance, HR, and Appointments helper functions

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/compliance/serializers.py`
- Modify: `TreatmentPathBackend/TreatmentPath/HR/serializers.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Appointments/serializers.py`

These files have local helper functions. Updating the helper covers all 6+ call sites in compliance and the Appointments call site in one change each.

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_compliance_hr_appointments_created_by.py`:

```python
from django.test import TestCase
from unittest.mock import MagicMock


def _make_superuser():
    u = MagicMock()
    u.user_type = "superuser"
    u.first_name = "Super"
    u.last_name = "Admin"
    u.email = "super@system.com"
    return u


class ComplianceHelperTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from compliance.serializers import get_user_display_name
        self.assertEqual(get_user_display_name(_make_superuser()), "System Admin")


class HRHelperTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from HR.serializers import get_user_full_name
        self.assertEqual(get_user_full_name(_make_superuser()), "System Admin")


class AppointmentsHelperTest(TestCase):
    def test_superuser_shows_system_admin(self):
        from Appointments.serializers import get_user_full_name
        self.assertEqual(get_user_full_name(_make_superuser()), "System Admin")
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPath.tests.test_compliance_hr_appointments_created_by -v 2
```

Expected: All 3 FAIL.

- [ ] **Step 3: Update compliance/serializers.py helper**

Find (line ~35):
```python
def get_user_display_name(user):
    """Helper function to get a user's display name."""
    if user.first_name or user.last_name:
        return f"{user.first_name or ''} {user.last_name or ''}".strip()
    return user.email
```

Replace with:
```python
def get_user_display_name(user):
    """Helper function to get a user's display name."""
    if not user:
        return None
    from TreatmentPath.utils import get_display_name_for_user
    return get_display_name_for_user(user)
```

- [ ] **Step 4: Update HR/serializers.py helper**

Find (line ~18):
```python
def get_user_full_name(user):
    """Helper to get full name from user (handles missing get_full_name method)"""
    if not user:
        return None
    if hasattr(user, 'get_full_name'):
        return user.get_full_name()
    # Fallback to first_name + last_name
    first = getattr(user, 'first_name', '') or ''
    last = getattr(user, 'last_name', '') or ''
```

Replace the entire function with:
```python
def get_user_full_name(user):
    """Helper to get full name from user. Superusers shown as System Admin."""
    if not user:
        return None
    from TreatmentPath.utils import get_display_name_for_user
    return get_display_name_for_user(user)
```

- [ ] **Step 5: Update Appointments/serializers.py helper**

Find (line ~17):
```python
def get_user_full_name(user):
    """Helper to get full name from user."""
    if not user:
        return None
    if hasattr(user, "get_full_name"):
        name = user.get_full_name()
```

Replace the entire function with:
```python
def get_user_full_name(user):
    """Helper to get full name from user. Superusers shown as System Admin."""
    if not user:
        return None
    from TreatmentPath.utils import get_display_name_for_user
    return get_display_name_for_user(user)
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPath.tests.test_compliance_hr_appointments_created_by -v 2
```

Expected: All 3 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/compliance/serializers.py \
        TreatmentPathBackend/TreatmentPath/HR/serializers.py \
        TreatmentPathBackend/TreatmentPath/Appointments/serializers.py \
        TreatmentPathBackend/TreatmentPath/TreatmentPath/tests/test_compliance_hr_appointments_created_by.py
git commit -m "feat: compliance/HR/Appointments helpers use get_display_name_for_user"
```

---

## Task 7: Run full test suite to check for regressions

- [ ] **Step 1: Run all tests**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test --verbosity=1 2>&1 | tail -20
```

Expected: All existing tests pass. If any fail, they will be in serializers that depend on a helper function you changed — read the error and check the specific serializer.

- [ ] **Step 2: Commit if clean**

Only needed if you had to make additional fixes. Otherwise proceed to Task 8.

---

## Task 8: Fix frontend GuestAccess.tsx — anonymous identity

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/GuestAccess.tsx`

The current code uses `data.admin_user` to build the stored user object, which leaks the superuser's name and email into the browser session. We replace this with a neutral guest identity.

- [ ] **Step 1: Read the current file**

Read `perfect-pixel-playground-project/src/pages/GuestAccess.tsx` lines 36–75 to confirm the current `data` shape handling before editing.

- [ ] **Step 2: Update GuestAccess.tsx**

Find the block starting at line ~43:
```typescript
          // Support both practice and individual guest responses (or minimal data)
          const practice = data.practice || data.current_practice || null;
          const userData = data.admin_user || data.user || data.practice_user || data;

          const user = {
            id: (userData?.id ?? practice?.id ?? "guest").toString(),
            email: userData?.email ?? "",
            firstName: userData?.first_name ?? "",
            lastName: userData?.last_name ?? "",
            userType: practice ? ("practice" as const) : ("individual" as const),
            practiceName: practice?.name,
            practiceId: practice?.id?.toString(),
            practiceSlug: practice?.slug,
            isVerified: true,
          };
```

Replace with:
```typescript
          // Build a neutral guest identity — never expose superuser personal details
          const practice = data.practice || data.current_practice || null;

          const user = {
            id: "guest",
            email: "",
            firstName: "System",
            lastName: "Admin",
            userType: practice ? ("practice" as const) : ("individual" as const),
            practiceName: practice?.name,
            practiceId: practice?.id?.toString(),
            practiceSlug: practice?.slug,
            isVerified: true,
          };
```

- [ ] **Step 3: Verify the app builds**

```bash
cd perfect-pixel-playground-project
npm run build 2>&1 | tail -20
```

Expected: Build succeeds with no TypeScript errors.

- [ ] **Step 4: Commit**

```bash
git add perfect-pixel-playground-project/src/pages/GuestAccess.tsx
git commit -m "fix: GuestAccess builds neutral guest identity — no superuser details in browser session"
```

---

## Task 9: Manual smoke test

- [ ] **Step 1: Start backend**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py runserver
```

- [ ] **Step 2: Start frontend**

```bash
cd perfect-pixel-playground-project
npm run dev
```

- [ ] **Step 3: Create a guest session**

1. Log in as superuser at `/admin/practices`
2. Open a practice detail page
3. Click **Guest Access**
4. Confirm a new tab opens at `/guest?token=...`

- [ ] **Step 4: Verify anonymous identity**

In the new guest tab:
- Open browser DevTools → Application → Local Storage
- Confirm the stored user has `firstName: "System"`, `lastName: "Admin"`, `email: ""`
- Confirm the user's real email/name is NOT present anywhere in storage

- [ ] **Step 5: Verify created_by shows "System Admin"**

In the guest session:
1. Create a new patient/intake/nurture record
2. Navigate to view it
3. Confirm the "Created by" field shows **"System Admin"** not the superuser's real name

- [ ] **Step 6: Verify practice members unaffected**

Log in as a practice admin (non-superuser):
1. Create a record
2. Confirm "Created by" shows their real name (e.g. "Alice Jones"), not "System Admin"
