# Standards & API Reference

> Project: Dental Practice Management · Generated: 2026-05-03

## Industry Standards & Specifications

### Terminology & Procedure Codes

**CDT — Current Dental Terminology (ADA)**
- Authority: American Dental Association
- URL: https://www.ada.org/publications/cdt
- The CDT code set is the HIPAA-mandated standard for describing dental procedures on claims and in clinical records. Updated annually; CDT 2026 introduced 31 new codes and 14 revisions covering saliva testing, prosthetics, implant maintenance, photobiomodulation, cracked-tooth diagnosis, and backup dentures. Any compliant dental billing system must update its code table each January. Commercial use of CDT codes requires an ADA licence.

**SNODENT — Systematized Nomenclature of Dentistry (ADA)**
- Authority: American Dental Association (official subset of SNOMED CT)
- URL: https://www.ada.org/resources/research/science-and-research-institute/dental-informatics/snodent
- SNODENT provides standardised clinical terminology for dental disease descriptions, patient characteristics, and clinical findings. It enables semantic interoperability between electronic dental records and medical EHR systems. Required for FHIR dental data exchange implementation. Commercial use requires an ADA SNODENT licence.

---

### Claims & Financial EDI

**ANSI X12 837D — Health Care Claim: Dental (005010X224A2)**
- Authority: ASC X12 / HIPAA mandate
- URL: https://x12.org/examples/005010x224
- The federally mandated EDI format for submitting dental claims electronically to insurance payers. Compliant with HIPAA 5010 (required since 2012). Specifies how dental procedure codes (CDT), patient demographics, provider NPI, and tooth-level detail are structured in an EDI transaction. Practices using EDI 837D report approximately 4 minutes saved per claim versus paper.

**ANSI X12 835 — Health Care Claim Payment/Advice**
- Authority: ASC X12 / HIPAA mandate
- URL: https://x12.org/examples/005010x221
- The standard EDI format for electronic remittance advice (ERA): the payer's response explaining what was paid, adjusted, or denied on each claim. Dental practice management software must parse 835 transactions to auto-post payments and identify denials.

**ANSI X12 270/271 — Eligibility Inquiry and Response**
- Authority: ASC X12 / HIPAA mandate
- URL: https://x12.org
- Used for real-time insurance eligibility verification before or at the time of appointment. Practices achieving above 95% claim acceptance rates rely on automated 270/271 verification workflows.

**NPI — National Provider Identifier**
- Authority: CMS / HIPAA mandate
- URL: https://www.cms.gov/Regulations-and-Guidance/Administrative-Simplification/NationalProvIDStand
- Every dental provider and practice must have a registered NPI that appears correctly on all X12 claims and patient records. Practice management software must maintain accurate provider credential records and validate NPI before claim submission.

---

### Health Record Interoperability

**HL7 FHIR R4 — Fast Healthcare Interoperability Resources**
- Authority: HL7 International
- URL: https://www.hl7.org/fhir/
- The modern, RESTful standard for healthcare data exchange. Increasingly required for medical-dental interoperability, patient access APIs, and payer-to-payer data exchange under CMS-0057-F. The FHIR dental data exchange implementation guide (see below) defines resource profiles for dental referral notes and consultation notes.

**HL7 FHIR Dental Data Exchange Implementation Guide (STU1 / v2.0.0-ballot)**
- Authority: HL7 International / CareQuest Institute
- URL: https://www.hl7.org/fhir/us/dental-data-exchange/STU1/ (STU1) and https://build.fhir.org/ig/HL7/dental-data-exchange/ (continuous build)
- Defines FHIR R4 resource profiles, extensions, and value sets for bi-directional data exchange between dental and medical providers. Based on ANS/ADA Specification No. 1084 (Reference Core Data Set). Uses SNODENT and CDT as terminology. Implementers must also support US Core and C-CDA profiles. Version 2.0.0 is in active ballot as of 2026.

**ISO 13606 — Electronic Health Record Communication**
- Authority: ISO / CEN
- URL: https://www.iso.org/standard/67868.html (ISO 13606-1:2019)
- A five-part ISO standard defining a reference model and archetype object model for interoperable EHR data exchange. Part 1 defines the reference model. Relevant for systems targeting EU markets or building on openEHR-derived architectures. Less commonly implemented in US dental software than HL7 FHIR, but referenced in EU Electronic Health Data Space (EHDS) planning.

**HL7 v2 Messaging**
- Authority: HL7 International
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185
- Older HL7 pipe-delimited message format still widely used for administrative messaging (ADT, ORM, ORU events) between practice management and lab, hospital, or referral systems. Legacy integrations in many established practice management platforms rely on HL7 v2.

---

### Imaging & Diagnostics

