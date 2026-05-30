# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Dental Practice Management · Created: 2026-05-25

## Philosophy

A dental practice generates a natural event stream: patients check in, hygienists chart periodontal measurements tooth by tooth, dentists record findings per tooth, treatment plans are presented and accepted, procedures are completed, charges are posted, claims are submitted, and remittances arrive. Each of these steps is a discrete, immutable event. An event-sourced architecture stores every state change as an event in an append-only store, and the current state of any entity — a patient's tooth chart, an encounter's findings, a claim's lifecycle — is derived by replaying the event stream.

This approach solves several hard problems in dental software. HIPAA requires immutable audit trails for every access to PHI. Clinical notes must support amendments without destroying the original (malpractice liability). Insurance claim denials need full lifecycle tracing for appeal management. Treatment plan acceptance, modification, and declination must be tracked for patient communication and financial forecasting. The event store provides all of this by construction: the audit trail is not a secondary log — it is the primary data.

For DSO-scale operations, the event stream enables real-time analytics projections: production per provider, procedure mix by location, claim denial rates by payer, hygiene reappointment rates — all computed by projecting events into materialised read models that serve specific reporting queries without impacting the operational database.

**Best for:** DSOs and multi-location practices requiring comprehensive HIPAA audit trails, insurance claim lifecycle tracking with full history, treatment plan acceptance analytics, and real-time production dashboards — all derived from a single immutable event stream.

**Trade-offs:**
- **Pro:** Complete, immutable HIPAA audit trail — compliance by construction
- **Pro:** Clinical note amendments preserve originals with reason (malpractice protection)
- **Pro:** Insurance claim lifecycle fully traceable (every status change with context)
- **Pro:** Treatment plan acceptance/declination tracking for patient engagement analytics
- **Pro:** Production dashboards and KPIs computed from event projections
- **Pro:** Temporal queries ("what was the tooth chart on date X?") are natural
- **Con:** Read models must be maintained and rebuilt when projections change
- **Con:** Higher write amplification — every tooth finding is an individual event
- **Con:** Perio charting generates many events (up to 192 per full-mouth exam)
- **Con:** Team must understand CQRS pattern — steeper learning curve
- **Con:** Event schema evolution requires versioning discipline

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CDT 2026 | Procedure events carry CDT codes; code table updates are versioned events |
| SNODENT | Finding events reference SNODENT concept IDs |
| ADA Universal Numbering | Tooth numbers in charting and procedure events |
| ANSI X12 837D | Claim submission events reference EDI transaction IDs |
| ANSI X12 835 | ERA receipt events parse 835 remittance data |
| ANSI X12 270/271 | Eligibility events store inquiry/response pairs |
| HL7 FHIR R4 | Read models project encounter events into FHIR Procedure and Claim resources |
| FHIR Dental Data Exchange IG | Referral/consultation events exportable as FHIR dental profiles |
| DICOM | Imaging events carry DICOM Study UIDs |
| HIPAA | Immutable event store satisfies audit trail requirement |
| CloudEvents v1.0 | Event envelope follows CloudEvents spec |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    sequence_number BIGINT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/dental-practice',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    created_by UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE TABLE event_store_2026_h1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_h2 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_creator ON event_store(created_by, created_at DESC);
CREATE INDEX idx_events_data ON event_store USING GIN (event_data jsonb_path_ops);
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_types (event_type, stream_type, description) VALUES
-- Patient stream
('patient.registered', 'patient', 'New patient registration with demographics'),
('patient.demographics_updated', 'patient', 'Patient demographics changed'),
('patient.insurance_added', 'patient', 'Insurance coverage added'),
('patient.insurance_updated', 'patient', 'Insurance coverage modified'),
('patient.insurance_removed', 'patient', 'Insurance coverage terminated'),
('patient.history_updated', 'patient', 'Medical/dental history updated'),
('patient.allergy_added', 'patient', 'Allergy recorded'),
('patient.medication_updated', 'patient', 'Current medications updated'),
('patient.recall_scheduled', 'patient', 'Recall appointment scheduled'),
('patient.deactivated', 'patient', 'Patient marked inactive'),

