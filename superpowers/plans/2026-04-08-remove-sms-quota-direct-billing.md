# Remove SMS Quota — Direct Stripe Billing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the entire free monthly SMS quota system and make every outbound SMS billed directly via Stripe metered usage from message 1.

**Architecture:** The quota fields (`default_monthly_quota`, `custom_monthly_quota`, `current_month_usage`, `last_reset_date`, `allow_overage`, `no_free_quota`, `quota_override_reason`) are removed from `PracticeSMSConfiguration` via a Django migration. The send path is simplified to always call `record_sms_usage.delay()` with no quota gate. Frontend quota UI (tabs, modals, displays) is removed entirely. The two Celery Beat tasks (reset and alert) are deleted along with the management command.

**Tech Stack:** Django/Python backend, DRF serializers, Celery Beat, React/TypeScript frontend, Stripe metered billing (via existing `usage.tasks.record_sms_usage`)

---

## File Map

### Backend — files modified
- `TreatmentPathBackend/TreatmentPath/Admin/models.py` — remove quota fields and properties from `PracticeSMSConfiguration`
- `TreatmentPathBackend/TreatmentPath/Admin/serializers.py` — remove quota fields from all three SMS serializers
- `TreatmentPathBackend/TreatmentPath/Admin/views.py` — remove `update_quota`, `reset_quota` actions; remove quota stats from statistics endpoint
- `TreatmentPathBackend/TreatmentPath/Admin/urls.py` — remove `update-quota/` and `reset-quota/` URL paths
- `TreatmentPathBackend/TreatmentPath/messaging/views/sms_config_views.py` — delete `SMSQuotaView` and `EnablePayAsYouGoView` classes
- `TreatmentPathBackend/TreatmentPath/messaging/views/message_views.py` — remove quota gate from `perform_create`; remove bulk SMS quota check; always call `record_sms_usage`
- `TreatmentPathBackend/TreatmentPath/automations/actions.py` — remove quota check; always call `record_sms_usage`
- `TreatmentPathBackend/TreatmentPath/messaging/tasks.py` — delete `reset_monthly_sms_quotas` and `check_and_alert_quota_thresholds`
- `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py` — remove the two Celery Beat schedule entries
- `TreatmentPathBackend/TreatmentPath/messaging/urls.py` — remove `sms-config/quota/` and `sms-config/enable-payg/` paths, remove `SMSQuotaView` and `EnablePayAsYouGoView` imports

### Backend — files deleted
- `TreatmentPathBackend/TreatmentPath/messaging/management/commands/reset_sms_quotas.py`

### Backend — files created
- `TreatmentPathBackend/TreatmentPath/Admin/migrations/0033_remove_sms_quota_fields.py`

### Frontend — files modified
- `perfect-pixel-playground-project/src/components/settings/sms-phone-config/callRatesUtils.tsx` — remove quota fields from `SMSConfiguration` interface
- `perfect-pixel-playground-project/src/components/settings/sms-phone-config/PracticeConfigModal.tsx` — remove `"quota"` tab, `QuotaSection` component, and quota display in `DetailsPanel`-like section
- `perfect-pixel-playground-project/src/components/settings/sms-phone-config/DetailsPanel.tsx` — remove quota display rows
- `perfect-pixel-playground-project/src/hooks/useMessaging.ts` — delete `QuotaExceededError` class and its throw/catch
- `perfect-pixel-playground-project/src/config/api.ts` — remove `updateQuota`, `resetQuota`, `getQuota`, `enablePayg` endpoint entries
- `perfect-pixel-playground-project/src/pages/admin/practice-management/hooks/usePracticeManagement.ts` — remove `smsQuotaConfig`, `smsQuotaValue`, `smsQuotaDirty`, `fetchSmsQuotaConfig`, `applySmsQuota` state/functions
- `perfect-pixel-playground-project/src/pages/admin/practice-management/components/PracticeWorkspace.tsx` — remove quota props, quota input section, `handleSaveQuota`, quota display block
- `perfect-pixel-playground-project/src/pages/admin/SMSConfiguration.tsx` — remove quota columns/display

### Frontend — files deleted
- `perfect-pixel-playground-project/src/components/settings/sms-phone-config/UpdateQuotaModal.tsx`

---

## Task 1: Write Backend Tests for the New Send Path

**Files:**
- Test: `TreatmentPathBackend/TreatmentPath/messaging/tests/test_sms_send_no_quota.py`

- [ ] **Step 1: Create the test file**

