# FHIR Resource Types That Matter for ICU Data

*The few dozen record kinds we actually need — and the three-way gap between what the US Core standard requires, what Epic's per-patient API exposes, and what Epic's bulk download actually hands you.*

## In plain words

A FHIR "resource type" is just a **kind of record**. A `Patient` is one kind, a `Condition` (a diagnosis) is another, an `Observation` (a lab value, a vital sign) is another, a `MedicationAdministration` (a drug actually given at the bedside) is another. The FHIR standard defines roughly 150 of these record kinds, but for intensive-care research only a few dozen carry any signal. Most of the 150 are billing, scheduling, or public-health plumbing we can ignore.

Three separate authorities decide what a given hospital will actually let us pull, and **they do not agree**:

1. **US Core** — the US national profile of FHIR. It says "a certified system SHOULD be able to serve these record kinds." Think of it as the floor.
2. **The (g)(10) certification rule** — the federal regulation ("Standardized API for Patient and Population Services") that forces vendors to expose a subset of US Core. Think of it as the *enforced* floor.
3. **Epic's actual product** — which exposes far *more* record kinds than US Core requires (Epic advertises around 55 resource types across roughly 450 API endpoints), including the ones that matter most in an ICU: the medication administration record, the flowsheet-style observations for lines/drains/airways, ventilator-adjacent assessments, and code status.

Here is the frustrating discovery that shapes this entire project: **the richest ICU records exist in Epic's API, but they do not come out of the bulk download.** Epic's bulk `$export` — the efficient "give me a whole population at once" mechanism — emits only about 20 record kinds, essentially the plain US Core set. The bedside medication administration record is *not* in the bulk export. The specialized flowsheet observations are *not* in the bulk export. To get those, you must fetch them **one patient at a time** through the regular REST API.

Fetching one patient at a time across a whole hospital would normally be unaffordable. It becomes affordable only because we first use the cheap bulk export to identify *which* patients were inpatient or observation-status, then spend the expensive per-patient calls only on that filtered cohort. Bulk to find the cohort; REST to enrich it. That two-phase shape is the heart of the middleware. See [doc 05](05-bulk-export-deep-dive.md) and [doc 08](08-what-hospitals-actually-provide.md).

## Technical detail

### Confidence and sourcing

`**[spec]**` = written in a normative standard or vendor spec page. `**[community]**` = reported by implementers / third parties, not vendor-published. `**[unverified]**` = we have not confirmed. Epic's per-API detail pages at `fhir.epic.com/Specifications?api=NNN` are a JavaScript single-page app; the resource-type index counts below were read through a summarizing fetch and should be re-checked against your Epic version. **[community]**

### The gap matrix

Legend: US Core column = has a US Core 6.1.0 profile. (g)(10) = required by the ONC Standardized API certification criterion. Epic REST = exposed as a first-class Epic R4 read/search API. Epic Bulk = observed in Epic Group/Patient `$export` output. Last column = **proposed** CLIF target table(s), not settled.

