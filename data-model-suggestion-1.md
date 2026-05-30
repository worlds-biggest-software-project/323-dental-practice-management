# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Dental Practice Management · Created: 2026-05-25

## Philosophy

A dental practice management platform orchestrates a complex clinical and financial pipeline: practices operate across locations with operatories, providers see patients on scheduled appointments, clinical encounters produce tooth-level charting and periodontal measurements, treatment plans sequence procedures across visits, completed procedures generate charges, charges flow into insurance claims via X12 837D, and remittance arrives via X12 835. A normalized relational model gives each concept — tooth, procedure, treatment plan, claim, payment — its own table with database-enforced referential integrity.

This mirrors how dental software actually works: the front desk schedules and verifies eligibility (270/271), the hygienist charts periodontal measurements, the dentist examines and records findings per tooth, a treatment plan is created with phased procedures, completed procedures are coded with CDT codes, claims are submitted to payers, and payments are posted. Each step maps to a table. CDT procedure codes, SNODENT diagnostic terms, and ADA tooth numbering (Universal Numbering System) are first-class reference tables, not embedded strings.

**Best for:** Teams building a HIPAA-compliant dental PMS where the charting → treatment plan → procedure → claim → payment pipeline needs strict relational integrity, where CDT code updates must be managed as reference data, and where multi-location DSO reporting requires consistent schema across all sites.

**Trade-offs:**
- **Pro:** Database-enforced clinical and financial pipeline integrity
- **Pro:** CDT codes as a reference table — annual updates are a data operation, not a schema change
- **Pro:** Tooth-level charting with explicit per-tooth procedure history
- **Pro:** Periodontal measurements as typed rows enable hygiene trending
- **Pro:** Insurance claims with line-level CDT codes enable denial analysis
- **Pro:** Multi-location operatory scheduling with provider assignments
- **Con:** 25+ tables — high complexity for a solo-practice deployment
- **Con:** Tooth charting requires many per-tooth rows for a full-mouth exam
- **Con:** High join count for "full patient chart" views
- **Con:** Periodontal charting (6 sites × 32 teeth = 192 probing depths per exam) generates high row counts

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CDT 2026 | `cdt_codes` reference table with annual version tracking |
| SNODENT | `snodent_terms` reference table for clinical finding terminology |
| ADA Universal Numbering | `tooth_number` columns use 1-32 (permanent) and A-T (primary) |
| ANSI X12 837D | Claims table structure maps to 837D transaction segments |
| ANSI X12 835 | ERA parsing maps to `claim_lines` adjudication columns |
| ANSI X12 270/271 | Eligibility verification results stored on `eligibility_checks` |
| HL7 FHIR R4 | Patient, Encounter, Procedure, Claim resources exportable from relational tables |
| FHIR Dental Data Exchange IG | Dental referral/consultation notes exportable from encounters |
| DICOM | Radiograph references stored with DICOM Study UID |
| HIPAA | Audit log, access controls, encryption requirements |
| NPI | Provider and practice NPI stored on `providers` and `practices` |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ISO 4217 | Currency codes for billing |

---

## Practice, Locations & Providers