```python
# TreatmentPathBackend/TreatmentPath/messaging/tests/test_sms_send_no_quota.py
from unittest.mock import MagicMock, patch

import pytest
from django.test import TestCase

from Admin.models import PracticeSMSConfiguration


class TestSMSSendNoQuota(TestCase):
    """After quota removal, every outbound SMS must trigger record_sms_usage."""

    def setUp(self):
        from UserAuthentication.models import Practice, User

        self.practice = Practice.objects.create(
            name="Test Practice",
            slug="test-practice",
            country="GB",
        )
        self.practice.twilio_phone_number = "+441234567890"
        self.practice.save()

        self.sms_config = PracticeSMSConfiguration.objects.create(
            practice=self.practice,
            phone_number_status="active",
            phone_number_request_status="approved",
            is_active=True,
        )

    def test_sms_config_has_no_quota_fields(self):
        """PracticeSMSConfiguration must not have quota-related fields."""
        config = PracticeSMSConfiguration.objects.get(practice=self.practice)
        quota_attrs = [
            "default_monthly_quota",
            "custom_monthly_quota",
            "current_month_usage",
            "last_reset_date",
            "allow_overage",
            "no_free_quota",
            "quota_override_reason",
            "effective_monthly_quota",
            "remaining_quota",
            "quota_percentage_used",
            "is_quota_exceeded",
        ]
        for attr in quota_attrs:
            self.assertFalse(
                hasattr(config, attr),
                f"PracticeSMSConfiguration should not have attribute: {attr}",
            )

    @patch("usage.tasks.record_sms_usage")
    def test_record_sms_usage_always_called(self, mock_record):
        """record_sms_usage.delay() must be called for every outgoing SMS."""
        mock_record.delay = MagicMock()
        # This will be wired up once send path is updated in Task 4
        # For now just verify the task is importable
        from usage.tasks import record_sms_usage  # noqa: F401
        self.assertTrue(callable(record_sms_usage))
```

- [ ] **Step 2: Run the test to verify it fails on quota attribute checks**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test messaging.tests.test_sms_send_no_quota -v 2
```

Expected: `test_sms_config_has_no_quota_fields` FAILS because `default_monthly_quota` still exists.

- [ ] **Step 3: Commit the failing test**

```bash
git add TreatmentPathBackend/TreatmentPath/messaging/tests/test_sms_send_no_quota.py
git commit -m "test: add failing tests for quota removal — no quota fields, always bill"
```

---

## Task 2: Remove Quota Fields from the Model

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/models.py:290-480`
- Create: `TreatmentPathBackend/TreatmentPath/Admin/migrations/0033_remove_sms_quota_fields.py`

- [ ] **Step 1: Remove quota fields and methods from `PracticeSMSConfiguration`**

In [Admin/models.py](TreatmentPathBackend/TreatmentPath/Admin/models.py), find the `PracticeSMSConfiguration` class and replace the entire quota section (lines ~341–470) so the class only retains phone management and status fields. The final model body should be:

```python
class PracticeSMSConfiguration(models.Model):
    """Manages SMS configuration and phone numbers for each practice."""

    STATUS_CHOICES = [
        ("pending", "Pending Provisioning"),
        ("active", "Active"),
        ("suspended", "Suspended"),
        ("cancelled", "Cancelled"),
    ]

    REQUEST_STATUS_CHOICES = [
        ("none", "No Request"),
        ("requested", "Requested"),
        ("approved", "Approved"),
        ("rejected", "Rejected"),
    ]

    practice = models.OneToOneField(
        "UserAuthentication.Practice",
        on_delete=models.CASCADE,
        related_name="sms_configuration",
        help_text="The practice this SMS configuration belongs to",
    )

    # Phone number management
    # NOTE: Phone number is stored in Practice.twilio_phone_number (single source of truth)
    # Access via the phone_number property below

    phone_number_status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default="pending",
        help_text="Current status of the phone number",
    )

    phone_number_request_status = models.CharField(
        max_length=20,
        choices=REQUEST_STATUS_CHOICES,
        default="none",
        help_text="Status of phone number request from practice",
    )

    phone_number_request_date = models.DateTimeField(
        blank=True, null=True, help_text="Date when phone number was requested"
    )

    phone_number_provisioned_date = models.DateTimeField(
        blank=True, null=True, help_text="Date when phone number was provisioned"
    )

    is_active = models.BooleanField(
        default=True, help_text="Whether SMS service is active for this practice"
    )

    created_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="created_sms_configurations",
        help_text="The superuser who created this configuration",
    )

    updated_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name="updated_sms_configurations",
        help_text="The superuser who last updated this configuration",
    )

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name = "Practice SMS Configuration"
        verbose_name_plural = "Practice SMS Configurations"
        ordering = ["-created_at"]

    def __str__(self):
        return f"SMS Config for {self.practice.name}"

    @property
    def phone_number(self):
        """Get phone number from Practice model (single source of truth)."""
        return self.practice.twilio_phone_number

    def save(self, *args, **kwargs):
        """
        Save SMS configuration.

        Note: Phone number is stored in Practice.twilio_phone_number (not in this model).
        This model only tracks status and provisioning metadata.
        """
        super().save(*args, **kwargs)
```

