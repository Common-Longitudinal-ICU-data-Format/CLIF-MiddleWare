# The Mapper Engine

*One engine per CLIF table. Free national services generate candidates, a language model ranks them, a clinician decides, and the decision is frozen forever. The AI is a research assistant, never an authority.*

---

## In plain words

The middleware's real job is translation. A hospital's record system says `"LEVOPHED DRIP 4MG/250ML"`. CLIF wants to hear the word `norepinephrine`. Somebody has to make that connection, and there are a few thousand such connections to make at every hospital.

Doing it entirely by hand is how it is done today, and it takes months. Doing it entirely by machine is unsafe: a language model that confidently maps `"dopamine"` to `norepinephrine` — both vasopressors, similar contexts, entirely different drugs — will produce a research dataset that looks perfectly clean and is wrong. Nobody downstream will catch it, because the raw string that would have revealed the error was thrown away.

So the engine is built on a division of labor that mirrors how a good research assistant works with a senior clinician.

**The machine proposes.** For each distinct signal in the catalog, we ask several independent sources what they think it is. The national drug database (RxNav) can tell us that `"LEVOPHED"` normalizes to the ingredient norepinephrine. The national lab-code database (LOINC) can tell us which of CLIF's 53 lab categories a given LOINC code belongs to. A language model can read the raw display string and make a judgment where the code databases have nothing to say. Each proposal comes with a confidence and a written rationale.

**The clinician disposes.** The proposals land on a dashboard, sorted so the signals affecting the most patient data appear first. A clinician sees the hospital's raw string, the observed value distribution, the proposed CLIF category, and the reasoning. They approve, correct, or reject. The top fifty signals typically account for the great majority of the data, so the first hour of review is enormously productive.

**The decision is frozen.** Once approved, the mapping is written into a versioned, append-only store with the approver's name and the date. From that point forward, converting the raw data into CLIF is a mechanical lookup with no model involved. Run it a thousand times and you get identical output.

That last property is the one that matters most for science. A language model is not reproducible — ask it twice and you may get two answers. A lookup table is perfectly reproducible. By confining the model to the *proposal* step and freezing its output before it touches any patient data, we get the model's speed without inheriting its non-determinism.

Two more design choices worth stating up front.

**The model never sees a patient.** It reads the catalog — a code, a name, a unit, a value distribution. It never reads a patient row, a name, a date of birth, or a medical record number. This is not merely a privacy nicety; it is what makes the whole approach affordable, since the catalog has thousands of rows and the data has billions.

**The model is swappable.** The engine talks to "a thing that ranks candidates," not to any specific vendor's API. A locally-hosted open model, a hosted commercial model, or no model at all (code databases plus a human) must all be valid configurations, chosen per deployment. A hospital that forbids sending any text off-premises must still be able to run this.

---

## Technical detail

### One engine per CLIF table

There is no universal mapper, because the tables ask different questions.

| Engine | Decides | Primary candidate source |
|---|---|---|
| `VitalsMapper` | which of 9 `vital_category` values | LOINC vital-signs codes; value distribution |
| `LabsMapper` | which of 53 `lab_category` values, plus `lab_order_category` | LOINC + LOINC Parts/COMPONENT axis |
| `MedContinuousMapper` | which of 75 `med_category`, plus `med_group`, plus `mar_action_category` | RxNav ingredient (IN RxCUI) → RxClass |
| `MedIntermittentMapper` | which of ~180 `med_category`, plus `med_group` | same |
| `AssessmentsMapper` | which of 72 `assessment_category` | LOINC survey codes; display-string matching |
| `RespiratorySupportMapper` | which of 17 numeric columns + `device_category` + `mode_category` | display-string matching; **no code system available** |
| `AdtMapper` | `location_category`, `location_type` | `Location.physicalType`; site unit list |
| `MicrobiologyMapper` | 545 `organism_category`, `fluid_category` | SNOMED via UMLS; NIH CDE lists |
| `DiagnosisMapper` | passthrough — ICD codes are already codes | none needed |

Note that `DiagnosisMapper` and `ProceduresMapper` are near-trivial: `hospital_diagnosis` and `patient_procedures` store raw ICD/CPT codes, not CLIF categories. And `RespiratorySupportMapper` is the hardest by a wide margin, because it has no code system to lean on and must pivot long events into a wide row ([doc 12](12-clif-events-and-signals.md)).

