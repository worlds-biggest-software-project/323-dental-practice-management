# Dental Practice Management — Feature & Functionality Survey

> Candidate #323 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Dentrix / Dentrix Ascend | Hybrid / Cloud SaaS | Proprietary — quote-only | https://www.dentrix.com / https://www.dentrixascend.com |
| Eaglesoft (Patterson) | Desktop / Hybrid | Proprietary — quote-only | https://www.pattersondental.com/cp/software/dental-practice-management-software/eaglesoft |
| Open Dental | Self-hosted / Cloud option | Open source (GPL) — maintenance fee | https://www.opendental.com |
| Curve Dental (Curve SuperHero) | Cloud SaaS | Proprietary — quote-only | https://www.curvedental.com |
| Planet DDS / Denticon | Cloud SaaS | Proprietary — quote-only | https://www.planetdds.com |
| CareStack | Cloud SaaS | Proprietary — ~$400–600/month/location | https://carestack.com |
| tab32 | Cloud SaaS | Proprietary — tiered pricing | https://tab32.com |
| Dovetail | Cloud SaaS | Proprietary — tiered pricing | https://softwarefinder.com/emr-software/dovetail |

---

## Feature Analysis by Solution

### Dentrix / Dentrix Ascend

**Core features**
- Clinical charting with colour-coded tooth diagram, freehand drawing, and perio charting
- Appointment scheduling with provider-level calendar management and Practice Growth booking suite
- Electronic claims (eClaims) submission, eligibility verification, and remittance posting
- Billing, line-item accounting, and patient statements
- Imaging integration bridging to major radiography vendors
- AI-powered radiograph analysis (VideaHealth integration)
- Patient engagement: automated appointment reminders, recalls, and two-way messaging
- Analytics dashboards and production reporting
- Dentrix Ascend (cloud variant) provides multi-location access from any device

**Differentiating features**
- Largest installed base (35,000+ practices) creating the widest third-party integration ecosystem
- Henry Schein One API Exchange connects Dentrix and Dentrix Ascend to 55+ authorised vendor integrations
- Dentrix Enterprise variant targets large DSOs with centralised administration

**UX patterns**
- Long-tenured desktop interface; Ascend is a modernised browser-based redesign
- Learning curve is notable; extensive certification programmes and online training modules
- On-screen prompts guide treatment planning and insurance eligibility workflows

**Integration points**
- API Exchange at ddp.dentrix.com — read-only stored procedures and views; Ascend API Exchange for bidirectional integrations
- Imaging bridges with 55+ authorised vendors (Dentsply Sirona, Carestream, DEXIS, Planmeca, etc.)
- Pearl AI for radiograph detection; Weave and RevenueWell for patient communications
- DentalXChange and other EDI clearinghouses for X12 837D claim submission

**Known gaps**
- High cost and complex interface are the most-cited complaints
- Desktop-first Dentrix lags in mobility and real-time multi-location data
- Ascend has fewer advanced reporting options than the legacy desktop product
- API access requires signed agreement and is not self-serve

**Licence / IP notes**
- Proprietary software; no publicly disclosed patents, but API usage governed by Henry Schein One API Agreement
- Dentrix SDK is downloadable from developer portal after agreement acceptance

---

### Eaglesoft (Patterson Dental)

**Core features**
- Clinical charting, periodontal templates, and SmartNote clinical documentation
- Patient scheduling with QuickFill for filling open appointment slots
- Advanced imaging with no additional bridge fees; integrated capture and storage
- Billing, line-item accounting, and MoneyFinder revenue opportunity tool
- InContact for digital patient lists, follow-up tasks, and outreach management
- Insurance eligibility and claim management
- Integration with Pearl Second Opinion AI for radiograph detection (announced 2024)

**Differentiating features**
- Deep integration with Patterson Dental supply chain (same vendor for equipment and software)
- 55+ authorised imaging integrations at no additional bridge cost
- MoneyFinder surfaces unbilled procedures and hidden revenue opportunities

**UX patterns**
- Legacy Windows desktop application; familiar interface to long-time Patterson customers
- Relies heavily on right-click context menus and task-based workflow panels
- Onboarding tied to Patterson sales and support representatives

**Integration points**
- Interfaces with Dentsply Sirona Sidexis, DEXIS, Planmeca, Air Techniques, Instrumentarium, Soredex, and Progeny
- Pearl AI radiograph detection (Second Opinion integration)
- EDI clearinghouse connectivity for X12 837D/835 claim flows

**Known gaps**
- Legacy architecture limits cloud and mobile access
- No native cloud deployment; server installation required
- API/developer ecosystem is less open than Dentrix API Exchange or Planet DDS

