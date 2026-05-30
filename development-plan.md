# Dental Practice Management — Phased Development Plan

> Project: 323-dental-practice-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. It adopts **Data Model Suggestion 2 (Hybrid Relational + JSONB)** as the primary schema — the financial pipeline (appointments, encounters, claims, payments) is strictly relational while clinical detail (tooth chart, perio, findings, AI output) lives in validated JSONB. This matches the target segment (solo to small-group practices) where schema simplicity and document-shaped clinical reads matter, while still enforcing referential integrity on the money path. The audit trail borrows the immutable, append-only pattern from Suggestion 3 (event-sourced) for HIPAA compliance, applied narrowly to PHI access logging rather than the whole system.

---

## Core Requirements Summary

**What it does:** An AI-native, cloud-native (optionally self-hosted) dental practice management platform covering scheduling, clinical charting and treatment planning, insurance eligibility and X12 837D/835 claims, patient billing, and recall/reminders — with transparent pricing and a self-serve developer API.

**Primary users:** Solo dentist owners, group practices, DSOs (10+ locations), and specialty practices (oral surgery, ortho, perio, endo). Front-desk staff (scheduling, billing), hygienists (perio charting), dentists (findings, treatment plans), billing staff (claims/RCM), and practice admins (reporting, access control).

**Key differentiators:** Transparent pricing, self-serve REST API + webhooks from day one, AI-native clinical workflow (CDT coding from notes, radiograph analysis, treatment-plan narratives, no-show prediction, fee/eligibility optimisation), and FHIR-first interoperability.

**Deployment model:** Multi-tenant SaaS as the default, with a single-tenant Docker Compose self-hosted mode. Same codebase for both.

**Integration surface:** EDI clearinghouse (X12 270/271, 837D, 835), SMS/email providers (Twilio, SendGrid), payment processor (Stripe), object storage for radiographs (S3-compatible), DICOM ingestion, LLM provider (Anthropic via the AI SDK), and outbound webhooks.

**Standards to implement:** CDT 2026 (reference table), X12 837D/835/270-271 EDI, NPI validation, HL7 FHIR R4 (export), DICOM (image storage/UIDs), HIPAA (encryption, RBAC, audit), OAuth 2.0/OIDC, TLS 1.2+/AES-256. SNODENT referenced where licensing permits.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | API + web UI + AI orchestration in one language; strong typing for clinical/financial data; rich healthcare/EDI library ecosystem; aligns with Vercel AI SDK for the AI-native features. |
| API framework | Fastify + `@fastify/swagger` | High-throughput, schema-first; JSON Schema route definitions auto-generate the OpenAPI 3.1 spec required for the self-serve developer API. |
| Schema/validation | Zod + `zod-to-json-schema` | Single source of truth for request/response validation and OpenAPI generation; validates JSONB clinical documents at the application layer (required by the hybrid model). |
| ORM / DB access | Drizzle ORM | Typed SQL with first-class JSONB support and SQL-level control needed for partitioned audit tables and GIN indexes; lightweight migrations. |
| Database | PostgreSQL 16 | JSONB + GIN indexing for tooth charts/perio, partitioning for audit log, `gen_random_uuid()`, strong transactional integrity for the claims pipeline. |
| Cache / queue | Redis 7 + BullMQ | Async workloads: EDI submission, ERA polling, reminder sends, AI calls, webhook delivery. BullMQ gives retries, backoff, scheduled (recall) jobs. |
| Object storage | S3-compatible (AWS S3 / MinIO self-hosted) | DICOM radiographs and intraoral photos are large binaries; keep out of Postgres. MinIO covers the self-hosted path. |
| Frontend | Next.js 16 (App Router) + React + shadcn/ui + Tailwind | Browser-based dashboard-first UI matching cloud-native competitors; server components for fast chart reads; shadcn for accessible clinical UI components. Tooth chart rendered with SVG. |
| Auth | OAuth 2.0 / OIDC via Clerk (or self-hosted Keycloak) | Delegated auth, SSO, SCIM offboarding (HIPAA same-day revocation). Patient portal and staff use separate realms/roles. |
| AI orchestration | Vercel AI SDK + Anthropic Claude | Structured output (CDT suggestions), vision (radiograph analysis), text generation (treatment-plan narratives). `generateObject` enforces typed AI output. |
| EDI | `node-x12` for parse/build + custom X12 builders | Generate 837D, parse 835/271; clearinghouse transport over SFTP/REST. |
| FHIR | `@types/fhir` + custom R4 mappers | Export Patient/Encounter/Procedure/Claim resources; align with Dental Data Exchange IG. |
| Testing | Vitest + Supertest + Testcontainers | Unit (Vitest), API integration (Supertest), real Postgres/Redis via Testcontainers; Playwright for web E2E. |
| Code quality | ESLint + Prettier + `tsc --noEmit` | Lint, format, strict type-check in CI. |
| Package manager / monorepo | pnpm + Turborepo | Monorepo: `api`, `web`, `worker`, shared `core`/`db`/`edi`/`fhir`/`ai` packages; pnpm workspaces. |
| Containerisation | Docker + Docker Compose | Self-hosted single-tenant mode; reproducible CI; production images per service. |
| Observability | Pino (structured logs) + OpenTelemetry | HIPAA-relevant audit separate from app logs; tracing across API → worker → EDI. |

### Project Structure