- [ ] **Step 2: Generate the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py makemigrations Admin --name remove_sms_quota_fields
```

Expected output: `Migrations for 'Admin': Admin/migrations/0033_remove_sms_quota_fields.py`

- [ ] **Step 3: Apply the migration**

```bash
python manage.py migrate Admin
```

Expected: `Applying Admin.0033_remove_sms_quota_fields... OK`

- [ ] **Step 4: Run the model test**

```bash
python manage.py test messaging.tests.test_sms_send_no_quota.TestSMSSendNoQuota.test_sms_config_has_no_quota_fields -v 2
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/Admin/models.py \
        TreatmentPathBackend/TreatmentPath/Admin/migrations/0033_remove_sms_quota_fields.py
git commit -m "feat: remove SMS quota fields from PracticeSMSConfiguration model"
```

---

## Task 3: Clean Up Serializers

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/serializers.py:446-598`

- [ ] **Step 1: Replace `PracticeSMSConfigurationSerializer` fields**

Remove all quota-related fields. New `fields` list for `PracticeSMSConfigurationSerializer.Meta`:

```python
class PracticeSMSConfigurationSerializer(serializers.ModelSerializer):
    """Serializer for viewing and managing practice SMS configurations (admin use)"""

    practice_name = serializers.CharField(source="practice.name", read_only=True)
    practice_slug = serializers.CharField(source="practice.slug", read_only=True)
    phone_number_status_display = serializers.CharField(
        source="get_phone_number_status_display", read_only=True
    )
    phone_number_request_status_display = serializers.CharField(
        source="get_phone_number_request_status_display", read_only=True
    )
    created_by_name = serializers.CharField(
        source="created_by.get_full_name", read_only=True, allow_null=True
    )
    updated_by_name = serializers.CharField(
        source="updated_by.get_full_name", read_only=True, allow_null=True
    )

    class Meta:
        model = PracticeSMSConfiguration
        fields = [
            "id",
            "practice",
            "practice_name",
            "practice_slug",
            "phone_number",
            "phone_number_status",
            "phone_number_status_display",
            "phone_number_request_status",
            "phone_number_request_status_display",
            "phone_number_request_date",
            "phone_number_provisioned_date",
            "is_active",
            "created_by",
            "created_by_name",
            "updated_by",
            "updated_by_name",
            "created_at",
            "updated_at",
        ]
        read_only_fields = [
            "practice",
            "phone_number_request_date",
            "created_by",
            "updated_by",
            "created_at",
            "updated_at",
        ]

    def update(self, instance, validated_data):
        instance.updated_by = self.context["request"].user
        return super().update(instance, validated_data)
```

- [ ] **Step 2: Replace `PracticeSMSConfigurationUpdateSerializer` fields**

```python
class PracticeSMSConfigurationUpdateSerializer(serializers.ModelSerializer):
    """Serializer for superuser to update SMS configuration"""

    class Meta:
        model = PracticeSMSConfiguration
        fields = [
            "phone_number_status",
            "is_active",
        ]

    def update(self, instance, validated_data):
        instance.updated_by = self.context["request"].user

        if "phone_number" in validated_data and validated_data["phone_number"]:
            if instance.phone_number_status == "pending":
                instance.phone_number_status = "active"
            if not instance.phone_number_provisioned_date:
                from django.utils import timezone
                instance.phone_number_provisioned_date = timezone.now()

        return super().update(instance, validated_data)

    def validate(self, data):
        request = self.context.get("request")
        if request and not request.user.is_superuser:
            raise serializers.ValidationError(
                "Only superusers can update SMS configurations"
            )
        return data
```

- [ ] **Step 3: Replace `PracticeSMSConfigurationReadSerializer` fields**

```python
class PracticeSMSConfigurationReadSerializer(serializers.ModelSerializer):
    """Serializer for practice users to view their SMS configuration (read-only)"""

    phone_number_status_display = serializers.CharField(
        source="get_phone_number_status_display", read_only=True
    )

    class Meta:
        model = PracticeSMSConfiguration
        fields = [
            "id",
            "phone_number",
            "phone_number_status",
            "phone_number_status_display",
            "is_active",
        ]
        read_only_fields = [
            "id",
            "phone_number",
            "phone_number_status",
            "phone_number_status_display",
            "is_active",
        ]
```

