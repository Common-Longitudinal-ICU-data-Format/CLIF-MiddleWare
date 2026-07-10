# Terminology resources: what we can reuse, and what we must build

*The free national dictionaries (LOINC, RxNorm, SNOMED, UCUM) can propose how a hospital's local codes map to CLIF's categories — but nobody has published that map, so the human-approved, versioned crosswalk is the core thing this middleware has to build.*

## In plain words

A single hospital calls potassium a dozen different things: `K`, `POTASSIUM, SERUM`, `K+ LEVEL`, `Potassium Lvl`, and a hundred local variants across its lab feeds. **LOINC** is the international dictionary that assigns every lab test one agreed number, so `K+ LEVEL` and `POTASSIUM, SERUM` both resolve to the same LOINC code. **RxNorm** does the same job for drugs: a thousand brand names, package sizes, and local formulary strings all collapse to one normalized ingredient like `norepinephrine`. **SNOMED CT** does it for organisms and body sites, and **UCUM** does it for units of measure.

Here is the catch that defines this whole project. CLIF does not ask *"what LOINC code is this?"* It asks *"is this potassium?"* — using its own short, curated list of category names (for labs, 53 of them). CLIF's controlled vocabularies (the mCIDE, see [doc 01](01-what-is-clif.md)) ship as small CSVs listing a `category`, a plain-language `description`, and a few free-text `name_examples`. They contain **no LOINC column, no RxNorm column, no SNOMED column** — the only code field anywhere in the CLIF schema is one optional `labs.lab_loinc_code`. So the translation the world actually needs — *"LOINC 2823-3 → CLIF `lab_category = potassium`"* — has never been published by anyone. Every existing CLIF site built its own by hand.

The free national services (all run by the US National Library of Medicine, all usable at no cost) can *propose* a translation: they can suggest that this LOINC belongs to that group, or that this drug is a vasopressor. But a proposal is not an answer. A clinician has to look at each proposal and approve it, and once approved it must be **written down, dated, and never quietly changed** — because a lab value that silently jumps categories between two data pulls is a patient-safety and reproducibility problem. That approved, versioned list *is* the middleware. The dictionaries are raw material; the crosswalk is the product.

## Technical detail

**The core gap.** Each mCIDE CSV holds `<x>_category`, a `description`, and free-text `<x>_name_examples` — and zero code-system columns. Existing sites map their source data to `*_category` with hand-authored per-site tables (CSV / SQL / a shared Google Sheet). The reference ETL, [CLIF-MIMIC](https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC), keeps its mapping decisions in an **external Google Sheet**, not in code. There is no published crosswalk to reuse. This is the work. **[spec]**

**LOINC — labs, vitals, microbiology.**
- Free **LOINC FHIR terminology server**, base `https://fhir.loinc.org`, needs a free LOINC account (credentials sent on every session). Supports `$lookup`, `$expand`, `$validate-code`, and subsumption queries. <https://loinc.org/fhir/> **[spec]**
- The **Top 2000+ lab observations** (~2170 codes ≈ 98% of US lab order volume; shipped in SI and US-conventional versions) is a pragmatic shortlist to pre-seed `lab_category`. <https://loinc.org/usage/obs/> ; [Mapper's Guide](https://lhncbc.nlm.nih.gov/publication/dataset-loinc-top-2000-lab-observations-and-mappers-guide). **[spec]**
- LOINC **Parts / Groups / hierarchy** expose the COMPONENT axis, letting many local LOINCs cluster into one CLIF `lab_category`. <https://loinc.org/get-started/mapping-resources/> **[spec]**

