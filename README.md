# Dental Practice Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open practice management platform for dental clinics covering charting, treatment planning, insurance claims, and scheduling.

Dental Practice Management is a candidate project for an end-to-end clinical and administrative system serving solo dentists, group practices, dental service organisations (DSOs), and specialist practices. It targets the core pain points of incumbent dental PMS suites: high cost, legacy desktop architectures, opaque pricing, and locked-down developer access.

---

## Why Dental Practice Management?

- Market leaders Dentrix and Eaglesoft are quote-only, expensive, and built on legacy desktop architectures with limited mobility and real-time multi-location data.
- Cloud-native alternatives such as Curve Dental and Planet DDS / Denticon carry premium pricing and remain overkill for solo or small group practices.
- Open Dental is the main open-source option but requires significant self-hosting and IT overhead, and its UI lags behind cloud-native competitors.
- Across the category, API access typically requires signed agreements rather than self-serve developer keys, blocking independent integrations.
- Total first-year cost of ownership for a full-featured implementation can reach USD 20,000–80,000, leaving room for a transparent, lower-cost AI-native alternative.

---

## Key Features

### Clinical Charting & Treatment Planning

- Graphical tooth chart with procedure recording and visual dentition history
- Periodontal charting and treatment planning workflows
- Clinical notes and progress documentation
- Specialty-aware charting hooks for oral surgery, orthodontics, periodontics, and endodontics

### Scheduling & Patient Engagement

- Provider and operatory calendar management with multi-location views
- Automated recall and appointment reminders via SMS and email
- Patient portal with online booking, digital intake forms, and secure messaging
- Two-way patient messaging and recall automation

### Billing, Insurance & Revenue Cycle

- Real-time insurance eligibility verification
- X12 837D electronic claim submission and 835 remittance posting
- Patient billing, statements, and payment collection
- Real-time fee schedule and expected out-of-pocket estimation at treatment plan creation

### Imaging & Interoperability

- Digital radiograph storage with bridges to major imaging hardware
- DICOM-compliant image handling
- FHIR-based data exchange for medical-dental interoperability
- Open data warehouse export for analytics

### Platform & Compliance

- HIPAA-compliant data storage, role-based access control, and audit trails
- Multi-provider and multi-location support with centralised dashboards
- Open REST API with self-serve developer key signup
- CDT 2026 procedure code set support, kept current with annual ADA updates

---

## AI-Native Advantage

AI is integrated directly into the clinical and administrative workflow rather than bolted on as a separate add-on. Native radiograph analysis flags potential caries, bone loss, and pathology at time of capture; clinical notes are parsed to suggest correct CDT procedure codes and reduce claim denials; treatment plans are translated into patient-friendly language with anticipated objections and financing options; and predictive no-show scoring adjusts reminder cadence per patient. Real-time insurance eligibility and fee schedule optimisation produce accurate financial estimates before any procedure begins.

---

## Tech Stack & Deployment

The project targets a cloud-native deployment with optional self-hosted operation, aligning with the >55% market share now held by cloud dental PMS. Core interoperability is built around CDT 2026, X12 837D / 835 EDI, DICOM for imaging, HL7 FHIR for medical-dental data exchange, NPI provider identifiers, and emerging ANSI/ADA standards for electronic records and treatment planning. A self-serve REST API and event-driven webhooks are intended from day one, addressing the gated developer access seen across most incumbents.

---

## Market Context

The global dental practice management software market is valued at approximately USD 2.62–2.64 billion in 2026, projected to reach USD 4.16–5.33 billion by 2031–2035 at a 7–11% CAGR, with North America holding more than 53% of share and the US alone exceeding USD 1 billion (Grand View Research; Fortune Business Insights; Mordor Intelligence; GlobeNewswire). Enterprise suites are quote-only, while smaller cloud platforms typically start at USD 200–500/month per location and CareStack publishes ~USD 400–600/month/location. Primary buyers are solo dentist owners, group practices, DSOs operating 10+ locations, and specialty practices such as orthodontists and oral surgeons.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context. Note that the CDT procedure code set and SNODENT terminology are registered trademarks of the American Dental Association and require licensing for any commercial redistribution.