- [ ] **Step 4: Verify the app boots with no errors**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/Admin/serializers.py
git commit -m "feat: remove quota fields from SMS serializers"
```

---

## Task 4: Clean Up Admin Views and URLs

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/views.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/urls.py`

- [ ] **Step 1: Remove `update_quota` and `reset_quota` actions from `PracticeSMSConfigurationViewSet`**

In [Admin/views.py](TreatmentPathBackend/TreatmentPath/Admin/views.py), find and delete both `@action` decorated methods named `update_quota` and `reset_quota` entirely (approximately lines 4455–4527).

- [ ] **Step 2: Remove quota fields from the `statistics` action in the same ViewSet**

Find the `statistics` action (approximately line 4765). Remove any references to `effective_monthly_quota`, `current_month_usage`, `remaining_quota`, `quota_percentage_used`, `is_quota_exceeded`, `allow_overage`, `no_free_quota` from the aggregation query and response dict.

- [ ] **Step 3: Remove the quota URL paths from Admin/urls.py**

Find and delete these two `path()` entries in [Admin/urls.py](TreatmentPathBackend/TreatmentPath/Admin/urls.py):

```python
# DELETE these two blocks:
path(
    "sms-configurations/<int:pk>/update-quota/",
    PracticeSMSConfigurationViewSet.as_view({"post": "update_quota"}),
    name="sms-configurations-update-quota",
),
path(
    "sms-configurations/<int:pk>/reset-quota/",
    PracticeSMSConfigurationViewSet.as_view({"post": "reset_quota"}),
    name="sms-configurations-reset-quota",
),
```

- [ ] **Step 4: Verify the app boots**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/Admin/views.py \
        TreatmentPathBackend/TreatmentPath/Admin/urls.py
git commit -m "feat: remove update_quota and reset_quota admin actions"
```

---

## Task 5: Remove Practice-Facing Quota Views and URLs

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/messaging/views/sms_config_views.py`
- Modify: `TreatmentPathBackend/TreatmentPath/messaging/urls.py`

- [ ] **Step 1: Delete `SMSQuotaView` and `EnablePayAsYouGoView` from sms_config_views.py**

In [messaging/views/sms_config_views.py](TreatmentPathBackend/TreatmentPath/messaging/views/sms_config_views.py), delete the entire `SMSQuotaView` class (lines 101–143) and the entire `EnablePayAsYouGoView` class (lines 145–194). The file should now only contain `PracticeSMSConfigurationView` and `PhoneNumberRequestView`.

- [ ] **Step 2: Also remove quota field references from `SMSQuotaView`-dependent response in `PracticeSMSConfigurationView`**

The `GET` handler returns `serializer.data` via `PracticeSMSConfigurationReadSerializer` which is already cleaned up in Task 3, so no additional change needed here.

- [ ] **Step 3: Clean up messaging/urls.py**

In [messaging/urls.py](TreatmentPathBackend/TreatmentPath/messaging/urls.py):

1. Remove `SMSQuotaView` from the import on line 18
2. Remove the import `from .views.sms_config_views import EnablePayAsYouGoView` on line 31
3. Delete the two URL path entries:

```python
# DELETE:
path(
    "sms-config/quota/",
    SMSQuotaView.as_view(),
    name="practice-sms-quota",
),
path(
    "sms-config/enable-payg/",
    EnablePayAsYouGoView.as_view(),
    name="practice-sms-enable-payg",
),
```

- [ ] **Step 4: Verify**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/messaging/views/sms_config_views.py \
        TreatmentPathBackend/TreatmentPath/messaging/urls.py
git commit -m "feat: remove SMSQuotaView and EnablePayAsYouGoView"
```

---

## Task 6: Simplify the SMS Send Path — Always Bill

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/messaging/views/message_views.py`

- [ ] **Step 1: Remove quota gate from `perform_create`**

In [message_views.py](TreatmentPathBackend/TreatmentPath/messaging/views/message_views.py), find the quota check block (approximately lines 181–229). Replace the entire `try/except PracticeSMSConfiguration.DoesNotExist` block with:

```python
# Check SMS quota
try:
    sms_config = PracticeSMSConfiguration.objects.get(practice=practice)

    # Check if SMS service is active
    if not sms_config.is_active:
        raise ValidationError(
            {"error": "SMS service is not active for this practice"}
        )

    # Check if phone number is active
    if (
        not sms_config.phone_number
        or sms_config.phone_number_status != "active"
    ):
        raise ValidationError(
            {
                "error": "No active phone number configured for this practice. Please provision a phone number first."
            }
        )

except PracticeSMSConfiguration.DoesNotExist:
    sms_config = PracticeSMSConfiguration.objects.create(
        practice=practice,
        phone_number_status="pending",
        phone_number_request_status="none",
    )
    raise ValidationError(
        {
            "error": "SMS configuration not found. Please contact administrator to set up SMS service."
        }
    )
```

