# What is CLIF — the target schema

*The 18 tables we must fill, the `_name`/`_category` pattern that defines all the work, and the validator that decides whether we succeeded.*

---

## In plain words

CLIF stands for Common Longitudinal ICU data Format. It is an agreed-upon way to lay out intensive care data so that researchers at different hospitals can run the same analysis without rewriting it for each hospital's database.

The problem CLIF solves is that every hospital records the same clinical fact differently. One hospital's electronic record calls a drug `"NOREPINEPHRINE 4 MG/250 ML IV INFUSION"`. Another calls it `"LEVOPHED DRIP"`. A third calls it `"norepi 16mcg/ml"`. All three are the same medication. A researcher who wants to study vasopressor use in septic shock cannot write one query that works everywhere, because there is no shared name.

CLIF fixes this by defining a short, closed list of allowed values for each clinically important field. For medications, that list has 75 entries for continuous drips, and `norepinephrine` is one of them. All three hospitals above must map their local string onto that one word. Once they do, the same analysis code runs at all three sites.

The crucial structural idea is that **CLIF keeps both names**. Every controlled field appears twice: once as `<something>_name`, which holds the hospital's original raw string exactly as it appeared, and once as `<something>_category`, which holds the CLIF word. So the medication table has a `med_name` column containing `"LEVOPHED DRIP"` and a `med_category` column containing `norepinephrine`, side by side, on the same row.

That pairing is not decoration. It is the audit trail. Three years from now, when a reviewer asks how you decided that `"LEVOPHED DRIP"` was norepinephrine, the raw string is still sitting there next to your answer. **Everything this middleware does is, at bottom, filling in the `_category` column given the `_name` column.** Read that sentence again — it is the entire project.

CLIF is federated by design. The data never leaves the hospital. Each site builds its own CLIF tables locally, runs the same analysis script locally, and only aggregate results are shared. That is why it matters that CLIF is a *format* and not a *database* — nobody is asking hospitals to send anyone their patients.

CLIF is developed by a consortium led by the University of Chicago, built by roughly a dozen academic health systems. The [proof-of-concept paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC11398431/) converted about 111,440 ICU admissions across 9 health systems and 39 hospitals.

One more thing to know before reading further, because it shapes this whole project: **today, every CLIF site builds its tables by writing SQL against Epic's Clarity or Caboodle data warehouse.** The consortium ships [pre-built SQL queries for Epic](https://clif-icu.com/tools). The CLIF paper states plainly that CLIF "is currently not linked to established interoperability standards like HL7 FHIR." We are proposing a different pipe. See [doc 08](08-what-hospitals-actually-provide.md) for whether that pipe can carry the water.

---

## Technical detail

### The tables

`clifpy`, the reference Python library, exposes **18 table classes**. The reference documentation calls it "20 CLIF tables"; the discrepancy is real and explained at the bottom of this section.

Files live flat in one directory, one file per table, named `clif_<table>.<parquet|csv>`. Parquet is strongly preferred.

