# Recall Treatment Table & Engine Refactor

## Summary

Add a `recall_treatment` table to store treatment plan items fetched from Dentally during recall sync. Move treatment flag computation (`has_crown`, `has_hygiene`, etc.) from `recall_patient` to `recall_record`, computed at generation time from local `recall_treatment` + `nomenclature_classification` data.

## Problem

1. The recall engine fetches treatment plan items from Dentally API during sync but throws them away after extracting boolean flags.
2. Treatment flags (`has_crown`, `has_implant`, etc.) are stored on `recall_patient`, which should be pure Dentally source data.
3. We have no timeline for when treatments happened — only whether they ever happened.
4. The existing `dentallyIntegration_treatmentplanitem` table is used by daylist and other systems, so recall needs its own table.

## Design

### New Table: `recall_treatment`

Pure Dentally data + housekeeping columns. All 27 fields from the Dentally `/treatment_plan_items` API response, plus our standard recall columns.

```sql
CREATE TABLE recall_treatment (
    -- Our housekeeping
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id                 BIGINT NOT NULL,
    recall_patient_id           UUID NOT NULL,
    dentally_patient_id         INTEGER NOT NULL,

    -- Dentally fields (all 27 from API response)
    dentally_id                 BIGINT NOT NULL,            -- Dentally's "id"
    appear_on_invoice           BOOLEAN NOT NULL DEFAULT false,
    base_chart                  BOOLEAN NOT NULL DEFAULT false,
    charged                     BOOLEAN NOT NULL DEFAULT false,
    completed_at                TIMESTAMPTZ,                -- actual completion timestamp
    completed                   BOOLEAN NOT NULL DEFAULT false,
    custom_fields               JSONB DEFAULT '[]'::jsonb,
    dentally_created_at         TIMESTAMPTZ,                -- Dentally's "created_at"
    duration                    INTEGER NOT NULL DEFAULT 0,
    import_id                   BIGINT,
    invoice_id                  BIGINT,                     -- Dentally's "invoice_id"
    nhs_treatment_cat           VARCHAR(50),
    nomenclature                VARCHAR(500) NOT NULL DEFAULT '',
    notes                       TEXT NOT NULL DEFAULT '',
    patient_nomenclature        VARCHAR(500) NOT NULL DEFAULT '',
    payment_plan_id             INTEGER,
    position                    BIGINT,
    practitioner_id             INTEGER,                    -- Dentally's "practitioner_id"
    price                       DECIMAL(10,2) NOT NULL DEFAULT 0,
    referrer_id                 INTEGER,
    region                      VARCHAR(50) NOT NULL DEFAULT '',
    surfaces                    JSONB,
    teeth                       JSONB DEFAULT '[]'::jsonb,
    treatment_appointment_id    BIGINT,                     -- Dentally's appointment link
    treatment_id                BIGINT,
    treatment_plan_id           BIGINT,
    uda_band                    VARCHAR(20),
    dentally_updated_at         TIMESTAMPTZ,                -- Dentally's "updated_at"

    -- Our timestamps
    last_synced_at              TIMESTAMPTZ,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Constraints
    CONSTRAINT fk_recall_treatment_practice
        FOREIGN KEY (practice_id) REFERENCES "UserAuthentication_practice"(id) DEFERRABLE INITIALLY DEFERRED,
    CONSTRAINT fk_recall_treatment_patient
        FOREIGN KEY (recall_patient_id) REFERENCES recall_patient(id) DEFERRABLE INITIALLY DEFERRED,
    CONSTRAINT uniq_recall_treatment_per_practice
        UNIQUE (practice_id, dentally_id)
);

-- Indexes
CREATE INDEX idx_rtreat_practice_patient ON recall_treatment (practice_id, dentally_patient_id);
CREATE INDEX idx_rtreat_patient_nomenclature ON recall_treatment (dentally_patient_id, nomenclature);
CREATE INDEX idx_rtreat_completed ON recall_treatment (practice_id, completed, completed_at);
CREATE INDEX idx_rtreat_recall_patient_id ON recall_treatment (recall_patient_id);
```

### Changes to `recall_patient`

Remove these columns (they are derived data, not source data):
- `has_crown`
- `has_implant`
- `has_cosmetic`
- `has_sedation`
- `has_hygiene`
- `has_perio`
- `has_ortho`

Also remove: `total_spend`, `outstanding_balance`, `late_payment_count`, `late_payment_ratio` — these are financial signals that belong on `recall_record`, not on the patient source table.

**Note:** Removing columns from `recall_patient` is deferred. We will add the new table and patch the engine first, then clean up `recall_patient` in a follow-up to avoid breaking anything mid-change.

### Changes to `recall_record`

No schema changes needed — `recall_record` already has `has_crown`, `has_implant`, etc. These flags will now be computed from `recall_treatment` + `nomenclature_classification` at generation time instead of being copied from `recall_patient`.

### Recall Engine Changes

**File: `recall_engine.go`**

1. **`fetchTreatmentFlags`** — After fetching treatment items from Dentally API, upsert each item into `recall_treatment`. Continue returning `treatmentFlags` as before for the current sync pass.

2. **`RunRecallGeneration`** (local-only regeneration) — Instead of reading `has_*` flags from `recall_patient`, compute them by querying `recall_treatment` joined with `nomenclature_classification`.

3. **New method: `upsertRecallTreatment`** — Upserts a single treatment plan item into `recall_treatment`, using ON CONFLICT on `(practice_id, dentally_id)`.

4. **New method: `computeTreatmentFlagsFromLocal`** — Queries `recall_treatment` + `nomenclature_classification` for a patient and returns `treatmentFlags`.

### Backfill Script

Standalone bash script that:
1. Reads all distinct `dentally_patient_id` values from `recall_patient` for a given practice
2. For each patient, calls Dentally API `/treatment_plan_items?patient_id=X` (paginated)
3. Upserts results into `recall_treatment`

This is a one-time script to populate `recall_treatment` for existing patients. After this, the recall engine keeps it updated during regular syncs.

## Data Flow (After)

```
Dentally API /patients         → recall_patient (pure patient data, no flags)
Dentally API /appointments     → recall_appointment
Dentally API /treatment_items  → recall_treatment (pure Dentally data)
Dentally API /payments         → recall_record (total_spend, late_payment_*)
Dentally API /invoices         → recall_record (late_payment_*)

recall_treatment + nomenclature_classification → has_* flags → recall_record
recall_appointment → appointment stats → recall_record
recall_patient + all above → scores, segments, priority → recall_record
```

## Migration Strategy

1. Create `recall_treatment` table (additive, no breaking changes)
2. Patch recall engine to write to `recall_treatment` during sync
3. Patch recall generation to compute flags from `recall_treatment`
4. Run backfill script on dev
5. (Future) Remove `has_*` columns from `recall_patient`