```sql
CREATE TABLE practices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    tax_id TEXT,
    npi TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city TEXT NOT NULL,
    state TEXT NOT NULL,
    postal_code TEXT NOT NULL,
    country TEXT NOT NULL DEFAULT 'US',
    phone TEXT,
    fax TEXT,
    place_of_service_code TEXT NOT NULL DEFAULT '11',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_locations_practice ON locations(practice_id);

CREATE TABLE operatories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    operatory_type TEXT NOT NULL DEFAULT 'general' CHECK (operatory_type IN (
        'general', 'hygiene', 'surgery', 'pediatric', 'ortho'
    )),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_operatories_location ON operatories(location_id);

CREATE TABLE providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN (
        'dentist', 'hygienist', 'oral_surgeon', 'orthodontist',
        'periodontist', 'endodontist', 'pedodontist',
        'assistant', 'receptionist', 'billing', 'admin'
    )),
    npi TEXT,
    license_number TEXT,
    license_state TEXT,
    dea_number TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_practice ON providers(practice_id);

CREATE TABLE provider_locations (
    provider_id UUID NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    PRIMARY KEY (provider_id, location_id)
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

CREATE TABLE snodent_terms (
    concept_id TEXT PRIMARY KEY,
    term TEXT NOT NULL,
    domain TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Patients

```sql
CREATE TABLE patients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    mrn TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    date_of_birth DATE NOT NULL,
    sex TEXT CHECK (sex IN ('male', 'female', 'other', 'unknown')),
    email TEXT,
    phone TEXT,
    phone_secondary TEXT,
    address_line1 TEXT,
    address_line2 TEXT,
    city TEXT,
    state TEXT,
    postal_code TEXT,
    country TEXT NOT NULL DEFAULT 'US',
    preferred_language TEXT DEFAULT 'en',
    ssn_last4 TEXT,
    emergency_contact_name TEXT,
    emergency_contact_phone TEXT,
    allergies TEXT[],
    medications TEXT[],
    medical_history TEXT[],
    dental_history TEXT[],
    preferred_provider_id UUID REFERENCES providers(id),
    recall_interval_months INT DEFAULT 6,
    next_recall_date DATE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_patients_practice ON patients(practice_id);
CREATE INDEX idx_patients_name ON patients(practice_id, last_name, first_name);
CREATE INDEX idx_patients_dob ON patients(practice_id, date_of_birth);
CREATE INDEX idx_patients_recall ON patients(practice_id, next_recall_date)
    WHERE next_recall_date IS NOT NULL AND is_active = TRUE;
```

---

## Insurance

```sql
CREATE TABLE insurance_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    payer_name TEXT NOT NULL,
    payer_id TEXT NOT NULL,
    plan_type TEXT NOT NULL DEFAULT 'dental' CHECK (plan_type IN ('dental', 'medical')),
    edi_payer_id TEXT,
    fee_schedule_name TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patient_insurances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
    plan_id UUID NOT NULL REFERENCES insurance_plans(id),
    priority TEXT NOT NULL DEFAULT 'primary' CHECK (priority IN (
        'primary', 'secondary', 'tertiary'
    )),
    subscriber_id TEXT NOT NULL,
    subscriber_name TEXT,
    group_number TEXT,
    effective_date DATE,
    termination_date DATE,
    annual_max_cents BIGINT,
    remaining_max_cents BIGINT,
    deductible_cents BIGINT,
    deductible_met_cents BIGINT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_insurance ON patient_insurances(patient_id);

CREATE TABLE eligibility_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_insurance_id UUID NOT NULL REFERENCES patient_insurances(id),
    checked_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL CHECK (status IN ('active', 'inactive', 'error')),
    response_data JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_eligibility ON eligibility_checks(patient_insurance_id, checked_at DESC);
```

---

## Scheduling

```sql
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL REFERENCES locations(id),
    operatory_id UUID NOT NULL REFERENCES operatories(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    appointment_type TEXT NOT NULL DEFAULT 'exam' CHECK (appointment_type IN (
        'exam', 'cleaning', 'restorative', 'crown', 'root_canal',
        'extraction', 'implant', 'ortho', 'pediatric',
        'emergency', 'consultation', 'follow_up'
    )),
    status TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
        'scheduled', 'confirmed', 'checked_in', 'in_operatory',
        'completed', 'cancelled', 'no_show'
    )),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end TIMESTAMPTZ NOT NULL,
    actual_start TIMESTAMPTZ,
    actual_end TIMESTAMPTZ,
    reason TEXT,
    notes TEXT,
    confirmed_at TIMESTAMPTZ,
    confirmation_method TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appts_location ON appointments(location_id, scheduled_start);