| # | clifpy class | Table | Grain | Key |
|---|---|---|---|---|
| 1 | `Patient` | `patient` | one row per patient | `patient_id` |
| 2 | `Hospitalization` | `hospitalization` | one row per encounter | `hospitalization_id` |
| 3 | `Adt` | `adt` | one row per location stay | `hospitalization_id` + `in_dttm` |
| 4 | `Vitals` | `vitals` | one row per measurement | `hospitalization_id` |
| 5 | `Labs` | `labs` | one row per result | `hospitalization_id` |
| 6 | `RespiratorySupport` | `respiratory_support` | one row per device/setting snapshot | `hospitalization_id` |
| 7 | `Position` | `position` | one row per position observation | `hospitalization_id` |
| 8 | `MedicationAdminContinuous` | `medication_admin_continuous` | one row per MAR action | `hospitalization_id` |
| 9 | `MedicationAdminIntermittent` | `medication_admin_intermittent` | one row per MAR action | `hospitalization_id` |
| 10 | `PatientAssessments` | `patient_assessments` | one row per assessment | `hospitalization_id` |
| 11 | `HospitalDiagnosis` | `hospital_diagnosis` | one row per diagnosis code | `hospitalization_id` |
| 12 | `CodeStatus` | `code_status` | one row per code-status change | **`patient_id`** |
| 13 | `CrrtTherapy` | `crrt_therapy` | one row per CRRT snapshot | `hospitalization_id` |
| 14 | `EcmoMcs` | `ecmo_mcs` | one row per device snapshot | `hospitalization_id` |
| 15 | `MicrobiologyCulture` | `microbiology_culture` | one row per organism | `organism_id` |
| 16 | `MicrobiologyNonculture` | `microbiology_nonculture` | one row per test | `hospitalization_id` |
| 17 | `MicrobiologySusceptibility` | `microbiology_susceptibility` | one row per organism × antimicrobial | **`organism_id` only** |
| 18 | `PatientProcedures` | `patient_procedures` | one row per procedure code | `hospitalization_id` |

Note the two key irregularities, both of which will bite an ETL author. **`code_status` and `patient` are keyed on `patient_id`, not `hospitalization_id`.** And **`microbiology_susceptibility` has no `hospitalization_id` at all** — it joins to the rest of the warehouse only through `organism_id` on `microbiology_culture`. If you lose `organism_id`, susceptibility data is orphaned and unrecoverable.

### The `_name` / `_category` pattern

Every mCIDE-controlled field is a pair:

- `<x>_name` — `VARCHAR`, `required: false`, the raw source string from the EHR, `is_category_column: false`
- `<x>_category` — `VARCHAR`, usually `required: true`, the normalized value, constrained by `permissible_values` in the schema, `is_category_column: true`

Some tables add `<x>_group`, a coarser rollup, sometimes with an in-schema `*_to_group_mapping` block (e.g. `med_category_to_group_mapping` maps all 75 continuous meds to their group).

A few fields break the pattern and have no `_name` partner: `adt.location_type`, `microbiology_culture.organism_category`, `hospitalization`'s geo columns.

### Column inventory, per table

Verified against `schemas/*.yaml` in the `clif-icu` skill. `[R]` = required, ⏰ = `DATETIME`.

**`patient`** — `patient_id [R]`, `birth_date [R]`⏰, `death_dttm [R]`⏰, `race_name`/`race_category [R]` (7 values), `ethnicity_name`/`ethnicity_category [R]` (Hispanic, Non-Hispanic, Unknown), `sex_name`/`sex_category [R]` (Male, Female, Unknown), `language_name`/`language_category [R]` (~50 values)

**`hospitalization`** — `patient_id [R]`, `hospitalization_id [R]`, `hospitalization_joined_id`, `admission_dttm [R]`⏰, `discharge_dttm [R]`⏰, `age_at_admission [R]` (INT), `admission_type_name`/`admission_type_category [R]` (ed, facility, osh, direct, elective, other), `discharge_name`/`discharge_category [R]` (17 values incl. Home, Expired, Hospice, Still Admitted), plus optional geo: `zipcode_nine_digit`, `zipcode_five_digit`, `census_block_code`, `census_block_group_code`, `census_tract`, `state_code`, `county_code`

**`adt`** — `hospitalization_id [R]`, `hospital_id [R]`, `hospital_type [R]` (academic, community, LTACH), `in_dttm [R]`⏰, `out_dttm [R]`⏰, `location_name`/`location_category [R]` (ed, ward, stepdown, icu, procedural, l&d, hospice, psych, rehab, radiology, dialysis, other), `location_type [R]` (general_icu, cardiac_icu, surgical_icu, medical_icu, neuro_icu, burn_icu, mixed_*)

