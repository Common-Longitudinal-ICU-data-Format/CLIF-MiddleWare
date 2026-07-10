# What hospitals actually provide

*The feasibility verdict. Which CLIF tables FHIR can fill, which it cannot, and the honest comparison against the path the CLIF consortium already uses.*

---

## In plain words

Everything else in this folder describes how the plumbing works. This document answers the only question that decides whether the project is worth doing: **if we connect this middleware to a real hospital, how much of CLIF actually comes out the other end?**

The short answer is: about half, and not the half that makes ICU data interesting.

Here is the shape of the problem. The US government requires every certified hospital records system to publish a standard set of data through FHIR. That mandated set — demographics, lab results, diagnoses, procedures, medication *orders*, basic charted vital signs — is genuinely available, genuinely standardized, and comes out of a single cheap bulk download. That is real, and it is enough to identify a cohort and describe it.

But the things that make intensive care *intensive* are not in the mandated set. Ventilator settings. The rate a norepinephrine drip is running at, right now, in micrograms per kilogram per minute. Whether the patient is proned. Whether they are on dialysis or ECMO. Their sedation score. None of these are required by regulation, and none of them appear in the bulk download.

Some of them exist in Epic's wider interface, one patient at a time, if the hospital has done specific configuration work that most hospitals have not done. Some of them appear to exist nowhere in FHIR at all — we searched, and could not find a single published example of any hospital anywhere exposing ventilator settings as queryable FHIR data.

There is a second, more uncomfortable fact. **The CLIF consortium does not use FHIR.** They ship ready-made SQL queries that read Epic's internal data warehouse directly, where every flowsheet row lives at full resolution with no configuration prerequisite. That path is not portable between vendors and requires database credentials and a friendly database administrator, but it *works*, and it works for all eighteen tables. We are proposing to walk away from a road that exists in order to build one that might not reach the destination.

That is not automatically the wrong call. A FHIR pipeline needs no database access, no DBA, no Epic-specific SQL, and would work at a Cerner site unchanged. Onboarding a new CLIF site could become a config file rather than a six-month project. That is a real prize. But it is a prize you win *only if* the data is there, and right now we do not know that it is.

The rest of this document is the evidence, table by table, with our confidence in each claim stated openly.

---

## Technical detail

### The three-layer reality

There are three nested sets, and confusing them is the most common error in this space.

```
   ┌──────────────────────────────────────────────────┐
   │  What Epic's REST API exposes                    │
   │  ~55 resource types, ~450 endpoints              │
   │  incl. MedicationAdministration, Observation      │
   │        (SmartData Elements / LDA / Assessments),  │
   │        Consent (Code Status), Flag                │
   │                                                   │
   │   ┌───────────────────────────────────────────┐  │
   │   │  What Epic's Bulk $export emits           │  │
   │   │  ~20 resource types ≈ the US Core set     │  │
   │   │  NO MedicationAdministration              │  │
   │   │  NO non-USCDI Observation flavors         │  │
   │   │                                            │  │
   │   │   ┌─────────────────────────────────┐     │  │
   │   │   │  What ONC (g)(10) requires      │     │  │
   │   │   │  US Core 6.1.0 / USCDI v3       │     │  │
   │   │   └─────────────────────────────────┘     │  │
   │   └───────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────┘
```

**US Core is a floor, not a ceiling.** You cannot conclude "the hospital doesn't have X" from "X isn't in US Core." Epic advertises roughly 55 resource types across ~450 endpoints, well beyond the mandate. **[community]** — from Epic's rendered index at [fhir.epic.com](https://fhir.epic.com/) and [open.epic.com/Interface/FHIR](https://open.epic.com/Interface/FHIR); the per-API spec pages are a JavaScript application that resists automated reading, so verify against the live pages for your Epic version.

