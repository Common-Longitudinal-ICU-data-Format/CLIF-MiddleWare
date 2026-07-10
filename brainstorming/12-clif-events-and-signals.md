# clif_events and clif_signals

*Two tables. One holds every fact we ever saw; the other holds every distinct kind of fact. The split is what makes an AI mapper affordable, reviewable, and replayable.*

---

## In plain words

When the middleware downloads a year of a hospital's data, it receives something like a billion individual clinical facts. A heart rate at 03:14. A potassium result. A dose of vancomycin. Each one arrives as a small record saying, in effect: *this patient, at this time, this measurement, this value, this unit.*

Those billion facts are made of only a few thousand distinct *kinds* of fact. The hospital does not have a billion different lab tests. It has maybe four hundred, measured over and over. It does not have a billion medications. It has a few thousand, given repeatedly.

That gap — a billion instances, a few thousand kinds — is the single most useful structural property of clinical data, and this design is built entirely around exploiting it.

So we keep two tables.

**`clif_events`** is the raw log. Every fact we ever received, stored exactly as received, one row each. We never edit it. We never delete from it. It is append-only, like a ledger. It is big.

**`clif_signals`** is the catalog. One row for each *distinct kind* of fact: this code system, this code, this display name, this unit. A few thousand rows. Small enough to open in a spreadsheet.

The mapping work — deciding that the hospital's `"POTASSIUM, SERUM"` is CLIF's `potassium` — happens **only** against the catalog. The AI never reads a patient row. A clinician reviewing the mapping never reads a patient row. They review a few thousand catalog entries, once, and the decisions are written down.

Then converting the billion events into CLIF tables is a mechanical join: look up each event's signal in the catalog, read off the approved CLIF category, write the row. No intelligence required, no AI in the loop, perfectly reproducible, and fast.

Three things fall out of this, and each one matters more than it first appears.

**Cost collapses.** Asking a language model to classify a few thousand catalog entries is cheap and finishes in minutes. Asking it to classify a billion rows is neither.

**Review becomes possible.** A clinician cannot audit a billion mapping decisions. They can audit four thousand, especially when sorted by how many events each one affects — the top fifty signals typically cover most of the data.

**Mistakes become cheap.** If someone discovers next year that a mapping was wrong, we correct one catalog row and re-run the conversion over `clif_events`. We do **not** go back to the hospital and ask for the data again — which matters enormously, because Epic will let us export roughly once per day (see [doc 09](09-data-freshness.md)). Without the raw log, a mapping bug is a re-extraction. With it, a mapping bug is a re-run.

There is a fourth consequence, and it is the reason this design will outlive the FHIR decision. **A signal is a signal regardless of where it came from.** A FHIR `Observation`, an HL7 v2 `OBX` segment, and a row from Epic's Clarity database are all the same shape: a timestamped fact with a source identifier, a value, and a unit. All three land in `clif_events`. All three get cataloged in `clif_signals`. All three flow through the same mapper. So when we discover that ventilator settings cannot come through FHIR ([doc 08](08-what-hospitals-actually-provide.md)) and must arrive over an HL7 v2 feed instead, nothing downstream changes. Only the adapter changes.

---

## Technical detail

### Layering

```
   FHIR bulk NDJSON ─┐
   FHIR REST         ├─► adapters ─► clif_events (append-only, raw)
   HL7 v2 ORU/ADT    │                     │
   Clarity SQL       ┘                     │ derive (distinct)
                                           ▼
                                     clif_signals (catalog)
                                           │
                                           │ MapperEngine per table
                                           │ (AI proposes, clinician approves)
                                           ▼
                                     signal_map (frozen, versioned)
                                           │
   clif_events ⋈ signal_map ────────────►  │
                                           ▼
                                     clif_*.parquet ─► clifpy.validate()
```

Four stores, three of which are ours and one of which is CLIF's:

| Store | Grain | Size | Mutability |
|---|---|---|---|
| `clif_events` | one clinical fact | ~10⁸–10⁹ rows/site-year | append-only |
| `clif_signals` | one distinct fact-kind | ~10³–10⁴ rows | insert + observed-stats update |
| `signal_map` | one approved mapping decision | ~10³–10⁴ rows | append-only, versioned |
| `clif_*.parquet` | CLIF schema | derived | fully rebuildable |