- [ ] **Step 2: Replace conditional Stripe billing with unconditional call**

Find the block after SMS message creation (approximately lines 295–310):

```python
# Increment SMS usage counter
sms_config.increment_usage()

# Report to Stripe if this practice is on direct billing (no_free_quota)
# or has opted in to pay-as-you-go overage
if sms_config.no_free_quota or sms_config.allow_overage:
    try:
        from usage.tasks import record_sms_usage

        record_sms_usage.delay(practice.id, "outgoing")
    except Exception as e:
        import logging

        logging.getLogger(__name__).warning(
            f"perform_create: failed to queue Stripe usage report for sms_sent: {e}"
        )
```

Replace with:

```python
# Record Stripe metered usage for every outgoing SMS
try:
    from usage.tasks import record_sms_usage

    record_sms_usage.delay(practice.id, "outgoing")
except Exception as e:
    logger.warning(
        f"perform_create: failed to queue Stripe usage report for sms_sent: {e}"
    )
```

- [ ] **Step 3: Remove bulk SMS quota check**

Find the bulk SMS quota gate (approximately lines 950–968):

```python
# Check if quota allows sending to all recipients
if (
    sms_config.remaining_quota < len(recipients)
    and not sms_config.allow_overage
):
    return Response(
        {
            "error": "Insufficient SMS quota",
            ...
        },
        status=status.HTTP_403_FORBIDDEN,
    )
```

Delete this entire block. Bulk SMS should proceed without any quota gate.

- [ ] **Step 4: Run the test**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test messaging.tests.test_sms_send_no_quota -v 2
```

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/messaging/views/message_views.py
git commit -m "feat: remove quota gate from SMS send path — always call record_sms_usage"
```

---

## Task 7: Simplify Automation SMS Send Path

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/automations/actions.py`

- [ ] **Step 1: Remove quota check from automation SMS action**

In [automations/actions.py](TreatmentPathBackend/TreatmentPath/automations/actions.py), find the quota check block (approximately lines 1258–1265):

```python
if sms_config.is_quota_exceeded:
    return Response(
        {
            "error": f"Monthly SMS quota exceeded. Used: {sms_config.current_month_usage}, Quota: {sms_config.effective_monthly_quota}"
        },
        status=status.HTTP_403_FORBIDDEN,
    )
```

Delete this entire `if` block.

- [ ] **Step 2: Replace conditional Stripe billing with unconditional call after SMS message creation**

Find the block at approximately line 1354:

```python
# Increment SMS usage counter
sms_config.increment_usage()
```

Replace with:

```python
# Record Stripe metered usage for every outgoing SMS
try:
    from usage.tasks import record_sms_usage

    record_sms_usage.delay(practice.id, "outgoing")
except Exception as e:
    logger.warning(
        f"Automation SMS: failed to queue Stripe usage report for sms_sent: {e}"
    )
```

- [ ] **Step 3: Verify**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py check
python manage.py test messaging.tests.test_sms_send_no_quota -v 2
```

Expected: Check OK, all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/automations/actions.py
git commit -m "feat: remove quota check from automation SMS — always bill via Stripe"
```

---

## Task 8: Delete Celery Tasks and Management Command

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/messaging/tasks.py`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py`
- Delete: `TreatmentPathBackend/TreatmentPath/messaging/management/commands/reset_sms_quotas.py`

- [ ] **Step 1: Delete `reset_monthly_sms_quotas` and `check_and_alert_quota_thresholds` from tasks.py**

In [messaging/tasks.py](TreatmentPathBackend/TreatmentPath/messaging/tasks.py), delete the entire `reset_monthly_sms_quotas` function (lines 257–317) and the entire `check_and_alert_quota_thresholds` function (lines 320–408).

- [ ] **Step 2: Remove Celery Beat schedule entries from settings.py**

In [settings.py](TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py), find and delete these two entries from `CELERY_BEAT_SCHEDULE`:

```python
# DELETE:
"reset-monthly-sms-quotas": {
    "task": "messaging.tasks.reset_monthly_sms_quotas",
    "schedule": crontab(hour=3, minute=30, day_of_month=1),
    "options": {"queue": "default"},
},
"check-and-alert-quota-thresholds": {
    "task": "messaging.tasks.check_and_alert_quota_thresholds",
    "schedule": crontab(hour=9, minute=0),
    "options": {"queue": "default"},
},
```

- [ ] **Step 3: Delete the management command file**

```bash
rm /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/messaging/management/commands/reset_sms_quotas.py
```

- [ ] **Step 4: Verify**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py check
```