**DICOM — Digital Imaging and Communications in Medicine**
- Authority: NEMA (National Electrical Manufacturers Association) / ISO 12052
- URL: https://www.dicomstandard.org/
- The universal standard for storing and transmitting medical and dental images (intraoral radiographs, panoramic X-rays, cone-beam CT). Dental involvement in DICOM has been active since 1996 when the ADA joined the DICOM Committee. Practice management systems that store or display radiographs must support DICOM to ensure vendor-neutral interoperability. Key supplement: DICOM PS3 Dental Use Cases.

**IHE Profiles — Integrating the Healthcare Enterprise**
- Authority: IHE International
- URL: https://www.ihe.net/
- IHE profiles define workflow integration patterns using DICOM, HL7, and other standards to achieve consistent interoperability between imaging devices, PACS, and practice management software. Relevant IHE profiles for dental software include Scheduled Workflow (SWF), Patient Information Reconciliation (PIR), and XDS.b for shared document registries.

---

### Security, Privacy & Compliance

**HIPAA — Health Insurance Portability and Accountability Act (45 CFR Parts 160 and 164)**
- Authority: U.S. Department of Health and Human Services (HHS)
- URL: https://www.hhs.gov/hipaa/
- Mandates privacy and security controls for all covered entities and business associates handling protected health information (PHI). All dental practice management software must comply with the HIPAA Security Rule (encryption, access controls, audit logs) and Privacy Rule (minimum necessary disclosure, patient rights). PHI includes patient records, radiographs, billing data, and appointment information.

**SOC 2 Type II**
- Authority: AICPA
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- SaaS dental software used in DSO and enterprise contexts is increasingly expected to hold a SOC 2 Type II attestation, which independently validates that security controls (confidentiality, availability, processing integrity) were operating effectively over a defined period. Curve Dental and CareStack both publish SOC 2 Type II compliance.

**ISO/IEC 27001 — Information Security Management**
- Authority: ISO / IEC
- URL: https://www.iso.org/standard/27001
- International standard for information security management systems. Provides a framework for risk management and security controls that complements HIPAA and SOC 2 requirements in global deployments.

**OAuth 2.0 / OpenID Connect (OIDC)**
- Authority: IETF / OpenID Foundation
- URLs: https://datatracker.ietf.org/doc/html/rfc6749 and https://openid.net/connect/
- Standard protocols for delegated authorisation (OAuth 2.0) and federated identity (OIDC). Required for any patient-facing portal, third-party API access, and SSO integrations. HIPAA requires same-day revocation of access; SCIM directory sync is needed for automated offboarding in multi-staff practices.

**TLS 1.2+ / AES-256 Encryption**
- Authority: IETF (RFC 8446 for TLS 1.3)
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- Minimum encryption standards for data in transit (TLS 1.2; TLS 1.3 preferred) and at rest (AES-256). Required by HIPAA Security Rule and PCI DSS 4.0. Curve Dental uses AES-256 at rest and TLS 1.3 in transit on AWS.

---

### Emerging & Interoperability Standards

**CMS-0057-F — CMS Interoperability and Prior Authorization Final Rule**
- Authority: CMS (Centers for Medicare & Medicaid Services)
- URL: https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-and-prior-authorization-final-rule-cms-0057-f
- Requires payers to implement Patient Access, Provider Access, Payer-to-Payer Data Exchange, and Prior Authorization APIs by 2026–2027 timelines using HL7 FHIR R4. Dental software with insurance connectivity must align with these timelines to remain compliant for federally-covered dental benefits.

**ADA Specification No. 1084 — Reference Core Data Set for Dental-Medical Communication**
- Authority: American Dental Association
- URL: https://www.ada.org/resources/research/science-and-research-institute/dental-informatics
- The foundational ADA data standard that underpins the HL7 FHIR dental data exchange implementation guide. Defines the minimum dataset required for dental referral notes and consultation notes exchanged with medical providers.

**ANSI/ADA Proposed Standards 2026**
- Authority: American Dental Association
- URL: https://www.ada.org/resources/practice/dental-standards
- Three new proposed standards under review in 2026 covering electronic records, diagnostics, and treatment planning integration. Software developers should monitor ADA standards publications for adoption timelines.

---

## Similar Products — Developer Documentation & APIs

### Open Dental

- **Description:** Open-source (GPL) dental practice management software with a self-hosted or cloud-hosted deployment model. Widest developer access of any dental PMS; API subscription model allows third-party integrations.
- **API Documentation:** https://www.opendental.com/site/apispecification.html
- **API Documents:** https://www.opendental.com/site/apidocuments.html
- **API Implementation Guide:** https://www.opendental.com/site/apiimplementation.html
- **API Subscriptions:** https://www.opendental.com/site/apisubscriptions.html
- **Developer Setup:** https://www.opendental.com/site/apisetup.html
- **Programming Resources:** https://www.opendental.com/site/programmingresources.html
- **Standards:** REST/JSON over HTTPS; endpoint at api.opendental.com; supports GET, POST, PUT, DELETE
- **Authentication:** Developer key required; obtained via developer portal after Open Dental licence verification