**`vitals`** — `hospitalization_id [R]`, `recorded_dttm [R]`⏰, `vital_name`/`vital_category [R]`, `vital_value [R]` (DOUBLE), `meas_site_name`
Categories (9): `temp_c`, `heart_rate`, `sbp`, `dbp`, `spo2`, `respiratory_rate`, `map`, `height_cm`, `weight_kg`
Schema also carries `vital_units` and `vital_ranges` per category (e.g. `temp_c` 25–44, `heart_rate` 0–300, `spo2` 50–100).

**`labs`** — `hospitalization_id [R]`, `lab_order_dttm [R]`⏰, `lab_collect_dttm [R]`⏰, `lab_result_dttm [R]`⏰, `lab_order_name`/`lab_order_category [R]` (blood_gas, bmp, cbc, coags, lft, misc), `lab_name`/`lab_category [R]` (**53 values**), `lab_value [R]` (VARCHAR — raw, may be non-numeric), `lab_value_numeric` (DOUBLE), `reference_unit [R]`, `lab_specimen_name`/`lab_specimen_category`, **`lab_loinc_code`** (optional)

`lab_loinc_code` is **the only standard-code column anywhere in CLIF.** Everything else is a bare string category.

**`medication_admin_continuous`** — `hospitalization_id [R]`, `med_order_id`, `admin_dttm [R]`⏰, `med_name`/`med_category [R]` (**75 meds**), `med_group [R]` (vasoactives, sedation, diuretics, anticoagulation, cardiac, paralytics, pulmonary vasodilators (IV), pulmonary vasodilators (inhaled), gastrointestinal, Inhaled, endocrine, fluids_electrolytes, others), `med_route_name`/`med_route_category [R]` (im, iv, inhaled), `med_dose [R]` (DOUBLE), `med_dose_unit [R]`, `mar_action_name`/`mar_action_category [R]` (dose_change, going, start, stop, verify, other), `mar_action_group [R]` (administered, not_administered, other)

**`medication_admin_intermittent`** — same shape; `med_category` has ~180 values; `med_group` ∈ {CMS_sepsis_qualifying_antibiotics, analgesia, antipsychotic, anxiolytic, car_t, other, paralytics, sedation, steroid, vasopressor}; `med_route_category` ∈ {enteral, im, iv, buccal_sublingual, intrapleural}; `mar_action_category` ∈ {given, bolus, not_given, other}; `med_dose` is FLOAT

**`respiratory_support`** — `hospitalization_id [R]`, `recorded_dttm [R]`⏰, `device_id`, `device_name`/`device_category [R]` (IMV, NIPPV, CPAP, High Flow NC, Face Mask, Trach Collar, Nasal Cannula, Room Air, Other), `vent_brand_name [R]`, `mode_name`/`mode_category [R]` (Assist Control-Volume Control, Pressure Control, PRVC, SIMV, Pressure Support/CPAP, Volume Support, Blow by, Other), `tracheostomy [R]` (0/1)
Settings, all `DOUBLE [R]`: `fio2_set`, `lpm_set`, `tidal_volume_set`, `resp_rate_set`, `pressure_control_set`, `pressure_support_set`, `flow_rate_set`, `peak_inspiratory_pressure_set`, `inspiratory_time_set`, `peep_set`
Observed, all `DOUBLE [R]`: `tidal_volume_obs`, `resp_rate_obs`, `plateau_pressure_obs`, `peak_inspiratory_pressure_obs`, `peep_obs`, `minute_vent_obs`, `mean_airway_pressure_obs`

This is a **wide** table, not a long one — one row is a snapshot of the whole ventilator, not one setting. That is a structural mismatch with FHIR `Observation`, which is one-fact-per-resource. Pivoting long FHIR Observations into a wide respiratory row requires a time-alignment rule. Flagged in [doc 15](15-edge-cases.md).

**`patient_assessments`** — `hospitalization_id [R]`, `recorded_dttm [R]`⏰, `assessment_name`/`assessment_category [R]` (**72 values**: gcs_total/eye/motor/verbal, RASS, SAS, cam_icu, braden_*, cpot_*, CIWA, COWS, WAT, sat_*, sbt_*, ...), `assessment_group`, and **three value columns**: `numerical_value [R]` (DOUBLE), `categorical_value [R]` (VARCHAR), `text_value [R]` (VARCHAR)