`clif_events` and `signal_map` are append-only. `clif_*.parquet` is a **pure function** of the two. That property is the whole point: it means the warehouse is always reproducible from the log plus the decisions, and both of those are auditable artifacts.

### `clif_events` — the raw log

Illustrative shape, not final DDL:

| Column | Type | Notes |
|---|---|---|
| `event_id` | UUID / ULID | surrogate, ours |
| `signal_key` | VARCHAR | **FK → `clif_signals`.** The hash described below |
| `source_system` | VARCHAR | `fhir_bulk`, `fhir_rest`, `hl7v2`, `clarity` |
| `source_resource_type` | VARCHAR | `Observation`, `MedicationAdministration`, `OBX`, … |
| `source_resource_id` | VARCHAR | the FHIR `id`, or v2 message control id. **Not length-capped at 64** — Epic ids can exceed the FHIR limit |
| `source_version_id` | VARCHAR | `meta.versionId`. Used for dedup |
| `patient_key` | VARCHAR | site MRN or FHIR `Patient.id`, pre-CLIF |
| `encounter_key` | VARCHAR | FHIR `Encounter.id`, pre-CLIF |
| `effective_dttm` | TIMESTAMPTZ | the clinical time the fact is about |
| `recorded_dttm` | TIMESTAMPTZ | when the system recorded it, if distinct |
| `value_numeric` | DOUBLE | populated only when the value parses as a number |
| `value_string` | VARCHAR | the raw value, always populated |
| `value_code_system` | VARCHAR | for `valueCodeableConcept` |
| `value_code` | VARCHAR | for `valueCodeableConcept` |
| `value_boolean` | BOOLEAN | |
| `unit_raw` | VARCHAR | as received |
| `ingest_batch_id` | UUID | FK to the export/pull that produced this |
| `ingest_transaction_time` | TIMESTAMPTZ | the manifest's `transactionTime`, not wall-clock |
| `raw_payload` | JSONB / TEXT | the whole original resource. Optional but strongly recommended |

Note the value columns. **This mirrors CLIF's own `patient_assessments` table**, which carries `numerical_value`, `categorical_value`, and `text_value` side by side. CLIF already solved polymorphic values this way; we are following its precedent rather than inventing one. FHIR's `Observation.value[x]` can be `valueQuantity`, `valueCodeableConcept`, `valueString`, `valueBoolean`, `valueRatio`, `valueSampledData`, or — for panels like blood pressure — absent entirely, with the values living in `component[]`. See [doc 15](15-edge-cases.md).

**Keep `raw_payload`.** It roughly doubles storage and it is the difference between "we can answer that" and "we would have to re-export." Given a 24-hour export throttle, that is a good trade. If storage is genuinely constrained, retain it for a rolling window and drop it for older partitions.

**Dedup key: `(source_resource_id, source_version_id)`.** Because the Epic incremental strategy uses overlapping `_typeFilter` date windows ([doc 09](09-data-freshness.md)), the same resource will arrive many times. Upsert idempotently on that pair. Never dedupe on content hash alone; an amended lab result has the same content shape and a different meaning.

### `clif_signals` — the distinct catalog

A **signal** is a distinct kind of clinical fact, identified by the tuple that determines its clinical meaning:

```
signal_key = hash(
    source_system,
    source_resource_type,
    code_system,        # e.g. http://loinc.org
    code,               # e.g. 2823-3
    display_normalized, # lowercased, whitespace-collapsed
    unit_raw            # e.g. mmol/L
)
```

| Column | Type | Notes |
|---|---|---|
| `signal_key` | VARCHAR | PK, the hash above |
| `source_system` | VARCHAR | |
| `source_resource_type` | VARCHAR | |
| `code_system` | VARCHAR | nullable — v2 and Clarity often have no code system |
| `code` | VARCHAR | nullable |
| `display_raw` | VARCHAR | **the hospital's exact string.** This becomes CLIF's `<x>_name` |
| `display_normalized` | VARCHAR | for hashing and matching |
| `unit_raw` | VARCHAR | |
| `observation_category` | VARCHAR | FHIR `Observation.category`, when present |
| `first_seen_dttm` | TIMESTAMPTZ | |
| `last_seen_dttm` | TIMESTAMPTZ | |
| `event_count` | BIGINT | **prevalence — drives review priority** |
| `distinct_patient_count` | BIGINT | |
| `value_kind` | VARCHAR | `numeric` / `coded` / `string` / `boolean` / `mixed` |
| `numeric_p01`, `p50`, `p99` | DOUBLE | observed distribution |
| `distinct_units` | VARCHAR[] | if >1, a data-quality flag |
| `example_values` | VARCHAR[] | up to 5, **scrubbed** |
| `status` | ENUM | `unmapped` / `proposed` / `approved` / `rejected` / `ignored` |

