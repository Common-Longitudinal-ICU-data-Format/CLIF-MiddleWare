# Open questions and roadmap: the things we're guessing, and the cheapest experiment for each

*Everything below is a guess dressed as a plan; the job of the roadmap is to turn each guess into a fact — and the two facts that decide the whole project, "can we get drip rates" and "can we get vent settings," can both be answered in an afternoon against the Epic sandbox.*

## In plain words

This document is a list of things we are currently **guessing** about, and for each one, the cheapest experiment that turns the guess into a fact. That framing matters: right now our architecture rests on assumptions about what a real Epic instance emits, and every one of those assumptions is a place the project can quietly fail after we have written a lot of code.

Two of those guesses matter more than all the others combined:

1. Can we get **continuous infusion drip rates** out of FHIR? (Without them, no vasopressor exposure, no `medication_admin_continuous`.)
2. Can we get **ventilator settings** out of FHIR? (Without them, no respiratory support, no SOFA respiratory subscore.)

Both can be answered in an **afternoon** against the free Epic sandbox — a couple of `GET` requests against a test ICU patient. Doing that **before** we write production code is the single highest-leverage hour on this project. If both answers are "no," we are not building a CLIF warehouse over FHIR; we are building a patient-finding index, and we should learn that for the price of an afternoon rather than a quarter.

So: guesses first, experiments to kill them second, and a build order that refuses to pour concrete until the load-bearing guesses are facts.

## Technical detail

Each open question below carries: the question, why it is load-bearing, exactly how to answer it, and what we do if the answer is "no."

