# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Dental Practice Management · Created: 2026-05-25

## Philosophy

Dental practice management straddles a highly structured financial pipeline (scheduling → procedure → claim → payment) and a highly variable clinical domain (tooth findings vary by exam type, periodontal charting has 6 sites × 32 teeth, radiograph AI findings vary by algorithm, treatment plan presentations vary by patient). A hybrid model keeps the financial pipeline relational — appointments, encounters, procedures, claims, payments have typed columns with foreign keys — while clinical detail, charting data, and variable configurations move into JSONB columns.

This is especially effective for dental charting because a full-mouth tooth chart is naturally a document: 32 teeth, each with conditions, restorations, and findings. Rather than 32+ rows in a tooth chart table plus hundreds of periodontal measurement rows, the chart becomes a single JSONB document on the patient record, and periodontal exams become JSONB arrays on the encounter. CDT code lookups remain relational (they're reference data updated annually), but tooth-level procedure detail uses JSONB to capture surface, quadrant, and tooth-specific notes without rigid column structures.

The hybrid approach also absorbs the variability in AI outputs: radiograph detection results from different AI models (Pearl, VideaHealth, custom) produce different field structures, and CDT coding suggestions have different confidence formats. JSONB columns on `ai_analyses` absorb this without per-model schemas.

**Best for:** Teams building a dental PMS for solo practices and small groups where schema simplicity matters, where tooth chart and perio data should be document-shaped for fast reads, and where the AI features produce variable output structures.

**Trade-offs:**
- **Pro:** 10 core tables vs. 28 in normalized — dramatically simpler
- **Pro:** Full-mouth tooth chart as a single JSONB document — one read, no joins
- **Pro:** Perio measurements inline on encounters — natural clinical document shape
- **Pro:** Treatment plans as self-contained JSONB documents with phased procedures
- **Pro:** New AI output formats require no schema migration
- **Con:** Tooth-level queries across patients require JSONB path extraction
- **Con:** Perio trending requires JSONB array unpacking instead of columnar queries
- **Con:** Application layer must validate clinical JSONB structures
- **Con:** CDT code validation on JSONB procedure entries requires application logic

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CDT 2026 | Reference table for code validation; JSONB procedures carry CDT codes as strings |
| SNODENT | Clinical finding terms referenced in tooth chart JSONB |
| ADA Universal Numbering | Tooth numbers as keys in chart JSONB (1-32, A-T) |
| ANSI X12 837D | Claim JSONB maps to 837D transaction segments |
| ANSI X12 835 | ERA data stored as JSONB on claims for reconciliation |
| ANSI X12 270/271 | Eligibility response stored as JSONB |
| HL7 FHIR R4 | Patient, Encounter, Procedure resources exportable from relational + JSONB |
| DICOM | Radiograph DICOM UIDs in imaging JSONB |
| HIPAA | Audit log kept relational for compliance |
| NPI | Provider NPI in credentials JSONB |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Practice & Providers

```sql
CREATE TABLE practices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    tax_id TEXT,
    npi TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "locations": [
    --     {"id": "loc-uuid", "name": "Main Office", "address": {...}, "phone": "...",
    --      "operatories": [{"id": "op-uuid", "name": "Op 1", "type": "general"}, {"id": "op-uuid-2", "name": "Hygiene 1", "type": "hygiene"}]}
    --   ],
    --   "insurance_plans": [
    --     {"id": "plan-uuid", "payer_name": "Delta Dental", "payer_id": "DD001", "edi_payer_id": "...", "fee_schedule": "ppo"}
    --   ],
    --   "fee_schedules": {
    --     "standard": {"D0120": 6500, "D0150": 9500, "D0210": 18000, "D1110": 11000, "D2391": 22000},
    --     "ppo": {"D0120": 5000, "D0150": 7500, "D0210": 14000, "D1110": 8500, "D2391": 18000}
    --   },
    --   "recall_defaults": {"interval_months": 6, "reminder_days": [30, 14, 3]}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    credentials JSONB NOT NULL DEFAULT '{}',
    -- credentials example:
    -- {
    --   "npi": "1234567890", "license_number": "DDS12345", "license_state": "CA",
    --   "dea_number": "FA1234567",
    --   "location_ids": ["loc-uuid-1"], "specialties": ["restorative", "cosmetic"]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_practice ON providers(practice_id);
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
    contact JSONB NOT NULL DEFAULT '{}',
    -- contact example:
    -- {
    --   "email": "jane@example.com", "phone": "555-0100",
    --   "address": {"line1": "123 Main", "city": "Portland", "state": "OR", "postal": "97201"},
    --   "preferred_language": "en",
    --   "emergency": {"name": "John Doe", "phone": "555-0102"}
    -- }
    medical_history JSONB NOT NULL DEFAULT '{}',
    -- medical_history example:
    -- {
    --   "allergies": ["penicillin", "latex"],
    --   "medications": ["metformin", "lisinopril"],
    --   "conditions": ["diabetes_type_2", "hypertension"],
    --   "dental_history": ["orthodontics_age_14", "wisdom_teeth_extracted_2018"]
    -- }
    tooth_chart JSONB NOT NULL DEFAULT '{}',
    -- tooth_chart example (Universal Numbering System):
    -- {
    --   "1": {"status": "missing", "conditions": ["extracted_2020"]},
    --   "2": {"status": "present", "conditions": ["amalgam_mo"], "restorations": [{"type": "amalgam", "surfaces": ["M","O"], "date": "2018-03-15"}]},
    --   "3": {"status": "present", "conditions": ["crown_pfm"], "restorations": [{"type": "crown", "material": "pfm", "date": "2021-07-22"}]},
    --   "14": {"status": "present", "conditions": ["caries_do"], "findings": [{"type": "caries", "surfaces": ["D","O"], "severity": "moderate", "found": "2026-05-25"}]},
    --   "19": {"status": "implant", "conditions": ["implant_crown"], "restorations": [{"type": "implant", "date": "2023-01-10"}, {"type": "crown", "material": "zirconia", "date": "2023-04-15"}]},
    --   "30": {"status": "present", "conditions": []}
    -- }
    insurance JSONB NOT NULL DEFAULT '[]',
    -- insurance example:
    -- [
    --   {"plan_id": "plan-uuid", "priority": "primary", "subscriber_id": "DD123456",
    --    "group_number": "GRP789", "effective_date": "2026-01-01",
    --    "annual_max_cents": 150000, "remaining_cents": 95000, "deductible_cents": 5000, "deductible_met_cents": 5000}
    -- ]
    recall JSONB NOT NULL DEFAULT '{}',
    -- recall example:
    -- {"interval_months": 6, "next_date": "2026-11-25", "last_cleaning": "2026-05-25", "preferred_provider_id": "provider-uuid"}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (practice_id, mrn)
);

CREATE INDEX idx_patients_practice ON patients(practice_id);
CREATE INDEX idx_patients_name ON patients(practice_id, last_name, first_name);
CREATE INDEX idx_patients_dob ON patients(practice_id, date_of_birth);
CREATE INDEX idx_patients_chart ON patients USING GIN (tooth_chart jsonb_path_ops);
```

---

## Appointments

```sql
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id UUID NOT NULL REFERENCES practices(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL,
    operatory_id UUID NOT NULL,
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appts_location ON appointments(location_id, scheduled_start);
CREATE INDEX idx_appts_provider ON appointments(provider_id, scheduled_start);
CREATE INDEX idx_appts_patient ON appointments(patient_id, scheduled_start DESC);
```

---

## Encounters

```sql
CREATE TABLE encounters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id UUID REFERENCES appointments(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    provider_id UUID NOT NULL REFERENCES providers(id),
    location_id UUID NOT NULL,
    encounter_type TEXT NOT NULL DEFAULT 'exam',
    status TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'completed', 'addendum', 'locked'
    )),
    chief_complaint TEXT,
    clinical_notes TEXT,
    findings JSONB NOT NULL DEFAULT '{}',
    -- findings example:
    -- {
    --   "tooth_findings": [
    --     {"tooth": "14", "surface": "DO", "type": "caries", "severity": "moderate", "snodent": "109723001"},
    --     {"tooth": "18", "type": "fracture", "severity": "mild", "notes": "Cracked cusp ML"}
    --   ],
    --   "soft_tissue": {"oral_cancer_screen": "negative", "gingival": "localized_inflammation_LL"},
    --   "occlusion": "class_1",
    --   "tmj": "normal"
    -- }
    perio_data JSONB,
    -- perio_data example (6 sites per tooth: DB, B, MB, DL, L, ML):
    -- {
    --   "exam_type": "full",
    --   "measurements": {
    --     "2":  {"probing": [3,3,3,3,3,3], "recession": [0,0,0,0,0,0], "bleeding": [false,false,false,false,false,false]},
    --     "3":  {"probing": [3,2,3,3,3,2], "recession": [0,0,0,0,0,0], "bleeding": [false,false,false,false,false,false]},
    --     "14": {"probing": [4,5,4,3,4,3], "recession": [1,1,0,0,0,0], "bleeding": [true,true,false,false,true,false]},
    --     "19": {"probing": [3,3,3,3,3,3], "recession": [0,0,0,0,0,0], "bleeding": [false,false,false,false,false,false]}
    --   },
    --   "summary": {"avg_probing": 3.2, "sites_bleeding": 3, "sites_total": 120}
    -- }
    procedures JSONB NOT NULL DEFAULT '[]',
    -- procedures example:
    -- [
    --   {"id": "proc-uuid", "cdt": "D0120", "description": "Periodic oral evaluation", "fee_cents": 6500,
    --    "performed_at": "2026-05-25T10:00:00Z"},
    --   {"id": "proc-uuid-2", "cdt": "D1110", "description": "Prophylaxis - adult", "fee_cents": 11000,
    --    "tooth": null, "performed_at": "2026-05-25T10:15:00Z"},
    --   {"id": "proc-uuid-3", "cdt": "D0274", "description": "Bitewings - four films", "fee_cents": 7000,
    --    "teeth": ["2","3","4","5","13","14","15","12"], "performed_at": "2026-05-25T10:20:00Z"}
    -- ]
    treatment_plan JSONB,
    -- treatment_plan example:
    -- {
    --   "id": "tp-uuid", "name": "Restorative Plan", "status": "presented",
    --   "phases": [
    --     {"phase": 1, "procedures": [
    --       {"cdt": "D2391", "tooth": "14", "surface": "DO", "priority": "urgent",
    --        "fee_cents": 22000, "insurance_est_cents": 11000, "patient_est_cents": 11000, "status": "planned"}
    --     ]},
    --     {"phase": 2, "procedures": [
    --       {"cdt": "D2740", "tooth": "3", "priority": "recommended",
    --        "fee_cents": 120000, "insurance_est_cents": 60000, "patient_est_cents": 60000, "status": "planned"}
    --     ]}
    --   ],
    --   "total_fee_cents": 142000, "insurance_est_cents": 71000, "patient_est_cents": 71000,
    --   "presented_at": "2026-05-25T10:30:00Z"
    -- }
    imaging JSONB NOT NULL DEFAULT '[]',
    -- imaging example:
    -- [
    --   {"type": "bitewing", "teeth": ["2","3","4","5"], "dicom_uid": "1.2.840...", "storage_path": "/images/...",
    --    "ai_findings": [
    --      {"tooth": "14", "finding": "caries", "surface": "D", "confidence": 0.92, "model": "pearl_v3"}
    --    ]}
    -- ]
    signed_by UUID REFERENCES providers(id),
    signed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id, created_at DESC);
CREATE INDEX idx_encounters_provider ON encounters(provider_id, created_at DESC);
CREATE INDEX idx_encounters_procedures ON encounters USING GIN (procedures jsonb_path_ops);
```

---

## Insurance Claims

```sql
CREATE TABLE claims (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id UUID NOT NULL REFERENCES encounters(id),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL,
    claim_type TEXT NOT NULL DEFAULT 'dental',
    claim_number TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'submitted', 'accepted', 'rejected', 'denied',
        'paid', 'partially_paid', 'appealed', 'voided'
    )),
    payer JSONB NOT NULL DEFAULT '{}',
    -- payer example:
    -- {"plan_id": "plan-uuid", "payer_name": "Delta Dental", "edi_payer_id": "...",
    --  "subscriber_id": "DD123456", "group_number": "GRP789"}
    service_lines JSONB NOT NULL DEFAULT '[]',
    -- service_lines example:
    -- [
    --   {"line": 1, "cdt": "D0120", "tooth": null, "surface": null,
    --    "description": "Periodic oral evaluation", "units": 1,
    --    "charge_cents": 6500, "allowed_cents": 5000, "paid_cents": 5000},
    --   {"line": 2, "cdt": "D1110", "tooth": null, "surface": null,
    --    "description": "Prophylaxis - adult", "units": 1,
    --    "charge_cents": 11000, "allowed_cents": 8500, "paid_cents": 8500}
    -- ]
    total_charge_cents BIGINT NOT NULL DEFAULT 0,
    allowed_cents BIGINT,
    paid_cents BIGINT,
    patient_responsibility_cents BIGINT,
    denial JSONB,
    submitted_at TIMESTAMPTZ,
    adjudicated_at TIMESTAMPTZ,
    era_835 JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id, created_at DESC);
CREATE INDEX idx_claims_status ON claims(status) WHERE status NOT IN ('paid', 'voided');
```

---

## Payments

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id UUID NOT NULL REFERENCES patients(id),
    location_id UUID NOT NULL,
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

## Audit Log

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
```

---

## Example Queries

### Full tooth chart for a patient

```sql
SELECT first_name, last_name, tooth_chart
FROM patients
WHERE id = 'patient-uuid';
```

### Find patients with caries on a specific tooth

```sql
SELECT p.id, p.first_name, p.last_name
FROM patients p
WHERE p.practice_id = 'practice-uuid'
  AND p.tooth_chart->'14' @> '{"conditions": ["caries_do"]}';
```

### Perio probing depths from an encounter

```sql
SELECT tooth_key, (tooth_data->'probing')::JSONB AS probing_depths,
       (tooth_data->'bleeding')::JSONB AS bleeding
FROM encounters e,
     jsonb_each(e.perio_data->'measurements') AS t(tooth_key, tooth_data)
WHERE e.id = 'encounter-uuid'
ORDER BY tooth_key::INT;
```

### Production report by provider

```sql
SELECT pr.name AS provider,
       SUM(jsonb_array_length(e.procedures)) AS procedure_count,
       SUM((SELECT COALESCE(SUM((p->>'fee_cents')::BIGINT), 0) FROM jsonb_array_elements(e.procedures) p)) AS production_cents
FROM encounters e
JOIN providers pr ON pr.id = e.provider_id
WHERE e.created_at >= CURRENT_DATE - 30
  AND e.status IN ('completed', 'locked')
GROUP BY pr.id, pr.name
ORDER BY production_cents DESC;
```

### Claim denial rate by payer

```sql
SELECT c.payer->>'payer_name' AS payer_name,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE c.status = 'denied') AS denied,
       ROUND(COUNT(*) FILTER (WHERE c.status = 'denied') * 100.0 / COUNT(*), 1) AS denial_pct