-- Appointment stream
('appointment.scheduled', 'appointment', 'Appointment created'),
('appointment.confirmed', 'appointment', 'Patient confirmed'),
('appointment.checked_in', 'appointment', 'Patient checked in'),
('appointment.roomed', 'appointment', 'Patient taken to operatory'),
('appointment.completed', 'appointment', 'Appointment completed'),
('appointment.cancelled', 'appointment', 'Appointment cancelled'),
('appointment.no_show', 'appointment', 'Patient did not show'),
('appointment.rescheduled', 'appointment', 'Moved to new time'),

-- Encounter stream
('encounter.started', 'encounter', 'Clinical encounter opened'),
('encounter.chief_complaint_recorded', 'encounter', 'Chief complaint documented'),
('encounter.tooth_finding_recorded', 'encounter', 'Finding recorded for specific tooth'),
('encounter.tooth_status_changed', 'encounter', 'Tooth status updated (missing, implant, etc.)'),
('encounter.soft_tissue_examined', 'encounter', 'Soft tissue / oral cancer screening'),
('encounter.perio_started', 'encounter', 'Periodontal charting begun'),
('encounter.perio_tooth_charted', 'encounter', 'Perio measurements for one tooth (6 sites)'),
('encounter.perio_completed', 'encounter', 'Periodontal charting finished with summary'),
('encounter.radiograph_captured', 'encounter', 'Radiograph taken or imported'),
('encounter.radiograph_ai_analysed', 'encounter', 'AI analysis completed on radiograph'),
('encounter.procedure_performed', 'encounter', 'Clinical procedure completed with CDT code'),
('encounter.note_documented', 'encounter', 'Clinical notes written'),
('encounter.note_amended', 'encounter', 'Clinical note amended with reason'),
('encounter.signed', 'encounter', 'Encounter signed by provider'),
('encounter.locked', 'encounter', 'Encounter locked — no further edits'),
('encounter.addendum_added', 'encounter', 'Addendum to locked encounter'),

-- Treatment plan stream
('treatment_plan.created', 'treatment_plan', 'Treatment plan drafted'),
('treatment_plan.procedure_added', 'treatment_plan', 'Procedure added to plan'),
('treatment_plan.procedure_removed', 'treatment_plan', 'Procedure removed from plan'),
('treatment_plan.presented', 'treatment_plan', 'Plan presented to patient'),
('treatment_plan.accepted', 'treatment_plan', 'Patient accepted plan'),
('treatment_plan.partially_accepted', 'treatment_plan', 'Patient accepted some procedures'),
('treatment_plan.declined', 'treatment_plan', 'Patient declined plan'),
('treatment_plan.procedure_completed', 'treatment_plan', 'Planned procedure performed'),
('treatment_plan.expired', 'treatment_plan', 'Plan expired without completion'),

-- Claim stream
('claim.drafted', 'claim', 'Insurance claim drafted'),
('claim.line_added', 'claim', 'Service line added with CDT code'),
('claim.line_modified', 'claim', 'Service line modified'),
('claim.submitted', 'claim', 'Claim submitted via EDI 837D'),
('claim.accepted', 'claim', 'Payer accepted claim for processing'),
('claim.rejected', 'claim', 'Claim rejected — EDI format error'),
('claim.adjudicated', 'claim', 'Payer adjudicated claim'),
('claim.paid', 'claim', 'Full payment received from payer'),
('claim.partially_paid', 'claim', 'Partial payment received'),
('claim.denied', 'claim', 'Claim denied with reason'),
('claim.appealed', 'claim', 'Appeal submitted'),
('claim.appeal_decided', 'claim', 'Appeal decision received'),
('claim.voided', 'claim', 'Claim voided'),
('claim.era_received', 'claim', 'ERA 835 remittance received'),

-- Payment stream
('payment.collected', 'payment', 'Patient payment collected'),
('payment.refunded', 'payment', 'Refund issued'),
('payment.insurance_posted', 'payment', 'Insurance payment posted from ERA'),

-- Eligibility stream
('eligibility.checked', 'eligibility', 'Eligibility verified via 270/271'),
('eligibility.active', 'eligibility', 'Patient eligible — benefits confirmed'),
('eligibility.inactive', 'eligibility', 'Patient not eligible'),
('eligibility.prior_auth_requested', 'eligibility', 'Prior authorisation submitted'),
('eligibility.prior_auth_approved', 'eligibility', 'Prior authorisation approved'),
('eligibility.prior_auth_denied', 'eligibility', 'Prior authorisation denied'),

