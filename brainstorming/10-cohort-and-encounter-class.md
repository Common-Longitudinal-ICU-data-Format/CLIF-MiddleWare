# Encounter Class, Cohort Definition, and the Group Problem

*"We only want observation/inpatient patients" sounds like a one-line filter, but patient class is a billing status rather than a place, the ICU-vs-ward distinction lives in a different FHIR field entirely, and — the hardest news — at Epic we do not even choose who is in the export: a hospital analyst builds the list out-of-band and we filter what they send.*

## In plain words

**Patient class is a legal and billing status, not a location.** When the hospital calls someone an "inpatient," that is a formal admission decision with insurance and regulatory consequences. "Observation status" means the patient occupies a bed and is being watched, but has *not* been formally admitted — it is how the stay is billed to Medicare, nothing more. The clinically jarring part: **an observation-status patient can be lying in an ICU bed receiving care identical to the formally-admitted inpatient in the next bed.** At the bedside you cannot tell them apart. The only difference is a code in the billing system.

So a research cohort has to make a choice it often does not realize it is making: **do we care about the bed, or about the billing code?** Different published studies decide differently. Some define "inpatient ICU cohort" strictly (formal admission only) and drop observation patients; others keep anyone physically in the unit regardless of billing status. Neither is wrong — but the choice changes who is in your denominator, and it must be **written down and defended**, not left as an accident of which code we happened to filter on.

The requirement we were handed — keep observation *and* inpatient — is the inclusive choice. It is defensible (the observation patient in the ICU bed is a real ICU patient by any clinical measure), but because many published cohorts exclude observation, we have to state the rule explicitly and stamp it into the warehouse so a future analyst can see exactly what we did.

Then there is the part nobody expects: **at Epic we do not get to pick who is in the export at all.** A hospital Epic analyst builds a patient list for us by hand (out-of-band, not through any API), hands us an ID, and *that* is our population. We can only narrow it down afterward, by filtering the records they send. So the cohort is really decided in two places by two different owners — the analyst who builds the list, and us who filter it — and understanding that split is the whole point of this document.

## Technical detail

