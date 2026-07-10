# Edge cases

*Every trap we found, with a worked example and a mitigation. Most of these fail silently, which is why they get their own document.*

---

## In plain words

A bug that crashes is a gift. It tells you something is wrong, it tells you where, and it refuses to produce a wrong answer.

Almost none of the problems in this document crash. They produce a dataset that loads cleanly, validates, looks plausible, and is wrong. A blood pressure column that is quietly empty. A creatinine value that is off by a factor of 88 because the units changed. A cohort that is missing every patient whose lab results arrived as part of a panel. A "real-time" feed that stopped updating six weeks ago because the server never told anyone.

Clinical data is unusually good at this. The values are all plausible-looking numbers. Nobody notices that the ventilator table has 400 rows instead of 40,000 until a reviewer asks why the median duration of mechanical ventilation is eleven minutes.

So this document is a list of the specific ways this pipeline will lie to us, grouped by where the lie originates. Each entry states the trap, shows what it looks like concretely, and says what to do about it. The ones marked **silent** produce no error of any kind.

If you read only one section, read the first two. Blood pressure and the missed-bump trap will each, on their own, invalidate a study.

---

## Technical detail

### Clinical data shape

---

**Blood pressure is a panel, not a value.** — *silent*

LOINC `85354-9` ("Blood pressure panel with all children optional") has **no top-level value**. The systolic reading is a component coded `8480-6`; the diastolic is `8462-4`; mean arterial pressure, when present, is `8478-0`. **[spec]** — [HL7 R4 BP example](https://www.hl7.org/fhir/R4/observation-example-bloodpressure.html), [US Core BP profile](http://hl7.org/fhir/us/core/STU1/Observation-blood-pressure.html)

CLIF wants **two separate rows**: `vital_category = 'sbp'` and `vital_category = 'dbp'`, each with its own `vital_value`.

```jsonc
{ "resourceType": "Observation",
  "code": { "coding": [{ "system": "http://loinc.org", "code": "85354-9" }] },
  // NOTE: no valueQuantity here
  "component": [
    { "code": {"coding":[{"code":"8480-6"}]}, "valueQuantity": { "value": 128, "unit": "mm[Hg]" } },
    { "code": {"coding":[{"code":"8462-4"}]}, "valueQuantity": { "value": 74,  "unit": "mm[Hg]" } }
  ]}
```

A flattener that reads `Observation.value[x]` and skips resources where it is absent will **drop every blood pressure in the dataset** and raise nothing. The `vitals` table will simply contain no `sbp` rows, and a `SELECT COUNT(*)` looks fine because heart rate and SpO₂ are there.

Worse: searching on `85354-9` returns the panel; searching on `8480-6` does not (you need `combo-code`). So a search-based extractor and a bulk-based extractor disagree about what exists.

*Mitigation:* define the signal grain on the **component** code, not the panel code, and give `signal_map` a `transform` column whose value here is `split_bp_components`. Assert non-zero `sbp` and `dbp` counts in the coverage report. See [doc 12](12-clif-events-and-signals.md).

---

**`Observation.value[x]` is polymorphic; CLIF's `vital_value` is one DOUBLE.** — *partially silent*

FHIR permits `valueQuantity`, `valueCodeableConcept`, `valueString`, `valueBoolean`, `valueInteger`, `valueRatio`, `valueSampledData`, `valuePeriod`, `valueTime`, `valueDateTime`, or nothing (with `component[]` carrying it). **[spec]**

What breaks:
- `valueCodeableConcept` — "positive", "no growth", an organism, a blood type. Not a number.
- `valueString` — "hemolyzed", "specimen clotted".
- `valueRatio` — microbiology titers, `1:64`.
- `valueSampledData` — a waveform snippet.
- absent + `component[]` — panels.

CLIF's own answer is instructive: `patient_assessments` carries **three value columns** side by side, `numerical_value`, `categorical_value`, and `text_value`. And `labs` carries both `lab_value` (VARCHAR, raw) and `lab_value_numeric` (DOUBLE, parsed).

*Mitigation:* follow CLIF's precedent. In `clif_events`, keep `value_numeric`, `value_string`, `value_code_system`, `value_code`, `value_boolean`. **Always populate `value_string` with the raw value.** Route to the CLIF column that fits. Never coerce a coded result into a number, and never discard the raw string to obtain one. Count and report every event whose value kind has no destination.

---

**Lab values are frequently not numbers.** — *silent if coerced*

`"<0.01"`, `">1000"`, `"POSITIVE"`, `"1:64"`, `"TNP"` (test not performed), `"HEMOLYZED"`. A naive `float()` either throws (good) or, worse, a "tolerant" parser strips the `<` and records `0.01` as if it were a measurement.

A censored value (`<0.01`) is *not* the same as `0.01`. For lactate or troponin this distinction changes the clinical interpretation.

*Mitigation:* `lab_value` (VARCHAR) always gets the raw string. `lab_value_numeric` gets a value only on clean parse. Record censoring separately if it matters to the analysis. Never let a coercion succeed silently.

---

**`respiratory_support` is wide; FHIR is long.** — *design mismatch*

One CLIF `respiratory_support` row is a snapshot of the entire ventilator: `fio2_set`, `peep_set`, `tidal_volume_set`, `resp_rate_set`, `mode_category`, `device_category`, and twelve more fields, all on one row at one `recorded_dttm`.

FHIR would deliver each of those as a separate `Observation` at a slightly different timestamp — FiO₂ charted at 08:00:03, PEEP at 08:00:11, mode at 08:01:47.

Pivoting long into wide requires a **time-bucketing rule**, and that rule is a clinical decision, not an engineering one. Bucket to the minute? To the hour? Forward-fill within a device/mode block?

*Mitigation:* clifpy already ships `RespiratorySupport.waterfall(bfill=False)`, which builds an hourly scaffold and forward-fills `fio2`/`peep`/`tidal_volume` within device and mode blocks (and helpfully auto-scales `fio2` from 40 to 0.40). Use it rather than inventing a rule. But note it operates on an already-built CLIF table, so the long→wide pivot is still ours. Get a clinician to sign off on the bucketing rule and record it in metadata. See [doc 01](01-what-is-clif.md).

---

**Units lie, and UCUM cannot catch it.** — *silent*

UCUM validates that a unit string is *well-formed*. It cannot tell you the unit is *true*.

A signal whose display is `"Temp"` and whose unit is `"Cel"` but whose median value is 98.6 is Fahrenheit. A norepinephrine infusion labelled `mcg/kg/min` with values clustering at 400 is not in `mcg/kg/min`.

Watch specifically for: creatinine mg/dL vs µmol/L (factor 88.4); hemoglobin g/dL vs g/L (factor 10); FiO₂ as `40` vs `0.40`; temperature C vs F; weight kg vs lb.

*Mitigation:* store `numeric_p01/p50/p99` on every `clif_signals` row and show the distribution to the reviewer. **The distribution is the lie detector; the unit field is the suspect.** Use [UCUM-LHC](https://github.com/LHNCBC/ucum-lhc) or the [NLM UCUM service](https://ucum.nlm.nih.gov/ucum-service.html) for well-formedness and conversion, and clifpy's `standardize_dose_to_base_units()` for medication doses. Range-check against the schema's `vital_ranges`. See [doc 14](14-terminology-resources.md).

---

**A unit change forks a signal.** — *loud, but looks like a bug*

Signal identity includes the unit ([doc 12](12-clif-events-and-signals.md)). If the hospital switches creatinine reporting on a Tuesday, a new `signal_key` appears with status `unmapped`.

This is **correct** — a human must confirm the new unit — but the on-call engineer will read it as a regression. Document it, and make the console say "unit changed for an existing concept" rather than "unknown signal."

---

### Extraction and transport

---

**The missed-bump trap.** — *silent, and the most dangerous item in this document*

`_since` and `_lastUpdated` return a resource only if the server **bumped `meta.lastUpdated`** when it changed. Many real changes do not bump it: a linked resource updates, a flowsheet value is corrected, a status changes without touching the resource envelope.

So incremental loading **silently misses updates**. No error. No warning. Your warehouse quietly diverges from the source, and the divergence is invisible because you have nothing to compare against.

Epic makes it worse: it **does not support `_since` for bulk at all**, and **under-populates `meta.lastUpdated`** in bulk output, so it is not usable as a watermark even if you wanted to. **[community]** — [cumulus #5](https://github.com/smart-on-fhir/cumulus/discussions/5)

The clinically nastiest instance: an **amended lab result**. A potassium of 7.2 corrected to 4.2 the next morning. If the amendment does not bump `lastUpdated`, your warehouse keeps 7.2 forever.

*Mitigation:* use the manifest's `transactionTime` as the only watermark. Re-pull **overlapping** trailing windows (30–90 days) with `_typeFilter` date ranges. Upsert idempotently on `(id, versionId)`. Run periodic **full historical backfills** — accept that you cannot rely on deltas for correctness. See [doc 09](09-data-freshness.md).

---

**`Prefer: handling=lenient` converts a filter error into silent full data.** — *silent*

Send a `_typeFilter` the server does not support. With `handling=lenient`, the server **drops the parameter and returns everything**, with a `200 OK`.

You believe you exported inpatients from January. You exported everyone, forever. Nothing tells you.

*Mitigation:* prefer strict handling. Catch the `422 Unprocessable Entity` and fail loudly. If lenient is required for some other reason, **assert the returned population against an independent count** before trusting it. See [doc 05](05-bulk-export-deep-dive.md).

---

**`error[]` in the manifest is where the bad news lives.** — *silent if ignored*

The completion manifest has `output[]`, and it also has `error[]` — an array of `OperationOutcome` NDJSON files describing what the server would not or could not give you. "You lack scope for resource X." "Type Y is not supported."

Naive clients download `output[]` and never fetch `error[]`. The export "succeeded."

*Mitigation:* always fetch, parse, persist, and surface `error[]`. Treat a non-empty `error[]` as a first-class run outcome, not a footnote.

---

**Epic hides nested Observations.** — *silent*

Observations referenced by `DiagnosticReport.result` and members of `Observation.hasMember` are not returned by default, and **Epic supports neither `_include:iterate` nor `_revinclude`**. **[community]**

Export `_type=DiagnosticReport` and you get report envelopes with no values in them.

*Mitigation:* export `Observation` alongside `DiagnosticReport` and stitch by reference. On REST, add `&_include=Observation:hasMember:Observation`. Verify by counting: a `DiagnosticReport` with a `result[]` of length 8 should resolve to 8 Observations.

---

**Epic resource ids exceed FHIR's 64-character `id` limit.** — *loud, at the worst moment*

FHIR constrains `id` to 64 characters. Epic sometimes emits longer ones; there is a server-side toggle. **[community]**

A `VARCHAR(64)` primary key truncates or rejects, mid-load, on a dataset you cannot re-export for 24 hours.

*Mitigation:* do not length-cap `source_resource_id`.

---

**Access tokens expire mid-export.** — *loud, but only on large exports*

SMART recommends `expires_in` ≤ 300 seconds. **[spec]** A real bulk export takes far longer than five minutes to complete and download.

Code that fetches one token at startup passes every sandbox test (small dataset, fast export) and fails on the first real hospital.

*Mitigation:* re-mint on demand during polling and downloading. The `client_assertion` JWT is single-use — fresh `jti` every time. See [doc 06](06-authentication.md).

---

**Losing the `Content-Location` URL burns a day.** — *loud, expensive*

Crash between the server accepting the kickoff and us persisting the polling URL, and the export runs to completion somewhere we cannot reach. Under Epic's ~24-hour per-Group throttle **[community]**, that is a lost day.

*Mitigation:* persist the polling URL before returning from kickoff. Record the *intent* to kick off before issuing the request.

---

**Retrying into a throttle looks like an attack.** — *loud, and gets you disabled*

"Request not allowed: The Client requested this Group too recently." That is not a transient error. A retry loop generates hundreds of rejections a day and will get the integration switched off by hospital IT.

*Mitigation:* persistent cooldown state that survives restarts. Treat throttle responses as a scheduled wait, never as a retry.

---

**Export files expire.** — *silent until you need them*

Oracle documents **30 days**. **[spec]** Epic's retention is shorter and configuration-dependent. **[unverified]**

*Mitigation:* download and checksum immediately on manifest completion. Never architect around re-fetching.

---

**One resource type spans many NDJSON files, and `count` is optional.** — *silent*

`output[]` may contain twelve `Observation` entries. Code that assumes one file per type processes one twelfth of the data.

*Mitigation:* iterate all of `output[]`. Never key on `type`.

---

### Cohort and identity

---

**OBSENC is not inpatient.** — *definitional*

`Encounter.class` `OBSENC` means observation status: the patient occupies a bed and is not formally admitted. It is a billing and regulatory status, not a location. An observation-status patient can be lying in an ICU bed receiving care identical to the inpatient beside them. **[spec]** — [ActEncounterCode](https://terminology.hl7.org/5.1.0/ValueSet-encounter-class.html)

Including or excluding OBSENC changes the cohort materially, and published studies differ.

*Mitigation:* make the cohort rule an explicit, versioned config value that is written into warehouse metadata. Whatever you choose, a downstream analyst must be able to see what you chose. See [doc 10](10-cohort-and-encounter-class.md).

---

**Encounter class does not tell you ICU.** — *silent*

Neither `class` nor `type` carries unit-level granularity. ICU-ness lives in `Encounter.location[]` and the referenced `Location`. And **population of `location[]` with full `period` history varies by site** — some collapse it to current-location-only. **[community]**

If it is collapsed, **CLIF's entire `adt` table is unbuildable from FHIR**, and with it every ICU-stay definition.

*Mitigation:* verify `Encounter.location[].period` at the site before promising the `adt` table. If collapsed, the transfer history must come from HL7 v2 `ADT^A02`.

---

**FHIR Encounters and CLIF hospitalizations do not line up.** — *silent*

Epic commonly emits a separate ED `Encounter` and inpatient `Encounter` for one continuous stay. CLIF wants one `hospitalization`.

*Mitigation:* clifpy's `stitch_encounters(..., time_interval=6)` exists precisely for this, and CLIF's schema has `hospitalization_joined_id`. Use them. But note that stitching is heuristic — a 6-hour window is a choice.

---

**An Encounter with no `period.end` is not an error.** — *definitional*

The patient is still admitted. CLIF has `discharge_category = "Still Admitted"` for exactly this.

---

**`code_status` is keyed on `patient_id`, not `hospitalization_id`.** — *silent, clinically serious*

A patient's DNR status crosses admissions. Attributing a code status to the wrong hospitalization is not a data-quality nit.

---

**`microbiology_susceptibility` has no `hospitalization_id`.** — *silent, unrecoverable*

Its only link to the rest of the warehouse is `organism_id` → `microbiology_culture`. An ETL that fails to mint stable `organism_id` values orphans the table permanently, and no amount of downstream cleverness recovers it.

---

**Group membership is evaluated at export time.** — *silent*

A Group defined as "currently admitted patients" contains a different population every night. Consecutive exports are not comparable, and the cohort is irreproducible.

*Mitigation:* ask the hospital analyst for a **stable, rule-based Group with an explicit time window**, and store the Group's definition text in warehouse metadata.

---

### Mapping

---

**A ranker given a list will always pick from the list.** — *silent*

Offer a model `["norepinephrine", "epinephrine", "dopamine"]` for a signal that is actually a saline flush, and it returns one of the three with a fluent rationale.

*Mitigation:* always include an explicit **"none of these"** candidate and instruct the model to prefer it under uncertainty. Without it, confidence scores are meaningless. See [doc 13](13-mapper-engine.md).

---

**Near-synonyms are where the model fails and the clinician does not.** — *silent*

Dopamine / dobutamine. Norepinephrine / epinephrine. Hydromorphone / hydrocodone. Sodium bicarbonate / sodium chloride.

Reported LLM Top-1 accuracy on terminology mapping is around **85%** — roughly one in seven wrong on first guess — while Top-5 reaches ~98%. That profile is a good candidate generator and a disastrous autonomous decider.

*Mitigation:* every vasopressor, sedative, and paralytic mapping gets individual clinician review **regardless of event volume**. Do not let a low `event_count` route a norepinephrine mapping past a human.

---

**Confidence scores are not probabilities.** — *silent*

LLM-stated confidence correlates with fluency, not correctness. Use it to order the review queue. **There must be no auto-approve threshold.**

---

**Reviewer fatigue launders an AI decision as a clinical one.** — *silent, and the worst outcome in this document*

A clinician clicking "approve" 400 times in ten minutes has not reviewed anything, but the audit log now says a physician approved it.

*Mitigation:* instrument median time-per-signal. Alert if it collapses. Group similar signals so that one considered decision covers many, and record it honestly as one decision applied to many, not as many decisions.

---

**Display strings can contain PHI.** — *silent, and a reportable incident*

Real flowsheet and lab display names sometimes contain identifiers or free-text notes. If the mapper uses a hosted model, `display_raw` and `example_values` leave the hospital.

*Mitigation:* a scrub-and-review gate before any egress. Support a fully local ranker and a null ranker. See [doc 16](16-security-and-governance.md).

---

**Cross-site map reuse is safe for codes and unsafe for strings.** — *silent*

`LOINC 2823-3 → potassium` is portable. `"K, SER" → potassium` is not — another hospital's `"K"` may be a different assay on a different specimen.

*Mitigation:* store code-keyed and string-keyed mappings separately. Only ship the code-keyed portion between sites.

---

### Time

---

**Timezone is applied globally by clifpy, once, at load.** — *silent*

No CLIF schema carries per-column timezone metadata. clifpy localizes **all** `DATETIME` columns to the single `timezone=` value. FHIR `instant` carries a UTC offset.

*Mitigation:* store UTC in `clif_events`. Convert exactly once, at parquet-write time. Two conversions produce a one-hour shift that appears twice a year and is nearly impossible to find later.

---

**`transactionTime`, not wall-clock, is the watermark.** — *silent*

The manifest's `transactionTime` is the server's snapshot instant. Your server's clock is not.

---

**Clock skew breaks authentication.** — *loud, unhelpfully*

The `client_assertion` carries `exp` (≤5 minutes). A drifting daemon clock produces authentication failures with opaque errors. Require NTP; alert on drift.

---

**Labs have three timestamps and they are not interchangeable.** — *silent*

CLIF's `labs` table has `lab_order_dttm`, `lab_collect_dttm`, and `lab_result_dttm`, all required. A lactate *collected* at 03:00 and *resulted* at 05:30 is a different clinical fact depending on which time you use. Sepsis bundle compliance is measured against collection time; a trajectory analysis wants collection time; an availability analysis wants result time.

*Mitigation:* populate all three. Never substitute one for another to fill a required column.

---

### Vocabulary

---

**mCIDE ships known typos.** — *loud, at the wrong moment*

`mCIDE/postion/` (not `position`). `clif_patient_ethinicity_categories.csv` (not `ethnicity`). Fluid category `stomatch` in `microbiology_nonculture` versus `stomach` in `microbiology_culture` — **the same anatomical site, spelled two ways in two tables.** A shared lookup keyed on the correct spelling fails on one of them.

---

**`med_group` is tagged as a category column in one table and a group column in another.** — *silent*

In `medication_admin_continuous`, `med_group` is marked `is_category_column: true`. In `medication_admin_intermittent` it is a group column. Validation behavior may differ. Do not assume symmetry.

---

**Nearly every numeric column in `respiratory_support`, `crrt_therapy`, and `ecmo_mcs` is `required: true`.** — *loud*

All seventeen ventilator fields. A source that provides `peep_set` but not `inspiratory_time_set` will not validate cleanly.

*Mitigation:* decide early whether to emit nulls and accept warnings, or omit rows. Test this against `clifpy.validate()` before designing around either answer. See [doc 01](01-what-is-clif.md).

---

## What we still don't know

- **Whether `clifpy.validate()` treats nulls in `required: true` numeric columns as errors or warnings.** Decides whether a partially-populated `respiratory_support` table is usable at all. Resolve in five minutes: build a one-row frame with nulls, call `.validate()`, read `.errors`.

- **Whether Epic bumps `meta.versionId` reliably** even when it under-populates `meta.lastUpdated`. If not, the `(id, versionId)` dedup key degrades and amended results may be missed entirely. **[unverified]**

- **Epic's actual bulk file retention window.** Oracle says 30 days; Epic is undocumented. Assume zero.

- **Whether `_typeFilter` on `Encounter.class` works at any real Epic site**, or is silently dropped under lenient handling. **[unverified]**

- **How many distinct signals a real hospital produces.** Every cost estimate in [doc 13](13-mapper-engine.md) assumes thousands, not hundreds of thousands. If a site's flowsheet catalog is an order of magnitude larger than expected, the human review burden changes character. Measure on the first real export.

- **Whether censored lab values (`<0.01`) matter for the target analyses.** If yes, CLIF has nowhere obvious to record the censoring, and that is a gap worth raising with the consortium.
