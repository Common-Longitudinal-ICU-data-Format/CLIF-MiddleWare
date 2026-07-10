# Free FHIR servers: where to build this before touching a hospital

*Never learn a new failure mode for the first time against a live EHR — you get roughly one bulk export per day at Epic, so rehearse every mechanic somewhere free first.*

## In plain words

We are building a pipeline that logs into a hospital's FHIR API, pulls the whole ICU population out in bulk, and reshapes it into CLIF. That pipeline has three hard parts that fail in different ways: the **authentication handshake**, the **asynchronous bulk-export dance** (kick it off, poll, download a pile of files), and the **mapping** of FHIR resources into correct CLIF rows. If we debug all three at once against a real Epic sandbox, we waste weeks — because Epic throttles us to about one export attempt per day (see [doc 05](05-bulk-export-deep-dive.md)), a new sandbox key can take an hour just to start working **[community]**, and the sandbox data is thin, so we would still not know whether our ICU mapping is *correct*.

So we split the problem across free servers, each testing one part. One reference server exists purely to prove auth and async plumbing. Separately, and this is the headline: **there is real, de-identified ICU data available as FHIR, for free, with no paperwork.** MIMIC-IV is the intensive-care database from Beth Israel Deaconess in Boston; a research group published a FHIR R4 version of it, and there is a 100-patient demo subset under an open license — no training, no data-use agreement, download it today. That matters because Epic's bulk export will *not* hand us the ICU-depth content (ventilator-cadence vitals, titrated drips); MIMIC does. Even better: someone has *also* already published a CLIF version of MIMIC. That gives us a **test oracle** — we can check whether our FHIR-to-CLIF pipeline produces the *right answer*, not merely whether it runs without crashing. That is a rare gift in health-data engineering and it should anchor our whole test strategy.

Only after auth, async, and mapping each work in isolation do we go rehearse the vendor-specific quirks against Epic's sandbox — the last step, not the first.

## Technical detail

**Purpose-built SMART / Bulk sandboxes — for testing auth + async in isolation.**