**RxNorm / RxNav / RxClass — medications.**
- **Ingredient normalization** for `med_category` (CLIF wants active ingredients, e.g. `norepinephrine`): [RxNav](https://www.nlm.nih.gov/research/umls/rxnorm/overview.html) is free, no API key. [`findRxcuiById`](https://lhncbc.nlm.nih.gov/RxNav/APIs/api-RxNorm.findRxcuiById.html) maps NDC→RxCUI (`idtype=NDC`); then fetch the ingredient with `tty=IN+PIN+MIN` (IN = single ingredient, PIN = precise ingredient, MIN = multi-ingredient). Free-text drug strings normalize via `approximateTerm` / `getRxcuiByString`. **[spec]**
- **`med_group`** (CLIF groups: vasoactives, sedation, paralytics, anticoagulation, diuretics, cardiac, fluids_electrolytes, …) via [**RxClass**](https://rxnav.nlm.nih.gov/RxClassAPIs.html): `getClassMembers`, `getClassByRxNormDrugId`. Class systems available include **ATC** (WHO), **VA/VANDF**, **MED-RT** (MoA / PE / EPC / PK), **FDA EPC**, and SNOMED. Background: [RxClass intro](https://lhncbc.nlm.nih.gov/RxNav/applications/RxClassIntro.html). **[spec]**
- Honest per-group assessment (researched judgement, **[community]** on the ICU-specificity):
  - *paralytics*: ATC **M03** — cleanest single-axis match.
  - *anticoagulation*: ATC **B01A** — clean.
  - *diuretics*: ATC **C03** (+ a few others) — mostly clean.
  - *vasoactives*: MED-RT MoA "Vasoconstrictor Agents" / Physiologic Effect, or ATC C01CA — **neither is 1:1 with CLIF "vasoactives."** Expect a curated allow-list.
  - *sedation*: straddles ATC N05 (hypnotics/anxiolytics) + N01AX (propofol, ketamine) + N02A (opioids used for sedation). ATC alone under- or over-includes.
- **Net: class systems get ~70–90% of `med_group` for free; the ICU-specific groupings force a hand-curated override layer at the ingredient (IN RxCUI) level.** The right pattern: normalize to IN RxCUI first, then hand-map the *finite* set of ICU ingredients (dozens, not thousands) to `med_group`.

**SNOMED CT / UMLS — organisms, specimens, cross-vocabulary fallback.**
- SNOMED CT US Edition is free under a no-cost UMLS license — useful for `organism_category` (microbiology) and body-site/specimen (`fluid_category`). **[spec]**
- The **UMLS Metathesaurus + UTS API** ([uts.nlm.nih.gov](https://uts.nlm.nih.gov/)), free with the same license, is the cross-vocabulary bridge (LOINC ↔ SNOMED ↔ RxNorm ↔ ICD) and the fallback normalizer when a FHIR code arrives in an unexpected code system. **[spec]**
- CLIF's organism/fluid groupings follow the **NIH Common Data Elements**. **[community]**

**UCUM — units.** Free and mature.
- [**UCUM-LHC**](https://github.com/LHNCBC/ucum-lhc) (NLM/LHC), a JS validate-and-convert library, `@lhncbc/ucum-lhc` on npm; hosted at <https://ucum.nlm.nih.gov/ucum-lhc/>. **[spec]**
- [**NLM UCUM REST web service**](https://ucum.nlm.nih.gov/ucum-service.html). **[spec]**
- Python: [`pyucum`](https://pypi.org/project/pyucum/) (wraps the service, LOINC molecular-weight aware). Java: Eclipse UOMo, PixelMed. **[spec]**
- Powers `lab_value` → CLIF reference-unit conversion and med-dose normalization. Watch LOINC SI-vs-conventional differences (the Top-2000 SI list flags molar-unit tests). CLIF's schema carries `lab_reference_units` per category, and clifpy exposes `standardize_dose_to_base_units(med_df, vitals_df=...)`, producing mcg/min, ml/min, u/min and weight-based `/kg` using vitals weight (see [doc 12](12-clif-events-and-signals.md)). **[spec]**

**The OMOP route (evaluate honestly; do not recommend blindly).**
- OHDSI [**Athena**](https://athena.ohdsi.org/) vocabularies and **ATLAS concept sets** already encode large LOINC→measurement and RxNorm→ingredient/class maps.
- Maintained FHIR→OMOP ETLs exist: [FhirToCdm](https://github.com/OHDSI/FhirToCdm), [ETL-German-FHIR-Core](https://github.com/OHDSI/ETL-German-FHIR-Core), [OMOPonFHIR](https://omoponfhir.org/), [MENDS-on-FHIR](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10690355/); HL7 + OHDSI are co-developing an official FHIR-to-OMOP IG.
- **Verdict:** routing FHIR→OMOP→CLIF reuses OMOP's vetted standard-concept maps but adds a **second lossy hop**, and the terminal OMOP-concept→CLIF-category step is *still unbuilt* because CLIF categories are not OMOP concept classes. Reasonable as an *internal normalization layer*; not a free lunch. There is currently **no published OMOP→CLIF bridge** — the [CLIF paper](https://pmc.ncbi.nlm.nih.gov/articles/PMC11398431/) lists OMOP linkage as a future direction. **[spec]**

**Inventory: what exists vs what we build.**

| Need | Exists (reuse) | Must build |
|---|---|---|
| CLIF schema + permissible categories | mCIDE Explorer; clifpy validation | — (consume as-is) |
| Source→category maps | Only per-site examples (CLIF-MIMIC's Google Sheet) | **BUILD — this is the core IP** |
| Ingredient normalization for `med_category` | RxNav (`findRxcuiById`, `tty=IN+PIN+MIN`) | Thin cached wrapper over RxNav |
| `med_group` classes | RxClass (ATC / MED-RT / VANDF / EPC) | **BUILD — hand-curated ICU-ingredient override layer** |
| Lab grouping `lab_category` | LOINC FHIR server, Top-2000, Parts/Groups, OMOP concept sets | **BUILD — LOINC→lab_category grouping table** |
| Organism / fluid | SNOMED / UMLS, NIH CDE | Map candidates onto CLIF's lists |
| Units | UCUM-LHC | Config of reference units per category |
| `value[x]` flattening + BP component splitting | — | **BUILD** |
| FHIR→CLIF end to end | **Nothing** | **BUILD** |

The mapper engine that consumes these tables at runtime is specified in [doc 13](13-mapper-engine.md); the resource-shape flattening (`value[x]`, BP components) that feeds them is in [doc 12](12-clif-events-and-signals.md) and [doc 15](15-edge-cases.md).

## Edge cases and how it breaks

- **The curated-grouping mismatch is structural, not a tuning problem.** CLIF's `*_category` / `*_group` sets are clinically curated ICU groupings that **do not line up 1:1 with any single standard class axis.** ATC alone mis-scopes `sedation`; no single MED-RT axis equals CLIF `vasoactives`. No amount of RxClass/LOINC-Parts automation removes the final hand-curation — it only shrinks it to a tractable, finite list (dozens of ICU ingredients, hundreds of common LOINCs). Budget for that curation as permanent work, not a one-time seed.
- **The free services are candidate generators, not the answer.** RxNav, RxClass, the LOINC server, and Athena *propose*; they do not decide. Treat every automated suggestion as a draft pending clinician sign-off, and persist the approved result — never re-derive a category live from an external API at query time, or the same drug can silently change groups when the upstream vocabulary updates.
- **Session-credentialed dependency.** The LOINC FHIR server requires account credentials on every session, and UMLS/SNOMED require an accepted license; a build that hard-depends on live calls to these breaks in offline or air-gapped hospital environments. Cache and vendor the slices we use.
- **NDC and free-text misses.** `findRxcuiById` returns nothing for a retired or repackaged NDC; `approximateTerm` returns a *ranked guess* for free-text drug names and can be confidently wrong (e.g. a compounded or investigational agent). Both need a human-reviewed fallback path, not silent drop.
- **Unit conversion traps.** SI-vs-conventional (mass vs molar) differences mean a numerically valid UCUM conversion can still be clinically wrong if the reference unit for the category is mismatched; the Top-2000 SI list flagging molar-unit tests is the tell. Conversions must be pinned per `lab_category`, not applied globally.
- **The second-hop tax on OMOP.** Even where OMOP concept maps are excellent, the OMOP-concept→CLIF-category step is unbuilt and unowned; adopting OMOP does not import a finished CLIF map, it imports a *different* intermediate vocabulary we then still have to bridge by hand.

## What we still don't know

- Exactly how many ICU ingredients and how many distinct LOINCs the hand-curated override layers will contain in practice — "dozens" and "hundreds" are the researched estimates, not a counted inventory against a real site's formulary and lab compendium.
- Whether RxClass's `getClassByRxNormDrugId` coverage is complete enough for the ICU long tail (compounded drips, investigational agents, non-US formulary items) or whether large gaps force manual entry.
- How much of CLIF's `organism_category` and `fluid_category` can actually be driven from SNOMED/NIH-CDE candidates versus needing bespoke microbiology mapping — this was asserted as feasible but not verified end to end. **[unverified]**
- The precise licensing posture for redistributing cached LOINC/UMLS/SNOMED slices inside a deployed middleware artifact (as opposed to querying live) — the free-to-use terms are clear; the vendor-and-ship terms need a license read.
- Whether a future official HL7 FHIR-to-OMOP IG (in co-development) changes the calculus enough to make the OMOP route worth the second lossy hop — currently unresolved, and there is no published OMOP→CLIF bridge to test against.
- Governance of the crosswalk store itself — who approves, how versions are pinned to a data release, and how a re-categorization is audited — is a design question owned by [doc 13](13-mapper-engine.md), not answered here.