CREATE INDEX idx_appts_operatory ON appointments(operatory_id, scheduled_start);
CREATE INDEX idx_appts_provider ON appointments(provider_id, scheduled_start);
CREATE INDEX idx_appts_patient ON appointments(patient_id, scheduled_start DESC);
```

---

## Encounters & Clinical Charting

```sql
CREATE TABLE encounters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    encounter_type TEXT NOT NULL DEFAULT 'exam',
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'completed', 'addendum', 'locked'
    )),
    chief_complaint TEXT,
    clinical_notes TEXT,
    signed_by UUID REFERENCES providers(id),
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id, created_at DESC);
CREATE INDEX idx_encounters_provider ON encounters(provider_id, created_at DESC);

CREATE TABLE tooth_chart (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    tooth_number TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'present' CHECK (status IN (
        'present', 'missing', 'impacted', 'unerupted',
        'primary', 'supernumerary', 'implant'
    )),
    conditions TEXT[] NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (patient_id, tooth_number)
);

CREATE INDEX idx_tooth_chart ON tooth_chart(patient_id);

CREATE TABLE tooth_findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id) ON DELETE CASCADE,
    tooth_number TEXT NOT NULL,
    surface TEXT,
    finding_type TEXT NOT NULL CHECK (finding_type IN (
        'caries', 'fracture', 'wear', 'erosion', 'abscess',
        'restoration_existing', 'restoration_defective', 'crown_existing',
        'root_canal_existing', 'mobility', 'furcation', 'recession',
        'bleeding_on_probing', 'suppuration', 'other'
    )),
    severity TEXT CHECK (severity IN ('mild', 'moderate', 'severe')),
    notes TEXT,
    snodent_concept_id TEXT REFERENCES snodent_terms(concept_id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_encounter ON tooth_findings(encounter_id);
CREATE INDEX idx_findings_tooth ON tooth_findings(tooth_number);

CREATE TABLE perio_exams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id) ON DELETE CASCADE,
    patient_id UUID NOT NULL REFERENCES patients(id),
    exam_type TEXT NOT NULL DEFAULT 'full' CHECK (exam_type IN ('full', 'limited')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_perio_encounter ON perio_exams(encounter_id);

CREATE TABLE perio_measurements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    perio_exam_id UUID NOT NULL REFERENCES perio_exams(id) ON DELETE CASCADE,
    tooth_number TEXT NOT NULL,
    site TEXT NOT NULL CHECK (site IN (
        'db', 'b', 'mb', 'dl', 'l', 'ml'
    )),
    probing_depth INT,
    recession INT,
    bleeding_on_probing BOOLEAN DEFAULT FALSE,
    suppuration BOOLEAN DEFAULT FALSE,
    furcation INT CHECK (furcation BETWEEN 0 AND 3),
    mobility INT CHECK (mobility BETWEEN 0 AND 3),
    plaque BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_perio_measurements ON perio_measurements(perio_exam_id);
```

---

## Treatment Plans & Procedures

```sql
CREATE TABLE treatment_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    name TEXT NOT NULL DEFAULT 'Treatment Plan',
    status TEXT NOT NULL DEFAULT 'proposed' CHECK (status IN (
        'proposed', 'presented', 'accepted', 'in_progress',
        'completed', 'declined', 'expired'
    )),
    presented_at TIMESTAMPTZ,
    accepted_at TIMESTAMPTZ,
    notes TEXT,
    total_fee_cents BIGINT NOT NULL DEFAULT 0,
    insurance_estimate_cents BIGINT NOT NULL DEFAULT 0,
    patient_estimate_cents BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tp_patient ON treatment_plans(patient_id, created_at DESC);

CREATE TABLE treatment_plan_procedures (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    treatment_plan_id UUID NOT NULL REFERENCES treatment_plans(id) ON DELETE CASCADE,
    cdt_code TEXT NOT NULL REFERENCES cdt_codes(code),
    tooth_number TEXT,
    surface TEXT,
    phase INT NOT NULL DEFAULT 1,
    priority TEXT NOT NULL DEFAULT 'recommended' CHECK (priority IN (
        'urgent', 'recommended', 'elective'
    )),
    status TEXT NOT NULL DEFAULT 'planned' CHECK (status IN (
        'planned', 'scheduled', 'completed', 'declined'
    )),
    fee_cents BIGINT NOT NULL,
    insurance_estimate_cents BIGINT NOT NULL DEFAULT 0,
    patient_estimate_cents BIGINT NOT NULL DEFAULT 0,
    notes TEXT,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tp_procedures ON treatment_plan_procedures(treatment_plan_id);

CREATE TABLE procedures_performed (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    treatment_plan_procedure_id UUID REFERENCES treatment_plan_procedures(id),
    cdt_code TEXT NOT NULL REFERENCES cdt_codes(code),
    tooth_number TEXT,
    surface TEXT,
    quadrant TEXT CHECK (quadrant IN ('UR', 'UL', 'LR', 'LL')),
    fee_cents BIGINT NOT NULL,
    notes TEXT,
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_procedures_encounter ON procedures_performed(encounter_id);
CREATE INDEX idx_procedures_patient ON procedures_performed(patient_id, performed_at DESC);
CREATE INDEX idx_procedures_cdt ON procedures_performed(cdt_code);
```

---

## Fee Schedules

```sql
CREATE TABLE fee_schedules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fee_schedule_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fee_schedule_id UUID NOT NULL REFERENCES fee_schedules(id) ON DELETE CASCADE,
    cdt_code TEXT NOT NULL REFERENCES cdt_codes(code),
    fee_cents BIGINT NOT NULL,
    insurance_allowable_cents BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fee_schedule_id, cdt_code)
);

CREATE INDEX idx_fee_entries ON fee_schedule_entries(fee_schedule_id);
```

---

## Insurance Claims

```sql
CREATE TABLE claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    insurance_id UUID NOT NULL REFERENCES patient_insurances(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    claim_number TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'accepted', 'rejected', 'denied',
        'paid', 'partially_paid', 'appealed', 'voided'
    )),
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial_reason TEXT,
    denial_code TEXT,
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id, created_at DESC);
CREATE INDEX idx_claims_status ON claims(status) WHERE status NOT IN ('paid', 'voided');