- **SMART Bulk Data Server** — [bulk-data.smarthealthit.org](https://bulk-data.smarthealthit.org/) ([source](https://github.com/smart-on-fhir/bulk-data-server)). R4, ~100 Synthea patients. Full `$export` at System, Group, and Patient level, and **full SMART Backend Services auth**: you paste a public key, it issues a `client_id`. This is *the* reference server for exercising backend auth and bulk export end to end (auth mechanics in [doc 06](06-authentication.md)). Carries AllergyIntolerance, CarePlan, CareTeam, Claim, Condition, Device, DiagnosticReport, DocumentReference, Encounter, ExplanationOfBenefit, Group, ImagingStudy, Immunization, MedicationRequest, Observation, Organization, Patient, Practitioner, Procedure — but **no ICU depth**. **[spec]**
- **SMART Bulk Data Client (CLI)** — [github](https://github.com/smart-on-fhir/bulk-data-client), [docs](https://docs.smarthealthit.org/bulk-data-client/). The reference *client*. Run it against a server to see what a correct exchange looks like, then validate our own implementation's behavior against it. **[spec]**
- **SMART Launcher** — [launch.smarthealthit.org](https://launch.smarthealthit.org/). EHR-launch simulator for user-facing SMART apps, not backend bulk. Largely out of scope for us. **[spec]**

**General-purpose public servers — for testing export *mechanics*, not auth.**

- **HAPI FHIR public** — [hapi.fhir.org](https://hapi.fhir.org), R4 base `https://hapi.fhir.org/baseR4` (also DSTU2/STU3/R5). Supports Bulk `$export` (Patient/Group/System, NDJSON). The public instance is **open / no-auth**, so it tests export mechanics but not authentication, over a shared, messy public data pool. HAPI shines **self-hosted**: enable bulk with `hapi.fhir.bulkdata.enabled=true`, `hapi.fhir.bulk_export_enabled=true`, `hapi.fhir.bulk_import_enabled=true`. This is our workhorse for loading MIMIC (below). **[spec]**
- **Firely Server public test** — [server.fire.ly](https://server.fire.ly), default R4. Bulk Data Export supports SMART on FHIR v2. A sandbox for testing/education; a private instance needs an eval license. **[spec]**
- HL7's canonical living list of public test servers: [confluence.hl7.org public test servers](https://confluence.hl7.org/spaces/FHIR/pages/35718859/Public+Test+Servers). **[spec]**

**Free-dev-tier platforms — production-like auth + bulk, self-hostable.**

- **Medplum** — [app.medplum.com](https://app.medplum.com) (free tier); [self-host](https://www.medplum.com/docs/self-hosting). R4, open-source, Postgres-backed. **Bulk FHIR 2.0.0 `$export`** (`GET /fhir/R4/Group/<id>/$export?_outputFormat=ndjson`, [docs](https://www.medplum.com/docs/api/fhir/operations/bulk-fhir)), full OAuth2 including `client_credentials`, and subscriptions. A strong second landing spot for MIMIC when we want auth *and* bulk together. **[spec]**
- **Aidbox (Health Samurai)** — free **dev license** since ~2024 (~5 GB). [Bulk API export](https://www.health-samurai.io/docs/aidbox/api/bulk-api/export): `$export` at patient/group/system levels, async status URL, NDJSON; SMART/OAuth; Docker self-host. **[spec]**
- **Smile CDR** — the commercial build of HAPI, with `$export` + SMART backend. **Not free**; noted only so we know the paid upgrade path exists. **[community]**

**Cloud managed FHIR — metered, BAA required before any real PHI.**

- **Google Cloud Healthcare API FHIR store** — [cloud.google.com/healthcare-api](https://cloud.google.com/healthcare-api). R4, bulk import/export to/from GCS. No permanent free tier; $300 new-customer credit. **[spec]**
- **Azure Health Data Services (FHIR service)** — R4, native `$export` to Azure Storage plus `$import`. Auth via Entra `client_credentials` — the **closest cloud analogue to SMART backend auth**. Metered. **[spec]**
- **AWS HealthLake** — R4, `$export`/`$import` via S3 NDJSON. Auth via **IAM/SigV4, not SMART**. No free tier; benchmarks report slower exports. **[community]**

**Vendor EHR sandboxes — for rehearsing vendor-specific quirks last.**

- **Epic sandbox** — [fhir.epic.com](https://fhir.epic.com/). R4. Backend Systems apps and **Group-level `$export`** are supported, but **Epic must give you the Group ID**. This is the only way to rehearse Epic's PEM/JWKS key registration and app-download choreography before a live hospital. New sandbox keys take ~60 min to propagate **[community]**. See [doc 08](08-what-hospitals-actually-provide.md) for what Epic actually exposes.
- **Oracle Health / Cerner** — open (no-auth) `https://fhir-open.cerner.com/r4/{tenant}/...`; secured code sandbox `https://fhir-ehr-code.cerner.com/r4/{tenant}/...`. System apps and `system/` scopes are set up via the [code Console](https://code-console.cerner.com/console). URLs are tenant-scoped; `cernerdemo` is the sandbox tenant. **[spec]**

**Dead / moved — do not target.**

- **Logica Sandbox** (`sandbox.logicahealth.org`) — **RETIRED Nov 1, 2024** ([EOL notice](https://www.logicahealth.org/logica-sandbox-end-of-life-and-migration/)). Successor is **MELD** at [meld.interop.community](https://meld.interop.community) (Interop.Community), plus **Meld Community Edition**, an Apache-2.0 self-hostable fork of Logica via HL7 FHIR Foundry. **[spec]**

**Synthetic data — good for volume and format, useless for ICU semantics.**

- **Synthea** — [site](https://synthetichealth.github.io/synthea/), [overview](https://mitre.github.io/fhir-for-research/modules/synthea-overview). `./run_synthea -p 1000` produces 1000 synthetic patients as FHIR R4 bundles (default) or CSV (`exporter.csv.export=true`), with Observations/Procedures/MedicationRequests/CarePlans coded in RxNorm/LOINC/SNOMED. Docker helper: [conceptant/synthea-fhir](https://github.com/conceptant/synthea-fhir). **CRITICAL CAVEAT: Synthea is not ICU-grade.** It does not model high-frequency chartevents, ventilator settings, titrated drips, or capacity constraints — it explicitly ignores resource limits, so its outputs are an *upper bound*. Use it to stress-test volume, NDJSON parsing, and pipeline throughput; do not use it to validate ICU meaning. **[spec]**
- **SyntheticMass** — MITRE-hosted Synthea dataset; same limitations. **[community]**

## Real ICU data as FHIR: MIMIC-IV (the headline finding)

This is the asset that changes our test strategy, so it gets its own section.

- **MIMIC-IV on FHIR v2.1** — [physionet.org/content/mimic-iv-fhir/2.1](https://physionet.org/content/mimic-iv-fhir/2.1/). FHIR R4, aligned to US Core v4.0.0 where possible, with ~30 custom profiles (22 core + 8 ED). Built from >60,000 patients across MIMIC-IV v2.2 + MIMIC-IV-ED v2.2 ([JAMIA 2023 paper](https://academic.oup.com/jamia/article/30/4/718/6998091)). The **ICU-relevant profiles are exactly the Tier B/C content Epic bulk export will not give us**: `MimicEncounterICU`, `MimicObservationChartevents` ("the majority of documented ICU information" — ICU-cadence vitals/observations), `MimicObservationDatetimeevents`, `MimicObservationOutputevents`, **`MimicMedicationAdministrationICU`** (from `inputevents` — drips/infusions), `MimicProcedureICU`, plus labs, microbiology, meds, and ED vitals. **[spec]**
  - *Access (full set):* create a PhysioNet account → become a **credentialed** user → complete **CITI "Data or Specimens Only Research"** training and upload the completion report → sign the PhysioNet Credentialed Health Data DUA (License 1.5.0). [Required training](https://www.physionet.org/content/mimic-iv-fhir/view-required-training/2.1/). **[spec]**
- **MIMIC-IV Clinical Database Demo on FHIR v2.1.0** — [physionet.org/content/mimic-iv-fhir-demo/2.1.0](https://physionet.org/content/mimic-iv-fhir-demo/2.1.0/). **~100 patients, OPEN license, NO DUA, NO CITI.** This is **the single most valuable dev asset for this project**: genuinely ICU-shaped FHIR data, free, instantly available, containing exactly the chartevents and ICU medication administrations that Epic will withhold. Ideal for CI — commit it to the test harness and run the mapper against it on every change. **[spec]**
- **Loading MIMIC into a bulk-capable server** (so we can exercise `$export`, not just read files): **[kind-lab/mimic-fhir](https://github.com/kind-lab/mimic-fhir)** generates resources in Postgres, validates them in HAPI, and exports NDJSON; **[ahagh/mimic-on-hapi-fhir](https://github.com/ahagh/mimic-on-hapi-fhir)**; and a two-step [walkthrough](https://nuchange.ca/2024/11/loading-mimic-dataset-onto-a-fhir-server-in-two-easy-steps.html). **[community]**
- **The test oracle.** There is already a MIMIC→CLIF ground truth: **[MIMIC-IV-Ext-CLIF](https://physionet.org/content/mimic-iv-ext-clif/1.1.0/)** on PhysioNet and the consortium's **[CLIF-MIMIC ETL](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC)**. This is the gift: we can run MIMIC-on-FHIR through *our* pipeline and **diff our CLIF output against the published CLIF-MIMIC output**, turning "does it run?" into "does it produce the correct CLIF?" No other data source in this doc gives us a known-correct answer to check against. **[spec]**

## Edge cases and how it breaks

- **Open public servers test the wrong thing.** HAPI-public and Firely-public are no-auth, so passing there proves nothing about the SMART Backend Services handshake — the part most likely to fail against Epic. Auth *must* be proven on the SMART Bulk Data Server or a self-hosted instance with OAuth enabled. **[unverified]**
- **Synthea green-lights a broken ICU mapper.** Because Synthea emits no chartevents, ventilator settings, or titrated drips, a pipeline can pass every Synthea test and still be blind to Tier B/C. Synthea validates *plumbing*; only MIMIC validates *ICU meaning*. **[spec]**
- **Shared public pools are non-deterministic.** hapi.fhir.org and server.fire.ly hold data anyone can write and delete, so record counts drift between runs — never assert on exact counts against them. Pin CI to the fixed 100-patient MIMIC demo instead. **[community]**
- **The demo subset is small.** ~100 patients is right for correctness and CI, but not for throughput/pagination/throttling behavior; use Synthea at volume, or the full credentialed MIMIC, for scale tests. **[unverified]**
- **Profile / US Core version skew.** MIMIC-on-FHIR targets US Core **v4.0.0**; a real hospital may be on a later US Core, and MIMIC uses ~30 *custom* Mimic\* profiles a live Epic will not emit. Passing validation against MIMIC does not guarantee validation against production profiles — treat MIMIC as a semantic oracle, not a conformance oracle. **[unverified]**
- **Oracle differs from Epic differ from the reference.** Group-only vs. Patient-list vs. System export, `_since` supported or not, retention windows — these diverge per vendor (see [doc 05](05-bulk-export-deep-dive.md)). A behavior that works on the SMART reference server can still break on Epic; the Epic-sandbox rehearsal step is not optional. **[community]**
- **Cloud managed stores need a BAA before real PHI.** Google/Azure/AWS are fine for synthetic and MIMIC (already de-identified), but the moment real hospital data is involved a Business Associate Agreement and metered billing apply — a procurement gate, not just a config flag. **[spec]**
- **Epic sandbox friction.** New keys can take ~60 min to propagate and you cannot self-serve a Group ID — Epic provisions it. Budget calendar time, not just engineering time, for this step. **[community]**

## Comparison table

| Server | FHIR version | `$export`? | SMART Backend auth? | ICU-realistic data? | Self-host Docker? | Free? | Notes |
|---|---|---|---|---|---|---|---|
| SMART Bulk Data Server | R4 | Yes (System/Group/Patient) | **Yes (full, paste public key)** | No (~100 Synthea) | Yes (source on GitHub) | Yes | THE reference for auth + async end to end |
| SMART Bulk Data Client (CLI) | R4 (client) | n/a (client) | Yes (as client) | n/a | Yes | Yes | Reference *client*; validate our impl against it |
| SMART Launcher | R4 | No | Launch, not backend | No | Yes | Yes | EHR-launch sim for user-facing apps; mostly out of scope |
| HAPI FHIR public | R4 (+DSTU2/STU3/R5) | Yes (NDJSON) | No (open, no-auth) | No (messy shared pool) | Yes (shines here) | Yes | Tests export mechanics only; workhorse when self-hosted |
| Firely Server public test | R4 default | Yes (SMART on FHIR v2) | Partial (SMARTv2) | No | Yes (eval license) | Yes (sandbox) | Private instance needs eval license |
| Medplum | R4 | Yes (Bulk FHIR 2.0.0) | **Yes (incl. client_credentials)** | No (until we load MIMIC) | Yes (open-source, Postgres) | Yes (free tier) | Production-like auth + bulk; subscriptions too |
| Aidbox | R4 | Yes (patient/group/system) | Yes (SMART/OAuth) | No (until loaded) | Yes | Yes (dev license ~5GB) | Free dev license since ~2024 |
| Smile CDR | R4 | Yes | Yes | No | Yes | **No** (commercial HAPI) | Paid upgrade path |
| Google Cloud Healthcare API | R4 | Yes (import/export via GCS) | Via GCP IAM | No (until loaded) | No (managed) | No ($300 credit) | BAA for real PHI |
| Azure Health Data Services | R4 | Yes (to Azure Storage) + `$import` | Entra `client_credentials` (closest cloud analogue) | No (until loaded) | No (managed) | No (metered) | BAA for real PHI |
| AWS HealthLake | R4 | Yes (`$export`/`$import` via S3) | **No — IAM/SigV4, not SMART** | No (until loaded) | No (managed) | No (metered) | Slower exports in benchmarks **[community]** |
| Epic sandbox | R4 | Yes (Group-level; Epic gives Group ID) | Yes (Backend Systems, PEM/JWKS) | Thin sandbox data | No | Yes | Rehearse vendor auth; ~60min key propagation **[community]** |
| Oracle Health / Cerner | R4 | Yes | Yes (`system/` scopes via code Console) | Thin sandbox data | No | Yes | Tenant-scoped URLs; `cernerdemo` tenant |
| Logica Sandbox | — | — | — | — | — | **RETIRED 2024-11-01** | Do not target; successor MELD / Meld CE |
| MELD (Interop.Community) | R4 | Varies | Yes (SMART) | No | Meld CE (Apache-2.0) yes | Yes | Logica successor |
| Synthea (generator) | R4 | n/a (generator) | n/a | **No — not ICU-grade** | Yes (Docker helper) | Yes | Volume/format testing only; upper-bound outputs |
| MIMIC-IV on FHIR v2.1 (full) | R4 (US Core 4.0.0) | Once loaded into a server | Depends on host server | **YES (real de-id ICU)** | Yes (via kind-lab/ahagh loaders) | Yes (credentialed: CITI + DUA) | >60k patients; ~30 Mimic\* profiles |
| **MIMIC-IV-on-FHIR demo v2.1.0** | R4 (US Core 4.0.0) | Once loaded | Depends on host | **YES (~100 real ICU pts)** | Yes | **Yes — OPEN, no DUA/CITI** | **Best CI asset; instantly available** |

## Recommended ladder (the conclusion)

Climb these in order. Each rung isolates one failure mode so we never debug two at once, and never meet a new one first against a live hospital.

1. **SMART Bulk Data Server** — prove the **auth + async mechanics** in isolation: paste a public key, get a `client_id`, kick off `$export`, poll, download NDJSON. Cross-check every step against the reference **`bulk-data-client` CLI** so we know our client behaves correctly before data semantics enter the picture.
2. **MIMIC-IV-on-FHIR demo on self-hosted HAPI or Medplum** — prove the **mapper against real ICU semantics** (chartevents, ICU medication administrations, ICU encounters — the Tier B/C content Epic will not export). Use **MIMIC-IV-Ext-CLIF / CLIF-MIMIC as the ground-truth oracle**: diff our FHIR→CLIF output against the published CLIF-MIMIC to confirm we produce the *right* answer, not just a running one. The open demo subset makes this a permanent, no-paperwork CI fixture.
3. **Epic sandbox** — rehearse the **vendor-specific auth and Group `$export`** (PEM/JWKS registration, Epic-provisioned Group ID, key propagation delay, one-export-per-day throttling) before go-live, so the only surprises left at the real hospital are the ones no sandbox can reproduce.

See [doc 05](05-bulk-export-deep-dive.md) for the export mechanics each rung exercises, [doc 06](06-authentication.md) for the auth handshake, [doc 08](08-what-hospitals-actually-provide.md) for why MIMIC has to fill the ICU-depth gap, and [doc 17](17-open-questions-and-roadmap.md) for the open questions this ladder does not close.

## What we still don't know

- **Whether MIMIC-IV-on-FHIR's profiles survive a round trip through a stock HAPI or Medplum bulk export.** The Mimic\* profiles are custom, and `MimicObservationChartevents` and `MimicMedicationAdministrationICU` are exactly the resources we most need to test. If a stock server's `$export` drops or mangles them, the Phase 1 oracle is weaker than it looks. **Resolve:** load the 100-patient demo, run `Group/$export`, and count resources in against resources out, per profile.

- **How closely the published CLIF-MIMIC output can actually be diffed against ours.** It is built from MIMIC's native tables, not from MIMIC-on-FHIR. Two independent conversions of the same underlying data should agree, but they may differ on encounter boundaries, timezone handling, and unit conventions in ways that are legitimate rather than erroneous. **Resolve:** diff on a single table (`labs`) first and characterize the disagreements before trusting the oracle broadly.

- **Whether the SMART Bulk Data Server accepts ES384 keys, or only RSA.** Its public form asks for a public key and the documentation emphasizes RSA. If it is RSA-only, we will not exercise the ES384 path there. **[unverified]**

- **Whether Epic's sandbox Group `$export` behaves like a production Group `$export`** — specifically whether the ~24 h throttle, the missing `_since`, and the ~20-type resource list all reproduce in the sandbox. If the sandbox is more permissive, it will teach us the wrong lessons. **[unverified]**

- **Whether Medplum's or Aidbox's free tiers permit enough storage for a realistic single-site event volume.** Aidbox's dev license is ~5 GB. That is comfortable for the MIMIC demo and probably not for a year of one hospital. Matters only if we intend to use them beyond development.

- **Whether any of the free servers can reproduce Epic's specific quirks** — hidden `hasMember` Observations, missing `meta.lastUpdated`, ids over 64 characters. Almost certainly not, which means [doc 15](15-edge-cases.md)'s Epic-specific traps cannot be regression-tested anywhere except against Epic.