```
dental-practice-management/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml           # postgres, redis, minio, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── core/                    # domain types, Zod schemas, tooth-numbering, CDT utils
│   │   └── src/
│   │       ├── schemas/         # zod schemas (patient, encounter, claim, toothChart, perio…)
│   │       ├── domain/          # business rules (eligibility, fee calc, no-show)
│   │       └── constants/       # tooth numbering, CDT categories, X12 codes
│   ├── db/                      # drizzle schema, migrations, seed, repositories
│   │   └── src/
│   │       ├── schema/
│   │       ├── migrations/
│   │       ├── repositories/
│   │       └── seed/            # CDT 2026 loader, demo practice
│   ├── edi/                     # X12 837D build, 835/271 parse, clearinghouse client
│   ├── fhir/                    # FHIR R4 resource mappers + export
│   ├── ai/                      # AI SDK wrappers + prompt templates
│   └── auth/                    # OIDC verification, RBAC policy
├── apps/
│   ├── api/                     # Fastify server
│   │   └── src/
│   │       ├── routes/          # patients, appointments, encounters, claims, billing, fhir, dev-api
│   │       ├── plugins/         # auth, audit, rate-limit, swagger, tenant-scope
│   │       ├── hooks/
│   │       └── server.ts
│   ├── worker/                  # BullMQ processors
│   │   └── src/
│   │       ├── queues/          # edi-submit, era-poll, reminders, ai-jobs, webhooks
│   │       └── schedulers/      # recall, reminder cadence
│   └── web/                     # Next.js dashboard + patient portal
│       └── src/app/
└── tests/
    ├── fixtures/                # sample 837D/835/271, DICOM, CDT subset, FHIR
    └── e2e/                     # Playwright specs
```

---

## Phase 1: Foundation — Monorepo, Database, Auth, Audit

### Purpose
Establish the monorepo, the multi-tenant PostgreSQL schema, OIDC authentication with role-based access control, and the HIPAA audit log. After this phase the system can authenticate a user, scope every query to a practice (tenant), and record every PHI access immutably. Nothing clinical works yet, but the security and data substrate every later phase depends on is in place.

### Tasks

#### 1.1 — Monorepo scaffolding and CI

**What:** Initialise the pnpm/Turborepo workspace with `core`, `db`, `auth`, `api`, `worker`, `web` packages, shared ESLint/Prettier/tsconfig, and a CI pipeline.

**Design:**
- `pnpm-workspace.yaml` lists `packages/*` and `apps/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test`, `test:e2e` with `dependsOn: ["^build"]`.
- Root `tsconfig.base.json` with `"strict": true`, `"noUncheckedIndexedAccess": true`.
- CI (GitHub Actions): install → `turbo lint typecheck test build`; Testcontainers requires Docker-in-CI.
- `.env.example` enumerating: `DATABASE_URL`, `REDIS_URL`, `S3_*`, `OIDC_ISSUER`, `OIDC_AUDIENCE`, `ANTHROPIC_API_KEY`, `TWILIO_*`, `SENDGRID_API_KEY`, `STRIPE_SECRET_KEY`, `CLEARINGHOUSE_*`, `APP_MODE` (`saas` | `self_hosted`).

**Testing:**
- `Unit: turbo build → all packages compile, exit 0`
- `CI smoke: pnpm install + turbo typecheck on clean checkout → exit 0`

#### 1.2 — Database schema (core relational tables)

**What:** Drizzle schema and migrations for `practices`, `providers`, `cdt_codes`, `patients`, `appointments`, `encounters`, `claims`, `payments`, `audit_log` per Data Model Suggestion 2.