Each engine implements the same interface conceptually:

```
propose(signal, context) -> [Candidate{category, confidence, rationale, proposer}]
validate(candidate)      -> bool   # is it in the schema's permissible_values?
```

`validate()` is not optional and is not the model's job. Every candidate is checked against the CLIF schema's `permissible_values` **before** it is ever shown to a human. A model that hallucinates `norepinephrin` (missing the final e) must fail closed, silently, at this gate. This single check eliminates the most common LLM failure mode in terminology mapping.

### Candidate generation, in priority order

Deterministic sources first. The model is the last resort, not the first.

**1. Exact code lookup.** If the signal carries a LOINC or RxNorm code and we have a curated crosswalk entry, use it. Confidence 1.0. No model.

**2. Code-database normalization.** **[spec]**
- Medications: RxNav `findRxcuiById` (NDC → RxCUI), then resolve to ingredient with `tty=IN+PIN+MIN`. Free text via `approximateTerm` / `getRxcuiByString`. [RxNav API](https://lhncbc.nlm.nih.gov/RxNav/APIs/api-RxNorm.findRxcuiById.html)
- `med_group`: RxClass `getClassByRxNormDrugId` over ATC / MED-RT / FDA EPC. [RxClass](https://rxnav.nlm.nih.gov/RxClassAPIs.html)
- Labs: the [LOINC FHIR terminology server](https://loinc.org/fhir/) (`$lookup`, subsumption) plus the LOINC Parts COMPONENT axis to cluster many local LOINCs into one `lab_category`. The [Top 2000 lab observations](https://loinc.org/usage/obs/) (~2170 codes ≈ 98% of US lab volume) pre-seeds most of it.
- Organisms and body sites: SNOMED CT US Edition via a UMLS license.

**3. Lexical / vector similarity** against the mCIDE `<x>_name_examples` column and against previously approved mappings at other sites.

**4. LLM ranking.** Given the signal (display string, code, unit, value distribution, observation category) and the *validated candidate set*, the model ranks and explains. **The model chooses among candidates; it does not invent them.** Where deterministic sources produced nothing — which is the common case for `RespiratorySupportMapper` and much of `AssessmentsMapper` — the model may propose freely, but its output still passes through `validate()`.

### Why the code databases are not enough

The researched honest assessment, per CLIF medication group:

- `paralytics` → ATC **M03**. Clean single-axis match.
- `anticoagulation` → ATC **B01A**. Clean.
- `diuretics` → ATC **C03** plus a few. Mostly clean.
- `vasoactives` → MED-RT "Vasoconstrictor Agents" or ATC C01CA. **Neither is 1:1 with CLIF's `vasoactives`.**
- `sedation` → straddles ATC N05 (hypnotics/anxiolytics), N01AX (propofol, ketamine), and N02A (opioids used for sedation). ATC alone under- or over-includes.

**Net: class systems deliver roughly 70–90% of `med_group` for free; the ICU-specific groupings force a hand-curated override layer at the ingredient level.** The right pattern is to normalize to RxNorm ingredient first — that reduces thousands of local medication strings to a few dozen ICU ingredients — and hand-curate *those*. Dozens of decisions, not thousands. See [doc 14](14-terminology-resources.md).

The same logic holds for labs. CLIF's `lab_category` is 53 clinically-curated values that do not correspond to any single LOINC axis. LOINC Parts gets you most of the way; a human closes it.

### The LLM-agnostic interface

The engine depends on an abstract ranker, not on a vendor:

```
rank(signal_context, candidates[], schema_constraints) -> ranked[] with rationale
```

Implementations: a locally-hosted open-weights model, a hosted commercial model, or a null ranker (deterministic sources plus human only). Configuration selects one. **The null ranker must be a first-class supported mode**, because some hospitals will not permit any text egress, and the system must remain usable for them — slower, not broken.

Requirements on any implementation:

- **Never receives a patient row.** Only catalog entries. Enforced structurally by having the ranker take a `Signal`, not an `Event`.
- **Output is constrained to the validated candidate set** wherever candidates exist.
- **Every proposal carries a rationale string**, which is stored and shown to the reviewer. An unexplained proposal is not reviewable.
- **The model identifier and version are recorded** in `signal_map.proposed_by` (e.g. `llm:<model-id>`). When a model changes, prior decisions remain attributed to the model that made them.

### Prior art: Usagi

OHDSI's [Usagi](https://www.ohdsi.org/analytic-tools/usagi/) ([source](https://github.com/OHDSI/Usagi)) is the closest existing design and it validates this shape: lexical and vector similarity generate candidate OMOP concepts, a domain expert approves or edits them in a review UI, and the output is a reusable `source_to_concept_map`. Recent work pairs Usagi-style candidate generation with LLM re-ranking.

**Reported LLM performance in terminology mapping**, for calibration:

- Agentic LLM against the LOINC Search API with query refinement: **Top-1 accuracy up to 85.4%, Top-5 up to 98.0%** ([Springer, 2025](https://link.springer.com/chapter/10.1007/978-3-032-26363-6_17))
- Google, [automated LOINC standardization with pre-trained LLMs](https://research.google/pubs/automated-loinc-standardization-using-pre-trained-large-language-models/)
- Pre-LLM baseline: ~85% unlabeled / 96% labeled by test frequency ([JAMIA 2018](https://academic.oup.com/jamia/article-pdf/25/10/1292/34150281/ocy110.pdf))
- LLM → OMOP: F1 0.68 (procedures) / 0.85 (medicines); Usagi ~90% on common medicines, ~70% on random ones, requiring heavy manual review ([ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0933365725001393))

**Consistently reported failure modes:** hallucinated or nonexistent codes; inconsistent ranking across runs; weak sensitivity to units and care context; poor performance on rare or local tests; confident-but-wrong on near-synonyms.

Read those numbers carefully. Top-1 of 85% means **roughly one in seven mappings is wrong on the first guess.** Top-5 of 98% means the right answer is almost always *in the list*. That is precisely the profile of a good candidate generator and a terrible autonomous decider — which is exactly how this design uses it.

### The review dashboard

Requirements, in priority order:

1. **Sort by `event_count` descending.** The reviewer's time is the scarce resource. Show cumulative coverage: *"you have now approved mappings covering 91.2% of lab events."*
2. **Show the evidence, not just the answer.** Raw display string. Code and system. Unit. Observed p01/p50/p99 and a small histogram. Up to five example values, scrubbed. The proposer and its rationale.
3. **Make rejection as easy as approval**, with `ignore` as a distinct, recorded outcome. "This signal is a nursing comment, not a vital sign" is a valuable, permanent decision.
4. **Show conflicts.** When two proposers disagree — RxClass says `cardiac`, the model says `vasoactives` — surface the disagreement rather than silently taking the higher confidence. Disagreement is signal.
5. **Bulk actions with guardrails.** Approving 200 lab mappings one at a time invites rubber-stamping. Group by proposed category and let the reviewer confirm a group — but require an explicit action per group, and record it per signal.
6. **The value distribution is a lie detector.** A signal displayed as `"temp"` with median 98.6 is Fahrenheit regardless of what the unit field says. Show it prominently.
7. **Never let one person both propose and approve.** `proposed_by` and `approved_by` are separate columns for a reason. See [doc 16](16-security-and-governance.md).

### Freezing and versioning

An approved mapping writes a row to `signal_map` ([doc 12](12-clif-events-and-signals.md)) with `approved_by`, `approved_dttm`, and `map_version`. Corrections append a new row and set `superseded_by` on the old one. Nothing is edited in place.

The CLIF output therefore carries a `signal_map` version in its metadata, and any published result can name the exact mapping version that produced it. Two runs with the same events and the same map version produce byte-identical parquet.

---

## Edge cases and how it breaks

**Near-synonym confusion is the killer.** Dopamine and dobutamine. Norepinephrine and epinephrine. Hydromorphone and hydrocodone. Sodium bicarbonate and sodium chloride. These are the errors a model makes and a human catches instantly. Weight review priority not only by `event_count` but by *clinical consequence* — every vasopressor, sedative, and paralytic mapping should be individually reviewed regardless of volume.

**A model asked to choose from a list will always choose from the list.** Given `["norepinephrine", "epinephrine", "dopamine"]` and a signal that is actually a saline flush, a ranker will return one of the three with a plausible rationale. **There must always be a "none of these" candidate**, and the model must be explicitly instructed and rewarded for choosing it. Absent that, the confidence scores are meaningless.

**Confidence scores are not probabilities.** An LLM's stated confidence is poorly calibrated and correlates with fluency, not correctness. Use it to *order* the review queue. Never use it as an auto-approve threshold. There should be no auto-approve threshold.

**The catalog can leak PHI.** `display_raw` and `example_values` come from real flowsheet and lab names, which occasionally contain identifiers or free-text notes. If a hosted model is used, those strings leave the hospital. A scrub-and-review gate before egress is mandatory, and it is the single most likely place for this project to cause a real incident. See [doc 16](16-security-and-governance.md).

**Unit strings lie.** `"mcg/kg/min"` in the unit field with values clustering around 0.05 is plausible for norepinephrine. The same unit string with values around 400 is not. Cross-check unit against distribution. UCUM ([doc 14](14-terminology-resources.md)) validates that a unit is *well-formed*, not that it is *true*.

**Blood pressure cannot be mapped one-to-one.** LOINC 85354-9 is a panel with no top-level value; systolic (8480-6) and diastolic (8462-4) live in `component[]`, and CLIF wants two rows. The mapper needs a `transform` concept, not just a category assignment. This is why `signal_map` has a `transform` column. See [doc 15](15-edge-cases.md).

**`mar_action_category` is not a terminology problem.** CLIF wants `start` / `stop` / `going` / `dose_change` for continuous meds. FHIR gives you `MedicationAdministration.status` and timing. Deriving MAR actions from administration records is a *state-machine* problem, not a mapping problem, and no terminology service will help. It probably deserves its own engine and its own doc.

**Cross-site map reuse is tempting and dangerous.** A code-keyed mapping (LOINC 2823-3 → `potassium`) is genuinely portable. A string-keyed mapping (`"K, SER"` → `potassium`) is not, because another hospital's `"K"` might be a different assay. Separate the two in storage and only ever ship the code-keyed portion between sites.

**Reviewer fatigue produces silent rubber-stamping**, which is worse than no review, because it launders an unreviewed AI decision as a clinical one. Instrument approval latency. If median time-per-signal drops below a few seconds, the review is not happening.

**The `RespiratorySupportMapper` has no code system.** LOINC does not meaningfully cover ventilator settings, and the source data may not exist at all ([doc 08](08-what-hospitals-actually-provide.md)). This engine will be mostly string matching plus human judgment, and it should be built last, after the tables that have codes prove the machinery.

---

## What we still don't know

- **Whether one signal shape can serve all engines.** A lab signal is keyed on LOINC + unit. A medication signal is keyed on an ingredient. A ventilator signal has neither. This may need two or three shapes. Prototype against the MIMIC-IV-on-FHIR demo before committing. See [doc 12](12-clif-events-and-signals.md).

- **How to evaluate the mapper.** We have an unusually good opportunity here and should not waste it: [MIMIC-IV-Ext-CLIF](https://physionet.org/content/mimic-iv-ext-clif/1.1.0/) and [CLIF-MIMIC](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC) are an existing, human-built MIMIC→CLIF mapping, and [MIMIC-IV-on-FHIR](https://physionet.org/content/mimic-iv-fhir-demo/2.1.0/) is the same data as FHIR. **Running our FHIR→CLIF pipeline over MIMIC and diffing against the published CLIF gives us a real accuracy number, per table, per category.** That turns "the mapper seems to work" into "the mapper agrees with expert humans on 94% of lab categories, and here are the 6% where it doesn't." Nothing else on the roadmap is worth as much. See [doc 17](17-open-questions-and-roadmap.md).

- **What accuracy is good enough**, and who decides. 85% Top-1 is unacceptable for autonomous mapping and irrelevant for assisted mapping. The number that matters is *post-review* accuracy, which nobody has measured for this design.

- **Whether a locally-hosted model is good enough** to keep everything on-premises. If yes, the entire PHI-egress problem dissolves. Worth testing early with the null-ranker and local-ranker configurations side by side on MIMIC.

- **How `mar_action_category` should actually be derived** from `MedicationAdministration` — assuming Epic populates the rate and period fields at all, which is **[unverified]** ([doc 08](08-what-hospitals-actually-provide.md)).

- **Whether the consortium wants this.** A validated, shareable, code-keyed FHIR→CLIF crosswalk would be a genuine contribution to CLIF — arguably more valuable than the middleware itself. It is also exactly the kind of artifact that should be built *with* the consortium rather than presented to it. `clif_consortium@uchicago.edu`.