### Henry Schein One — Dentrix / Dentrix Ascend API Exchange

- **Description:** The API Exchange is Henry Schein One's integration marketplace connecting Dentrix and Dentrix Ascend to 55+ authorised partner solutions. Provides read-only and bidirectional access depending on product version and API tier.
- **Developer Portal:** https://ddp.dentrix.com/
- **API Exchange Overview:** https://www.henryscheinone.com/dental-solutions/api-exchange/
- **Vendor Programme:** https://www.henryscheinone.com/dental-solutions/api-exchange/api-exchange-vendors/
- **SDK:** Dentrix SDK v2.2 — downloadable from Resources > Downloads after agreement acceptance
- **Standards:** Proprietary stored procedures and views (Dentrix); REST-based for Dentrix Ascend API Exchange
- **Authentication:** API Agreement required; API key issued post-approval

### Planet DDS — Denticon API

- **Description:** Purpose-built cloud dental platform for DSOs. API program launched 2024 provides comprehensive access to patient, appointment, financial, and clinical data with event-driven architecture.
- **Developer Portal:** https://developer.planetdds.com/
- **API Resources:** https://developer.planetdds.com/apis
- **Programme Announcement:** https://www.planetdds.com/blog/planet-dds-api-program/
- **Standards:** REST/JSON; OpenAPI-described endpoints; event-driven webhooks for real-time synchronisation
- **Authentication:** Self-service API key signup via developer portal

### CareStack

- **Description:** All-in-one cloud dental PMS with AI suite. Open API strategy for custom patient care and practice management integrations.
- **Developer Portal:** https://developer.carestack.com/
- **Integrations Overview:** https://carestack.com/dental-software/integrations
- **Standards:** REST/JSON
- **Authentication:** Developer portal registration required

### tab32

- **Description:** Cloud-native dental platform with FHIR-compliant architecture and open data warehouse (Google BigQuery). First dental platform to standardise EHR data.
- **Platform:** https://tab32.com/
- **FHIR & Interoperability Article:** https://www.tensorlinks.com/blog/fhir-ehr-ai-dental-interoperability-2026/
- **Standards:** FHIR R4; REST/JSON; BigQuery integration for open data warehouse analytics
- **Authentication:** By arrangement; no public self-serve developer portal identified

### Curve Dental

- **Description:** AWS-native cloud PMS targeting independent practices and DSOs. Partner integration marketplace with 20+ integration partners.
- **Partner List:** https://www.curvedental.com/partner-list
- **GitHub Organisation:** https://github.com/curvedental (26 repositories)
- **API Tracker Entry:** https://apitracker.io/a/curvedental
- **Standards:** REST/JSON (per API tracker); no public OpenAPI spec identified
- **Authentication:** Partner agreement required

### Sikka ONE API (Cross-Platform Dental Data Aggregator)

- **Description:** HIPAA-compliant aggregator API that connects to 400+ practice management systems (including Dentrix, Eaglesoft, Open Dental, and others) through a single REST API. Also supports MCP server connection for AI agent integration. Enables developers to build applications for dental, veterinary, chiropractic, optometry, and physician practices without per-PMS integrations.
- **API Documentation:** https://apidocs.sikkasoft.com/
- **Developer Portal:** https://www.sikkasoft.com/oneapi
- **Postman Collection:** https://documenter.getpostman.com/view/3967924/RW1dExDy
- **API Portal:** https://api.sikkasoft.com/
- **Packages & Pricing:** https://www.sikkasoft.com/api-packages
- **Standards:** REST/JSON; 100+ endpoints; HIPAA-compliant; MCP server integration for AI agents
- **Authentication:** API key via developer account signup; HIPAA Business Associate Agreement required

---

## Notes

- **CDT licence fees** are a meaningful cost consideration for any commercial dental software product. The ADA charges annual licensing for CDT codes included in production software. Open-source projects that need CDT codes must either license from the ADA or rely on users providing their own CDT data files.
- **SNODENT licensing** is similarly controlled by the ADA and is required for any software that surfaces SNODENT clinical terms.
- **FHIR dental interoperability is nascent but accelerating**: the dental data exchange IG v2.0.0 is in active ballot as of 2026, and CMS-0057-F timelines create regulatory pressure for adoption through 2027. AI-native platforms that implement FHIR early will gain a durable competitive advantage.
- **Sikka ONE API** is the fastest path to multi-platform dental PMS integration for developers who do not want to build individual integrations with each system. It is particularly relevant for AI agents that need to read appointment, patient, and production data across heterogeneous practice environments.
- **MCP (Model Context Protocol) compatibility**: Sikka.ai has published an MCP server for their ONE API, enabling Claude and other LLM-based agents to directly query dental practice data. This is a relevant integration pattern for any AI-native dental practice management tool.