Note the three-value-column design. **This is CLIF's own answer to polymorphic values, and it is the precedent we should imitate** when flattening FHIR's `Observation.value[x]`. See [doc 15](15-edge-cases.md).

**`position`** — `hospitalization_id [R]`, `recorded_dttm [R]`⏰, `position_name`/`position_category [R]` (prone, not_prone)

**`microbiology_culture`** — `patient_id [R]`, `hospitalization_id [R]`, `organism_id [R]`, `order_dttm [R]`⏰, `collect_dttm [R]`⏰, `result_dttm [R]`⏰, `fluid_name`/`fluid_category [R]` (~45 sites), `method_name`/`method_category [R]` (culture, gram_stain, smear), `organism_category [R]` (**545+ organisms**), `organism_group [R]` (~110 groups)

**`microbiology_nonculture`** — `patient_id [R]`, `hospitalization_id [R]`, `result_dttm [R]`⏰, `collect_dttm [R]`⏰, `order_dttm [R]`⏰, `fluid_name`/`fluid_category [R]`, `method_name`/`method_category [R]` (pcr), `micro_order_name`, `organism_category [R]` (clostridium_difficile, sars_cov2, respiratory_syncytial_virus), `organism_group [R]`, `result_name`/`result_category` (detected, not_detected, indeterminant)

**`microbiology_susceptibility`** — `organism_id`, `antimicrobial_name`/`antimicrobial_category [R]` (~180), `sensitivity_name`, `susceptibility_name`/`susceptibility_category [R]` (susceptible, non_susceptible, intermediate, NA). No timestamps. No `hospitalization_id`.

**`crrt_therapy`** — `hospitalization_id [R]`, `device_id`, `recorded_dttm [R]`⏰, `crrt_mode_name`/`crrt_mode_category [R]` (scuf, cvvh, cvvhd, cvvhdf, avvh), `dialysis_machine_name`, and FLOAT `[R]`: `blood_flow_rate`, `pre_filter_replacement_fluid_rate`, `post_filter_replacement_fluid_rate`, `dialysate_flow_rate`, `ultrafiltration_out`

**`ecmo_mcs`** — `hospitalization_id [R]`, `recorded_dttm [R]`⏰, `device_name`/`device_category [R]` (Impella_*, iVAC, TandemHeart_*, CentriMag_*, IABP, VA_ECMO, VV_ECMO, VAV_ECMO, LAVA_ECMO, HeartWare, HMII, HMIII), `mcs_group [R]` (ECMO, IABP, RVAD, durable_LVAD, temporary_LVAD), `device_metric_name`, and FLOAT `[R]`: `device_rate`, `sweep`, `flow`, `fdO2`

**`hospital_diagnosis`** — `hospitalization_id [R]`, `diagnosis_code [R]`, `diagnosis_code_format [R]` (ICD10CM, ICD9CM), `diagnosis_primary [R]` (0/1), `poa_present [R]` (0/1, present-on-admission)

**`code_status`** — `patient_id [R]`, `start_dttm [R]`⏰, `code_status_name`/`code_status_category [R]` (DNR, DNAR, UDNR, DNR/DNI, DNAR/DNI, AND, Full, Presume Full, Other)

**`patient_procedures`** — `hospitalization_id [R]`, `billing_provider_id`, `performing_provider_id`, `procedure_billed_dttm [R]`⏰, `procedure_code [R]`, `procedure_code_format [R]` (CPT, ICD10PCS, HCPCS)

### mCIDE

mCIDE = "minimum Common ICU Data Elements." It is the authoritative vocabulary backing every `permissible_values` list in the schemas. Per the standardization summary: **37 files, 1,543 category rows, 5 files with a group column.**