**Licence / IP notes**
- Proprietary software; developer access tightly restricted

---

### Open Dental

**Core features**
- Graphical tooth chart with visual dentition history for patient communication
- Full scheduling module: provider calendars, operatory utilisation, recall tracking, confirmation management
- Treatment planning and clinical notes
- Insurance plan management, claim creation, EDI submission via clearinghouse integrations
- Patient billing, statements, and payment processing
- eServices suite: patient portal, web scheduling, web forms, secure messaging, integrated texting, automated reminders, mobile app
- RESTful API for custom integrations (api.opendental.com)
- Imaging bridges with most major vendors via open plugin model

**Differentiating features**
- Open-source GPL codebase allows practices and developers to customise and extend without vendor lock-in
- Lowest total cost (self-hosted, minimal licensing fee; optional maintenance contract)
- Active community forum and user-contributed plugin library
- Open Data Warehouse partnership with tab32 for standardised data export

**UX patterns**
- Functional but dated Windows interface; significant configuration required
- Highly configurable but setup complexity is a barrier for practices without IT support
- Community documentation supplements official manual

**Integration points**
- REST API at api.opendental.com; developer key required; supports GET, POST, PUT, DELETE
- Clearinghouse connections for X12 837D (DentalXChange and others)
- Imaging bridges for major vendors
- Third-party patient communication and analytics tools via API

**Known gaps**
- Self-hosting complexity and IT overhead deter smaller or non-technical practices
- UI modernisation lags behind cloud-native competitors
- Cloud-hosted version (via third-party hosts) adds cost without the managed-service benefits of native SaaS

**Licence / IP notes**
- GPL v2 or v3 open-source licence; source available on SourceForge
- Third-party plugins may carry their own licences; clearinghouse integrations are proprietary

---

### Curve Dental (Curve SuperHero)

**Core features**
- Scheduling across all locations on a single screen; multi-location calendar switching
- Clinical charting and treatment planning
- Imaging integration (cloud-stored, accessible from any location)
- Billing, insurance eligibility verification, and claims management
- Patient engagement: reminders, recalls, online booking, two-way messaging
- Consolidated reporting dashboards with macro and micro views for DSO analytics
- SOC 2 Type II compliant; data encrypted at rest (AES-256) and in transit (TLS 1.3) on AWS

**Differentiating features**
- Pure-cloud architecture built on AWS from inception — not a desktop port
- Single unified patient record across all locations (no per-site record silos)
- Centralised backup management eliminates per-location server overhead
- Remote management of scheduling, records, and production from any device

**UX patterns**
- Modern browser-based interface; low hardware requirements
- Progressive onboarding with structured implementation programme
- Partner marketplace accessible from within the application

**Integration points**
- Partner network (Weave, Amplify360, Vyne Dental, Birdeye, AirPay, Affirm, MMG Fusion, TNT Dental)
- API tracker lists Curve Dental API; GitHub organisation (github.com/curvedental) with 26 repositories
- DentalHQ integration partnership announced 2025 for membership plan management
- Availity and Cigna Healthcare EDI connectivity

**Known gaps**
- Premium pricing relative to Open Dental and smaller cloud tools
- API documentation is not publicly self-serve; developer access requires partner agreement
- Fewer specialty-specific charting features than enterprise competitors for oral surgery or orthodontics

**Licence / IP notes**
- Proprietary SaaS; no publicly disclosed open-source components

---

### Planet DDS / Denticon

**Core features**
- Cloud-based scheduling, clinical charting, and treatment planning
- Voice-driven clinical charting (single-provider documentation added 2026)
- Centralised analytics dashboards: production and KPIs across all locations in real time
- Billing and revenue cycle management; centralised billing hub for DSOs
- Patient engagement via Legwork (acquired): reminders, reviews, recall automation
- Imaging via Apteryx (acquired): digital radiography and cloud image storage
- AI agents (early access 2026) connected directly to Denticon and Cloud 9 orthodontic module

**Differentiating features**
- Purpose-built for DSOs managing 5–200+ locations
- Full ecosystem acquisition strategy: Denticon (PM) + Apteryx (imaging) + Legwork (engagement) + Cloud 9 (ortho)
- Comprehensive open API program launched 2024 for patient, appointment, financial, and clinical data
- Event-driven API architecture for real-time data synchronisation

**UX patterns**
- Enterprise-grade interface; steeper learning curve accepted in DSO context
- Centralised dashboards surfaced to executives and regional managers
- Location-level staff access scoped to their own operatories and schedules

**Integration points**
- Developer portal at developer.planetdds.com; interactive API explorer; self-service key signup
- Event-driven webhooks for patient and appointment data synchronisation
- Third-party RCM, patient communication, and clinical tool integrations via API