FROM claims c
WHERE c.location_id = 'location-uuid'
  AND c.created_at >= CURRENT_DATE - 90
GROUP BY c.payer->>'payer_name'
ORDER BY denial_pct DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Practice & Providers | 2 | practices (locations, operatories, plans, fee schedules in JSONB), providers (credentials in JSONB) |
| Reference Data | 1 | cdt_codes |
| Patients | 1 | patients (contact, medical history, tooth chart, insurance, recall in JSONB) |
| Scheduling | 1 | appointments |
| Encounters | 1 | encounters (findings, perio, procedures, treatment plan, imaging in JSONB) |
| Claims | 1 | claims (payer, service lines, denial, ERA in JSONB) |
| Payments | 1 | payments |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **9** | Plus CDT reference table |

---

## Key Design Decisions

1. **Tooth chart as patient-level JSONB** — The full-mouth tooth chart lives on the `patients` table as a single JSONB document keyed by tooth number (1-32, A-T). Each tooth entry tracks status, conditions, restorations with dates, and findings. This eliminates the 32-row-per-patient tooth chart table and makes "show me the full chart" a single row read.

2. **Perio measurements as encounter JSONB** — A full-mouth perio exam (6 sites × 32 teeth) becomes a single `perio_data` JSONB document on the encounter, with measurements keyed by tooth number. Arrays of 6 values represent the 6 probing sites in order (DB, B, MB, DL, L, ML). This is more compact than 192 separate rows.