**Where "class" comes from.** `Encounter.class` in FHIR R4 is drawn from the HL7 v3 **ActEncounterCode** value set ([terminology.hl7.org ValueSet-encounter-class](https://terminology.hl7.org/5.1.0/ValueSet-encounter-class.html)) **[spec]**. The codes that matter here:

| Code | Meaning | Bucket |
|---|---|---|
| `IMP` | inpatient encounter | **inpatient** |
| `ACUTE` | inpatient acute | **inpatient** (subtype) |
| `NONAC` | inpatient non-acute | **inpatient** (subtype) |
| `OBSENC` | observation encounter | **NOT inpatient** — bed occupied, not formally admitted; a billing/regulatory status, not a location |
| `EMER` | emergency | ED |
| `AMB` | ambulatory / outpatient | outpatient |
| `SS` | short stay | — |
| `PRENC` | pre-admission | — |
| `VR` | virtual | — |
| `HH` | home health | — |

**Our filter, stated plainly:** `Encounter.class ∈ {IMP, ACUTE, NONAC, OBSENC}`. This deliberately **unions** true inpatients with observation-status patients. That is a legitimate research decision — but because it merges a billing distinction that many cohort definitions keep separate, we recommend recording it as a **versioned config value that lands in the warehouse metadata**, so a downstream analyst reading the CLIF tables can see the exact inclusion rule and its version rather than reverse-engineering it.

**How Epic populates `class` vs `type`** **[community]**:
- `Encounter.class` maps from Epic's **patient class / base class** → typically `IMP` for admitted inpatients, `EMER` for ED, `AMB` for clinic, `OBSENC` for observation status. It is a single high-level code.
- `Encounter.type` carries the finer, **site-configured** Epic encounter/visit type — often local codes, sometimes CPT or SNOMED.
- **CRITICAL: unit-level / ICU granularity is in NEITHER `class` NOR `type`.** "Was this patient in the ICU?" is answered by **`Encounter.location[]`** — the bed/unit reference and its `Location` resource / `location.physicalType`. See the resource anatomy in [doc 03](03-fhir-resource-types.md).

**Mapping to CLIF** (see [doc 01](01-what-is-clif.md) for the schema):
- `Encounter` → CLIF **`hospitalization`**: one row per encounter, carrying `admission_dttm`, `discharge_dttm`, `admission_type_category`, `discharge_category`. `Encounter.class` feeds the inpatient/observation inclusion decision, not a location field.
- `Encounter.location[]` → CLIF **`adt`**: one row per location stay, with `in_dttm`, `out_dttm`, `location_category` (icu / ward / ed / stepdown / …) and `location_type` (general_icu / cardiac_icu / …). `location.period` start/stop is what becomes `in_dttm`/`out_dttm`.

`Encounter.location[]` is *designed* to hold the ordered sequence of locations with `location.period`, i.e. the intra-stay transfer history — exactly CLIF's `adt` table. **But population varies by site**: some populate the full array with periods, others collapse to current-location-only **[community]**. The complete, reliable transfer history is the **HL7 v2 ADT feed** (`A02` transfer messages), not FHIR `Encounter`. **Validate `Encounter.location[].period` population per site before trusting it** to build `adt`; if it is collapsed, the `adt` table has to come from the v2 ADT stream instead (cross-ref [doc 08](08-what-hospitals-actually-provide.md)).

**Encounter boundaries will not line up with CLIF hospitalizations.** Epic may emit **multiple Encounters for one continuous stay** — a separate ED encounter, then an inpatient encounter, for a patient who never left the building. CLIF handles this: `hospitalization` rows carry a **`hospitalization_joined_id`**, and clifpy provides a **`stitch_encounters(time_interval=6h)`** utility that links related hospitalizations for the same patient across inter-hospital transfers. Plan for stitching as a required step, not an edge case — the raw FHIR-Encounter-to-CLIF-hospitalization mapping is many-to-one.

**Can we filter by class at the export kickoff?** In principle `_typeFilter=Encounter%3Fclass=IMP` is a valid per-type filter. In practice this is unreliable: `_typeFilter` support in bulk is **server-specific**, and Epic's `_typeFilter` is battle-tested mainly for **date** ranging, not `class` **[community]**. **Do not assume class filtering works at kickoff — pull Encounters, then filter post-hoc in the ETL.** The `Prefer: handling=lenient` hazard makes this non-optional: a server may **silently drop** an unsupported `_typeFilter` and return everything while you believe you filtered — a silent correctness bug, not a visible error. Full treatment in [doc 05](05-bulk-export-deep-dive.md).

### The Group problem

At Epic, Bulk `$export` is **Group-level only**, and **Groups are defined out-of-band by a hospital Epic analyst — they are not created via API.** You are handed a Group ID (Epic's Cooper Thompson, [chat.fhir.org implementers thread](https://chat-archive.fhir.org/stream/179166-implementers/topic/Epic.20Bulk.20Data.20Export.20questions.3F.html)). Group IDs appear as `mylist|<id>` (a *My List*) or `systemlist|<id>` (a system-level registry list) **[community]** ([community.open-emr.org](https://community.open-emr.org/t/epic-bulk-data-export-or-patient-list/21359)). The Group is typically backed by an Epic **Registry** or a Reporting-Workbench-style rule-based cohort (e.g. "all patients with condition X discharged last week").

**"Currently admitted ICU patients" as a Group** is architecturally *possible* via a rule-based registry keyed on ICU units / bed types, but we found **no public Epic doc or community post demonstrating an ICU-unit bulk-export Group** — mark **[unverified]**. Worse, bulk export is a **point-in-time snapshot**, poorly suited to a *currently-admitted* cohort whose membership changes hourly.

**Oracle / Cerner** provisions Groups via **Ignite Management Tooling**; Patient `$export` needs an explicit patient-ID list, **max 20,000** **[spec]** ([docs.oracle.com bulk_data_access](https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfbda/bulk_data_access.html)).

### The architectural consequence

There are **two places the cohort can be narrowed, with different owners**:

- **Server-side (the Group)** — owned by the **hospital analyst**. Cheap, but we cannot change it, cannot iterate on it, and cannot make it depend on anything the analyst cannot express as a registry rule.
- **Client-side (post-export filter on `Encounter.class`)** — owned by **us**. Costs a full export of a wider population, but is **versionable, auditable, and reproducible**.

**Recommendation: ask for the broadest defensible Group** (e.g. "all hospital encounters in the last N days"), then do the inpatient/observation filter **client-side**, where we control it and can version it. This costs bandwidth and buys reproducibility — the correct trade for research. And the filter pays for itself immediately: it is what **shrinks the cohort before the expensive REST enrichment leg** (cross-ref [doc 04](04-api-access-modes.md)), so its cost is recovered downstream.

## Edge cases and how it breaks

- **Observation-to-inpatient conversion mid-stay.** A patient admitted under observation is later "converted" to formal inpatient. The class changes, and Epic may represent this as **two Encounters** (one `OBSENC`, one `IMP`) or as one Encounter carrying only the final class. Either way, a naive class filter can keep, drop, or double-count the same physical stay. Stitching (`hospitalization_joined_id`) is what rescues this.
- **ED-to-inpatient transitions.** A separate `EMER` Encounter precedes the `IMP` Encounter for one continuous presentation. If the filter excludes `EMER`, the ED portion of the `adt` timeline vanishes even though it is part of the same ICU stay.
- **`class` is a single code.** A stay whose status changes over time is squeezed into one code per Encounter. There is no in-Encounter class history — the transition is only visible if the site emitted a second Encounter.
- **`Encounter.location[]` collapsed to current-location-only.** Kills the `adt` table: no periods, no transfer sequence, so ICU-vs-ward time cannot be reconstructed from FHIR. Fall back to the HL7 v2 ADT feed.
- **`_typeFilter` on `class` silently dropped under `handling=lenient`.** You believe you exported only inpatients; you actually got everyone, and the client-side filter is now doing *all* the work. If the client-side filter is also missing, the warehouse is silently contaminated. Never rely on kickoff class filtering.
- **Group membership evaluated at export time.** A "currently admitted" Group returns a **different population every night**. Re-running the export is not reproducible unless the Group is defined over a fixed historical window.
- **Patients in the Group with no Encounter in the window.** The analyst's list can include people who have no qualifying Encounter for our date range — they arrive in the export as patients with nothing to attach, and must be dropped cleanly rather than producing empty `hospitalization` rows.
- **Encounter with no `period.end`.** Still admitted → the stay has no discharge time. Maps to CLIF `discharge_category = "Still Admitted"`; do not treat the missing end as a data error or impute a discharge.

## What we still don't know

- **Whether our target sites can/will stand up an ICU-unit or "currently-admitted" bulk-export Group at all** — the ICU-unit Group is **[unverified]** with no public precedent, and the point-in-time nature of bulk export may make a live-ICU cohort impractical regardless. We need to negotiate the Group definition with the site's Epic analyst directly.
- **The exact registry/Reporting-Workbench rule the analyst will use**, and therefore the true membership semantics of "our" Group — rule-based vs. My-List, refresh cadence, and whether it is windowed or live.
- **Per-site `Encounter.location[].period` population.** Whether each target site populates the full location array with periods (usable for `adt`) or collapses to current-location — decides whether `adt` comes from FHIR or requires the HL7 v2 ADT feed (see [doc 08](08-what-hospitals-actually-provide.md)).
- **How cleanly `stitch_encounters` will resolve real Epic Encounter fragmentation** (ED→observation→inpatient chains) into single CLIF hospitalizations, and whether the default `time_interval=6h` is right for our sites.
- **Whether Oracle sites (if any) can express a research population cohort** beyond the explicit ≤20,000 patient-ID list, which does not scale to an open "all inpatients" definition.
- **Whether any observation-status nuance is lost in Epic's single `class` code** — e.g. observation stays that never carry `OBSENC` because of local base-class configuration, which would make our union filter miss real observation patients.