**Known gaps**
- Overkill and cost-prohibitive for solo or two-dentist practices
- Imaging (Apteryx) requires separate licensing from core Denticon
- AI features are still in early access and limited to select customers

**Licence / IP notes**
- Proprietary SaaS; DentalOS branding under Planet DDS parent

---

### CareStack

**Core features**
- Practice management: scheduling, clinical charting, treatment planning
- Insurance eligibility verification (real-time, automated)
- Billing with Kaylie AI integration for AI-powered claims management and payer adjustment identification
- Patient engagement: online booking, two-way SMS/email reminders, recalls
- Collections automation: overdue account segmentation, SMS/email with embedded pay-links, text-to-pay
- AI Suite (2026): 24/7 AI answering service that books and triages visits automatically
- Centralised billing hub for multi-location groups
- Analytics and reporting dashboards

**Differentiating features**
- Most comprehensive AI layer of any current-generation platform: Kaylie AI for revenue cycle, AI phone agent for 24/7 new-patient capture
- No per-user fees; single per-location monthly price (~$400–600)
- Collections-first philosophy: identifies payer adjustments and patient balances in real time
- Open developer portal at developer.carestack.com for custom integrations

**UX patterns**
- Modern browser-based interface with dashboard-first layout
- Progressive disclosure: admin-level analytics vs. front-desk scheduling views
- Steep learning curve for the full feature set, noted as a drawback for solo practices

**Integration points**
- Open API at developer.carestack.com
- Kaylie AI (revenue cycle AI), Pearl AI (radiograph analysis)
- DentalXChange and similar clearinghouses for X12 837D/835
- Payment processing, patient financing, and reputation management partners

**Known gaps**
- Feature depth can overwhelm small solo practices
- AI features are add-ons that increase effective price
- Smaller install base than Dentrix or Eaglesoft limits community support resources

**Licence / IP notes**
- Proprietary SaaS; AI modules (Kaylie, Pearl) are third-party integrations with their own terms

---

### tab32

**Core features**
- Cloud scheduling, clinical charting, and treatment planning
- Imaging integration with cloud storage
- Billing and insurance claims (EDI via DentalXChange, covering ~80% of insurers)
- Patient communication via HelloPatient: two-way messaging (SMS, email, calls, voicemail), appointment reminders, recall automation, online booking, digital intake forms, reputation management
- Pearl AI integration for real-time radiograph condition detection
- Open Data Warehouse powered by Google BigQuery for standardised EHR analytics
- FHIR-compliant data model (first dental platform to standardise EHR data per published claims)

**Differentiating features**
- First dental platform to standardise EHR data and offer a cloud open data warehouse (BigQuery)
- FHIR-first architecture aligned with CMS-0057-F interoperability timelines
- HelloPatient is a fully integrated patient engagement module rather than an add-on
- Pearl AI detection drives 30–37% higher treatment acceptance (per platform claims)

**UX patterns**
- Modern browser-based interface targeting independent practices and small groups
- Embedded analytics reduce need for external BI tools
- Onboarding focused on independent practice owners rather than DSO IT teams

**Integration points**
- Pearl AI, Looker Studio, Stripe, DentalXChange, Hatch, Google BigQuery, Birdeye
- FHIR-compliant data exchange for medical-dental integration
- Note: third-party API endpoint access not prominently self-documented; developer access by arrangement

**Known gaps**
- Smaller install base creates less community support compared to Dentrix or Open Dental
- API access for external developers not prominently self-serve
- Limited multi-location DSO-scale reporting compared to Planet DDS or Curve

**Licence / IP notes**
- Proprietary SaaS; BigQuery and Pearl integrations governed by respective vendor terms

---

### Dovetail

**Core features**
- Cloud EHR and practice management built by dentists for dentists
- Treatment planning, clinical charting, and progress notes
- Punch clock / staff time tracking
- In-platform chat for internal team communication
- Patient portal and online booking
- E-claims and image management
- Mobile payments
- Seamless integration with leading dental imaging hardware

**Differentiating features**
- All-in-one design including staff chat and time clock not typically bundled in competing tools
- Paperless-first design: eliminates manual data entry via browser-based forms
- Mobile-accessible from computer, tablet, or phone
- Created by practising dentists, emphasising clinical workflow alignment

**UX patterns**
- Emphasis on intuitive design and simplicity for patient engagement
- Minimal setup friction targeted at small independent practices
- Fully browser-based; no hardware requirements beyond a device with a browser

**Integration points**
- Integrations with major dental imaging software and equipment vendors
- E-claims through standard EDI clearinghouses
- Limited published API or developer ecosystem documentation