**But bulk export is stuck at the floor.** The observed Epic Group `$export` resource list is: AllergyIntolerance, Condition, Device, DiagnosticReport, DocumentReference, Encounter, EpisodeOfCare, Immunization, Location, Medication, MedicationDispense, MedicationRequest, Observation, Organization, Patient, Practitioner, PractitionerRole, Procedure, ServiceRequest, Specimen, Binary. **[community]** — [smart-on-fhir/cumulus discussion #5](https://github.com/smart-on-fhir/cumulus/discussions/5), corroborated by [interopiO](https://support.interopio.com/hc/en-us/articles/4419488428052). **This is not Epic-published. It must be verified empirically per site** — see [doc 17](17-open-questions-and-roadmap.md), question 1.

`MedicationAdministration` is absent. And `_type=Observation` in bulk returns predominantly the US Core categories — `vital-signs`, `laboratory`, `social-history`, SDOH — not arbitrary flowsheet rows.

**This asymmetry is the single most important engineering fact in the project.** It is why the pipeline is bulk-then-REST rather than bulk-only: bulk is wide and cheap and reaches the floor; REST is narrow and expensive and reaches the ceiling. Filtering to inpatient and observation encounters between the two legs is what makes the expensive leg affordable. See [doc 04](04-api-access-modes.md).

### How Epic surfaces flowsheet data

Nearly all ICU-specific documentation in Epic lives in **flowsheets** — rows identified by `FLO_MEAS_ID`. Some flowsheet rows surface through FHIR `Observation` flavors: `Observation (Vital Signs)`, `Observation (SmartData Elements)`, `Observation (Lines, Drains, Airways)`, `Observation (Assessments)`, `Observation (Activities of Daily Living)`.

Reading arbitrary flowsheet rows is **partial and gated, not turnkey**:

- `Observation.Search (Vital Signs)` returns the USCDI vitals set plus rows the organization has categorized as vitals. It filters by FHIR `category=vital-signs` and by **LOINC**. A flowsheet row appears only if it has been **mapped to a LOINC code**.
- `Observation.Search (SmartData Elements)` is the closest thing to arbitrary flowsheet read, but it reads **SmartData Elements**, not raw flowsheet rows, returned by Epic SDE identifier, and queryability is configuration-dependent.
- **There is no single "give me every flowsheet row as FHIR" read API.** This is the crux.

For a hospital to expose, say, ventilator PEEP as a FHIR Observation, an Epic analyst must map that flowsheet row to LOINC in **Doc Flowsheet Builder → Concept Mapping** (row → LOINC, values → SNOMED), *and* the row must belong to a category exposed by one of the Observation flavors. Most ICU respiratory rows are **not LOINC-mapped out of the box**.

Epic's documented [flowsheet interface](https://fhir.epic.com/Documentation?docId=rpm_devices_software&section=example_flowsheet_interface) describes `Observation.Create (Flowsheet Value)` — but that is fundamentally a **write** API, oriented to remote-patient-monitoring and MyChart-connected devices, posting one reading at a time. It is how data goes *in*, not how it comes back out. Third-party integration guidance treats FHIR `Observation` as an inbound device path alongside HL7 v2 `ORU^R01`, and does not claim general read-back ([Mindbowser](https://www.mindbowser.com/epic-device-integration-hl7-oru-vs-fhir-observation/)).

**We found no public example of any Epic site exposing full ventilator, CRRT, or ECMO flowsheet data as queryable FHIR Observations.** **[unverified]** — absence of evidence, not evidence of absence, but consistent with the architecture. This is the largest single risk to the project and it must be tested before anyone writes production code.

### MedicationAdministration — the drip problem

CLIF's `medication_admin_continuous` table needs, per row: the drug, the dose, the dose unit, the MAR action (`start`, `stop`, `going`, `dose_change`), and the timestamp. To reconstruct vasopressor exposure you need the **rate** and the **start/stop interval**.

- Epic **does** expose `MedicationAdministration.Search` and `.Read` (R4). This is the MAR. **[community]**
- FHIR R4 supports `dosage.rate[x]` as `rateQuantity` or `rateRatio`, and `effectiveDateTime` or `effectivePeriod` ([HL7 R4 MedicationAdministration](https://hl7.org/fhir/R4/medicationadministration.html)) **[spec]**.
- **Whether Epic populates `dosage.rate[x]` for continuous infusions is not documented and not verified.** **[unverified]** Community experience is that Epic's MedicationAdministration is strong on discrete "given" events (drug, dose, route, time, nurse) and thin or inconsistent on continuous-infusion rate and clean start/stop semantics.
- `MedicationAdministration` is **not in Epic's bulk export list** **[community]**. So bulk gives you `MedicationRequest` (orders) and `MedicationDispense` (pharmacy) — neither of which is what was actually infused into the patient.

**A prescribed drug is not a given drug.** Building `medication_admin_continuous` from `MedicationRequest` is a well-known way to produce a research dataset that is confidently wrong.

The pragmatic alternative source for administered doses and rates is Epic Clarity's `MAR_ADMIN_INFO` table, also present in Epic's [EHI Export](https://open.epic.com/EHITables/GetTable/MAR_ADMIN_INFO.htm).

### The reachability tiers

Confidence is stated per table. **This table is the deliverable.** It is what you take to a hospital IT contact.

| CLIF table | Tier | Source | Confidence |
|---|---|---|---|
| `patient` | **A** | `Patient` (bulk) | high |
| `hospitalization` | **A** | `Encounter` (bulk) | high |
| `adt` | **A*** | `Encounter.location[]` + `Location` (bulk) | high *if* `location[].period` is populated — varies by site **[community]** |
| `labs` | **A** | `Observation` (category=laboratory) + `DiagnosticReport` + `Specimen` (bulk) | high |
| `vitals` | **A*** | `Observation` (category=vital-signs) (bulk) | high for the 9 CLIF categories; charted cadence only, not monitor-frequency |
| `hospital_diagnosis` | **A** | `Condition` (bulk) | high |
| `patient_procedures` | **A** | `Procedure` (bulk) | high |
| `medication_admin_intermittent` | **B** | `MedicationAdministration` (REST only) | medium — resource exists; discrete "given" events are its strong suit |
| `medication_admin_continuous` | **B?** | `MedicationAdministration` (REST only) | **low** — rate and start/stop population **[unverified]** |
| `patient_assessments` | **B** | `Observation (Assessments)` / `(SmartData Elements)` (REST only) | **low** — requires site to have LOINC-mapped RASS, GCS, CAM-ICU, etc. |
| `code_status` | **B** | `Consent (Code Status)` (REST only) | medium — Epic exposes it; mapping to CLIF's 10 categories unexamined |
| `microbiology_culture` | **B** | `Observation` / `DiagnosticReport` + `Specimen` | medium — organism coding to 545 categories is substantial work |
| `microbiology_nonculture` | **B** | `Observation` / `DiagnosticReport` | medium |
| `respiratory_support` | **C** | none found | **no public example exists** |
| `crrt_therapy` | **C** | none found | **no public example exists** |
| `ecmo_mcs` | **C** | none found; possibly `Device (LDA)` | **no public example exists** |
| `position` | **C** | none found | **no public example exists** |
| `microbiology_susceptibility` | **C** | none found | needs `organism_id` linkage FHIR does not obviously provide |

**Tier A** — reachable from bulk export, high confidence.
**Tier B** — reachable only through interactive REST on a filtered cohort, and only if the site has done the configuration. Site-dependent.
**Tier C** — no FHIR path identified at any site. Requires HL7 v2, device middleware, or direct warehouse SQL.

### What this means for the science

A Tier-A-only warehouse can: identify an ICU cohort, describe demographics, compute comorbidity indices (Charlson, Elixhauser) from `hospital_diagnosis`, chart lab trajectories, and compute length of stay and mortality.

It **cannot** compute a SOFA score. SOFA needs six components: respiratory (PaO₂/FiO₂ — needs `respiratory_support.fio2_set`, Tier C), coagulation (platelets — Tier A), liver (bilirubin — Tier A), cardiovascular (**vasopressor dose** — Tier B?, low confidence), CNS (**GCS** — Tier B, low confidence), renal (creatinine — Tier A, plus urine output). Four of six are at risk.

It cannot characterize mechanical ventilation, prone positioning, sedation depth, or renal replacement therapy.

**Say this plainly to stakeholders: a FHIR-only CLIF is a cohort-discovery index, not an ICU physiology warehouse.**

### The honest alternative: Clarity and Caboodle

The CLIF consortium's [tools page](https://clif-icu.com/tools) states: *"Jump-start your CLIF 2.0.0 implementation with our pre-built SQL queries for Epic's Caboodle/Clarity systems … Optimized for Epic systems."* The [CLIF paper](https://pubmed.ncbi.nlm.nih.gov/39281737/) (9 health systems, 39 hospitals, 111,440 admissions) describes building CLIF from institutional EHR warehouses.

Why Clarity wins on data:

- **Flowsheet fidelity.** Clarity's `IP_FLWSHT_MEAS` / `IP_FLWSHT_REC` expose **every flowsheet row by `FLO_MEAS_ID` at full temporal resolution**, with no LOINC-mapping prerequisite and no per-row FHIR build. This single fact resolves every Tier C table.
- **Medication administration.** `MAR_ADMIN_INFO` gives dose, rate, route, and time.
- **Arbitrary cohorts and joins.** SQL over the warehouse gives you any cohort, full history, no `$export` throttle, no missing `_since`, no hidden-Observation quirks.

Why FHIR wins on everything else:

- **Portability.** The same client works against Epic, Oracle Health, MEDITECH, athenahealth. Clarity SQL works against Epic only, and against *that site's* Clarity build.
- **Access.** FHIR needs an OAuth client and an app registration. Clarity needs database credentials, a DBA relationship, and typically a named analyst. At many institutions the second is harder to obtain than the first.
- **Stability.** FHIR is a versioned public standard. Clarity's schema is Epic-internal and changes across upgrades.
- **Legal clarity.** FHIR access is exactly the pathway the government mandated for exactly this kind of use. That is not nothing when you are in front of an IRB.

Also relevant: Epic's **EHI Export** ([open.epic EHI tables](https://open.epic.com/EHITables/GetTable/IP_FLWSHT_EDITED.htm)) is a relational dump mandated for information-blocking compliance that mirrors Clarity's structure — a third, SQL-shaped path with a regulatory basis.

For orientation: **Chronicles** is Epic's live operational database. **Clarity** is the nightly relational SQL warehouse. **Caboodle** is the dimensional warehouse. **Interconnect** is the API gateway that hosts the FHIR endpoints — plumbing, not a data store. Research warehouse teams at Epic sites overwhelmingly use Clarity and Caboodle.

### The recommendation

Do not choose yet. **Design the middleware so the source is pluggable, then let the pilot site's answers choose for you.**

Concretely: a FHIR `Observation`, an HL7 v2 `OBX` segment, and a Clarity `IP_FLWSHT_MEAS` row are all the same thing — a timestamped clinical fact with a source-specific identifier, a value, and a unit. Land all three in the same `clif_events` table. Catalog all three in the same `clif_signals` table. Map all three with the same MapperEngine. **The signal catalog does not care where a signal came from.** See [doc 12](12-clif-events-and-signals.md).

If that abstraction holds, the FHIR-versus-Clarity question stops being an architecture decision and becomes a deployment decision made per site. That is the correct place for it, because the answer genuinely differs per site.

---

## Edge cases and how it breaks

**The 20-type bulk list is community-reported, not Epic-published.** Our entire Tier A/B split rests on it. If the target site's Epic emits `MedicationAdministration` in bulk, the architecture simplifies considerably and we will have over-engineered. If it emits *fewer* than 20 types, Tier A shrinks. **Probe before believing.** [doc 17](17-open-questions-and-roadmap.md), question 1.

**"Epic exposes X" does not mean "this hospital exposes X."** Every Epic API is per-site enabled, and every flowsheet-to-LOINC mapping is per-site work. Two hospitals on the same Epic version can have completely different FHIR surfaces. There is no shortcut around asking the specific site.

**`Observation.Search (Vital Signs)` returns *charted* vitals, not monitor vitals.** A patient's heart rate is sampled continuously by the bedside monitor and written to the flowsheet every 15 minutes to an hour. FHIR gives you the flowsheet, not the monitor. CLIF's `vitals` table will therefore be correct but sparse. For most retrospective research this is fine; for anything resembling waveform analysis it is useless.

**Waveforms are not in FHIR at all.** ECG, plethysmography, arterial pressure tracings, breath-by-breath ventilator data. FHIR has `Device`, `DeviceMetric`, and the [Point-of-Care Device IG](https://build.fhir.org/ig/HL7/uv-pocd/mappingv2.html) with ISO/IEEE 11073 mappings, but US EHRs do not expose ICU device data through them and USCDI does not require it. That data lives in device-integration middleware (Capsule, Philips, GE) and reaches the EHR as HL7 v2 `ORU^R01`.

**`DeviceUseStatement` / `DeviceUsage` is effectively vaporware in US EHRs.** Do not plan to reach ECMO or CRRT device usage through it.

**Epic hides nested Observations.** Members of `DiagnosticReport.result` and `Observation.hasMember` are not returned by default, and Epic supports neither `_include:iterate` nor `_revinclude` **[community]**. A naive `_type=DiagnosticReport` export silently loses the individual lab values. Export `Observation` alongside and stitch by reference; add `&_include=Observation:hasMember:Observation`.

**Tier B is only affordable because of the cohort filter.** REST enrichment costs roughly one HTTP round trip per patient per resource type. On an unfiltered population that is prohibitive; on inpatients and observation patients from a single hospital it is a few thousand calls. If the hospital's Group turns out to be broader than expected, the cost model breaks. See [doc 10](10-cohort-and-encounter-class.md).

**The bulk throttle makes exploration expensive.** Roughly one Group export per 24 hours **[community]**. You cannot iterate on `_type` and `_typeFilter` combinations interactively. Every probe costs a day. Plan probes carefully, and do all exploration on the sandbox first. See [doc 05](05-bulk-export-deep-dive.md).

**Partial tables may be worse than absent tables.** A `respiratory_support` table populated only for the handful of patients whose site happened to LOINC-map their vent rows produces a silently biased cohort. Any table below full population must carry an explicit coverage statistic in warehouse metadata, and analyses must be able to see it.

---

## What we still don't know

These are ranked by how much they change the answer. All are cheap to resolve, and all should be resolved before production code is written.

- **Does Epic populate `MedicationAdministration.dosage.rateQuantity` and `effectivePeriod` for continuous infusions?** Decides whether `medication_admin_continuous` — and therefore SOFA cardiovascular, and therefore most sepsis research — is reachable via FHIR. **Resolve:** `GET MedicationAdministration?patient=<ICU test patient>` against the Epic sandbox and inspect `dosage.rate[x]` and `effective[x]`. An afternoon's work.

- **Has any site LOINC-mapped its ventilator, RASS, GCS, CRRT, or ECMO flowsheet rows so they are readable as FHIR Observations?** Decides Tier C entirely. **Resolve:** `Observation.Search` with known LOINCs (RASS ≈ 71387-8, GCS total ≈ 9269-2) against a test patient, and ask the site's Epic analyst directly whether Doc Flowsheet Builder concept mapping has been done.

- **Does the target site's bulk export really emit only the ~20 types?** **Resolve:** kick off a probe Group `$export` with no `_type` and enumerate `output[]` in the manifest.

- **Is `Encounter.location[]` populated with full `period` history at this site, or collapsed to current location?** Single-handedly decides whether the `adt` table is buildable from FHIR. **[community]** says it varies.

- **Whether Epic's non-US-Core Observation flavors are reachable with the scopes a research client can obtain.** Existence in the API catalog is not the same as being grantable to us.

- **Whether the CLIF consortium would recognize a FHIR-derived warehouse as conforming.** A social question, not a technical one, and worth asking early. `clif_consortium@uchicago.edu`.

- **Whether the hospital would give us Clarity access anyway.** If yes, much of this document is moot and the honest engineering answer is to use it. Ask before building.