3. **Procedures inline on encounters** — Completed procedures live in the encounter's `procedures` JSONB array with CDT codes, tooth numbers, surfaces, and fees. Each procedure has a generated UUID for cross-referencing from claims. The CDT code is validated against the `cdt_codes` reference table at the application layer.

4. **Treatment plans as encounter JSONB** — Treatment plans with phased procedures live inline on the encounter where they were created. This captures the clinical context (findings that drove the treatment plan) in one document. Active treatment plans are tracked by the application layer querying for encounters with `treatment_plan.status = 'accepted'`.

5. **CDT codes kept as a relational reference table** — Despite the JSONB-heavy design, CDT codes remain a separate table because they're externally managed reference data (updated annually by the ADA) and need category-based lookups for coding assistance.

6. **Imaging inline on encounters** — Radiographs with DICOM references and AI findings live in the encounter's `imaging` JSONB array. Each image entry includes tooth numbers, AI detection results with confidence scores, and storage paths.

7. **Insurance on patients, payer on claims** — Patient insurance details (annual max, deductible, remaining benefits) live as JSONB on the patient record. Each claim snapshots the payer details at submission time, ensuring claim integrity independent of later coverage changes.

8. **Audit log kept relational** — The HIPAA audit log remains a separate partitioned table, not merged into encounter JSONB, because compliance audits require independent, tamper-evident logging.