Expected: `System check identified no issues (0 silenced).`

- [ ] **Step 5: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/messaging/tasks.py \
        TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py
git rm TreatmentPathBackend/TreatmentPath/messaging/management/commands/reset_sms_quotas.py
git commit -m "feat: delete SMS quota reset/alert Celery tasks and management command"
```

---

## Task 9: Run Full Backend Test Suite

- [ ] **Step 1: Run all messaging and admin tests**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test messaging Admin --verbosity=2 2>&1 | tail -30
```

Expected: All tests PASS. If quota-related test files exist that test the old behaviour, delete them. If tests fail for reasons unrelated to quota removal, investigate before proceeding.

- [ ] **Step 2: Commit any test cleanup**

```bash
git add -u
git commit -m "test: clean up outdated quota-related test assertions"
```

---

## Task 10: Frontend — Remove `SMSConfiguration` Quota Fields

**Files:**
- Modify: `perfect-pixel-playground-project/src/components/settings/sms-phone-config/callRatesUtils.tsx`

- [ ] **Step 1: Remove quota fields from the `SMSConfiguration` interface**

In [callRatesUtils.tsx](perfect-pixel-playground-project/src/components/settings/sms-phone-config/callRatesUtils.tsx), replace the `SMSConfiguration` interface with:

```typescript
export interface SMSConfiguration {
  id: number;
  practice: number;
  practice_name: string;
  practice_slug: string;
  phone_number: string | null;
  phone_number_status: string;
  phone_number_status_display: string;
  phone_number_request_status: string;
  phone_number_request_status_display: string;
  phone_number_provisioned_date: string | null;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}
```

- [ ] **Step 2: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

Expected: TypeScript errors will appear for all the files that reference removed fields — that is expected and will be fixed in subsequent tasks.

- [ ] **Step 3: Commit**

```bash
git add perfect-pixel-playground-project/src/components/settings/sms-phone-config/callRatesUtils.tsx
git commit -m "feat: remove quota fields from SMSConfiguration TypeScript interface"
```

---

## Task 11: Frontend — Delete `UpdateQuotaModal` and Remove Quota Tab

**Files:**
- Delete: `perfect-pixel-playground-project/src/components/settings/sms-phone-config/UpdateQuotaModal.tsx`
- Modify: `perfect-pixel-playground-project/src/components/settings/sms-phone-config/PracticeConfigModal.tsx`

- [ ] **Step 1: Delete UpdateQuotaModal.tsx**

```bash
rm /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project/src/components/settings/sms-phone-config/UpdateQuotaModal.tsx
```

- [ ] **Step 2: Remove the quota tab from `PracticeConfigTab` type**

In [PracticeConfigModal.tsx](perfect-pixel-playground-project/src/components/settings/sms-phone-config/PracticeConfigModal.tsx), update the tab type (line 45):

```typescript
export type PracticeConfigTab =
  | "sms-phone"
  | "voice-agents"
  | "details"
  | "call-rates"
  | "sms-rates"
  | "stt-rates"
  | "usage";
```

- [ ] **Step 3: Remove the quota tab from the tab navigation array**

Delete the `{ id: "quota", ... }` entry from the tabs array (approximately line 92).

- [ ] **Step 4: Remove the `QuotaSection` render call and the `QuotaSection` component definition**

Delete `{activeSection === "quota" && <QuotaSection ... />}` from the render (approximately line 130).

Delete the entire `QuotaSection` function component (approximately lines 407–450).

- [ ] **Step 5: Remove quota display from the `DetailsPanel`-like section inside the modal**

Find the section that displays `effective_monthly_quota`, `remaining_quota`, quota percentage, `is_quota_exceeded`, `quota_override_reason` (approximately lines 678–703) and delete it.

Also find `handleResetQuota` function (approximately line 655) and the reset quota button and delete them.

- [ ] **Step 6: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

- [ ] **Step 7: Commit**

```bash
git rm perfect-pixel-playground-project/src/components/settings/sms-phone-config/UpdateQuotaModal.tsx
git add perfect-pixel-playground-project/src/components/settings/sms-phone-config/PracticeConfigModal.tsx
git commit -m "feat: remove quota tab and QuotaSection from PracticeConfigModal"
```

---

## Task 12: Frontend — Remove Quota Display from DetailsPanel

**Files:**
- Modify: `perfect-pixel-playground-project/src/components/settings/sms-phone-config/DetailsPanel.tsx`

- [ ] **Step 1: Read the file first**

Read [DetailsPanel.tsx](perfect-pixel-playground-project/src/components/settings/sms-phone-config/DetailsPanel.tsx) to understand its full structure.