Files are named `mCIDE/<table>/clif_<table>_<variable>_categories.csv`. The standardized CSV columns are:

1. `<variable>_category` — the normalized value
2. `description` — short clinical description (often blank)
3. `<variable>_name_examples` — up to 5 representative raw EHR strings, **intended to be filled in by each site**, frequently empty
4. `<variable>_group` — optional, only in 5 files

**There are no LOINC, RxNorm, SNOMED, or ICD columns in any mCIDE file.** The labs CSV carries a `reference_unit` and an `lab_order_category`; that is the extent of structured metadata. Standardization is to string tokens plus free-text examples.

This is the finding that defines the middleware. There is no crosswalk to download. See [doc 14](14-terminology-resources.md) for what free services can *propose* a crosswalk, and [doc 13](13-mapper-engine.md) for how a clinician approves one.

### clifpy: config, validation, utilities

Config, auto-detected as `clif_config.json` in the working directory or passed via `config_path=`:

```json
{
  "data_directory": "/path/to/clif/tables",
  "filetype": "parquet",
  "timezone": "America/Chicago",
  "output_directory": "/path/to/output"
}
```

`data_directory`, `filetype`, `timezone` are required. A YAML variant uses `tables_path` instead of `data_directory`.

**There is a real validator, and it checks category membership.** On any table object:

- `table.validate()` — runs the full suite
- `table.isvalid()` → bool; `table.errors` → structured error dicts; `table.get_summary()`

The validator checks column presence; data-type validation with cast-ability (exact → `datatype_castable` warning → `datatype_mismatch` error); missing-data analysis; **categorical value validation against `permissible_values`**; duplicate checking; numeric range validation against `vital_ranges` / the outlier config; statistical and unit validation; cohort analysis.

**This is our acceptance test.** The middleware succeeds when `clifpy` validates its output with zero category errors. That is an objective, external, non-negotiable gate, and we should wire it into CI from day one.

Utilities worth knowing:

- `stitch_encounters(hospitalization=..., adt=..., time_interval=6)` — links a patient's hospitalizations that are close in time (inter-hospital transfers), returning hospitalization and adt frames annotated with `encounter_block`, plus a mapping. Relates to the `hospitalization_joined_id` column.
- `create_wide_dataset(clif_instance=co, category_filters={...}, hospitalization_ids=[...])` — requires `ClifOrchestrator`, pivots categories into columns.
- `convert_wide_to_hourly(...)` — hourly aggregation with `max`/`min`/`mean`/`last`/`boolean`/`one_hot_encode` config.
- `RespiratorySupport.waterfall(bfill=False)` — hourly scaffold, forward-fills `fio2`/`peep`/`tidal_volume` within device/mode blocks, auto-scales `fio2` 40 → 0.40.
- `standardize_dose_to_base_units(med_df, vitals_df=...)` — mcg/min, ml/min, u/min; weight-based `/kg` using vitals weight.
- `apply_outlier_handling(table)` / `get_outlier_summary(table)` — driven by `schemas/outlier_config.yaml`.
- SOFA needs `clifpy.utils.sofa.REQUIRED_SOFA_CATEGORIES_BY_TABLE`.

### The "20 tables" discrepancy

The CLIF reference documentation is titled "all 20 CLIF tables" but documents 18. The missing two — **`invasive_hemodynamics`** (`measure_category`) and **`key_icu_orders`** (`order_category`) — exist in the mCIDE vocabulary but have **no schema YAML and no clifpy class** in this version. If the middleware targets the full 20-table CLIF spec it would emit `clif_invasive_hemodynamics.parquet` and `clif_key_icu_orders.parquet`, but clifpy will neither load nor validate them. Treat as out of scope for v1 and revisit.

---

## Edge cases and how it breaks

**Timezone is global, not per-column.** No schema carries per-column timezone metadata. clifpy localizes *all* `DATETIME` columns to the single `timezone=` value passed at load. FHIR `instant` values carry a UTC offset. Convert once, at the ingestion boundary, and record which zone you chose in warehouse metadata. If a hospital spans time zones or you get a DST transition wrong, timestamps silently shift by an hour and nobody notices until a length-of-stay looks odd.