-- Access stream (HIPAA)
('access.patient_chart_viewed', 'access', 'User viewed patient chart'),
('access.phi_exported', 'access', 'PHI data exported'),
('access.phi_printed', 'access', 'PHI data printed'),
('access.break_glass', 'access', 'Emergency access to restricted record');
```

---

## Stream Snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    snapshot_data JSONB NOT NULL,
    last_sequence_number BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Projection Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data

```sql
CREATE TABLE cdt_codes (
    code TEXT PRIMARY KEY,
    category TEXT NOT NULL,
    subcategory TEXT,
    description TEXT NOT NULL,
    version_year INT NOT NULL DEFAULT 2026,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cdt_category ON cdt_codes(category);
```

---

## Read Model: Patients

```sql
CREATE TABLE rm_patients (
    id UUID PRIMARY KEY,
    practice_id UUID NOT NULL,
    mrn TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    date_of_birth DATE NOT NULL,
    sex TEXT,
    contact JSONB NOT NULL DEFAULT '{}',
    medical_history JSONB NOT NULL DEFAULT '{}',
    tooth_chart JSONB NOT NULL DEFAULT '{}',
    active_insurance JSONB NOT NULL DEFAULT '[]',
    recall JSONB NOT NULL DEFAULT '{}',
    last_visit_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_rm_patients_practice ON rm_patients(practice_id);
CREATE INDEX idx_rm_patients_name ON rm_patients(practice_id, last_name, first_name);
CREATE INDEX idx_rm_patients_recall ON rm_patients(practice_id, (recall->>'next_date'))
    WHERE (recall->>'next_date') IS NOT NULL AND is_active = TRUE;
```

---

## Read Model: Encounters

```sql
CREATE TABLE rm_encounters (
    id UUID PRIMARY KEY,
    appointment_id UUID,
    patient_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    location_id UUID NOT NULL,
    encounter_type TEXT NOT NULL,
    status TEXT NOT NULL,
    chief_complaint TEXT,
    clinical_notes TEXT,
    findings JSONB NOT NULL DEFAULT '{}',
    perio_data JSONB,
    procedures JSONB NOT NULL DEFAULT '[]',
    imaging JSONB NOT NULL DEFAULT '[]',
    amendments JSONB NOT NULL DEFAULT '[]',
    signed_by UUID,
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_enc_patient ON rm_encounters(patient_id, created_at DESC);
CREATE INDEX idx_rm_enc_provider ON rm_encounters(provider_id, created_at DESC);
```

---

## Read Model: Treatment Plans

```sql
CREATE TABLE rm_treatment_plans (
    id UUID PRIMARY KEY,
    patient_id UUID NOT NULL,
    patient_name TEXT NOT NULL,
    provider_id UUID NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL,
    procedures JSONB NOT NULL DEFAULT '[]',
    total_fee_cents BIGINT NOT NULL DEFAULT 0,
    insurance_estimate_cents BIGINT NOT NULL DEFAULT 0,
    patient_estimate_cents BIGINT NOT NULL DEFAULT 0,
    timeline JSONB NOT NULL DEFAULT '[]',
    -- timeline example:
    -- [
    --   {"event": "treatment_plan.created", "at": "2026-05-25T10:30:00Z", "by": "provider-uuid"},
    --   {"event": "treatment_plan.presented", "at": "2026-05-25T10:35:00Z"},
    --   {"event": "treatment_plan.accepted", "at": "2026-05-25T10:40:00Z", "notes": "Patient accepted phase 1"},
    --   {"event": "treatment_plan.procedure_completed", "at": "2026-06-10T14:00:00Z", "cdt": "D2391", "tooth": "14"}
    -- ]
    presented_at TIMESTAMPTZ,
    accepted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_tp_patient ON rm_treatment_plans(patient_id, created_at DESC);
CREATE INDEX idx_rm_tp_status ON rm_treatment_plans(status)
    WHERE status IN ('presented', 'accepted', 'in_progress');
```

---

## Read Model: Claims Lifecycle

```sql
CREATE TABLE rm_claims_lifecycle (
    id UUID PRIMARY KEY,
    encounter_id UUID NOT NULL,
    patient_id UUID NOT NULL,
    patient_name TEXT NOT NULL,
    location_id UUID NOT NULL,
    claim_number TEXT,
    payer_name TEXT NOT NULL,
    status TEXT NOT NULL,
    service_lines JSONB NOT NULL DEFAULT '[]',
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial JSONB,
    timeline JSONB NOT NULL DEFAULT '[]',
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_claims_patient ON rm_claims_lifecycle(patient_id, created_at DESC);
CREATE INDEX idx_rm_claims_status ON rm_claims_lifecycle(status)
    WHERE status NOT IN ('paid', 'voided');
CREATE INDEX idx_rm_claims_payer ON rm_claims_lifecycle(payer_name, status);
```

---

## Read Model: Production Dashboard

```sql
CREATE TABLE rm_production_daily (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL,
    location_id UUID NOT NULL,
    provider_id UUID NOT NULL,
    production_date DATE NOT NULL,
    procedure_count INT NOT NULL DEFAULT 0,
    production_cents BIGINT NOT NULL DEFAULT 0,
    collections_cents BIGINT NOT NULL DEFAULT 0,
    adjustments_cents BIGINT NOT NULL DEFAULT 0,
    procedures_by_category JSONB NOT NULL DEFAULT '{}',
    -- procedures_by_category example:
    -- {"diagnostic": {"count": 3, "cents": 23500}, "preventive": {"count": 2, "cents": 19000},
    --  "restorative": {"count": 1, "cents": 22000}}
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (location_id, provider_id, production_date)
);

CREATE INDEX idx_rm_prod_practice ON rm_production_daily(practice_id, production_date);
CREATE INDEX idx_rm_prod_provider ON rm_production_daily(provider_id, production_date);
```

---

## Read Model: Perio Trends

```sql
CREATE TABLE rm_perio_trends (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL,
    measurements JSONB NOT NULL DEFAULT '[]',
    -- measurements example:
    -- [
    --   {"date": "2025-05-20", "exam_type": "full", "avg_probing": 3.1, "sites_bleeding": 8, "sites_total": 180,
    --    "max_probing": 5, "pockets_4plus": 6, "pockets_5plus": 1},
    --   {"date": "2026-05-25", "exam_type": "full", "avg_probing": 2.9, "sites_bleeding": 3, "sites_total": 180,
    --    "max_probing": 4, "pockets_4plus": 3, "pockets_5plus": 0}
    -- ]
    trend_direction TEXT CHECK (trend_direction IN ('stable', 'improving', 'worsening', 'insufficient_data')),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (patient_id)
);

CREATE INDEX idx_rm_perio_patient ON rm_perio_trends(patient_id);
```

---

## Example Queries

### Replay encounter from events

```sql
SELECT event_type, event_data, created_by, created_at
FROM event_store
WHERE stream_type = 'encounter'
  AND stream_id = 'encounter-uuid'
ORDER BY sequence_number;
```

### Treatment plan acceptance rate

```sql
SELECT COUNT(*) AS total_presented,
       COUNT(*) FILTER (WHERE status = 'accepted') AS accepted,
       COUNT(*) FILTER (WHERE status = 'declined') AS declined,
       ROUND(COUNT(*) FILTER (WHERE status = 'accepted') * 100.0 / NULLIF(COUNT(*), 0), 1) AS acceptance_pct
FROM rm_treatment_plans
WHERE provider_id = 'provider-uuid'
  AND created_at >= CURRENT_DATE - 90;
```

### Production by provider for month

```sql
SELECT pr.provider_id, SUM(pr.production_cents) AS production,
       SUM(pr.collections_cents) AS collections,
       SUM(pr.procedure_count) AS procedures
FROM rm_production_daily pr
WHERE pr.practice_id = 'practice-uuid'
  AND pr.production_date >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY pr.provider_id
ORDER BY production DESC;
```

### Amendment history for encounter

```sql
SELECT event_data->>'original_text' AS original,
       event_data->>'amended_text' AS amended,
       event_data->>'reason' AS reason,
       created_by, created_at
FROM event_store
WHERE stream_type = 'encounter'
  AND stream_id = 'encounter-uuid'
  AND event_type = 'encounter.note_amended'
ORDER BY created_at;
```

### HIPAA access audit

```sql
SELECT event_data->>'user_name' AS who,
       event_type,
       event_data->>'patient_name' AS patient,
       created_at
FROM event_store
WHERE stream_type = 'access'
  AND created_at >= CURRENT_DATE - 30
ORDER BY created_at DESC;
```

---

## Event-Driven Automation Patterns

### Auto-Charge Capture

When `encounter.procedure_performed` fires, the charge projection:
1. Looks up the CDT code fee from the practice fee schedule
2. Emits `claim.drafted` with pre-populated service lines
3. Updates `rm_claims_lifecycle` in draft status

### Recall Scheduling

When `encounter.signed` fires with a cleaning procedure, the recall projection:
1. Reads the patient's recall interval (default 6 months)
2. Calculates `next_recall_date`
3. Emits `patient.recall_scheduled`
4. Updates `rm_patients.recall`

### Tooth Chart Auto-Update

When `encounter.procedure_performed` fires with a restorative CDT code:
1. The tooth chart projection updates the tooth's conditions and restorations
2. Updates `rm_patients.tooth_chart`
3. The chart always reflects the latest procedures

### Treatment Plan Tracking

When `encounter.procedure_performed` fires with a `treatment_plan_procedure_id`:
1. Emits `treatment_plan.procedure_completed`
2. Updates `rm_treatment_plans` procedure status
3. If all procedures complete, updates plan status to `completed`

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), event_types, stream_snapshots |
| Projection Infrastructure | 1 | projection_checkpoints |
| Reference Data | 1 | cdt_codes |
| Read Model: Patients | 1 | rm_patients (demographics, tooth chart, insurance, recall) |
| Read Model: Encounters | 1 | rm_encounters (findings, perio, procedures, imaging) |
| Read Model: Treatment Plans | 1 | rm_treatment_plans (lifecycle timeline) |
| Read Model: Claims | 1 | rm_claims_lifecycle (full claim timeline) |
| Read Model: Production | 1 | rm_production_daily (provider-level daily stats) |
| Read Model: Perio Trends | 1 | rm_perio_trends (longitudinal perio data) |
| **Total** | **12** | 5 infrastructure + 7 read models |

---

## Key Design Decisions

1. **Granular tooth-level events** — Each tooth finding, status change, and procedure is a separate event with tooth number and surfaces. This enables reconstruction of the tooth chart at any point in time ("what did tooth #14 look like before the filling?") and per-tooth audit trails.

2. **Perio charting as per-tooth events** — `encounter.perio_tooth_charted` events carry 6-site measurements for one tooth. A full-mouth exam generates up to 32 events. The `rm_perio_trends` read model pre-computes summary statistics (average probing depth, bleeding sites, pockets ≥4mm) for longitudinal trending.

3. **Treatment plan lifecycle as events** — Creation, presentation, acceptance, procedure completion, and expiration are all events. The `rm_treatment_plans` read model includes a `timeline` array showing the complete history. This enables treatment plan acceptance rate analytics and patient engagement scoring.

4. **Production dashboard as a daily read model** — `rm_production_daily` aggregates per-provider daily production, collections, and procedure mix. This avoids expensive event replays for the KPI dashboards that DSO executives check daily. The projection updates incrementally as procedure events arrive.

5. **Amendment events preserve originals** — `encounter.note_amended` events carry `original_text`, `amended_text`, and mandatory `reason`. The encounter's event stream is the definitive clinical record. The `rm_encounters` read model shows the current version with an `amendments` array for history.

6. **HIPAA access events as a separate stream** — Chart views, PHI exports, and break-glass access are events on the `access` stream type. These never merge into clinical read models — they're queried directly from the event store for compliance audits.

7. **CDT codes as relational reference** — Despite the event-sourced architecture, CDT codes remain a separate relational table because they're externally managed reference data. Procedure events carry the CDT code as a string; the reference table provides category lookups and annual version tracking.

8. **Auto-charge and recall from event projections** — Charge capture, recall scheduling, and tooth chart updates are implemented as event projections. When a procedure event fires, the charge projection generates a claim draft event. This makes the automation auditable and recoverable.
