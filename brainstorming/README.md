# CLIF-MiddleWare — feasibility brainstorm

*Can we build a CLIF research warehouse out of a hospital's FHIR API? Partly. Here is exactly which parts, and what breaks.*

---

## Read this first

**The idea.** A service runs on a Linux server inside a hospital. It authenticates to the hospital's electronic health record as a machine — no human logs in — and downloads clinical data. It keeps only the patients who were admitted as inpatients or placed under observation. It catalogs every distinct kind of clinical fact it sees. A mapper engine, with an AI assistant proposing and a clinician approving, translates those facts into [CLIF](01-what-is-clif.md), the Common Longitudinal ICU data Format used for federated critical-care research. A web console handles configuration, mapping review, and monitoring.

**The verdict.** The plumbing works. The data is only half there.

FHIR reliably delivers demographics, encounters, laboratory results, diagnoses, procedures, and charted vital signs. It does not deliver ventilator settings, continuous infusion rates, sedation scores, prone positioning, dialysis, or ECMO — the things that make intensive care intensive. Roughly seven of CLIF's eighteen tables come out of a bulk export cleanly. Five more are reachable only through expensive per-patient requests, and only if the hospital has done configuration work most hospitals have not done. Five appear to be unreachable through FHIR at any hospital we could find.

**That is not a reason to stop.** It is a reason to build the middleware so the data source is a plug, not an assumption — and to spend an afternoon against Epic's sandbox answering two questions before anyone writes production code.

**These documents contain no code.** They are a feasibility study. Every non-obvious claim carries a source link and a confidence tag: **[spec]** means it is published in an HL7 or vendor specification, **[community]** means implementers report it but the vendor has not published it, and **[unverified]** means we could not confirm it and it must be tested against a real server.

---

## The three findings that shape everything

### 1. US Core is a floor, not a ceiling — but bulk export is stuck at the floor

Epic exposes roughly 55 FHIR resource types across ~450 endpoints, far beyond what regulation requires. That includes `MedicationAdministration` — the medication administration record, where infusion data would live — plus several flowsheet-derived `Observation` flavors and `Consent (Code Status)`.

But Epic's Group `$export` emits only about twenty resource types, and that list is essentially the US Core set. `MedicationAdministration` is not among them. Neither are the non-USCDI Observation flavors. **[community]** — [cumulus discussion #5](https://github.com/smart-on-fhir/cumulus/discussions/5), corroborated by [interopiO](https://support.interopio.com/hc/en-us/articles/4419488428052). This is not Epic-published and **must be verified per site**.

So you cannot conclude "the hospital doesn't have it" from "it isn't in US Core." But you also cannot get it from the bulk download. That asymmetry is the central engineering fact of this project, and it produces the architecture below.

### 2. The CLIF consortium does not use FHIR