**Design:**
Use the DDL from `data-model-suggestion-2.md` verbatim for `practices` (locations/operatories/insurance_plans/fee_schedules in `settings` JSONB), `providers` (`credentials` JSONB), `cdt_codes`, `patients` (`contact`/`medical_history`/`tooth_chart`/`insurance`/`recall` JSONB), `appointments`, `encounters` (`findings`/`perio_data`/`procedures`/`treatment_plan`/`imaging` JSONB), `claims` (`payer`/`service_lines`/`denial`/`era_835` JSONB), `payments`, and partitioned `audit_log`. Add a `users` table linking OIDC subjects to providers/patients:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oidc_subject TEXT NOT NULL UNIQUE,
    practice_id UUID REFERENCES practices(id) ON DELETE CASCADE,
    provider_id UUID REFERENCES providers(id),     -- staff
    patient_id  UUID REFERENCES patients(id),       -- portal users
    user_type TEXT NOT NULL CHECK (user_type IN ('staff','patient','developer')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
GIN indexes on `patients.tooth_chart` and `encounters.procedures` per the model. Migrations are SQL files under `packages/db/src/migrations`; a `pnpm db:migrate` script applies them.

**Testing:**
- `Integration (Testcontainers Postgres): run all migrations → all tables and indexes exist (query pg_catalog)`
- `Integration: insert practice + patient with tooth_chart JSONB → row round-trips, GIN index used (EXPLAIN)`
- `Unit: rolling-down then re-applying migrations is idempotent`

#### 1.3 — CDT 2026 reference loader

**What:** A seed command that loads the CDT 2026 code set into `cdt_codes`, with a documented bring-your-own-data path for the ADA-licensed full set.

**Design:**
- `cdt_codes` seeded from `packages/db/src/seed/cdt-2026.json` (`{code, category, subcategory, description, version_year}`). Ship a small non-infringing public subset (D0120, D0150, D0210, D0274, D1110, D2391, D2740, etc.) for tests/demo; document that production deployments must supply a licensed ADA CDT file via `CDT_DATA_PATH`.
- Loader is idempotent (`INSERT ... ON CONFLICT (code) DO UPDATE`).
- Helper `getCdtCategory(code)` and `isCdtActive(code, year)` in `packages/core`.

**Testing:**
- `Unit: load seed subset → 12+ codes present, version_year=2026`
- `Unit: getCdtCategory('D2391') → 'restorative'`
- `Unit: loader run twice → no duplicates, counts stable`
- `Unit: CDT_DATA_PATH override → loads from external file`

#### 1.4 — OIDC auth + RBAC

**What:** A Fastify auth plugin that verifies OIDC bearer tokens, resolves the `users` row, and enforces a role-based access policy.

**Design:**
- `packages/auth` exposes `verifyToken(jwt): Promise<AuthContext>` (JWKS validation against `OIDC_ISSUER`) and a policy engine.
- `AuthContext`:
```ts
interface AuthContext {
  userId: string;
  userType: 'staff' | 'patient' | 'developer';
  practiceId: string | null;
  providerId: string | null;
  patientId: string | null;
  role: ProviderRole | 'patient' | 'developer';
  scopes: string[]; // for developer API keys
}
type ProviderRole =
  | 'dentist' | 'hygienist' | 'oral_surgeon' | 'orthodontist'
  | 'periodontist' | 'endodontist' | 'pedodontist'
  | 'assistant' | 'receptionist' | 'billing' | 'admin';
```
- Policy: `can(ctx, action, resource)` matrix. e.g. `billing`/`admin` → claims write; `hygienist`/`dentist` → perio/findings write; `receptionist` → scheduling write, no clinical write; `patient` → only own record (read + portal actions).
- Fastify decorator `request.auth`; preHandler rejects with 401 (no/invalid token) or 403 (policy fail).

**Testing:**
- `Unit: valid JWT → AuthContext with correct role/practiceId`
- `Unit: expired/invalid-signature JWT → throws → 401`
- `Unit: can(receptionist, 'write', 'claim') → false`
- `Unit: can(billing, 'write', 'claim') → true`
- `Integration: request without Authorization header → 401`

#### 1.5 — Tenant scoping + HIPAA audit middleware

**What:** Middleware that injects `practice_id` into every query and writes an immutable audit-log row for every PHI access.

**Design:**
- A `withTenant(ctx)` repository wrapper appends `WHERE practice_id = ctx.practiceId` to all reads/writes; cross-tenant access throws.
- Audit hook (Fastify `onResponse`) writes to `audit_log`: `{practice_id, user_id, action, resource_type, resource_id, details, ip_address}`. Append-only — no UPDATE/DELETE grants on the table for the app role.
- Audit actions enumerated: `patient.chart_viewed`, `phi.exported`, `phi.printed`, `claim.submitted`, `record.created/updated`, `access.break_glass`.
- Partition-creation job (monthly) in worker scheduler.

**Testing:**
- `Integration: GET patient chart → exactly one audit_log row with action='patient.chart_viewed', resource_id=patientId`
- `Integration: user from practice A requests patient of practice B → 403, audit row with denied detail`
- `Unit: app DB role has no DELETE privilege on audit_log (information_schema check)`

---

## Phase 2: Patients, Scheduling & Recall

### Purpose
Deliver the front-desk core: patient registration with demographics, insurance, and medical history; the operatory/provider scheduling calendar with conflict detection; and recall tracking. This is the entry point for every downstream clinical and financial workflow and is independently demoable.

### Tasks

#### 2.1 — Patient CRUD + search

**What:** REST endpoints to create, read, update, search, and deactivate patients, validating all JSONB documents.

**Design:**
- Zod schemas in `core/schemas/patient.ts` for `contact`, `medicalHistory`, `insurance[]`, `recall`, and `toothChart` (validated but defaulted empty at creation).
```ts
const Contact = z.object({
  email: z.string().email().optional(),
  phone: z.string().optional(),
  address: z.object({ line1: z.string(), city: z.string(), state: z.string(), postal: z.string() }).partial().optional(),
  preferredLanguage: z.string().default('en'),
  emergency: z.object({ name: z.string(), phone: z.string() }).optional(),
});
const PatientInsurance = z.object({
  planId: z.string().uuid(), priority: z.enum(['primary','secondary','tertiary']),
  subscriberId: z.string(), groupNumber: z.string().optional(),
  effectiveDate: z.string().date().optional(),
  annualMaxCents: z.number().int().optional(), remainingCents: z.number().int().optional(),
  deductibleCents: z.number().int().optional(), deductibleMetCents: z.number().int().optional(),
});
```
- Endpoints:
  - `POST /v1/patients` → 201 `{id}`; auto-assign `mrn` (per-practice sequence).
  - `GET /v1/patients/:id` → patient document (audited).
  - `PATCH /v1/patients/:id` → partial update (JSONB merge).
  - `GET /v1/patients?q=&page=` → name/DOB/MRN search (uses `idx_patients_name`).
  - `DELETE /v1/patients/:id` → soft delete (`is_active=false`).
- `mrn` uniqueness per `(practice_id, mrn)`.

**Testing:**
- `Unit: valid patient payload → Zod parse succeeds`
- `Unit: invalid email in contact → ValidationError naming 'contact.email'`
- `Integration: POST then GET → same document, mrn assigned`
- `Integration: duplicate mrn within practice → 409`
- `Integration: search 'Smith' → returns matching patients only within tenant`
- `Integration: DELETE → is_active=false, excluded from default search`

#### 2.2 — Appointment scheduling with conflict detection

**What:** Endpoints to book, reschedule, cancel appointments with operatory/provider double-booking prevention.

**Design:**
- `POST /v1/appointments` body: `{patientId, providerId, locationId, operatoryId, appointmentType, scheduledStart, scheduledEnd, reason?}`.
- Conflict check in a transaction: reject if any non-cancelled appointment overlaps `[start,end)` for the same `operatory_id` OR same `provider_id`. Use exclusion logic via `tstzrange` overlap query.
- Status state machine: `scheduled → confirmed → checked_in → in_operatory → completed`; side branches `cancelled`, `no_show`, `rescheduled`. `PATCH /v1/appointments/:id/status` validates legal transitions.
- `GET /v1/appointments?locationId=&date=&providerId=&operatoryId=` returns calendar slice ordered by `scheduled_start`.

**Testing:**
- `Integration: book non-overlapping slots same operatory → both 201`
- `Integration: book overlapping operatory → 409 conflict`
- `Integration: book overlapping provider, different operatory → 409`
- `Unit: status transition scheduled→completed direct → rejected (must pass through checked_in)`
- `Unit: status transition scheduled→cancelled → allowed`
- `Integration: calendar query for a day → ordered, tenant-scoped`

#### 2.3 — Recall tracking and scheduling job

**What:** Maintain each patient's `recall` JSONB and a scheduled worker that surfaces due recalls.

**Design:**
- On `appointment.status=completed` for a cleaning/exam type, recompute `recall.nextDate = lastCleaning + intervalMonths`.
- BullMQ repeatable job `recall-scan` (daily): query `idx_patients_recall` for `next_date <= today + 30` and enqueue reminder jobs (Phase actual send in 2.4).
- `GET /v1/recall/due?within=30` for front-desk worklist.

**Testing:**
- `Unit: completing cleaning with interval 6 → nextDate = +6 months`
- `Integration (real Redis): recall-scan finds due patients → enqueues reminder jobs (count matches)`
- `Integration: GET /recall/due → returns only active patients with due dates in window`

#### 2.4 — Automated reminders (SMS + email)

**What:** Reminder send pipeline over Twilio (SMS) and SendGrid (email) with templated messages and cadence.

**Design:**
- `reminders` queue; job payload `{patientId, appointmentId?, channel, templateId, cadenceStep}`.
- Cadence from `practice.settings.recall_defaults.reminder_days` (default `[30,14,3]`) for recalls and per-appointment confirmations.
- Provider abstraction `NotificationProvider.send({channel,to,template,vars})`; Twilio/SendGrid adapters; a `console` adapter for dev/tests.
- Idempotency: dedupe key `{patientId, appointmentId, cadenceStep, channel}` in Redis to avoid double-sends.
- Records outcome to `notification_log` table `{id, patient_id, channel, template, status, provider_message_id, sent_at}`.

**Testing:**
- `Unit (mocked Twilio): SMS job → adapter.send called with rendered template`
- `Unit: dedupe key present → job is a no-op`
- `Integration: appointment confirmed → confirmation reminder not re-sent on duplicate event`
- `Unit: template render with patient name + appt time → expected string`

---

## Phase 3: Clinical Charting & Treatment Planning

### Purpose
Deliver the clinical heart of the product: encounters, the graphical tooth chart, per-tooth findings, periodontal charting, and treatment plans with phased procedures and fee/estimate calculation. After this phase a dentist can examine a patient, chart findings, and present a costed treatment plan — the core value proposition.

### Tasks

#### 3.1 — Encounter lifecycle

**What:** Open, document, sign, and lock clinical encounters.

**Design:**
- `POST /v1/encounters` `{patientId, appointmentId?, providerId, locationId, encounterType, chiefComplaint?}` → status `in_progress`.
- `PATCH /v1/encounters/:id` updates `clinical_notes`, `findings`, `perio_data`, `procedures`, `imaging` JSONB (each validated by its Zod schema).
- `POST /v1/encounters/:id/sign` → requires dentist/hygienist role; sets `signed_by`, `signed_at`, status `completed`.
- `POST /v1/encounters/:id/lock` → status `locked`; further edits only via addendum (`POST /v1/encounters/:id/addendum` appends to an `amendments` array preserving original — borrowed from Suggestion 3).
- State machine: `in_progress → completed → locked`; `addendum` permitted post-lock.

**Testing:**
- `Integration: open → patch notes → sign → status completed, signed_at set`
- `Integration: edit after lock → 409; addendum after lock → 200, original preserved`
- `Unit: receptionist attempts sign → 403`

#### 3.2 — Tooth chart + per-tooth findings

**What:** Persist and update the full-mouth tooth chart and record per-encounter findings, validated against ADA Universal Numbering.

**Design:**
- `ToothChart` Zod schema: keys `'1'..'32'` and `'A'..'T'`; value `{status, conditions[], restorations[], findings[]}` per Suggestion 2 example.
- Tooth-number validator in `core/constants/tooth-numbering.ts`: permanent 1–32, primary A–T; reject others.
- `PUT /v1/patients/:id/tooth-chart/:tooth` updates one tooth (merges into `patients.tooth_chart` JSONB).
- Findings recorded on the encounter `findings.tooth_findings[]`: `{tooth, surface?, type, severity?, snodent?, notes?}` with `type` enum from Suggestion 1's `finding_type` list. Optional `snodent` concept id (string; SNODENT table optional/licensed).
- When an encounter is signed, a projection updates `patients.tooth_chart` from completed restorative procedures (auto tooth-chart update borrowed from Suggestion 3).

**Testing:**
- `Unit: tooth '33' → validation error; tooth '32' and 'A' → valid`
- `Unit: finding with type 'caries', surface 'DO', severity 'moderate' → parses`
- `Integration: PUT tooth 14 status implant → chart reflects change, GIN-indexed query finds it`
- `Integration: sign encounter with restorative procedure on tooth 3 → chart tooth 3 gains restoration`

#### 3.3 — Periodontal charting

**What:** Capture 6-site probing depths, recession, bleeding, furcation, mobility per tooth and compute summary stats.

**Design:**
- `PerioData` Zod schema (Suggestion 2): `{examType, measurements: Record<tooth, {probing:number[6], recession:number[6], bleeding:boolean[6], furcation?:number, mobility?:number}>, summary}`. Sites order DB,B,MB,DL,L,ML.
- `POST /v1/encounters/:id/perio` validates arrays length 6, depths 0–15.
- Compute `summary`: `avgProbing`, `sitesBleeding`, `sitesTotal`, `maxProbing`, `pockets4plus`, `pockets5plus`.
- Perio trend read (`GET /v1/patients/:id/perio-trends`) aggregates `summary` across the patient's signed encounters for longitudinal display.

**Testing:**
- `Unit: probing array length 5 → validation error`
- `Unit: 6 sites with depths → summary.avgProbing correct, pockets4plus counted`
- `Integration: two perio exams over time → trend endpoint returns ordered series`

#### 3.4 — Treatment plans with fee & estimate calculation

**What:** Create phased treatment plans, compute total fee and insurance vs. patient estimates from the practice fee schedule and patient benefits.

**Design:**
- Treatment plan stored on the originating encounter's `treatment_plan` JSONB (Suggestion 2 shape): `{id, name, status, phases:[{phase, procedures:[{cdt, tooth?, surface?, priority, feeCents, insuranceEstCents, patientEstCents, status}]}], totalFeeCents, insuranceEstCents, patientEstCents, presentedAt?}`.
- Status machine: `proposed → presented → accepted | partially_accepted | declined → in_progress → completed | expired`.
- Fee calc (`core/domain/estimate.ts`):
  1. Look up `feeCents` from `practice.settings.fee_schedules[planFeeSchedule][cdt]` (fallback `standard`).
  2. `insuranceAllowable = fee_schedules[planName][cdt]`; apply coverage %, then cap against `remainingCents` (annual max) and subtract unmet `deductibleCents`.
  3. `patientEstCents = feeCents - insuranceEstCents`.
- `POST /v1/encounters/:id/treatment-plan`, `PATCH .../treatment-plan` (add/remove procedures recompute totals), `POST .../treatment-plan/present`, `.../accept`.

**Testing:**
- `Unit: D2391 fee 22000, coverage 50%, remaining 95000, no deductible → insuranceEst 11000, patientEst 11000`
- `Unit: coverage exceeds remaining annual max → insuranceEst capped at remaining`
- `Unit: unmet deductible 5000 → subtracted before coverage`
- `Integration: add procedure to plan → totals recomputed`
- `Unit: present before accept enforced; accept sets accepted_at`

---

## Phase 4: Billing, Insurance Eligibility & X12 Claims

### Purpose
Connect the clinical record to revenue: real-time eligibility (270/271), generate and submit X12 837D claims to a clearinghouse, post 835 remittances and denials, and handle patient billing and payments. This is the second core value pillar and the highest-stakes correctness path.

### Tasks

#### 4.1 — Eligibility verification (X12 270/271)

**What:** Build a 270 inquiry, send it to the clearinghouse, parse the 271 response, and store benefits.

**Design:**
- `packages/edi/x12/270.ts` builds a 270 from `{provider NPI, payer EDI id, subscriberId, patient}`; `271.ts` parses coverage/benefits.
- Clearinghouse transport `ClearinghouseClient` (REST or SFTP per `CLEARINGHOUSE_TRANSPORT`); a `mock` transport returns fixture 271s for tests.
- `POST /v1/patients/:id/eligibility/:insuranceId` → enqueues `eligibility` job → on 271 updates the patient's `insurance[]` entry (`annualMaxCents`, `remainingCents`, `deductibleCents`, coverage) and records raw response.
- NPI validation (Luhn-based check digit) before any claim/eligibility transaction.

**Testing:**
- `Unit: build 270 → valid segment structure (ISA/GS/ST/HL/EQ/SE/GE/IEA)`
- `Unit: parse fixture 271 → extracts annual max, deductible, remaining`
- `Unit: NPI '1234567893' valid checksum → pass; '1234567890' → fail`
- `Integration (mock transport): eligibility request → patient insurance updated`

#### 4.2 — Claim generation (X12 837D)

**What:** Generate a draft claim from a signed encounter's completed procedures and build the 837D transaction.

**Design:**
- On encounter sign, a projection drafts a `claims` row: `payer` snapshot from patient insurance, `service_lines[]` from `encounters.procedures` (CDT, tooth, surface, charge), `total_charge_cents`, status `draft`.
- `packages/edi/x12/837d.ts` `build837D(claim, provider, practice): string` emits 005010X224A2 segments (ISA/GS/ST, billing provider NPI, subscriber, patient, CLM, tooth-level LX/SV3 lines with CDT codes, SE/GE/IEA).
- Claim status machine: `draft → submitted → accepted | rejected → adjudicated → paid | partially_paid | denied → appealed`.
- `POST /v1/claims/:id/submit` validates required fields (NPI, payer id, ≥1 line) then enqueues `edi-submit`.

**Testing:**
- `Unit: build 837D for 2-line claim → contains CLM, two SV3 with CDT codes, balanced control segments`
- `Unit: claim missing provider NPI → submit rejected 422`
- `Integration: sign encounter with 2 procedures → draft claim with 2 service lines`
- `Unit: submit on draft → status submitted; submit on paid → 409`

#### 4.3 — Remittance posting (X12 835)

**What:** Poll/receive 835 ERAs, match to claims, post payments, and flag denials.

**Design:**
- `era-poll` worker job pulls 835 files from clearinghouse; `packages/edi/x12/835.ts` parses CLP/CAS/SVC segments.
- Match by claim control number; update `claims`: `allowed_cents`, `paid_cents`, `patient_responsibility_cents`, per-line adjudication into `service_lines`, status (`paid`/`partially_paid`/`denied`), `denial` JSONB (CARC/RARC codes + reason). Store raw in `era_835`.
- Auto-post insurance payment to `payments` (`payment_method='insurance_payment'`, link `claim_id`).
- Denial worklist `GET /v1/claims?status=denied`.

**Testing:**
- `Unit: parse fixture 835 → claim allowed/paid/patient amounts correct`
- `Unit: 835 with CARC denial code → claim.denial populated, status denied`
- `Integration: post ERA → payment row created, claim status updated`
- `Unit: 835 referencing unknown claim → logged as unmatched, no crash`

#### 4.4 — Patient billing, statements & payments

**What:** Patient ledger, statement generation, and payment collection via Stripe.

**Design:**
- Ledger derived per patient: charges (procedures) − insurance payments − patient payments = balance. `GET /v1/patients/:id/ledger`.
- `POST /v1/payments` `{patientId, amountCents, paymentMethod, claimId?}`; card payments create a Stripe PaymentIntent; on webhook success record `payments` row.
- `GET /v1/patients/:id/statement?from=&to=` → statement document (line items, balance, aging buckets 0–30/31–60/61–90/90+).
- Stripe webhook endpoint with signature verification.

**Testing:**
- `Unit: ledger with 1 charge 22000, insurance 11000, patient 5000 → balance 6000`
- `Integration (mocked Stripe): card payment intent succeeds → payment row created`
- `Unit: Stripe webhook bad signature → 400, no payment recorded`
- `Unit: statement aging buckets computed from charge dates`

---

## Phase 5: Developer REST API, Webhooks & OpenAPI

### Purpose
Deliver the headline differentiator: a self-serve, documented, versioned REST API with developer key signup and event-driven webhooks. This phase formalises the public API surface (which earlier phases already built internally) and exposes it under stable contracts. Can be developed in parallel with Phase 6.

### Tasks

#### 5.1 — Self-serve developer keys & scopes

**What:** Developer account signup that issues scoped API keys without manual approval.

**Design:**
- `api_keys` table `{id, practice_id, name, key_hash, scopes text[], rate_limit_tier, last_used_at, revoked_at, created_at}`. Key shown once; stored as SHA-256 hash.
- Scopes: `patients:read`, `patients:write`, `appointments:read/write`, `claims:read`, `clinical:read`, `webhooks:manage`. Map to the same RBAC policy as `user_type='developer'`.
- `POST /v1/developer/keys`, `GET`, `DELETE /:id` (revoke). HIPAA: revoke takes effect immediately (key cache TTL ≤ 60s).
- API-key auth path in the auth plugin parallel to OIDC bearer.

**Testing:**
- `Integration: create key → returned once, subsequent GET shows masked`
- `Integration: request with key lacking 'claims:read' → 403`
- `Integration: revoked key → 401 within cache TTL`

#### 5.2 — OpenAPI 3.1 spec + rate limiting

**What:** Auto-generate the OpenAPI spec from Zod/JSON-Schema route definitions and serve interactive docs; enforce per-tier rate limits.

**Design:**
- `@fastify/swagger` + `zod-to-json-schema` produce `/openapi.json` and Swagger UI at `/docs`.
- All `/v1` routes carry JSON Schema for params/body/response.
- `@fastify/rate-limit` keyed by API key with tiers (`free`: 60/min, `pro`: 600/min) from `api_keys.rate_limit_tier`; 429 with `Retry-After`.

**Testing:**
- `Integration: GET /openapi.json → valid OpenAPI 3.1, includes /v1/patients with schemas`
- `Integration: exceed free-tier rate → 429 with Retry-After`
- `Contract: every /v1 route has a documented response schema (spec lint)`

#### 5.3 — Event-driven webhooks

**What:** Outbound webhooks for domain events with signed payloads, retries, and delivery logs.

**Design:**
- `webhooks` table `{id, practice_id, url, event_types text[], secret, is_active}`; `webhook_deliveries` `{id, webhook_id, event_type, payload, status, attempts, response_code, next_retry_at}`.
- Events: `patient.created`, `appointment.scheduled`, `appointment.cancelled`, `encounter.signed`, `claim.submitted`, `claim.paid`, `claim.denied`, `payment.collected`.
- Domain emits events to a `webhooks` queue; worker POSTs JSON with `X-Signature` (HMAC-SHA256 over body using webhook `secret`); exponential backoff retries (max 6), then dead-letter.
- `POST/GET/DELETE /v1/webhooks`.

**Testing:**
- `Unit: HMAC signature matches recomputed digest`
- `Integration (mock receiver): claim.paid event → POST delivered with valid signature, delivery logged`
- `Integration: receiver returns 500 → retry scheduled with backoff; success on retry → status delivered`
- `Unit: webhook only fires for subscribed event_types`

---

## Phase 6: AI-Native Clinical Features

### Purpose
Deliver the AI-native advantage that differentiates the platform: CDT code suggestion from clinical notes, radiograph analysis, treatment-plan narrative generation, no-show prediction, and eligibility/fee optimisation. Each writes to `encounters.imaging[].ai_findings` or a dedicated `ai_analyses` record and is always provider-reviewable. Requires Phases 3–4. Can parallel Phase 5.

### Tasks

#### 6.1 — AI infrastructure & ai_analyses store

**What:** AI SDK wrapper, prompt templates, and an `ai_analyses` table for auditable AI output.

**Design:**
- `ai_analyses` table (from Suggestion 1): `{id, encounter_id?, patient_id?, analysis_type, content, score, details JSONB, model_version, created_at}`; `analysis_type` enum: `radiograph_detection`, `cdt_suggestion`, `treatment_plan_narrative`, `no_show_prediction`, `fee_optimisation`, `clinical_note_draft`.
- `packages/ai` wraps Vercel AI SDK + Anthropic; `generateObject` with Zod schemas for structured output. Every result persisted with `model_version` and surfaced as a suggestion (never auto-applied to clinical records).
- All AI calls run in the `ai-jobs` worker queue (async, retriable).

**Testing:**
- `Unit (mocked model): generateObject returns typed object → persisted to ai_analyses with model_version`
- `Integration: AI job failure → retried, error recorded, no partial clinical write`

#### 6.2 — CDT code suggestion from clinical notes

**What:** Read free-text clinical notes and suggest CDT codes with confidence and rationale.

**Design:**
- Input: encounter `clinical_notes` + `findings`. System prompt: "You are a dental coding assistant. Given clinical documentation, suggest CDT 2026 procedure codes. Only suggest codes present in the provided CDT list. Return code, description, confidence (0–1), and rationale." Provide the active `cdt_codes` subset as context.
- Output schema: `z.array(z.object({ cdt: z.string(), description: z.string(), confidence: z.number(), rationale: z.string() }))`. Validate every `cdt` against `cdt_codes`; drop hallucinated codes.
- `POST /v1/encounters/:id/ai/cdt-suggestions` → suggestions presented for provider acceptance into `procedures`.

**Testing:**
- `Unit (mocked): notes "two-surface composite tooth 14 DO" → suggests D2391, confidence > 0.7`
- `Unit: model returns code not in cdt_codes → filtered out`
- `Integration: suggestions endpoint → ai_analyses row, no auto-write to procedures`

#### 6.3 — Radiograph analysis

**What:** Run vision analysis on a stored radiograph to flag caries, bone loss, and pathology per tooth.

**Design:**
- Trigger on `encounter.radiograph_captured`; fetch image from object storage; call vision model.
- Output schema: `z.array(z.object({ tooth: z.string(), finding: z.enum(['caries','bone_loss','periapical_lesion','calculus','other']), surface: z.string().optional(), confidence: z.number() }))`.
- Write to that image's `imaging[].ai_findings` and an `ai_analyses` row. Findings render as overlay flags for provider confirmation.

**Testing:**
- `Unit (mocked vision): fixture image → findings array with tooth+confidence`
- `Integration: capture radiograph → ai-job enqueued → imaging[].ai_findings populated`
- `Unit: low-confidence (<0.5) findings flagged as low-confidence, not auto-added to chart`

#### 6.4 — Treatment-plan narrative generation

**What:** Translate a treatment plan into patient-friendly language with anticipated objections and financing options.

**Design:**
- Input: accepted/presented `treatment_plan` JSONB + patient estimate. Prompt produces `{summary, perPhaseExplanation[], anticipatedObjections[], financingOptions[]}` in plain language at ~8th-grade reading level.
- `POST /v1/encounters/:id/ai/plan-narrative` → stored as `ai_analyses` (`treatment_plan_narrative`), available to portal/print.

**Testing:**
- `Unit (mocked): plan with 2 phases → narrative covers both phases and patient cost`
- `Integration: narrative endpoint → ai_analyses row, retrievable`

#### 6.5 — No-show prediction & adaptive reminders

**What:** Score appointment no-show risk and adjust reminder cadence/channel.

**Design:**
- Feature vector from history: prior no-shows, lead time, appointment type, day/time, prior confirmations. A lightweight logistic model in `core/domain/no-show.ts` (deterministic, testable) computes `riskScore 0–1`; optionally augmented by AI for new patients with sparse history.
- High risk (>0.6) → escalate cadence (add phone-step, more SMS) via the reminders queue; store score in `ai_analyses` (`no_show_prediction`).

**Testing:**
- `Unit: patient with 3 prior no-shows, short lead time → risk > 0.6`
- `Unit: long-tenured patient, always confirmed → risk < 0.2`
- `Integration: high-risk appointment → escalated reminder cadence enqueued`

#### 6.6 — Eligibility & fee optimisation at plan time

**What:** Compute expected reimbursement and out-of-pocket at treatment-plan creation using live eligibility + fee schedule.

**Design:**
- Combines Phase 4.1 eligibility data + Phase 3.4 fee calc. `GET /v1/encounters/:id/treatment-plan/estimate` returns per-procedure and total `{feeCents, insuranceEstCents, patientEstCents}` using the patient's current `remainingCents`/deductible.
- Flags when annual max will be exceeded mid-plan and suggests phase resequencing across benefit years.

**Testing:**
- `Unit: plan total 142000, remaining max 71000 → flags max-exceeded, recommends phasing`
- `Integration: estimate endpoint reflects latest eligibility check`

---

## Phase 7: Imaging (DICOM) & FHIR Interoperability

### Purpose
Add DICOM-compliant radiograph storage and HL7 FHIR R4 export, enabling imaging-hardware bridges and medical-dental interoperability aligned with the Dental Data Exchange IG and CMS-0057-F. Requires Phases 2–3.

### Tasks

#### 7.1 — Radiograph storage & DICOM ingestion

**What:** Upload, store (object storage), and serve radiographs with DICOM metadata.

**Design:**
- `POST /v1/encounters/:id/imaging` multipart upload; accept DICOM (`.dcm`) and standard images. Parse DICOM tags (Study/Series/SOP UID, modality) with a DICOM parser; store binary in object storage at `practiceId/patientId/encounterId/uid`.
- Record into `encounters.imaging[]`: `{type, teeth[], dicomUid, storagePath, capturedAt}`.
- `GET /v1/imaging/:id` streams the image (audited as PHI access).

**Testing:**
- `Unit: parse fixture .dcm → extracts Study UID, modality`
- `Integration: upload radiograph → imaging[] entry + object in MinIO`
- `Integration: GET image → audit row written`

#### 7.2 — FHIR R4 export

**What:** Export Patient, Encounter, Procedure, and Claim as FHIR R4 resources aligned to the Dental Data Exchange IG.

**Design:**
- `packages/fhir` mappers: `toFhirPatient`, `toFhirEncounter`, `toFhirProcedure` (CDT as `code.coding`), `toFhirClaim`. Tooth references use FHIR `bodySite`/tooth extensions; SNODENT where licensed.
- `GET /v1/fhir/Patient/:id`, `GET /v1/fhir/Encounter/:id`, `GET /v1/fhir/metadata` (CapabilityStatement).
- Validate output against FHIR R4 JSON schema in tests.

**Testing:**
- `Unit: toFhirProcedure(D2391) → Procedure with CDT coding system`
- `Unit: exported Patient validates against FHIR R4 schema`
- `Integration: GET /v1/fhir/Patient/:id → valid resource, tenant-scoped, audited`

---

## Phase 8: Web Application (Dashboard & Patient Portal)

### Purpose
Deliver the browser-based UI: a staff dashboard (schedule, charting, billing, reporting) and a patient portal (booking, intake forms, statements, secure messaging). Requires the relevant API phases; UI areas can be built incrementally as their APIs land.

### Tasks

#### 8.1 — Staff dashboard shell, auth & scheduling calendar

**What:** Next.js app with OIDC login, role-aware navigation, and the multi-location/operatory calendar.

**Design:**
- App Router; server components fetch via the `/v1` API with the user's token. Layout: schedule, patients, charting, billing, reports.
- Calendar: day/week views by operatory and provider; drag-to-create calls `POST /v1/appointments` with conflict feedback; status chips reflect the appointment state machine.

**Testing:**
- `E2E (Playwright): staff logs in → sees only their practice's schedule`
- `E2E: create appointment → appears on calendar; overlapping → conflict error shown`

#### 8.2 — Charting UI (tooth chart, perio, treatment plan)

**What:** Interactive SVG tooth chart, perio chart grid, and treatment-plan builder with live estimates.

**Design:**
- SVG odontogram (Universal Numbering); clicking a tooth opens findings/condition editor → `PUT tooth-chart`. Color-codes conditions (caries, restoration, missing, implant).
- Perio grid: 6 inputs/tooth → `POST .../perio`; renders trend sparkline from `/perio-trends`.
- Treatment-plan builder: add procedures (with CDT autocomplete + AI suggestions from 6.2), phase assignment, live estimate from 6.6. Radiograph viewer shows AI overlay flags (6.3).

**Testing:**
- `E2E: chart caries on tooth 14 → persists, reflected on reload`
- `E2E: build treatment plan → totals and patient estimate display; present/accept flow works`
- `E2E: AI CDT suggestion appears and can be accepted into procedures`

#### 8.3 — Billing UI & reporting dashboards

**What:** Claims worklist, denial management, patient ledger/statements, and production dashboards.

**Design:**
- Claims list filterable by status (draft/submitted/denied); submit and resubmit actions; denial detail with CARC/RARC.
- Patient ledger with aging; record payment (Stripe element).
- Reports: production by provider/location, collections, claim denial rate by payer, treatment-plan acceptance rate, recall reappointment rate (queries over relational + JSONB per Suggestion 2 example queries).

**Testing:**
- `E2E: submit claim from worklist → status updates to submitted`
- `E2E: production report renders provider totals for date range`

#### 8.4 — Patient portal

**What:** Patient-facing online booking, digital intake forms, statements, and secure messaging.

**Design:**
- Separate patient auth realm; portal scoped to own record only (enforced by RBAC + tenant scope).
- Online booking against available operatory/provider slots; intake forms write to patient `medical_history`/`contact`; statement view from 4.4; secure messaging thread stored per patient with staff inbox.

**Testing:**
- `E2E: patient books an available slot → appointment created, confirmation reminder enqueued`
- `E2E: patient submits intake form → patient record updated`
- `E2E: patient cannot access another patient's record → 403`

---

## Phase 9: Multi-Location/DSO, Hardening & Compliance

### Purpose
Make the platform DSO-ready and production-grade: centralised multi-location dashboards, SOC 2/HIPAA hardening, encryption verification, observability, and deployment artefacts for both SaaS and self-hosted modes. Requires all prior phases.

### Tasks

#### 9.1 — Multi-location centralised dashboards

**What:** Cross-location rollups for DSO administrators.

**Design:**
- Aggregate production, collections, denial rates, and recall metrics across all locations of a practice; location filter and roll-up totals. Reuse Phase 8.3 queries grouped by `location_id` from `practice.settings.locations`.
- Role `admin` sees all locations; location staff scoped to their `location_ids` (from provider `credentials`).

**Testing:**
- `Integration: admin dashboard aggregates two locations → combined totals correct`
- `Integration: location-scoped user → sees only their location's data`

#### 9.2 — Security hardening & encryption

**What:** Enforce TLS 1.2+, AES-256 at rest, secrets management, and least-privilege DB roles.

**Design:**
- TLS termination config (TLS 1.3 preferred); object storage server-side encryption (AES-256); Postgres at-rest encryption documented (volume/KMS).
- Separate DB roles: app role (no DELETE on `audit_log`), migration role, read-only reporting role.
- Security headers (HSTS, CSP), input size limits, dependency audit in CI (`pnpm audit`).
- Break-glass access flow writes `access.break_glass` audit event.

**Testing:**
- `Integration: app DB role cannot DELETE from audit_log → permission denied`
- `Unit: security headers present on responses`
- `Integration: break-glass access → audit event recorded with justification`

#### 9.3 — Observability & operations

**What:** Structured logging, tracing, health checks, and backup/restore runbook.

**Design:**
- Pino JSON logs (PHI-scrubbed); OpenTelemetry traces across api/worker/edi; `/healthz` and `/readyz`.
- Backup: nightly `pg_dump` + object-storage replication; documented restore procedure; audit-log partition rotation job verified.

**Testing:**
- `Integration: /healthz → 200 when DB+Redis reachable; 503 when DB down`
- `Unit: log redaction removes patient names/SSN from log lines`
- `Integration: backup → restore into fresh DB → row counts match`

#### 9.4 — Deployment (SaaS + self-hosted)

**What:** Production Docker images, Compose stack for self-hosted, and SaaS deploy config.

**Design:**
- `Dockerfile.api/worker/web` multi-stage builds; `docker-compose.yml` wires postgres/redis/minio/api/worker/web with healthchecks for the self-hosted (`APP_MODE=self_hosted`) path.
- SaaS deploy (Vercel for `web`; containerised api/worker on a managed platform); managed Postgres/Redis/S3.
- One-command bootstrap script: migrate → seed CDT → create demo practice.

**Testing:**
- `Integration: docker compose up → all services healthy, /healthz green`
- `E2E (self-hosted): bootstrap → log in → create patient → book appointment → sign encounter → submit claim (mock clearinghouse) → all succeed`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, DB, audit, tenant)      ─── required by everything
    │