| Resource | In US Core 6.1.0? | Required by (g)(10)? | Epic REST exposes? | Epic Bulk $export emits? | Which CLIF table it feeds (PROPOSED) |
|---|---|---|---|---|---|
| Patient | Yes | Yes | Yes | Yes | `patient` |
| Encounter | Yes | Yes | Yes | Yes | `hospitalization` + `adt` (via `Encounter.location[]`) |
| Observation (Vital Signs) | Yes | Yes | Yes (Read/Search/**Create**) | Yes | `vitals` |
| Observation (Lab Result) | Yes | Yes | Yes | Yes | `labs` |
| Observation (Social History / SDOH / Core Characteristics) | Yes | Yes | Yes | Yes | (cohort/demographics context) |
| Observation (SmartData Elements) | No | No | Yes (Read/Search) | **No** | `patient_assessments`; possibly `respiratory_support` / `position` |
| Observation (Lines, Drains, Airways) | No | No | Yes (R/S/Create/Update) | **No** | `respiratory_support` (airways), device-adjacent |
| Observation (Assessments) | No | No | Yes (Read/Search) | **No** | `patient_assessments` |
| Observation (Activities of Daily Living) | No | No | Yes (Read/Search) | **No** | `patient_assessments` |
| Observation (Genomics / OB-GYN / Newborn / DICOM / Phenotype / …) | No | No | Yes (Read/Search) | **No** | mostly out of scope |
| DiagnosticReport (Lab + Note) | Yes | Yes | Yes | Yes | `labs`; `microbiology_culture` (micro reports) |
| Specimen | Yes (added 6.x) | No | Yes | Yes | `labs`; `microbiology_culture` |
| Condition (Problems + Encounter Diagnosis) | Yes | Yes | Yes | Yes | `hospital_diagnosis` |
| Procedure | Yes | Yes | Yes | Yes | `patient_procedures` |
| MedicationRequest | Yes | Yes | Yes | Yes | medication **orders**, NOT administrations |
| MedicationDispense | Yes (added ~6.1) | No | Yes | Yes | pharmacy dispense, NOT administration |
| Medication | (referenced) | n/a | Yes | Yes | join target for med codes |
| **MedicationAdministration** | **No** | **No** | **Yes (Read/Search) — this is the MAR** | **No** | `medication_admin_continuous` + `medication_admin_intermittent` |
| AllergyIntolerance | Yes | Yes | Yes | Yes | (allergy context) |
| Immunization | Yes | Yes | Yes | Yes | (history context) |
| DocumentReference (Clinical Notes) | Yes | Yes | Yes | Yes | note text / `Binary` |
| ServiceRequest | Yes | No | Yes | Yes | orders context |
| Device (Implantable) | Yes | No | Yes | Yes | device context |
| Device (Lines, Drains, Airways) | No | No | Yes | Yes (`Device` generic) | possibly `ecmo_mcs` / `crrt_therapy` |
| Device (External Devices) | No | No | Yes | Yes (`Device` generic) | device context |
| DeviceRequest | No | No | Yes | **No** | device orders |
| DeviceUseStatement / DeviceUsage | **No** | No | Listed but effectively vaporware | **No** | (do not plan on it) |
| Coverage | Yes | No | Yes | **No** | out of scope |
| Provenance (Basic) | Yes | **Yes** | (referenced) | partial | audit/lineage, not clinical |
| CareTeam / CarePlan / Goal | Yes | partial | Yes | **No** | out of scope |
| Location / Organization / Practitioner / PractitionerRole | Yes (must-support refs) | as referenced | Yes | Yes | `adt` (Location), joins |
| RelatedPerson / QuestionnaireResponse | Yes | partial | Yes | **No** | out of scope |
| EpisodeOfCare | No | No | Yes | Yes | `hospitalization` grouping |
| Encounter (Outside Record) | No | No | Yes | (n/a) | external context |
| **Consent (Code Status)** | **No** | **No** | **Yes (limited to Code Status + Document)** | **No** | `code_status` |
| **Flag** (Patient FYI / Isolation / Infection) | **No** | **No** | **Yes** | **No** | isolation/infection context |
| NutritionOrder | No | No | Yes | **No** | nutrition orders |
| Account / Claim / Contract / ExplanationOfBenefit | No/partial | No | Yes | **No** | billing, out of scope |
| ResearchStudy / ResearchSubject / Task / RequestGroup / Communication(Request) | No/partial | No | Yes | **No** | out of scope |
| Binary | n/a | n/a | Yes | Yes | note/attachment payloads |
| RiskAssessment | No | No | **Not** a first-class Epic R4 read API | **No** | — |
| ChargeItem | No | No | **Not** found | **No** | — |
| MedicationStatement | **No** | No | (not primary) | **No** | patient-reported meds |

Sources: US Core 6.1.0 server CapabilityStatement `https://hl7.org/fhir/us/core/STU6.1/CapabilityStatement-us-core-server.html` **[spec]**; Epic API index `https://fhir.epic.com/` and `https://open.epic.com/Interface/FHIR` **[community]** (SPA caveat above); Epic bulk `$export` observed type list from `https://github.com/smart-on-fhir/cumulus/discussions/5` corroborated by `https://support.interopio.com/hc/en-us/articles/4419488428052` **[community]** — this list is NOT Epic-published.

### The three columns, restated

- **US Core 6.1.0 has profiles for**: Patient, Encounter, Condition, Procedure, Observation (Vital Signs / Lab Result / Social History / SDOH / Core Characteristics), DiagnosticReport (Lab + Note), MedicationRequest, MedicationDispense, AllergyIntolerance, Immunization, DocumentReference, ServiceRequest, Specimen, Coverage, Device (Implantable), Provenance, CareTeam, CarePlan, Goal, Location/Organization/Practitioner/PractitionerRole, RelatedPerson, QuestionnaireResponse. **[spec]** Note that **Basic Provenance is one of the few things (g)(10) explicitly *requires*.** **[spec]**
- **Critically NOT in US Core**: `MedicationAdministration`, `MedicationStatement`, `DeviceUseStatement`/`DeviceUsage`, `Flag`, `Consent`, `RiskAssessment`. **[spec]** Several of these (the MAR, code-status Consent, isolation Flags) are exactly the ICU-relevant records.
- **Epic exposes beyond US Core** (~55 types / ~450 endpoints): the whole family of specialized `Observation` flavors (SmartData Elements; Lines, Drains, Airways; Assessments; Activities of Daily Living; Study Finding; Genomics; and OB-GYN/Newborn/Periodontal/Phenotype/DICOM variants), `MedicationAdministration`, `Device (External Devices)` and `Device (Lines, Drains, Airways)`, `DeviceRequest`, `DeviceUseStatement`, `Flag`, `Consent (Code Status / Document)`, `EpisodeOfCare`, `Encounter (Outside Record)`, `NutritionOrder`, `Account`/`Claim`/`Contract`/`ExplanationOfBenefit`, `ResearchStudy`/`ResearchSubject`, `Task`, `RequestGroup`, `Communication`/`CommunicationRequest`. Epic also supports FHIR `Subscription` but it is **registration-gated / restricted**. **[community]**
- **Epic bulk `$export` emits ~20 types** (essentially US Core): AllergyIntolerance, Condition, Device, DiagnosticReport, DocumentReference, Encounter, EpisodeOfCare, Immunization, Location, Medication, MedicationDispense, MedicationRequest, Observation, Organization, Patient, Practitioner, PractitionerRole, Procedure, ServiceRequest, Specimen, Binary. **[community]** Epic launched Group `$export` in its **August 2021** release (Epic's Cooper Thompson, `https://chat-archive.fhir.org/stream/179166-implementers/topic/Epic.20Bulk.20Data.20Export.20questions.3F.html`). **[community]**

The single most consequential row: **`MedicationAdministration` — the actual bedside MAR — is exposed by Epic REST but is absent from bulk export.** Continuous drips and intermittent doses (vasopressors, sedation, antibiotics) all live here. This is why the CLIF `medication_admin_continuous` and `medication_admin_intermittent` tables cannot be populated from bulk alone.

### What `_type=Observation` gives you in bulk

Requesting `_type=Observation` in a bulk export does **not** dump arbitrary flowsheet rows. It returns predominantly the US Core Observation categories: `vital-signs`, `laboratory`, `social-history`, and SDOH. **[community]** The wider `Observation.category` value set (`exam`, `therapy`, `procedure`, `survey`, `activity`, `imaging`) — where much ICU flowsheet content actually lives — is reachable only through the specialized Epic REST Observation APIs, per-patient. **[spec]** for the value set (`http://hl7.org/fhir/ValueSet/observation-category`).

### Non-Epic note

Oracle Health / Cerner also serves R4. Its Group export conforms to Bulk Data IG v1.0.1 (with experimental v2.0.0 params), and Patient `$export` requires an **explicit patient-ID list capped at 20,000**. `https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfbda/bulk_data_access.html` **[spec]** This matters if the middleware ever targets a Cerner site: the cohort-first strategy is *mandatory* there, not just an optimization.

### CLIF coverage summary (18 tables)

Plausibly populated from FHIR (bulk and/or REST): `patient`, `hospitalization`, `adt`, `vitals`, `labs`, `medication_admin_continuous`, `medication_admin_intermittent`, `hospital_diagnosis`, `patient_procedures`, `patient_assessments`, `code_status`, `microbiology_culture`, `microbiology_nonculture`.

**No plausible FHIR source found** (state plainly): `respiratory_support` (ventilator settings — mode, FiO₂, PEEP, tidal volume are not reliably exposed as discrete FHIR observations), `crrt_therapy`, `ecmo_mcs`, `position` (e.g., prone/supine), and `microbiology_susceptibility` (antibiotic sensitivities as structured resistance data). These are the hardest gaps and likely require site-specific flowsheet extraction outside standard FHIR. See [doc 08](08-what-hospitals-actually-provide.md).

## Edge cases and how it breaks

- **Order ≠ administration (the classic research-data error).** `MedicationRequest` is a *prescription/order* — an intent that a drug be given. It is **not** evidence the drug reached the patient. A norepinephrine order at 08:00 does not mean norepinephrine ran. Only `MedicationAdministration` records the actual event with dose, rate, and time. Populating `medication_admin_continuous` from `MedicationRequest` would silently fabricate exposures. **[spec]**
- **Dispense ≠ administration either.** `MedicationDispense` is the *pharmacy* handing out the medication. It is closer to reality than an order but still not a bedside give-event — dispensed doses get wasted, held, or returned. **[spec]**
- **The MAR isn't in bulk, so cost is real.** Because `MedicationAdministration` requires per-patient REST calls, ICU medication exposure data scales with cohort size × calls-per-patient. This is the primary justification for filtering to inpatient/observation encounters *before* enrichment. See [doc 05](05-bulk-export-deep-dive.md).
- **Hidden nested Observations.** Epic returns panel results nested inside `DiagnosticReport.result` and `Observation.hasMember`, and **[community]** reportedly supports **neither `_include:iterate` nor `_revinclude`** for walking those references in one call. You may retrieve a report and still be missing the child values, forcing extra fetches. This directly threatens `labs` completeness. Forward ref: [doc 15](15-edge-cases.md).
- **`Observation.value[x]` polymorphism.** The value can be `valueQuantity`, `valueString`, `valueCodeableConcept`, `valueBoolean`, and more. A parser that assumes `valueQuantity` will drop string/coded results. **[spec]**
- **Blood pressure is a panel, not a number.** Systolic and diastolic arrive as `Observation.component[]` entries under one BP Observation, not as two top-level Observations — a frequent mapping bug when filling `vitals`. Forward ref: [doc 15](15-edge-cases.md). **[spec]**
- **`DeviceUseStatement`/`DeviceUsage` is effectively vaporware** in US EHRs. It is listed in specs but not meaningfully populated; do not architect `ecmo_mcs`/`crrt_therapy` around it. **[community]**
- **Consent is narrow.** Epic's `Consent` is limited to Code Status and Document flavors — usable for `code_status`, but do not expect general treatment-consent modeling. **[community]**

## What we still don't know

- **Exact per-version Epic reality.** The 55-type / 450-endpoint counts and the parenthetical API labels came through a summarizing fetch of a JavaScript SPA index; the authoritative per-API pages must be checked on the live spec for the target Epic version. **[community]** Which flavors are enabled is also a **site licensing/config** decision, not a global Epic fact.
- **Whether any site exposes ventilator settings as FHIR.** We found no standard resource reliably carrying vent mode/FiO₂/PEEP/tidal volume. Some sites may surface these as `Observation (SmartData Elements)` or `Observation (Lines, Drains, Airways)`, but coverage and coding are unconfirmed — this is the biggest open question for `respiratory_support`.
- **CRRT and ECMO/MCS sourcing.** No confirmed FHIR path. Possibly `Device (Lines, Drains, Airways)` plus associated Observations, but unverified.
- **Microbiology structure.** Whether organism identification, culture, and antibiotic **susceptibility** come through as structured `Observation`/`DiagnosticReport` with `Specimen` links — versus free text in a `DiagnosticReport (Note)` — is site-dependent and unverified. `microbiology_susceptibility` in particular has no confirmed structured source.
- **Position / prone data.** No confirmed FHIR representation for patient positioning.
- **Bulk `$export` type list is community-observed.** Epic does not publish the authoritative bulk type list; the ~20 types could vary by version/config. **[community]**
- **The final CLIF mapping.** Every entry in the last matrix column is a **proposed** intuition, not a validated mapping. Category codes, unit normalization, and the split between continuous vs. intermittent medication tables all need empirical validation against real extracts.