Two columns carry more weight than they look like they do.

`event_count` is the review priority. Sort the catalog descending and a clinician's first hour of review covers most of the data volume. It also lets us report honest coverage: *"96.4% of lab events are mapped; the unmapped 3.6% are these 812 rare signals."*

`numeric_p01/p50/p99` is a free sanity check. A signal whose display says `"temperature"` with a median of 98.6 is Fahrenheit, not Celsius, whatever the unit string claims. The distribution catches unit lies that the unit field does not.

### `signal_map` — the frozen decisions

Append-only. Never updated in place; a correction is a new row with a new `map_version` and the old row retained.

| Column | Type | Notes |
|---|---|---|
| `map_id` | UUID | |
| `signal_key` | VARCHAR | FK → `clif_signals` |
| `clif_table` | VARCHAR | `vitals`, `labs`, `medication_admin_continuous`, … |
| `clif_category_column` | VARCHAR | `vital_category`, `lab_category`, … |
| `clif_category_value` | VARCHAR | must be in the schema's `permissible_values` |
| `unit_conversion` | JSONB | UCUM expression + factor, or null |
| `transform` | VARCHAR | e.g. `split_bp_components`, `identity` |
| `proposed_by` | VARCHAR | `llm:<model-id>` / `rxnav` / `loinc-parts` / `human` |
| `proposal_confidence` | DOUBLE | |
| `proposal_rationale` | TEXT | what the proposer said and why |
| `approved_by` | VARCHAR | the clinician's identity |
| `approved_dttm` | TIMESTAMPTZ | |
| `map_version` | INT | |
| `superseded_by` | UUID | nullable, FK to the replacing row |

`proposed_by` and `approved_by` are separate columns and they must never be the same principal. An AI proposal that nobody approved must not be able to write a CLIF row. See [doc 16](16-security-and-governance.md) on separation of duties.

### The transform

```
clif_events ⋈ signal_map ON signal_key WHERE status='approved'
    → apply unit_conversion
    → apply transform
    → resolve patient_key/encounter_key → patient_id/hospitalization_id
    → project to the CLIF table's columns
    → write clif_<table>.parquet
    → clifpy.Table.from_file(...).validate()
```

Deterministic. No network. No model. Given the same `clif_events` and the same `signal_map` version, byte-identical output. That is what makes the warehouse defensible: you can hand a reviewer the log, the decisions, and the code, and they can reproduce your dataset exactly.

Events whose `signal_key` has no approved mapping are **not dropped**. They stay in the log, and they are counted in a per-table coverage report emitted alongside the parquet. Silent dropping is the failure mode that produces confidently wrong research.

### Why not map inline and skip the log?

Because the log buys three things that are individually worth its storage cost:

1. **Replay.** A mapping fix costs a re-run, not a re-export. Under a 24-hour export throttle, this is decisive.
2. **Determinism.** An LLM invoked per-row is non-reproducible by construction. An LLM invoked per-catalog-entry, once, with the answer frozen, is reproducible forever.
3. **Auditability.** Three years later, `clif_events` still holds the hospital's exact original string next to the timestamp, and `signal_map` holds who decided what it meant and when. That is the artifact an IRB or a reviewer actually wants.

---

## Edge cases and how it breaks

**The same clinical concept splits across many signals.** A hospital may have twelve distinct flowsheet rows all meaning "heart rate," differing only by the recording device or the unit. Twelve signals, one CLIF category. That is fine and expected — the mapping is many-to-one. The reverse is the problem: see the next item.