**Known gaps**
- Lacks advanced features needed by large or multi-location practices
- Smaller developer ecosystem and partner network
- Limited analytics and DSO-scale reporting
- No publicly documented API for third-party developers

**Licence / IP notes**
- Proprietary SaaS (Gaargle Solutions, Inc.); no disclosed open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Graphical tooth chart / clinical charting with procedure recording
- Appointment scheduling with provider and operatory calendars
- Insurance eligibility verification and X12 837D electronic claim submission
- Patient billing, statements, and payment collection
- Recall management and automated appointment reminders (SMS/email)
- Digital radiograph storage and bridge to imaging hardware
- HIPAA-compliant data handling, access controls, and audit trails
- Reporting and production dashboards

### Differentiating Features
- AI-powered radiograph analysis integrated in the clinical workflow (Pearl, VideaHealth)
- AI-driven revenue cycle management and claims scrubbing (Kaylie AI)
- 24/7 AI phone agent for new-patient capture and triage (CareStack)
- Open data warehouse / FHIR-first architecture for analytics and interoperability (tab32)
- Purpose-built DSO multi-location management with event-driven APIs (Planet DDS)
- Open-source extensibility with GPL codebase (Open Dental)
- All-in-one ecosystem through acquisitions: imaging + engagement + ortho + PM (Planet DDS)

### Underserved Areas / Opportunities
- **Transparent, predictable pricing** — all enterprise and mid-market tools hide pricing behind sales calls
- **Self-serve developer API access** — most platforms require signed agreements and manual approval before any API access
- **Specialty-specific workflows** — oral surgery, orthodontics, periodontics, and endodontics each require custom charting and procedure logic rarely well-served by general PMS platforms
- **Patient financial transparency** — real-time expected out-of-pocket calculation before treatment is rare and poorly executed
- **True medical-dental interoperability** — FHIR dental data exchange is specified but implementation is nascent
- **AI-native clinical documentation** — voice-to-note with automatic CDT coding is still early-stage across all platforms
- **No-show prediction and intelligent recall** — rule-based reminders dominate; predictive no-show models are missing
- **Small-practice simplicity at scale** — tools are either too simple (Dovetail) or too complex (Planet DDS) for the solo-to-small-group segment

### AI-Augmentation Candidates
- Automated CDT code suggestion from free-text clinical notes
- Radiograph analysis for caries, bone loss, and pathology detection
- Treatment plan narrative generation in patient-friendly language
- Insurance eligibility pre-check and fee schedule optimisation at plan creation time
- No-show risk scoring to adjust reminder cadence per patient
- Claim denial prediction and pre-submission claim scrubbing
- 24/7 conversational patient intake and scheduling
- Revenue cycle anomaly detection (unexpected payer adjustments, write-off patterns)

---

## Legal & IP Summary

All major commercial platforms (Dentrix, Eaglesoft, Curve Dental, Planet DDS, CareStack, tab32, Dovetail) are proprietary SaaS or hybrid products with standard commercial licensing. No publicly disclosed patents were identified that would restrict independent implementation of equivalent features. Open Dental is licensed under the GPL; any derivative or linked work must comply with GPL terms if distributing. The CDT procedure code set and SNODENT terminology are registered trademarks of the American Dental Association; licence fees apply to any commercial product that redistributes these code sets or their definitions. Third-party AI integrations (Pearl, VideaHealth, Kaylie AI) carry independent commercial terms. No other copyright or licence compatibility issues were identified for building an open-source alternative from scratch.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Appointment scheduling with provider and operatory calendar management
- Clinical charting (graphical tooth chart, procedure recording, treatment planning)
- Insurance eligibility verification and X12 837D/835 EDI claim submission and remittance
- Patient billing, statements, and payment collection
- Automated recall and appointment reminders (SMS and email)
- HIPAA-compliant data storage with role-based access control and audit trail

**Should-have (v1.1)**
- Digital radiograph storage with bridge to major imaging hardware (DICOM-compliant)
- AI-assisted CDT code suggestion from clinical notes
- Patient portal with online booking, digital intake forms, and secure messaging
- Multi-provider and multi-location support with centralised dashboards
- Open REST API with self-serve developer key signup

**Nice-to-have (backlog)**
- AI radiograph analysis (caries, bone loss, pathology flagging)
- Predictive no-show scoring and adaptive reminder cadence
- Real-time fee schedule and out-of-pocket estimate calculator at treatment plan creation
- Voice-driven clinical documentation (speech-to-note with CDT code suggestions)
- Open data warehouse / FHIR export for analytics and medical-dental interoperability