Phase 2: Patients, Scheduling, Recall              ─── requires Phase 1
    │
Phase 3: Clinical Charting & Treatment Planning    ─── requires Phase 2  (CORE VALUE)
    │
Phase 4: Billing, Eligibility & X12 Claims         ─── requires Phase 3  (CORE VALUE)
    │
    ├── Phase 5: Developer API & Webhooks           ─── requires Phase 4; can parallel Phase 6
    ├── Phase 6: AI-Native Clinical Features         ─── requires Phases 3–4; can parallel Phase 5
    └── Phase 7: Imaging (DICOM) & FHIR Export       ─── requires Phases 2–3; can parallel Phases 5–6
         │
Phase 8: Web App (Dashboard & Patient Portal)      ─── requires APIs from Phases 2–7 (built incrementally)
    │
Phase 9: Multi-Location/DSO, Hardening, Compliance ─── requires all prior phases
```

**Parallelism opportunities:**
- Phases 5, 6, and 7 can be developed concurrently once Phase 4 is complete (Phase 7 needs only 2–3).
- Within Phase 8, each UI area (8.1–8.4) can be built as soon as its backing API phase lands.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks implemented.
2. All unit and integration tests pass (`turbo test`).
3. ESLint and Prettier pass with no errors.
4. `tsc --noEmit` (strict) passes across all packages.
5. Database migrations created, applied cleanly, and idempotent.
6. New `/v1` endpoints appear in the auto-generated OpenAPI 3.1 spec with request/response schemas.
7. New JSONB clinical documents have Zod validators and round-trip tests.
8. Every PHI-accessing endpoint writes an `audit_log` row (verified by test).
9. New config/environment variables documented in `.env.example`.
10. Docker build succeeds for affected services.
11. The phase's feature works end-to-end (relevant E2E test green) for both `saas` and `self_hosted` modes where applicable.
12. No secrets, patient data, or PHI in logs (redaction test passes).
```
