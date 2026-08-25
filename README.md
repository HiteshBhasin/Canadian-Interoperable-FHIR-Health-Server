# Canadian Interoperable FHIR Health Server

> **A Canadian, standards-based, consent-aware, provenance-preserving health-information interoperability prototype.**
>
> This project demonstrates how authorized clinical systems can retrieve a structured, clinically useful patient summary across organizational or provincial boundaries without treating a central database as the universal source of truth.

**Status:** Architecture and synthetic-data MVP / pilot preparation  
**FHIR version:** HL7 FHIR R4  
**Primary stack:** Java 17, Spring Boot 3.x, HAPI FHIR R4, PostgreSQL, Keycloak, Redis, Docker  
**Initial interoperability scope:** Ontario ↔ British Columbia simulated gateway exchange  
**Data policy:** Synthetic or formally approved de-identified data only in development and demonstrations

---

## Table of Contents

- [Overview](#overview)
- [The Canadian Interoperability Problem](#the-canadian-interoperability-problem)
- [Project Goals and Non-Goals](#project-goals-and-non-goals)
- [How the Solution Works](#how-the-solution-works)
- [Clinical Use Cases](#clinical-use-cases)
- [Standards Baseline](#standards-baseline)
- [Architecture](#architecture)
- [Core Capabilities](#core-capabilities)
- [FHIR Resource Scope](#fhir-resource-scope)
- [Identity, Consent, and Authorization](#identity-consent-and-authorization)
- [Terminology and Data Quality](#terminology-and-data-quality)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Local Development](#local-development)
- [API and FHIR Operations](#api-and-fhir-operations)
- [Example Payloads](#example-payloads)
- [Gateway Federation](#gateway-federation)
- [Testing Strategy](#testing-strategy)
- [Security, Privacy, and Safety](#security-privacy-and-safety)
- [Observability and Operations](#observability-and-operations)
- [Deliverables](#deliverables)
- [Roadmap](#roadmap)
- [Definition of Done](#definition-of-done)
- [Contributing](#contributing)
- [References](#references)
- [Important Disclaimer](#important-disclaimer)

---

## Overview

Canadian health information is frequently distributed across provincial, territorial, regional, hospital, primary-care, laboratory, pharmacy, and vendor systems. Although many of these systems have electronic records, an authorized clinician cannot always obtain a complete, current, structured, and interpretable patient summary when care crosses organizational or provincial boundaries.

This project is a **federated interoperability prototype**. It uses HL7 FHIR R4 and Canadian interoperability artifacts to demonstrate a safe pattern for exchanging high-value clinical information, such as allergies, medications, conditions, immunizations, laboratory observations, and diagnostic reports.

The system is designed around the following principle:

> The platform creates a **trusted federated longitudinal view**. It does not replace provincial source systems, silently merge patient records, or declare itself the authoritative source for every clinical fact.

Each returned resource should retain information about:

- The source organization and source system
- The source resource identifier
- The clinical effective time and the technical retrieval time
- The FHIR profile and terminology version used
- Whether the data was returned live, cached, imported, partial, stale, invalid, withheld, or unavailable
- The provenance and audit context associated with the exchange

---

## The Canadian Interoperability Problem

The project addresses a combination of technical, semantic, governance, and workflow gaps:

| Layer | Problem | Project response |
|---|---|---|
| Technical interoperability | Data is often exchanged as PDFs, faxes, scanned documents, or proprietary formats | Provide structured, queryable FHIR R4 resources and document Bundles |
| Semantic interoperability | Different systems may encode the same clinical concept differently | Validate and map coded data using Canadian terminology artifacts and versioned value sets |
| Patient identity | Provincial identifiers are jurisdiction-scoped; demographics can be incomplete or ambiguous | Support multiple identifiers, conservative matching, candidate review, and merge/unmerge audit history |
| Consent and authorization | A valid login does not by itself prove a clinician may access a patient record | Evaluate practitioner, organization, purpose of use, patient context, consent, jurisdiction, and emergency policy |
| Provenance and freshness | A record without source or update context may be clinically unsafe | Preserve source, timestamps, transformation information, and data-state indicators |
| Gateway federation | Jurisdictions and organizations may have different APIs, profiles, and operational limits | Use gateway adapters, capability discovery, contract testing, timeouts, retries, and partial-response handling |
| Clinical workflow | A technically valid response can still be unsafe or unusable | Design around emergency, medication-reconciliation, laboratory, and transition-of-care workflows |

---

## Project Goals and Non-Goals

### Goals

1. Provide an FHIR R4 server and interoperability broker for **synthetic-data** demonstrations.
2. Generate and retrieve a structured, validated patient summary using a pinned Canadian standards baseline.
3. Demonstrate read-only Ontario ↔ British Columbia exchange using a simulated remote gateway.
4. Enforce SMART on FHIR/OIDC authentication and policy-based authorization.
5. Support multiple patient identifiers and safe candidate matching.
6. Preserve provenance, audit evidence, clinical source, and data freshness.
7. Validate FHIR profiles and coded terminology during ingestion and exchange.
8. Return clear `OperationOutcome` responses for validation, authorization, identity, and remote-gateway failures.
9. Demonstrate partial-response safety: a failed remote gateway must never appear as “no known information.”
10. Supply a repeatable engineering, testing, and delivery foundation for an approved future pilot.

### Non-Goals for the MVP

- Replace an EMR, provincial repository, laboratory information system, pharmacy system, or imaging archive.
- Establish real production connectivity to all provinces or territories.
- Treat a provincial health number as a pan-Canadian universal patient identifier.
- Centrally replicate every patient record from all source systems.
- Automatically merge patients using name/date-of-birth or low-confidence probabilistic matching.
- Provide billing, claims, insurance, scheduling, or full patient-portal features.
- Make autonomous diagnostic, prescribing, triage, or clinical decisions.
- Process real personal health information without formal legal, privacy, security, clinical-safety, and operational approval.

---

## How the Solution Works

The prototype follows this exchange sequence:

1. An authorized clinician or client application authenticates through SMART on FHIR / OIDC.
2. The server validates the access token, client identity, scopes, issuer, audience, expiry, and required claims.
3. A policy service evaluates role, organization, purpose of use, patient context, consent, requested resource types, jurisdiction, and emergency state.
4. The patient identity service uses known identifiers and approved demographic rules to find a deterministic match or a candidate match.
5. The federation broker selects the relevant provincial gateway adapter.
6. The gateway adapter retrieves a permitted patient summary or selected resources.
7. Incoming resources are validated against the selected FHIR profile, terminology, and data-quality rules.
8. The platform preserves source provenance and freshness metadata rather than overwriting source truth.
9. The platform returns a structured summary and any partial-response, validation, or policy warnings.
10. The platform writes complete audit evidence with correlation identifiers.

---

## Clinical Use Cases

### 1. Emergency patient-summary retrieval

A patient previously treated in British Columbia presents to an Ontario emergency department. The clinician needs allergy, medication, condition, immunization, laboratory, and diagnostic-report information before making urgent care decisions.

**Expected capabilities:**

- Authorized read-only summary retrieval
- Patient identity resolution
- PS-CA-aligned summary Bundle
- Source organization and freshness for high-risk facts
- Explicit warning if a source fails or data is partial
- Full audit trail

### 2. Medication reconciliation at care transition

A patient is discharged from hospital and sees a community provider. Medication orders, patient-reported medication use, pharmacy dispensing, and prior allergy information may not agree.

**Expected capabilities:**

- Separate representation of `MedicationRequest`, `MedicationStatement`, `MedicationDispense`, and `MedicationAdministration`
- Clinical provenance and dates
- Terminology validation
- Reconciliation workflow that exposes conflicts instead of silently replacing records

### 3. Laboratory and diagnostic-report sharing

A specialist needs results that were produced in another jurisdiction or organization.

**Expected capabilities:**

- `Observation` for individual laboratory results
- `DiagnosticReport` for reports and result references
- Coded terminology and source laboratory information
- Clear reporting, effective, and last-updated dates
- Optional controlled `DocumentReference` support for legacy reports

### 4. Planned cross-provincial transition of care

A patient relocates from Ontario to British Columbia. The new care team retrieves a structured, source-preserving patient summary instead of relying only on scanned documents or patient recall.

### 5. Emergency break-glass and privacy review

A patient is unable to communicate during an emergency. A clinician invokes an approved emergency pathway with mandatory justification, enhanced auditing, expiry, and retrospective review.

---

## Standards Baseline

The project uses four complementary layers of standards.

| Standard | Role in this project | Link |
|---|---|---|
| HL7 FHIR R4 | Base resource model, REST API, search, Bundles, operations, history, and interoperability framework | [FHIR R4 specification](https://hl7.org/fhir/R4/index.html) |
| CA Core+ | Pan-Canadian foundational FHIR profiles, extensions, and terminology artifacts | [CA Core+ Implementation Guide](https://simplifier.net/guide/ca-core) |
| PS-CA | Pan-Canadian Patient Summary FHIR implementation guidance | [PS-CA FHIR Implementation Guide](https://infoscribe.infoway-inforoute.ca/display/PSCAV1/FHIR-Implementation-Guide) |
| CACDI | Canadian Core Data for Interoperability: core data elements and value sets supporting connected care | [CIHI CACDI overview](https://www.cihi.ca/en/what-is-the-canadian-core-data-for-interoperability-or-cacdi) |
| CACDI Version 2 | Current published core data-content material and logical modelling reference | [CACDI Version 2 PDF](https://www.cihi.ca/sites/default/files/document/canadian-core-data-for-interoperability-version-2-en.pdf) |
| Canadian FHIR Registry | Canadian FHIR profiles, extensions, value sets, and related artifacts | [Canadian FHIR Registry](https://simplifier.net/organization/canadianfhirregistry) |
| Infoway FHIR community | Canadian implementation and collaboration community | [InfoCentral FHIR Implementations](https://infocentral.infoway-inforoute.ca/en/collaboration/wg/fhir-implementations) |

### Versioning rule

Do not implement against “latest” without recording it. Maintain `docs/standards/standards-baseline.md` with:

```text
FHIR: HL7 FHIR R4 (4.0.1)
PS-CA: <selected implementation-guide release>
CA Core+: <selected implementation-guide release>
CACDI: Version 2
SNOMED CT CA: <selected release>
pCLOCD / LOINC: <selected release>
Canadian Clinical Drug Data Set: <selected release>
FHIR validator packages: <exact package names and versions>
```

Canadian implementation guides and terminology releases evolve. A profile, value set, or terminology package should be upgraded through a documented change-control and regression-testing process.

---

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│ Clinician EMR / Portal / Authorized SMART Application        │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS / OAuth2 / OIDC
┌──────────────────────────────▼──────────────────────────────┐
│ API Gateway                                                   │
│ Rate limits · routing · request size limits · correlation ID │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ SMART Authentication and Token Validation                     │
│ Keycloak/OIDC · issuer/audience/signature/expiry validation   │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ Consent and Policy Decision Service                           │
│ Role · organization · purpose · patient context · jurisdiction│
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ Patient Identity Service                                      │
│ Multiple identifiers · deterministic/candidate matching       │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ Federation Broker                                             │
│ Gateway registry · routing · timeout · retry · partial status │
└───────────────┬───────────────────────────┬─────────────────┘
                │                           │
┌───────────────▼──────────────┐ ┌──────────▼─────────────────┐
│ Ontario Gateway Adapter       │ │ BC Gateway Adapter          │
│ Local / simulated source      │ │ Remote / WireMock simulator │
└───────────────┬──────────────┘ └──────────┬─────────────────┘
                └───────────────┬───────────┘
                                │
┌───────────────────────────────▼─────────────────────────────┐
│ FHIR and Terminology Services                                 │
│ HAPI FHIR R4 · profile validation · terminology validation    │
└───────────────────────────────┬─────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────┐
│ Data, Provenance, Audit, and Observability                    │
│ PostgreSQL JSONB · indexed projections · AuditEvent · traces  │
│ bounded Redis cache · metrics · alerts                        │
└─────────────────────────────────────────────────────────────┘
```

### Architectural principles

- **Federated, not universal-centralized:** participating sources retain authority for their own records.
- **Source-preserving:** return source, source system, source resource identifier, and timestamps.
- **Policy-aware:** authorization is based on more than a valid token.
- **Versioned:** profile, terminology, resource, and source-data versions are tracked.
- **Failure-transparent:** partial and stale results are visible to users and systems.
- **Clinical-safety oriented:** high-risk data such as allergies and medications must never silently disappear due to integration failure.
- **Synthetic-first:** prototype development is based on synthetic fixtures and simulators.

---

## Core Capabilities

### FHIR server capabilities

- FHIR R4 REST API under `/fhir`.
- Public capability discovery at `/fhir/metadata`.
- SMART discovery at `/.well-known/smart-configuration` or the deployment-specific configured location.
- Read/search for approved resources.
- Resource history and versioning where implemented.
- Conditional operations and ETag/optimistic concurrency where implemented.
- Structured `OperationOutcome` error responses.
- Patient-summary operation, such as `GET /fhir/Patient/{id}/$summary`.
- Validation against pinned FHIR profiles and terminology packages.

### Federation capabilities

- Provincial gateway capability discovery.
- Gateway-specific adapters.
- Patient discovery and candidate identity matching.
- Authorized remote patient-summary retrieval.
- Timeout, bounded retry, circuit-breaker, and source-health handling.
- Correlation IDs across local and remote calls.
- Partial-response representation.
- Remote source provenance and freshness capture.

### Clinical safety capabilities

- Explicit data-state labels: `live`, `cached`, `partial`, `stale`, `unavailable`, `invalid`, `withheld`, and `unverified`.
- Allergy and medication source/freshness visibility.
- No low-confidence automatic patient merges.
- Conflict-aware medication and allergy representation.
- Emergency-access justification and enhanced audit pathway.
- Synthetic safety scenarios for identity, consent, stale data, gateway outage, and invalid FHIR responses.

---

## FHIR Resource Scope

### MVP resources

```text
Patient
Organization
Practitioner
PractitionerRole

Condition
AllergyIntolerance
MedicationRequest
MedicationStatement
MedicationDispense
MedicationAdministration
Immunization
Observation
DiagnosticReport

Composition
Bundle
DocumentReference

Consent
Provenance
AuditEvent
Endpoint
OperationOutcome
CapabilityStatement
```

### Why medication resources are separated

Medication information has different clinical meanings:

| Resource | Meaning |
|---|---|
| `MedicationRequest` | A prescription or order |
| `MedicationStatement` | A statement that the patient is taking or has taken a medication |
| `MedicationDispense` | A dispensing event |
| `MedicationAdministration` | A medication administration event, often in a facility |

Do not reduce all medication data to a single table containing only `medication_name`, `code`, and `status`. That loses important clinical context and can create medication-reconciliation risk.

---

## Identity, Consent, and Authorization

### Patient identity model

Provincial health numbers are not a universal Canada-wide patient key. The identity service must support multiple identifiers and source context.

```text
Person
- internal_person_id
- legal and preferred names
- date of birth
- approved demographic attributes
- verification state

PersonIdentifier
- person_id
- identifier system URI
- identifier value
- identifier type
- assigning jurisdiction
- validity period
- source system
- verification date

IdentityMatchEvent
- candidate records
- deterministic or probabilistic method
- match score
- evidence/rationale
- decision
- reviewer
- timestamp
- merge/unmerge history
```

### Matching policy

| Level | Example | Required action |
|---|---|---|
| Deterministic | Trusted identifier confirmed by source | May link or retrieve according to policy |
| Strong candidate | Several verified demographics and identifiers agree | Return candidate; apply governed policy |
| Probabilistic candidate | Similar name, date of birth, and historical address | Human review required; do not automatically merge |
| Low confidence | Name only, name + DOB only, or conflicting attributes | Do not link or retrieve as a confirmed patient |

### Authorization model

SMART scopes are necessary but insufficient. Every sensitive request should be evaluated using:

```text
Who is requesting access?
Which application is requesting access?
Which organization does the requester represent?
What is the requester’s role?
Which patient is being requested?
What resource types are requested?
What is the declared purpose of use?
Does a permitted care relationship or exception apply?
Does patient consent permit the disclosure?
Which source jurisdiction’s policy applies?
Is emergency break-glass being used?
What audit and expiry obligations apply?
```

Example internal access context:

```json
{
  "practitionerId": "Practitioner/ed-physician-001",
  "organizationId": "Organization/ontario-emergency-department",
  "roles": ["emergency-physician"],
  "purposeOfUse": "TREATMENT",
  "requestingJurisdiction": "ON",
  "sourceJurisdiction": "BC",
  "requestedResources": [
    "AllergyIntolerance",
    "MedicationStatement",
    "Condition"
  ],
  "emergencyOverride": false,
  "correlationId": "9f8c1e2d"
}
```

### Suggested least-privilege scopes

```text
patient/Patient.rs
patient/AllergyIntolerance.rs
patient/Condition.rs
patient/MedicationStatement.rs
patient/MedicationRequest.rs
patient/Observation.rs
patient/DiagnosticReport.rs
patient/Immunization.rs
patient/Patient.$summary
```

The server must still apply its own policy checks even when a token includes a valid SMART scope.

---

## Terminology and Data Quality

### Terminology principles

- Use the terminology artifacts required by the selected Canadian profiles and use cases.
- Preserve source code, source system, source display, and source version.
- Store normalized Canadian code, system, display, version, mapping status, method, and confidence separately.
- Validate codes against the appropriate value set when required.
- Flag unknown, unmapped, deprecated, ambiguous, and invalid codes.
- Do not silently change the clinical meaning of source data.
- Record terminology-version changes and run regression tests after updates.

### Example coded-data representation

```text
source_code: "LOCAL-HTN-01"
source_system: "https://source.example.ca/local-diagnoses"
source_display: "High blood pressure"
source_version: "2026.1"

normalized_code: "38341003"
normalized_system: "http://snomed.info/sct"
normalized_display: "Hypertensive disorder"
normalized_version: "SNOMED CT CA <release>"

mapping_status: "verified"
mapping_method: "approved-concept-map"
mapping_confidence: 1.0
```

### Data-quality rules

The platform should validate or expose:

- Required profile fields.
- FHIR cardinality and reference integrity.
- Code-system and value-set conformance.
- Effective time, recorded time, and synchronization time.
- Resource status and verification state.
- Duplicate or conflicting resources.
- Missing versus unknown versus negative clinical statements.
- Source and provenance presence.
- Stale cache and partial-response indicators.

---

## Repository Structure

```text
canadian-fhir-interoperability/
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── docker-compose.yml
├── pom.xml
├── docs/
│   ├── architecture/
│   │   ├── target-architecture.md
│   │   ├── data-flow.md
│   │   └── architecture-decision-records/
│   ├── clinical/
│   │   ├── emergency-summary-use-case.md
│   │   ├── medication-reconciliation-use-case.md
│   │   └── clinical-safety-case.md
│   ├── governance/
│   │   ├── privacy-impact-template.md
│   │   ├── threat-risk-assessment.md
│   │   ├── consent-policy.md
│   │   └── data-sharing-model.md
│   ├── standards/
│   │   ├── standards-baseline.md
│   │   ├── ps-ca-summary-spec.md
│   │   ├── ca-core-mapping.md
│   │   ├── cacdi-fhir-mapping.csv
│   │   └── terminology-strategy.md
│   └── runbooks/
│       ├── gateway-outage.md
│       ├── privacy-incident.md
│       └── emergency-access-review.md
├── implementation-guide/
│   ├── input/
│   │   ├── profiles/
│   │   ├── valuesets/
│   │   └── examples/
│   └── README.md
├── test-data/
│   ├── valid/
│   ├── invalid/
│   ├── identity/
│   └── gateway-fixtures/
├── api/
│   ├── postman/
│   ├── openapi/
│   └── capability-statement.json
├── infra/
│   ├── docker/
│   ├── terraform/
│   ├── kubernetes/
│   └── monitoring/
├── scripts/
│   ├── validate-fhir.sh
│   ├── seed-synthetic-data.sh
│   └── run-contract-tests.sh
├── src/
│   ├── main/
│   │   ├── java/ca/health/interoperability/
│   │   │   ├── InteroperabilityApplication.java
│   │   │   ├── config/
│   │   │   ├── fhir/
│   │   │   │   ├── provider/
│   │   │   │   ├── operation/
│   │   │   │   ├── validation/
│   │   │   │   └── mapper/
│   │   │   ├── identity/
│   │   │   ├── policy/
│   │   │   ├── gateway/
│   │   │   ├── terminology/
│   │   │   ├── provenance/
│   │   │   ├── audit/
│   │   │   ├── persistence/
│   │   │   └── common/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── db/migration/
│   │       └── fhir-packages/
│   └── test/
│       ├── java/
│       │   ├── unit/
│       │   ├── integration/
│       │   ├── contract/
│       │   ├── security/
│       │   └── conformance/
│       └── resources/
└── .github/workflows/
    ├── ci.yml
    └── security.yml
```

---

## Getting Started

### Prerequisites

For local synthetic-data development:

- Java 17 or newer
- Maven 3.9+
- Docker and Docker Compose
- Git
- At least 8 GB RAM recommended for local PostgreSQL, Keycloak, WireMock, and the application
- Optional: Postman/Bruno, jq, curl, a FHIR validation tool, and a code editor with Java support

### Clone and configure

```bash
git clone <your-repository-url>
cd canadian-fhir-interoperability
cp .env.example .env
```

Update local-only environment values in `.env`. Never put production credentials, real patient data, private signing keys, or unapproved endpoints in this file.

### Start supporting services

```bash
docker compose up -d postgres keycloak wiremock
```

### Start the application

```bash
./mvnw spring-boot:run
```

Or, if Maven Wrapper is not included:

```bash
mvn spring-boot:run
```

### Verify health and capability endpoints

```bash
curl http://localhost:8080/actuator/health
curl -H "Accept: application/fhir+json" http://localhost:8080/fhir/metadata
```

### Run tests

```bash
./mvnw test
./mvnw verify
```

### Seed synthetic data

```bash
./scripts/seed-synthetic-data.sh
```

### Run gateway contract tests

```bash
./scripts/run-contract-tests.sh
```

---

## Local Development

### Local-only Docker Compose services

| Service | Local purpose |
|---|---|
| PostgreSQL | FHIR resource persistence, indexed projections, identity records, audit metadata |
| Keycloak | Local OIDC/OAuth2/SMART-style authentication and realm/client configuration |
| WireMock | Simulated British Columbia provincial gateway and failure scenarios |
| Redis | Bounded local cache for approved metadata and test cases |
| FHIR server | Spring Boot + HAPI FHIR R4 application |

### Example `.env.example`

```dotenv
DB_URL=jdbc:postgresql://postgres:5432/fhir_db
DB_USER=fhir_user
DB_PASSWORD=replace-with-local-secret
OIDC_ISSUER_URI=http://keycloak:8080/realms/health
FHIR_BASE_URL=http://localhost:8080/fhir
BC_GATEWAY_URL=http://wiremock:8080/bc/fhir
```

### Development data rules

- Use synthetic data by default.
- Label all fixtures clearly as synthetic.
- Do not place PHI in Git repositories, CI logs, screenshots, Slack/Teams messages, issue trackers, or test assertions.
- Do not use production cloud subscriptions or production key vaults for unapproved experimentation.
- Do not rely on Keycloak `start-dev`, publicly exposed databases, or local passwords in a production environment.

---

## API and FHIR Operations

### Core endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/fhir/metadata` | Public FHIR capability discovery |
| `GET` | `/.well-known/smart-configuration` | SMART discovery, depending on deployment configuration |
| `GET` | `/fhir/Patient/{id}` | Read an authorized patient resource |
| `GET` | `/fhir/Patient?identifier={system}\|{value}` | Search using an approved patient identifier |
| `GET` | `/fhir/Patient?family={family}&birthdate={date}` | Restricted candidate search; never treat as final identity proof alone |
| `GET` | `/fhir/AllergyIntolerance?patient={id}` | Retrieve allergies, subject to policy |
| `GET` | `/fhir/Condition?patient={id}` | Retrieve conditions, subject to policy |
| `GET` | `/fhir/MedicationStatement?patient={id}` | Retrieve medication statements, subject to policy |
| `GET` | `/fhir/Observation?patient={id}&code={system}\|{code}` | Retrieve observations/labs, subject to policy |
| `GET` | `/fhir/DiagnosticReport?patient={id}` | Retrieve diagnostic reports, subject to policy |
| `GET` | `/fhir/Patient/{id}/$summary` | Retrieve a validated patient summary Bundle |

### Request headers

```http
Authorization: Bearer <access-token>
Accept: application/fhir+json
X-Correlation-ID: <uuid>
X-Purpose-Of-Use: TREATMENT
```

The actual allowed headers and policy context must be documented in the project’s gateway contract and security design. Do not place sensitive consent decisions or PHI in unprotected client-controlled headers.

### Error response pattern

Use FHIR `OperationOutcome` for structured errors and warnings.

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "warning",
      "code": "transient",
      "details": {
        "text": "Remote source did not respond within the configured timeout."
      },
      "diagnostics": "Correlation ID: 9f8c1e2d"
    }
  ]
}
```

A partial result must additionally state which sources responded, which did not, whether permitted cached information was used, and when any cached source was last refreshed.

---

## Example Payloads

### Example `Patient`

```json
{
  "resourceType": "Patient",
  "id": "synthetic-ontario-001",
  "meta": {
    "profile": [
      "<pinned CA Core+ Patient profile URL>"
    ],
    "lastUpdated": "2026-08-25T12:00:00Z"
  },
  "identifier": [
    {
      "system": "https://example.ca/identifier/ontario-phn",
      "value": "SYN-ON-000123",
      "use": "official"
    }
  ],
  "name": [
    {
      "use": "official",
      "family": "Demo",
      "given": ["Avery"]
    }
  ],
  "birthDate": "1985-03-14",
  "gender": "other"
}
```

### Example `AllergyIntolerance`

```json
{
  "resourceType": "AllergyIntolerance",
  "id": "allergy-synthetic-001",
  "clinicalStatus": {
    "coding": [{ "code": "active" }]
  },
  "verificationStatus": {
    "coding": [{ "code": "confirmed" }]
  },
  "type": "allergy",
  "criticality": "high",
  "code": {
    "coding": [
      {
        "system": "http://snomed.info/sct",
        "code": "91936005",
        "display": "Allergy to penicillin"
      }
    ]
  },
  "patient": {
    "reference": "Patient/synthetic-ontario-001"
  },
  "recordedDate": "2026-02-02",
  "note": [{ "text": "Synthetic example only." }]
}
```

### Example `Provenance`

```json
{
  "resourceType": "Provenance",
  "target": [
    {
      "reference": "AllergyIntolerance/allergy-synthetic-001"
    }
  ],
  "recorded": "2026-08-25T12:05:00Z",
  "agent": [
    {
      "type": { "text": "source organization" },
      "who": {
        "reference": "Organization/synthetic-bc-gateway"
      }
    }
  ],
  "entity": [
    {
      "role": "source",
      "what": {
        "reference": "https://synthetic-bc.example.ca/fhir/AllergyIntolerance/remote-123"
      }
    }
  ]
}
```

### Example summary Bundle structure

```text
Bundle (type: document)
├── Composition
├── Patient
├── AllergyIntolerance (0..*)
├── MedicationStatement / MedicationRequest (0..*)
├── Condition (0..*)
├── Immunization (0..*)
├── Observation / DiagnosticReport (as in scope)
├── Practitioner / Organization (as required for attribution)
└── Provenance (as required for source lineage)
```

Use the exact Bundle, Composition, section, profile, cardinality, terminology, and extension requirements from the pinned PS-CA release selected for the project.

---

## Gateway Federation

### Gateway adapter interface

```java
public interface ProvincialGatewayAdapter {
    GatewayCapabilities discoverCapabilities();

    PatientMatchResult findPatient(
        PatientIdentityQuery query,
        AccessContext context
    );

    RemoteSummaryResult fetchPatientSummary(
        RemotePatientMatch match,
        AccessContext context
    );

    GatewayHealth getHealth();
}
```

### Required gateway behavior

Each adapter should define:

- Base endpoint and environment.
- FHIR version and supported profiles.
- Supported resources, searches, and operations.
- Client and gateway authentication mechanism.
- Trust and certificate requirements.
- Required request context and scopes.
- Pagination, payload, and rate-limit limits.
- Timeout, retry, circuit-breaker, and idempotency rules.
- Error semantics and expected `OperationOutcome` behavior.
- Source freshness and provenance behaviour.
- Maintenance windows, support contacts, and incident escalation path.

### Gateway outage behavior

| Condition | Required behavior |
|---|---|
| Remote gateway timeout | Return explicit partial response; identify unavailable source; do not claim there is no data |
| Invalid remote FHIR | Quarantine/reject affected content; return actionable validation outcome |
| Remote authorization denial | Return policy-aware denial without leaking protected details |
| Identity ambiguous | Do not retrieve a confirmed patient summary; require review or controlled candidate workflow |
| Terminology validation warning | Preserve source data, mark warning, and follow approved clinical display policy |

---

## Testing Strategy

### Required test layers

| Test layer | Scope |
|---|---|
| Unit tests | Services, mappers, policy evaluation helpers, validation utilities |
| Integration tests | Spring Boot endpoints with PostgreSQL/Testcontainers and synthetic resources |
| FHIR conformance tests | Selected PS-CA and CA Core+ profiles, structure, cardinality, value-set bindings |
| Terminology tests | Valid, invalid, unknown, deprecated, mapped, and unmapped codes |
| Identity tests | Deterministic, candidate, false positive, false negative, review, merge/unmerge scenarios |
| Security tests | Token validation, role/purpose checks, consent denial, cross-jurisdiction access, break-glass |
| Gateway contract tests | Simulated Ontario/BC requests, timeouts, retries, partial response, malformed Bundles |
| Resilience tests | Circuit breakers, cache safety, outage recovery, duplicate request suppression, chaos cases |
| Performance tests | Separate local read, search, write, remote summary, cache hit/miss, and error-rate targets |
| Clinical usability tests | Source/freshness comprehension, medication/allergy workflow, partial-data interpretation |
| Disaster recovery tests | Backup restoration, recovery objectives, runbook execution |

### Synthetic fixtures

The test-data package should include at minimum:

```text
SYN-001 Exact trusted identifier match and complete summary
SYN-002 Multiple provincial identifiers after relocation
SYN-003 Same name/date of birth as another patient
SYN-004 Confirmed severe allergy from a remote source
SYN-005 Medication order conflicts with patient-reported use
SYN-006 Remote gateway unavailable / timeout
SYN-007 Invalid profile or invalid terminology code
SYN-008 Consent restriction / redaction / denial
SYN-009 Emergency break-glass access
SYN-010 Stale cached source information
```

---

## Security, Privacy, and Safety

### Baseline security controls

- TLS 1.3 for external connections.
- Mutual TLS for approved gateway-to-gateway exchange where required.
- OAuth2/OIDC token validation with issuer, audience, signature, expiry, and key-rotation checks.
- Least-privilege SMART scopes and server-side policy evaluation.
- MFA and privileged-access controls for administration.
- Managed secrets and encryption-key lifecycle management.
- Private networking and database access restrictions for approved environments.
- Dependency, secret, container, and infrastructure scanning.
- Signed build artifacts and software bill of materials.
- Tamper-evident or otherwise protected audit evidence.
- PHI redaction in logs, traces, exceptions, CI output, and support tooling.
- Tested backup, restoration, incident response, and disaster-recovery procedures.

### Privacy and governance work required before real PHI

Before any real patient data or real provincial gateway integration is used, the project needs appropriate governance work, which may include:

- Privacy impact assessment.
- Threat and risk assessment.
- Data-flow inventory.
- Custodian/service-provider role determination.
- Data-sharing and service agreements.
- Consent and emergency-access policy approval.
- Data residency, backup, logging, and support-access review.
- Retention, correction, deletion, and legal-hold design.
- Clinical safety assessment.
- Appropriate Indigenous data-governance engagement if applicable.
- Accessibility and French-language requirements review.

### Clinical safety rules

- Do not state “no known allergies” when the relevant source is unavailable.
- Do not treat an incomplete summary as a complete record.
- Do not silently merge patient identities from weak demographic matches.
- Do not overwrite source data during terminology normalization.
- Do not use AI to make autonomous patient-care decisions.
- Do not use stale medication or allergy data without clearly displaying freshness and source state.
- Do not use a cache to extend expired or revoked authorization.

---

## Observability and Operations

### Key metrics

```text
fhir_request_latency_ms{resource, operation, status}
gateway_request_latency_ms{jurisdiction, operation, outcome}
gateway_availability{jurisdiction}
fhir_profile_validation_failures_total{profile}
terminology_validation_failures_total{system}
identity_match_candidates_total{match_method, decision}
authorization_decisions_total{decision, purpose, jurisdiction}
partial_summary_responses_total{source}
break_glass_access_total{organization}
audit_event_write_failures_total
cache_data_state_total{state}
```

### Required operational runbooks

- Provincial gateway outage.
- Authentication/OIDC outage.
- Consent/policy-engine failure.
- Identity-match issue and incorrect linkage report.
- Suspected privacy incident.
- Suspected PHI disclosure in logs or support artifacts.
- Validation/terminology release failure.
- Stale cache concern.
- Database recovery and backup restore.
- Emergency break-glass retrospective review.

### Service-level objectives

Define separate targets for separate operations. Do not apply one generic latency target to everything.

| Service behaviour | Example target to define |
|---|---|
| Local single-resource read | p95 latency under agreed threshold under defined load |
| Local search | Separate p95/p99 threshold and maximum search complexity |
| Remote patient summary | Measured against remote gateway SLA and timeout policy |
| Authorization decision | Low-latency internal SLO with fail-closed rules for sensitive flows |
| Audit write | Durable before or within approved transaction boundary |
| Recovery | Explicit RPO/RTO by data class and service |

---

## Deliverables

### Foundation and standards

- Project charter and MVP scope.
- Clinical use-case specifications.
- Stakeholder and jurisdiction map.
- Standards baseline.
- FHIR implementation guide or profile mapping document.
- PS-CA summary specification.
- CA Core+ mapping.
- CACDI-to-FHIR mapping matrix.
- Terminology strategy.
- Requirements traceability matrix.

### Governance and safety

- Privacy impact assessment.
- Threat and risk assessment.
- Data-flow diagram.
- Data-sharing/governance model.
- Consent and authorization policy.
- Identity and duplicate-resolution specification.
- Clinical safety case.
- Accessibility and language plan.
- Indigenous data-governance engagement plan where applicable.

### Engineering

- Architecture diagram and architecture decision records.
- Database/storage design.
- Gateway adapter contract.
- FHIR server implementation.
- CapabilityStatement.
- API contract and error model.
- Patient-summary operation.
- Provenance/audit implementation.
- Synthetic data and gateway fixtures.
- Docker/infrastructure-as-code configuration.
- CI/CD pipeline.

### Verification and pilot

- Unit, integration, conformance, terminology, identity, policy, security, contract, resilience, and performance test evidence.
- Clinician usability report.
- Monitoring and incident runbooks.
- Backup/disaster-recovery test report.
- Pilot plan and go-live checklist.
- Benefits dashboard and post-pilot evaluation.

---

## Roadmap

### Phase 0 — Discovery and baseline

- Select one clinical use case.
- Identify the pilot boundary and what is simulated.
- Pin standards and terminology versions.
- Create synthetic test scenarios.
- Draft governance, privacy, identity, and policy requirements.

### Phase 1 — Local FHIR foundation

- Build the Spring Boot/HAPI FHIR service.
- Add canonical resource storage and search projections.
- Implement synthetic Patient, AllergyIntolerance, Condition, medication, Observation, and DiagnosticReport resources.
- Add profile validation, `Provenance`, and `AuditEvent`.
- Build a local patient-summary operation.

### Phase 2 — Safety and semantics

- Implement terminology validation and mapping.
- Implement source/freshness data model.
- Implement identity candidates and review flow.
- Implement consent/policy decisions.
- Add partial response and stale-data handling.

### Phase 3 — Federation simulation

- Implement Ontario and BC gateway adapters.
- Use WireMock or equivalent remote gateway simulator.
- Add capability discovery, mTLS-ready trust design, retries, circuit breakers, and contract tests.
- Demonstrate end-to-end audited summary retrieval.

### Phase 4 — Pilot readiness

- Complete security, privacy, clinical-safety, and operational assessments.
- Add monitoring, backup/recovery, release controls, and runbooks.
- Conduct clinician usability testing.
- Run conformance and interoperability testing.

### Phase 5 — Controlled pilot and expansion

- Run a limited approved pilot.
- Measure availability, completeness, freshness, usability, safety, and adoption.
- Remediate issues.
- Add additional domains or jurisdictions only after governance and conformance work is complete.

---

## Definition of Done

The synthetic-data MVP is complete when all of the following are true:

- [ ] Standards baseline is versioned and reproducible.
- [ ] Supported resources and summary operation are documented in the CapabilityStatement.
- [ ] Synthetic PS-CA-aligned patient summary validates against the pinned profile baseline.
- [ ] Terminology validation is automated for in-scope coded resources.
- [ ] Multiple identifiers are supported.
- [ ] Ambiguous patient matches do not automatically merge.
- [ ] SMART/OIDC authentication and policy evaluation protect patient data.
- [ ] Audit records include actor, organization, patient, purpose, decision, resource type, timestamp, source jurisdiction, and correlation ID.
- [ ] Returned clinical resources include source and freshness information.
- [ ] Gateway timeout produces an explicit partial response rather than a false absence of data.
- [ ] Synthetic Ontario ↔ BC gateway contract tests pass.
- [ ] Unit, integration, FHIR validation, terminology, identity, policy, security, and resilience tests run in CI.
- [ ] No PHI exists in repository content, test fixtures, logs, or CI artifacts.
- [ ] A clinician-oriented demonstration shows source, freshness, warning, and partial-data behaviour clearly.

---

## Contributing

### Before submitting a change

1. Read the current standards baseline and architecture decision records.
2. Do not add real patient information, real identifiers, secrets, or protected endpoints.
3. Add or update synthetic fixtures for a clinical behaviour change.
4. Update profile mappings or terminology documentation when data semantics change.
5. Add tests for positive, negative, and failure paths.
6. Consider privacy, identity, consent, provenance, and clinical-safety implications.
7. Update the relevant runbook if operational behaviour changes.

### Pull request expectations

Every pull request should include:

- Clear problem statement and scope.
- Test evidence.
- FHIR/profile/terminology impact statement.
- Security and privacy impact statement.
- Migration plan if persistence changes.
- Backward-compatibility impact.
- Rollback approach for operational changes.

---

## References

### Core standards

- [HL7 FHIR R4 Specification](https://hl7.org/fhir/R4/index.html)
- [CA Core+ Implementation Guide](https://simplifier.net/guide/ca-core)
- [PS-CA FHIR Implementation Guide](https://infoscribe.infoway-inforoute.ca/display/PSCAV1/FHIR-Implementation-Guide)
- [PS-CA publication area](https://simplifier.net/guide/ps-ca)
- [Canadian Core Data for Interoperability (CACDI)](https://www.cihi.ca/en/what-is-the-canadian-core-data-for-interoperability-or-cacdi)
- [CACDI Version 2](https://www.cihi.ca/sites/default/files/document/canadian-core-data-for-interoperability-version-2-en.pdf)
- [Canadian FHIR Registry](https://simplifier.net/organization/canadianfhirregistry)
- [Canada Health Infoway FHIR Implementations community](https://infocentral.infoway-inforoute.ca/en/collaboration/wg/fhir-implementations)

### Project documents

- `docs/standards/standards-baseline.md`
- `docs/clinical/emergency-summary-use-case.md`
- `docs/governance/consent-policy.md`
- `docs/architecture/target-architecture.md`
- `docs/runbooks/gateway-outage.md`
- `test-data/README.md`

---

## Important Disclaimer

This repository is an interoperability engineering prototype and educational/project-planning foundation. It is **not** a certified clinical system, a legal interpretation of Canadian privacy law, an approval to use personal health information, or a substitute for provincial, territorial, organizational, clinical, privacy, security, and Indigenous data-governance requirements.

Before processing real patient information, connecting to real provincial gateways, or using the platform in a clinical workflow, obtain appropriate approvals and complete required privacy, security, legal, clinical-safety, operational, and interoperability conformance activities.