CREATE TABLE claim_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id UUID NOT NULL REFERENCES claims(id) ON DELETE CASCADE,
    procedure_id UUID NOT NULL REFERENCES procedures_performed(id),
    line_number INT NOT NULL,
    cdt_code TEXT NOT NULL,
    tooth_number TEXT,
    surface TEXT,
    description TEXT NOT NULL,
    units INT NOT NULL DEFAULT 1,
    charge_cents BIGINT NOT NULL,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_lines ON claim_lines(claim_id);
```

---

## Payments

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL REFERENCES locations(id),
    amount_cents BIGINT NOT NULL,
    payment_method TEXT NOT NULL CHECK (payment_method IN (
        'cash', 'credit_card', 'debit_card', 'check',
        'insurance_payment', 'care_credit', 'account_credit'
    )),
    reference_number TEXT,
    claim_id UUID REFERENCES claims(id),
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_patient ON payments(patient_id, created_at DESC);
```

---

## Imaging

```sql
CREATE TABLE radiographs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    image_type TEXT NOT NULL CHECK (image_type IN (
        'periapical', 'bitewing', 'panoramic', 'cephalometric',
        'cbct', 'intraoral_photo', 'extraoral_photo'
    )),
    tooth_numbers TEXT[],
    dicom_study_uid TEXT,
    storage_path TEXT NOT NULL,
    notes TEXT,
    ai_findings JSONB,
    captured_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_radiographs_encounter ON radiographs(encounter_id);
CREATE INDEX idx_radiographs_patient ON radiographs(patient_id, captured_at DESC);
```

---

## Audit Log & AI

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL,
    user_id UUID NOT NULL,
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    details JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_h1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_h2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2027-01-01');

CREATE INDEX idx_audit_practice ON audit_log(practice_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);

CREATE TABLE ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID REFERENCES encounters(id),
    patient_id UUID REFERENCES patients(id),
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'radiograph_detection', 'cdt_suggestion', 'treatment_plan_narrative',
        'no_show_prediction', 'fee_optimisation', 'clinical_note_draft'
    )),
    content TEXT NOT NULL,
    score REAL,
    details JSONB,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_encounter ON ai_analyses(encounter_id);