**Q1 — Does the target Epic instance's Bulk `$export` really emit only ~20 US Core resource types?**
*Load-bearing:* the whole scope of what CLIF tables are even reachable depends on which resources come out. The 20-type list is **[community]**-reported from [smart-on-fhir/cumulus discussion #5](https://github.com/smart-on-fhir/cumulus/discussions/5), not Epic-published, so we are planning against a forum post.
*How to answer:* kick off a probe Group `$export` with **no** `_type` parameter and enumerate the resource types in the manifest `output[]`. See [doc 05](05-bulk-export-deep-dive.md) for the mechanics.
*If "no" (fewer types than hoped):* the enrichment leg gets more expensive and some CLIF tables move out of reach. Conversely, if `MedicationAdministration` appears in that manifest, the whole medication-enrichment leg gets cheaper.

**Q2 — Does Epic populate `MedicationAdministration.dosage.rateQuantity` (or `rateRatio`) for continuous infusions, with clean `effectivePeriod` start/stop?**
*Load-bearing:* this single fact determines whether CLIF `medication_admin_continuous` is reachable via FHIR **at all**. No rate, no vasopressor exposure. **[unverified]**.
*How to answer:* `GET MedicationAdministration?patient=<test ICU patient>` on the Epic sandbox **and** on the site's non-prod instance; inspect `dosage.rate[x]` and `effective[x]`. See [doc 03](03-fhir-resource-types.md).
*If "no":* `medication_admin_continuous` needs HL7 v2 or Clarity (`MAR_ADMIN_INFO`), not FHIR. This is one of the two project-deciding questions.

**Q3 — Has the site LOINC-mapped its ventilator / RASS / GCS / CRRT / ECMO flowsheet rows, and are they queryable as FHIR Observations?**
*Load-bearing:* the ICU physiologic core of CLIF lives in these rows. Research found **no public example of any Epic site exposing vent / CRRT / ECMO flowsheets as queryable FHIR Observations** — absence of evidence, not evidence of absence, but consistent with how Epic flowsheets work.
*How to answer:* `Observation` search with `category=` variants and known LOINCs (RASS ≈ `71387-8`, GCS total ≈ `9269-2`) against a test patient; **and** ask the site's Epic analyst whether Doc Flowsheet Builder concept mapping has been done for those rows. The human question is as important as the API call.
*If "no":* `respiratory_support`, `crrt_therapy`, `ecmo_mcs`, and `position` are out of FHIR reach entirely. This is the second project-deciding question.

**Q4 — Will the hospital create a Group we can actually use?**
*Load-bearing:* Epic Groups are built by a human analyst, not via API. Our whole cohort model assumes a usable Group. **[unverified]**.
*How to answer:* ask the site — can they express "all inpatient + observation encounters, rolling 90 days"? Is there precedent for an ICU-unit Group? See [doc 08](08-what-hospitals-actually-provide.md).
*If "no" (only a narrow Group):* our client-side cohort filter becomes moot, and reproducibility suffers because the cohort definition now lives in an analyst's Group build we can't see or version.

**Q5 — What is the site's actual bulk kickoff throttle window?**
*Load-bearing:* default is roughly 24h, server-tunable **[community]**. It sets the floor on how fresh the warehouse can be and whether incremental export is viable.
*How to answer:* ask the site's Epic team the configured throttle for our client, and observe `429` / retry-after behavior on the non-prod instance.
*If "no" (can't be widened):* the incremental strategy changes materially — we design around a fixed daily-ish cadence rather than on-demand refresh. See [doc 05](05-bulk-export-deep-dive.md).

**Q6 — Does the site's Epic honor granular SMART v2 scopes (`system/Observation.rs?category=laboratory`)?**
*Load-bearing:* the HIPAA minimum-necessary story (see [doc 16](16-security-and-governance.md)) leans on being able to request narrow categories rather than all Observations. **[unverified]**.
*How to answer:* request the granular scope during the sandbox backend-auth flow and inspect the granted scopes in the token response.
*If "no":* we request broader scopes and lean harder on `_typeFilter` and post-download filtering to satisfy minimum-necessary, and we document that the narrowing happens client-side.

**Q7 — Is `Encounter.location[]` populated with full `period` history, or collapsed to current location?**
*Load-bearing:* this single fact decides whether CLIF's `adt` (admit/discharge/transfer) table is buildable from FHIR at all. **[community]** says it varies by site.
*How to answer:* `GET Encounter?patient=<test patient>` and inspect whether `location[]` carries multiple entries with distinct `period` start/stop, or a single current location.
*If "no" (collapsed):* the `adt` transfer timeline can't be reconstructed from FHIR and needs Clarity (`ADT` / `CLARITY_ADT`) or HL7 v2 ADT feeds.

**Q8 — Is there an OMOP instance at the site already?**
*Load-bearing:* if the site already maintains OMOP, FHIR→CLIF may be the wrong pipe entirely — an OMOP→CLIF path could be shorter and better-supported.
*How to answer:* ask. It's an org question, not an API call.
*If "yes":* re-evaluate the whole pipeline choice before building.

**Q9 — The uncomfortable one: should this be Clarity/Caboodle SQL instead of FHIR?**
*Load-bearing:* it questions the project's core technical premise. The CLIF consortium ships **prebuilt SQL for Epic Caboodle/Clarity** ([clif-icu.com/tools](https://clif-icu.com/tools)), and the CLIF paper states CLIF "is currently not linked to established interoperability standards like HL7 FHIR" ([PMC11398431](https://pmc.ncbi.nlm.nih.gov/articles/PMC11398431/)). Clarity `IP_FLWSHT_MEAS` gives every flowsheet row at full temporal resolution with **no LOINC-mapping prerequisite**; `MAR_ADMIN_INFO` gives dose/rate/route/time directly.
*The honest trade:* **FHIR** buys portability across vendors and needs no database access or DBA — you talk to a standard API. **Clarity** buys completeness and fidelity but is Epic-specific, per-site, and requires warehouse credentials and a DBA relationship. See [doc 13](13-mapper-engine.md) for how the mapper differs between the two.
*Recommendation:* decide this **before** building, and consider the hybrid — FHIR for the portable backbone (demographics, labs, diagnoses, encounters), Clarity or HL7 v2 for the ICU physiologic core (vents, drips, CRRT, ADT).

## Edge cases and how it breaks

- **Q2 and Q3 come back "no" but only at the pilot site.** The sandbox says yes, the real site says no, and we've already built assuming yes. Mitigation: answer Q2/Q3 on the site's **non-prod** instance (Phase 3), not just the sandbox, before writing site-specific code — sandbox behavior is necessary but not sufficient evidence.
- **The probe export in Q1 is throttled.** We kick off the no-`_type` probe and hit the 24h window (Q5) before we get a clean manifest. Mitigation: sequence Q1 and Q5 together on first contact so the first successful export doubles as the throttle observation.
- **Q7 varies by encounter, not just by site.** Some encounters carry full `location[]` history, others are collapsed — so a single test patient gives a false "yes." Mitigation: sample several encounters, including transfers, before concluding `adt` is buildable.
- **OMOP exists but is stale (Q8).** The site has OMOP but it lags weeks behind clinical reality, so it's not actually a better pipe for a freshness-sensitive warehouse. Ask about refresh cadence, not just existence.
- **Granular scopes granted but silently down-scoped (Q6).** Epic accepts the scope request and returns a token, but the granted scopes are narrower or broader than asked. Mitigation: always diff requested vs granted scopes in the token response; never assume the request was honored.
- **The MIMIC oracle diverges for a benign reason.** In Phase 1 the mapper output disagrees with `MIMIC-IV-Ext-CLIF` because of a version or unit convention difference, not a real bug. Mitigation: pin the exact CLIF version and treat the first diff pass as calibration, not pass/fail.

## The roadmap

Each phase is gated on the questions above; do not start a phase until the prior phase's questions are answered.

**Phase 0 — Prove the protocol.** Against the [SMART Bulk Data Server](https://bulk-data.smarthealthit.org/): backend auth + kickoff + poll + manifest + NDJSON download + `error[]` handling + cancellation. Cross-check the implementation against the [reference bulk-data client](https://docs.smarthealthit.org/bulk-data-client/). No CLIF yet — this phase proves we can move bytes correctly.

**Phase 1 — Prove the semantics.** Load the [MIMIC-IV-on-FHIR demo](https://physionet.org/content/mimic-iv-fhir-demo/2.1.0/) (100 patients, open access, no DUA) into self-hosted HAPI or Medplum. Build `clif_events` + `clif_signals`, run the mapper, and **diff the output against the existing MIMIC→CLIF ground truth** ([MIMIC-IV-Ext-CLIF](https://physionet.org/content/mimic-iv-ext-clif/1.1.0/), [CLIF-MIMIC repo](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC)). This is the **single most valuable thing on the roadmap**: it gives us a real test oracle, turning "does it run" into "is it right." Everything before this is plumbing; this is where correctness becomes measurable.

**Phase 2 — Prove the vendor.** Epic sandbox: register a Backend Systems app, wire up PEM/JWKS, run a Group `$export`. Answer questions **1, 2, 3, 6** empirically. This is the afternoon that decides whether FHIR carries the ICU core.

**Phase 3 — Prove the site.** Non-production instance at one partner hospital. Answer questions **4, 5, 7**. IRB and BAA in writing **first** (see [doc 16](16-security-and-governance.md)) — no PHI, even non-prod, moves before the paperwork.

**Phase 4 — Decide.** With real answers in hand, choose the pipe: FHIR-only, FHIR+HL7v2, or FHIR+Clarity hybrid. **Only now write production code.** The point of Phases 0–3 is to make this decision on evidence rather than hope.

## What would make us abandon FHIR

If questions **2, 3, and 7** all come back negative at the pilot site, FHIR yields only demographics, labs, diagnoses, and intermittent vitals — no drip rates, no ventilator settings, no reconstructable transfer timeline. A warehouse built from that **cannot compute a SOFA respiratory subscore or vasopressor exposure**. That is not a CLIF warehouse; it is a cohort-discovery index — useful for finding patients, useless for ICU physiology.

To be plain: this would be a reason to **pivot the pipe, not the project.** The CLIF target stays; the source changes to Clarity or HL7 v2 for the ICU physiologic core, with FHIR retained only for the portable backbone. Learning this early — in the Phase 2 afternoon and the Phase 3 site probe — is exactly why the roadmap refuses to write production code until questions 2, 3, and 7 are facts.

## What we still don't know

- Whether the sandbox faithfully predicts a real site's behavior for Q2/Q3 — sandbox data is synthetic and may be populated more completely (or less) than production flowsheets.
- Whether any single partner site will grant non-prod access on a timeline that lets Phase 3 happen before organizational patience runs out.
- Whether the MIMIC→CLIF ground truth is complete enough to serve as an oracle for the CLIF tables we most care about, or whether it too is thin exactly where FHIR is thin (vents, drips).
- The real cost/effort of a Clarity path if we pivot — it needs a DBA relationship and warehouse credentials per site, and we have not scoped that.
- Whether a hybrid pipe (FHIR backbone + Clarity core) can be kept temporally consistent, or whether joining two sources with different refresh cadences reintroduces the freshness problems from [doc 09](09-data-freshness.md).