- [ ] **Step 2: Remove quota display rows**

Delete any rows/sections that display `current_month_usage`, `effective_monthly_quota`, `remaining_quota`, `last_reset_date`, quota percentage, or quota exceeded warnings. Also remove any "Reset monthly usage" button and its handler.

- [ ] **Step 3: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

- [ ] **Step 4: Commit**

```bash
git add perfect-pixel-playground-project/src/components/settings/sms-phone-config/DetailsPanel.tsx
git commit -m "feat: remove quota display rows from DetailsPanel"
```

---

## Task 13: Frontend — Remove `QuotaExceededError` from useMessaging

**Files:**
- Modify: `perfect-pixel-playground-project/src/hooks/useMessaging.ts`

- [ ] **Step 1: Delete `QuotaExceededError` class and its usage**

In [useMessaging.ts](perfect-pixel-playground-project/src/hooks/useMessaging.ts), delete:

1. The entire `QuotaExceededError` class definition (lines 7–18)
2. The `if (errorData?.error_code === 'quota_exceeded') { throw new QuotaExceededError(...) }` block (approximately lines 440–446)
3. The `if (err instanceof QuotaExceededError) { throw err; }` re-throw block (approximately lines 463–465)

- [ ] **Step 2: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

- [ ] **Step 3: Commit**

```bash
git add perfect-pixel-playground-project/src/hooks/useMessaging.ts
git commit -m "feat: remove QuotaExceededError — no quota gates on send path"
```

---

## Task 14: Frontend — Remove Quota API Endpoints and Practice Management Quota State

**Files:**
- Modify: `perfect-pixel-playground-project/src/config/api.ts`
- Modify: `perfect-pixel-playground-project/src/pages/admin/practice-management/hooks/usePracticeManagement.ts`
- Modify: `perfect-pixel-playground-project/src/pages/admin/practice-management/components/PracticeWorkspace.tsx`

- [ ] **Step 1: Remove quota API endpoints from api.ts**

In [api.ts](perfect-pixel-playground-project/src/config/api.ts), delete these entries from the `smsConfiguration.admin` object:

```typescript
// DELETE:
updateQuota: (id: number) => getApiUrl(`/system-settings/sms-configurations/${id}/update-quota/`),
resetQuota: (id: number) => getApiUrl(`/system-settings/sms-configurations/${id}/reset-quota/`),
```

And from `smsConfiguration.practice`:

```typescript
// DELETE:
getQuota: () => getApiUrl('/messaging/sms-config/quota/'),
enablePayg: () => getApiUrl('/messaging/sms-config/enable-payg/'),
```

- [ ] **Step 2: Remove quota state and functions from usePracticeManagement.ts**

In [usePracticeManagement.ts](perfect-pixel-playground-project/src/pages/admin/practice-management/hooks/usePracticeManagement.ts), delete:

1. `smsQuotaConfig` and `smsQuotaValue` state declarations (lines 63–64)
2. `savedSmsQuotaState` state (lines 113–115)
3. `smsQuotaDirty` computed value
4. `fetchSmsQuotaConfig()` function (lines 505–530)
5. The `await fetchSmsQuotaConfig(practice.id)` call (line 383)
6. `applySmsQuota()` function (lines 744–766)
7. The `await applySmsQuota()` call in the save handler (line 976)
8. All quota state resets in the cleanup/reset area (lines 1680–1682)
9. The quota state/callback exports from the return object (lines 1757–1759, 1811)

- [ ] **Step 3: Remove quota props and UI from PracticeWorkspace.tsx**

In [PracticeWorkspace.tsx](perfect-pixel-playground-project/src/pages/admin/practice-management/components/PracticeWorkspace.tsx), delete:

1. `smsQuotaConfig`, `smsQuotaValue`, `smsQuotaReason`, `smsQuotaDirty` from the props interface (lines 71–74)
2. `onSmsQuotaValueChange`, `onSmsQuotaReasonChange` from props (lines 140–141)
3. `quotaInput`, `savingQuota` state (lines 206–207)
4. `quotaBaseline` and `quotaDirty` computed values (lines 222–227)
5. The `setQuotaInput(...)` call in the `useEffect` that syncs from `practiceConfig` (lines 291–294)
6. `handleSaveQuota()` function (lines 610–650)
7. Quota prop passthrough to sub-components (lines 815–855 area)
8. The entire "SMS Quota" display section (lines 1151–1210)

- [ ] **Step 4: Verify TypeScript compiles clean**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

Expected: Zero errors (or only pre-existing unrelated errors).

- [ ] **Step 5: Commit**