```

---

## Example Queries

### Full patient chart with tooth conditions

```sql
SELECT p.first_name, p.last_name,
       tc.tooth_number, tc.status, tc.conditions
FROM patients p
JOIN tooth_chart tc ON tc.patient_id = p.id
WHERE p.id = 'patient-uuid'
ORDER BY tc.tooth_number::INT;
```

### Periodontal trending for a tooth

```sql
SELECT pe.created_at::DATE AS exam_date,
       pm.site, pm.probing_depth, pm.recession,
       pm.bleeding_on_probing
FROM perio_measurements pm
JOIN perio_exams pe ON pe.id = pm.perio_exam_id
WHERE pe.patient_id = 'patient-uuid'
  AND pm.tooth_number = '3'
ORDER BY pe.created_at, pm.site;
```

### Production report by provider

```sql
SELECT pr.name AS provider,
       COUNT(*) AS procedures,
       SUM(pp.fee_cents) AS production_cents
FROM procedures_performed pp
JOIN providers pr ON pr.id = pp.provider_id
WHERE pp.performed_at >= CURRENT_DATE - 30
GROUP BY pr.id, pr.name
ORDER BY production_cents DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Practice & Providers | 5 | practices, locations, operatories, providers, provider_locations |
| Reference Data | 2 | cdt_codes, snodent_terms |
| Patients | 1 | patients |
| Insurance | 3 | insurance_plans, patient_insurances, eligibility_checks |
| Scheduling | 1 | appointments |
| Encounters & Charting | 5 | encounters, tooth_chart, tooth_findings, perio_exams, perio_measurements |
| Treatment Plans | 3 | treatment_plans, treatment_plan_procedures, procedures_performed |
| Fee Schedules | 2 | fee_schedules, fee_schedule_entries |
| Claims | 2 | claims, claim_lines |
| Payments | 1 | payments |
| Imaging | 1 | radiographs |
| Audit & AI | 2 | audit_log (partitioned), ai_analyses |
| **Total** | **28** | |

---

## Key Design Decisions

1. **CDT codes as a reference table** — The ADA updates CDT annually (31 new codes in 2026). Storing codes in `cdt_codes` with a `version_year` column means annual updates are INSERT/UPDATE operations on reference data, not schema migrations. Treatment plan procedures and claim lines reference CDT codes by foreign key.

2. **Tooth chart as persistent state** — `tooth_chart` stores the current condition of each tooth for a patient (present, missing, implant). `tooth_findings` stores per-encounter observations. This separation means the chart persists across visits while findings are encounter-specific.

3. **Periodontal measurements at the site level** — Each tooth has 6 probing sites (DB, B, MB, DL, L, ML). A full-mouth perio exam produces up to 192 measurement rows. This granularity enables site-level trending for periodontal treatment monitoring — essential for insurance documentation of periodontal disease progression.

4. **Treatment plans with phased procedures** — `treatment_plan_procedures` includes a `phase` column for sequencing work across visits and a `priority` column (urgent, recommended, elective). When a procedure is completed, `procedures_performed` links back to the treatment plan procedure, closing the loop.

5. **Operatories as a scheduling resource** — Dental scheduling is operatory-based (the room), not just provider-based. Appointments reference both `operatory_id` and `provider_id` because a provider may work across operatories and an operatory is shared among providers.

6. **Fee schedules per practice** — `fee_schedules` supports multiple named fee schedules (standard, PPO, Medicaid) with per-CDT-code fees. Insurance plans reference a fee schedule name for estimate calculation at treatment plan creation.

7. **Radiographs with AI findings** — `radiographs` stores image references with DICOM Study UIDs. The `ai_findings` JSONB column stores AI detection results (caries probability, bone loss measurements) from radiograph analysis, linked to the encounter for clinical context.

8. **HIPAA audit log** — Partitioned `audit_log` records every access to PHI with user, action, resource type/ID, and IP address. Half-yearly partitions enable retention management per HIPAA requirements.