[clif-icu.com/tools](https://clif-icu.com/tools) ships pre-built SQL queries for Epic's Caboodle and Clarity data warehouses. The [CLIF paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC11398431/) states that CLIF "is currently not linked to established interoperability standards like HL7 FHIR."

There is no published FHIR→CLIF ETL, no OMOP→CLIF bridge, and no real-time CLIF. We are proposing a path nobody has taken, walking away from one that already works. The prize is portability — a FHIR pipeline needs no database credentials, no DBA, no Epic-specific SQL, and works unchanged at a Cerner site. Onboarding a new CLIF site could become a config file instead of a six-month project. That prize is real, and it is only collectable if the data is actually there.

### 3. CLIF's vocabularies carry no code crosswalk

CLIF's controlled vocabularies (mCIDE) contain a category name, a description, and free-text examples. **Zero LOINC, RxNorm, or SNOMED columns.** The only code field anywhere in the CLIF schema is the optional `labs.lab_loinc_code`.

So mapping FHIR codes to CLIF categories is work nobody has done and no lookup table exists for. It is the core intellectual property of the middleware, and it is exactly where an AI-proposes / clinician-approves design earns its place.

---

## The pipeline

```
  1. DISCOVER   /metadata + /.well-known/smart-configuration + a probe export.
                Ask the server what it has. Never assume from the spec.
  2. AUTH       SMART Backend Services. A signed, 5-minute, single-use JWT.
                No username, no password. The hard part is the paperwork.
  3. BULK       Group $export → poll → NDJSON. Wide, cheap, ~once per 24h.
                Gets the backbone: patient, encounter, labs, diagnoses.
  4. FILTER     Encounter.class ∈ {IMP, ACUTE, NONAC, OBSENC}
                Shrinks the cohort ~10×. THIS IS WHAT MAKES STEP 5 AFFORDABLE.
  5. ENRICH     Targeted REST on the small cohort only.
                MedicationAdministration, flowsheet Observations, Consent.
  6. LAND       clif_events   — append-only raw log, one row per fact.
                clif_signals  — the DISTINCT kinds of fact. A few thousand rows.
  7. MAP        MapperEngine, one per CLIF table.
                RxNav/LOINC/UCUM propose. An LLM ranks. A clinician decides.
                The decision is frozen, versioned, and never re-asked.
  8. EMIT       clif_events ⋈ signal_map → clif_*.parquet → clifpy.validate()
                Deterministic. No model. Byte-identical on re-run.
```

Two structural ideas hold this together.

**Filter between bulk and enrich.** Bulk costs O(1) requests for O(N) patients; REST costs O(N) requests. Shrink N first. Reverse the order and the enrichment leg is unaffordable and trips rate limits.

**A billion facts, a few thousand kinds.** The signal catalog is what makes an AI mapper viable. The model reads the catalog, never a patient row. A clinician reviews a few thousand entries sorted by how much data each affects. The decisions are frozen, and the transform from raw log to CLIF becomes a mechanical join with no model in the loop. When a mapping turns out to be wrong, we fix one row and re-run — we never ask the hospital for the data again, which matters because Epic will only give it to us about once a day.

And one idea that will outlive the FHIR decision: **a signal is a signal regardless of where it came from.** A FHIR `Observation`, an HL7 v2 `OBX` segment, and a Clarity flowsheet row are the same shape. All three land in `clif_events`. All three flow through the same mapper. So when the ventilator data turns out to need an HL7 v2 feed, only the adapter changes.

---

## Reachability: the table to take to a hospital IT contact

| Tier | Meaning | CLIF tables |
|---|---|---|
| **A** | Bulk export. High confidence. | `patient`, `hospitalization`, `adt`\*, `labs`, `vitals`\*, `hospital_diagnosis`, `patient_procedures` |
| **B** | Interactive REST only. Site-configuration dependent. | `medication_admin_intermittent`, `medication_admin_continuous`†, `patient_assessments`†, `code_status`, `microbiology_culture`, `microbiology_nonculture` |
| **C** | No FHIR path found at any site. Needs HL7 v2, device middleware, or Clarity SQL. | `respiratory_support`, `crrt_therapy`, `ecmo_mcs`, `position`, `microbiology_susceptibility` |

\* `adt` depends on `Encounter.location[].period` being populated, which varies by site. `vitals` is charted cadence, not monitor frequency.
† Low confidence. Whether Epic populates infusion rates, and whether any site has LOINC-mapped its RASS/GCS flowsheet rows, are both **[unverified]**.

**What this means for the science.** A Tier-A-only warehouse identifies a cohort, describes it, computes comorbidity indices, and charts lab trajectories. It **cannot compute a SOFA score** — four of the six components are at risk. It cannot characterize mechanical ventilation, sedation depth, vasopressor exposure, or renal replacement therapy.

Say it plainly to stakeholders: **a FHIR-only CLIF is a cohort-discovery index, not an ICU physiology warehouse.**

---

## The two questions worth an afternoon

Both are answerable against Epic's free sandbox, before a line of production code exists. They are the highest-leverage hour on this project.

1. **Does Epic populate `MedicationAdministration.dosage.rateQuantity` and `effectivePeriod` for continuous infusions?** Decides whether vasopressor exposure — and therefore most sepsis research — is reachable via FHIR.
2. **Has any site LOINC-mapped its ventilator, RASS, GCS, CRRT, or ECMO flowsheet rows so they are readable as FHIR Observations?** Decides Tier C entirely. We searched and found no published example anywhere.

If both come back negative, along with `Encounter.location[]` being collapsed, then FHIR yields demographics, labs, diagnoses, and intermittent vitals. That would be a reason to **pivot the pipe, not the project** — the same `clif_events` log and the same mapper sit behind an HL7 v2 or Clarity adapter unchanged.

See [doc 17](17-open-questions-and-roadmap.md) for the full list and the experiment for each.

---

## Where to develop

Never learn a new failure mode for the first time against a live hospital, because you get roughly one bulk export attempt per day.

1. **[SMART Bulk Data Server](https://bulk-data.smarthealthit.org/)** — the reference implementation. Proves backend authentication and the async export protocol with nothing else in the way.
2. **[MIMIC-IV-on-FHIR demo](https://physionet.org/content/mimic-iv-fhir-demo/2.1.0/)** — 100 patients of real, de-identified ICU data as FHIR. **Open access: no training, no data-use agreement.** It carries `MimicObservationChartevents` and `MimicMedicationAdministrationICU` — exactly the Tier B and C content we need to test against.
3. **Epic sandbox** — rehearse the app registration, key upload, and Group export before touching a hospital.

**The test oracle.** There is already a published, human-built [MIMIC→CLIF mapping](https://physionet.org/content/mimic-iv-ext-clif/1.1.0/) ([ETL source](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC)), and MIMIC also exists as FHIR. Run our FHIR→CLIF pipeline over MIMIC and diff the output against the published CLIF. That converts "does it run" into "it agrees with expert humans on 94% of lab categories, and here are the 6% where it doesn't." **Nothing else on the roadmap is worth as much.**

---

## The documents

Each one opens with a plain-English section a physician or IRB reviewer can read, followed by technical detail, then failure modes, then explicit open questions.

**Start here**
- [08 — What hospitals actually provide](08-what-hospitals-actually-provide.md) — *the feasibility verdict. If you read one document, read this one.*
- [15 — Edge cases](15-edge-cases.md) — *every trap we found. Most of them fail silently.*
- [17 — Open questions and roadmap](17-open-questions-and-roadmap.md) — *what we're guessing about, and the cheapest experiment for each.*

**The target**
- [01 — What is CLIF](01-what-is-clif.md) — the 18 tables, exact columns, the `_name`/`_category` pattern, the validator that grades us.

**The source**
- [02 — FHIR fundamentals](02-fhir-fundamentals.md) — versions, US Core, USCDI, ONC (g)(10), floor-not-ceiling.
- [03 — FHIR resource types](03-fhir-resource-types.md) — the gap matrix: US Core vs Epic REST vs Epic bulk.
- [04 — API access modes](04-api-access-modes.md) — bulk vs REST vs subscriptions vs HL7 v2 vs device middleware.
- [05 — Bulk export deep dive](05-bulk-export-deep-dive.md) — the `$export` protocol end to end, and every vendor deviation.
- [06 — Authentication](06-authentication.md) — SMART Backend Services, the signed JWT, key custody. The hard part is the paperwork.
- [07 — Free FHIR servers](07-free-fhir-servers.md) — where to develop. Includes the MIMIC test oracle.
- [09 — Data freshness](09-data-freshness.md) — how often we can pull, and why "real-time CLIF over bulk export" is not a thing.
- [10 — Cohort and encounter class](10-cohort-and-encounter-class.md) — inpatient vs observation, and who really controls the cohort.

**The build**
- [11 — Middleware architecture](11-middleware-architecture.md) — systemd service, state machine, resumability, the web console.
- [12 — clif_events and clif_signals](12-clif-events-and-signals.md) — the raw log and the distinct-signal catalog. Why the split.
- [13 — The Mapper Engine](13-mapper-engine.md) — AI proposes, clinician disposes, the decision is frozen.
- [14 — Terminology resources](14-terminology-resources.md) — LOINC, RxNav, RxClass, UCUM, SNOMED. What exists vs what we build.

**The governance**
- [16 — Security and governance](16-security-and-governance.md) — HIPAA, IRB, BAA, key custody, and the LLM as a PHI egress risk.

---

## The thing that should worry you most

It is not a protocol bug. It is that this pipeline can produce a dataset that loads cleanly, passes validation, and is wrong.

Blood pressure silently absent because it arrives as a panel with no top-level value. Creatinine off by a factor of 88 because the units changed on a Tuesday. An amended potassium result never updated because the server didn't bump a timestamp. A cohort filter silently dropped by the server because `Prefer: handling=lenient` was set. A clinician approving 400 AI mappings in ten minutes, which the audit log then records as physician review.

Every one of those is documented in [doc 15](15-edge-cases.md) with a mitigation. None of them raise an error. Clinical data is unusually good at looking plausible while being false, and the counter-measures — keep the raw string next to the answer, keep the raw log so you can replay, make a human sign every clinical decision, and report coverage honestly — are the reason this design has the shape it does.