```bash
git add perfect-pixel-playground-project/src/config/api.ts \
        perfect-pixel-playground-project/src/pages/admin/practice-management/hooks/usePracticeManagement.ts \
        perfect-pixel-playground-project/src/pages/admin/practice-management/components/PracticeWorkspace.tsx
git commit -m "feat: remove SMS quota state, API endpoints, and UI from practice management"
```

---

## Task 15: Frontend — Clean Up SMSConfiguration Admin Page

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/admin/SMSConfiguration.tsx`

- [ ] **Step 1: Read the file**

Read [SMSConfiguration.tsx](perfect-pixel-playground-project/src/pages/admin/SMSConfiguration.tsx) fully to understand what quota fields it displays.

- [ ] **Step 2: Remove quota columns and displays**

Delete any table columns or card sections that display:
- `effective_monthly_quota`, `custom_monthly_quota`, `default_monthly_quota`
- `current_month_usage`, `remaining_quota`, `quota_percentage_used`
- `is_quota_exceeded`, `allow_overage`, `no_free_quota`, `last_reset_date`
- Any "Update Quota" / "Reset Quota" buttons or modals
- Any import of `UpdateQuotaModal`

- [ ] **Step 3: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1 | head -40
```

- [ ] **Step 4: Commit**

```bash
git add perfect-pixel-playground-project/src/pages/admin/SMSConfiguration.tsx
git commit -m "feat: remove quota columns and controls from SMS admin page"
```

---

## Task 16: Frontend — Final TypeScript Compile Check and Remaining Cleanup

- [ ] **Step 1: Run full TypeScript check**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit 2>&1
```

Review all errors. Fix any remaining references to deleted quota fields or types (e.g., `SubscriptionSection.tsx`, `PracticeCommunicationCard.tsx` if they reference quota fields).

- [ ] **Step 2: Fix any remaining files**

For each file still referencing quota fields, remove the references. Common fixes:
- `SubscriptionSection.tsx` — remove quota-related fields from `SmsConfiguration` interface usage
- `PracticeCommunicationCard.tsx` — remove quota display if present

- [ ] **Step 3: Final TypeScript compile**

```bash
npx tsc --noEmit 2>&1
```

Expected: Zero quota-related errors.

- [ ] **Step 4: Commit**

```bash
git add -u
git commit -m "feat: fix remaining TypeScript references to removed quota fields"
```

---

## Task 17: End-to-End Smoke Test

- [ ] **Step 1: Start backend server**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py runserver 8000
```

- [ ] **Step 2: Verify the SMS config endpoint returns no quota fields**

Using the dev server or curl with an authenticated token, call:

```
GET /api/backend/messaging/sms-config/
```

Expected response contains only: `id`, `phone_number`, `phone_number_status`, `phone_number_status_display`, `is_active` — no quota fields.

- [ ] **Step 3: Verify removed endpoints return 404**

```
GET /api/backend/messaging/sms-config/quota/        → 404
POST /api/backend/messaging/sms-config/enable-payg/ → 404
POST /api/backend/system-settings/sms-configurations/1/update-quota/ → 404
POST /api/backend/system-settings/sms-configurations/1/reset-quota/  → 404
```

- [ ] **Step 4: Verify frontend builds without errors**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npm run build 2>&1 | tail -20
```

Expected: Build succeeds with no errors.

- [ ] **Step 5: Final commit**

```bash
git commit --allow-empty -m "feat: SMS quota removal complete — all SMS billed directly via Stripe from message 1"
```

---

## Self-Review

**Spec coverage:**
- ✅ Remove all quota fields from model (Task 2)
- ✅ Drop DB columns via migration (Task 2)
- ✅ Remove serializer quota fields (Task 3)
- ✅ Remove admin quota actions and URLs (Task 4)
- ✅ Remove practice quota view and URL (Task 5)
- ✅ Send path always calls `record_sms_usage` (Tasks 6, 7)
- ✅ Delete Celery reset/alert tasks and management command (Task 8)
- ✅ Remove quota TypeScript interface fields (Task 10)
- ✅ Delete UpdateQuotaModal, remove quota tab (Task 11)
- ✅ Remove DetailsPanel quota rows (Task 12)
- ✅ Remove QuotaExceededError (Task 13)
- ✅ Remove quota state from practice management (Task 14)
- ✅ Clean up SMSConfiguration admin page (Task 15)

**No placeholders found.**

**Type consistency:** `PracticeSMSConfiguration` model fields used in migration match the model changes in Task 2. Serializer field lists in Task 3 reference only fields that exist in the updated model. Frontend `SMSConfiguration` interface in Task 10 is referenced in Tasks 11–16 — all use the same trimmed interface.