**One signal splits into many CLIF rows.** Blood pressure. LOINC 85354-9 is a panel; the systolic (8480-6) and diastolic (8462-4) values live in `component[]`, and CLIF wants two separate `vitals` rows with categories `sbp` and `dbp`. So the signal grain must accommodate *components*, not just top-level codes. **Recommendation: define the signal on the component code, not the panel code, and treat the panel as a container.** Otherwise the catalog has one blood-pressure entry that cannot be mapped to a single category. This is the most likely place for the design to break, and it should be prototyped first. See [doc 15](15-edge-cases.md).

**Signal identity includes the unit, which means a unit change forks the signal.** If the hospital switches creatinine from mg/dL to µmol/L on a Tuesday, a new `signal_key` appears and shows up as `unmapped`. That is the *correct* behavior — it forces a human to notice — but it will look like a bug to whoever is on call. Document it.

**Display strings can carry PHI.** Real flowsheet and lab display names occasionally contain patient identifiers, initials, or free-text nurse notes. `display_raw` and `example_values` therefore leave the hospital if the mapper uses a hosted model. **The catalog needs a scrub-and-review step before any egress.** This is not hypothetical. See [doc 16](16-security-and-governance.md).

**`event_count` tempts you to ignore the tail.** The rare signals are rare in volume and not rare in importance. `norepinephrine` in a general ward population is a low-count signal; it is also the one your sepsis study depends on. Prioritize by count, but never *truncate* by count, and log exactly what was left unmapped.

**Unmapped events accumulate silently unless you look.** Emit a coverage report per table per run: mapped events, unmapped events, unmapped signals ranked by count. If coverage drops between runs, something upstream changed.

**Epic resource ids can exceed FHIR's 64-character `id` limit** **[community]**. Do not declare `source_resource_id VARCHAR(64)`.

**Dedup on `(id, versionId)` fails if the server does not bump `versionId`.** Epic under-populates `meta.lastUpdated` in bulk output and may be inconsistent about `versionId` **[unverified]**. If `versionId` is absent, fall back to `(id, ingest_transaction_time)` and accept that amended results may be missed — and *log that you are in this degraded mode*. See [doc 09](09-data-freshness.md).

**`clif_events` grows without bound.** A million events per patient-year is not unusual for ICU flowsheet data. Partition by `effective_dttm` month and by `source_system`. Plan for the `raw_payload` column to dominate storage, and make its retention a config value.

**Timezone.** Store everything as `TIMESTAMPTZ` in UTC in `clif_events`. Convert once, at the very end, to the single timezone clifpy expects. Never store naive timestamps in the log. See [doc 01](01-what-is-clif.md).

---

## What we still don't know

- **Whether the signal grain should be the component code or the panel code**, and whether a single grain can serve labs, vitals, medications, and assessments simultaneously. Blood pressure argues for components. Medications argue for something else entirely — a `MedicationAdministration` signal is identified by the drug, not by an observation code, and its "unit" is a dose unit. **This may need two or three signal *shapes* rather than one.** Prototype against the MIMIC-IV-on-FHIR demo ([doc 07](07-free-fhir-servers.md)) before committing to a schema.

- **What identifies a medication signal.** RxNorm code? Epic medication id? The free-text `med_name`? Probably the RxNorm ingredient after normalization ([doc 14](14-terminology-resources.md)), but the raw string must still be preserved because CLIF wants it in `med_name`.

- **Whether `raw_payload` retention is acceptable to the hospital's privacy office.** It is a complete copy of the source resource. Some institutions will object; others will consider it the safest possible provenance record. Ask early.

- **The right storage engine.** DuckDB over partitioned parquet is attractive (no server, clifpy already speaks parquet). Postgres is attractive (transactional, familiar, good for `signal_map` and the review UI). A split — Postgres for the small mutable stores, parquet for the large immutable log — is probably correct but adds a moving part. See [doc 11](11-middleware-architecture.md).

- **How to represent `respiratory_support`.** It is a *wide* CLIF table: one row is a whole-ventilator snapshot with seventeen numeric columns. But `clif_events` is long: one row per fact. The pivot from long events to a wide row needs a time-bucketing rule, and that rule is a clinical decision. Unresolved.

- **Whether `signal_map` should be shareable across sites.** Two hospitals' raw strings differ, but their LOINC and RxNorm codes often do not. A code-keyed portion of the map may be a consortium asset rather than a site asset. That would be genuinely valuable and is worth designing for even if v1 does not use it.