**`labs.lab_value` is VARCHAR and `lab_value_numeric` is DOUBLE.** Not every lab result is a number: `"<0.01"`, `"POSITIVE"`, `"HEMOLYZED"`, `"1:64"` (a titer). Write the raw string to `lab_value` always; populate `lab_value_numeric` only when parsing succeeds. Discarding the raw string to get a number is data loss you cannot undo.

**`respiratory_support` is wide; FHIR is long.** One CLIF row holds seventeen numeric ventilator fields. FHIR would give you seventeen separate `Observation` resources at seventeen slightly different timestamps. Collapsing them into one row requires an explicit time-bucketing rule, and that rule is a clinical decision, not an engineering one.

**`med_group` is tagged inconsistently in the schema.** In `medication_admin_continuous`, `med_group` is marked `is_category_column: true` and appears under `category_columns`, not `group_columns` — unlike `medication_admin_intermittent`, where it is a group column. Validation behavior may differ. Do not assume symmetry.

**Known typos in the shipped vocabulary.** `mCIDE/postion/` (not `position`). `clif_patient_ethinicity_categories.csv` (not `ethnicity`). Fluid category `stomatch` in `microbiology_nonculture` versus `stomach` in `microbiology_culture` — meaning the *same* anatomical site has two spellings across two tables, and a naive shared lookup will fail on one of them.

**Required means required.** Nearly every numeric column in `respiratory_support`, `crrt_therapy`, and `ecmo_mcs` is `required: true`, including all seventeen ventilator fields. A source that provides `peep_set` but not `inspiratory_time_set` will not validate cleanly. Decide early whether to emit nulls and accept warnings, or to omit rows.

**`microbiology_susceptibility` is join-orphaned.** No `hospitalization_id`, no timestamps. Its only link to a patient runs through `organism_id` → `microbiology_culture`. Any ETL that fails to mint stable `organism_id` values destroys this table irrecoverably.

**`code_status` is keyed on `patient_id`, not `hospitalization_id`.** A patient's DNR status crosses admissions. Attributing a code status to the wrong hospitalization is a clinically serious error.

**Encounter boundaries will not line up.** FHIR `Encounter` and CLIF `hospitalization` are not the same object. Epic commonly emits a separate ED Encounter and inpatient Encounter for one continuous stay. `stitch_encounters` exists precisely because this is hard. See [doc 10](10-cohort-and-encounter-class.md).

---

## What we still don't know

- **Whether `hospitalization_joined_id` should be populated by us or left to `stitch_encounters` downstream.** The schema has the column; the utility computes `encounter_block`. Their relationship is not documented. Resolve by reading `clifpy/utils/stitching_encounters.py` and by asking the consortium.

- **Whether emitting nulls in `required: true` numeric columns produces validator errors or warnings.** This determines whether a partially-populated `respiratory_support` table is usable at all. Resolve empirically: build a one-row frame with nulls, call `.validate()`, read `.errors`.

- **Whether the consortium will accept a FHIR-derived CLIF warehouse as a conforming CLIF instance.** Passing `clifpy.validate()` is a technical gate, not a social one. The consortium's ETL guide assumes Clarity. Resolve by contacting `clif_consortium@uchicago.edu` before investing in a pipeline they will not recognize.

- **What CLIF version we target.** This document reflects the schema shipped in the `clif-icu` skill. The published data dictionary is at [v2.1.0](https://clif-icu.com/data-dictionary/data-dictionary-2.1.0) and the tools page references CLIF 2.0.0; an mCIDE v3.0 expansion process is underway. Pin a version explicitly and record it in warehouse metadata.

- **Whether `invasive_hemodynamics` and `key_icu_orders` are coming.** They have vocabulary but no schema. If they land mid-project the table count changes.
