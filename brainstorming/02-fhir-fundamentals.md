# FHIR Fundamentals

*What FHIR is, which version to build against, and the US regulatory stack that forces hospitals to expose it — with the single load-bearing idea that US Core is a floor, not a ceiling.*

## In plain words

FHIR stands for **Fast Healthcare Interoperability Resources**. It is a standard published by **HL7** (Health Level Seven, the standards body that has governed healthcare data exchange for decades). FHIR is the modern way hospital software lets outside programs read clinical data over the web, using the same kind of technology that powers ordinary websites and phone apps.

The core idea is the **resource**. A resource is a single, self-contained record about one clinical thing: a `Patient`, an `Observation` (a lab result or a vital sign), an `Encounter` (a hospital stay or visit), a `MedicationAdministration`, and so on. Each resource arrives as a **JSON document** — plain text structured as labelled fields, readable by both people and machines. Every resource carries an `id` (its unique name on that server) and a small `meta` block that records housekeeping facts, most importantly `lastUpdated`, the timestamp of the last change to that record. We will come back to that timestamp; it matters more than it looks.

Resources do not repeat each other's data. Instead they **point** at each other with references. An `Observation` for a potassium level does not copy the patient's name into itself; it says `"subject": {"reference": "Patient/abc123"}`, meaning "this result belongs to the patient whose id is abc123." To assemble a complete clinical picture you follow these pointers from resource to resource, the way web pages link to one another.

When you ask a FHIR server a question — for example, "all lab observations for this patient" — it does not hand back a loose pile of records. It returns a **Bundle**: one envelope containing an `entry[]` list of the matching resources. Because a patient may have thousands of results, the server returns them a page at a time and includes a `link[]` section; when that section contains a link marked `relation: "next"`, there is another page to fetch. You keep following "next" until it disappears. Handling this paging correctly is routine but non-optional.

Two more ideas a physician should hold onto. First, a **profile** (packaged into an **Implementation Guide**, or **IG**) is a rulebook that narrows the generic FHIR standard for a specific country or purpose. Plain FHIR says a `Patient` *may* have a race field; a US profile can say a `Patient` *must* support one, coded a particular way. The IG that matters in the United States is **US Core**. Second, within a profile each field is tagged with an obligation. The important one is **"must support"**: it means "if your system holds this piece of data, your API is required to expose it here." It does not promise the data exists for every patient — only that the plumbing is present when the data is.

Now the regulatory stack, which is why any of this is available to us at all. US federal rules (the **ONC** / **ASTP** Health IT Certification Program — Office of the National Coordinator, recently renamed the Assistant Secretary for Technology Policy) require certified hospital record systems to expose a standardized FHIR API. The specific rule, criterion **§170.315(g)(10)**, forces every certified system to serve a defined set of data, both one patient at a time and for whole populations at once, using FHIR version 4.0.1 shaped by US Core. The list of *what* data must be available is a separate standard called **USCDI** (United States Core Data for Interoperability). This is why Epic, Oracle Health (formerly Cerner), athenahealth, and MEDITECH all offer a FHIR endpoint at all: the government requires it.

Here is the single most important idea in this entire document, and the reason our whole architecture is shaped the way it is. **US Core and USCDI are a floor, not a ceiling.** Certification says "you must serve *at least* this." It never says "you may serve *only* this." Epic, in practice, exposes roughly 55 resource types across about 450 endpoints — vastly more than US Core requires. So you can never reason "the hospital doesn't have data X because X isn't in US Core." That inference is invalid. The only way to know what a given server holds is to **ask that server**. (See [doc 03](03-fhir-resource-types.md) and [doc 08](08-what-hospitals-actually-provide.md).)

And then the sting. The floor is not applied evenly across the two access paths. Epic's ordinary one-patient-at-a-time REST API is wide and generous. But Epic's **Bulk Export** — the population path that lets you pull a whole ICU cohort in one job — is restricted roughly to the narrow US Core set. So the floor effectively binds bulk but not REST. That asymmetry is the hinge of our design: **use Bulk Export to cheaply find and outline the cohort, then use the wider REST API to enrich each patient with the ICU-specific detail bulk won't give us.** [Doc 05](05-bulk-export-deep-dive.md) works through the bulk mechanics.

One last plain-language warning about that `lastUpdated` timestamp. It is meant to be our **watermark** for incremental loading: on each run we would ask only for records changed since last time, instead of re-pulling everything. That only works if the server sets the timestamp faithfully — and Epic reportedly does not populate it reliably. [Doc 09](09-data-freshness.md) is where that problem lives.

## Technical detail

**What FHIR is.** FHIR (Fast Healthcare Interoperability Resources) is an HL7 standard for representing and exchanging health data as RESTful resources. Each resource is a JSON (or XML) document with a `resourceType`, an `id`, a `meta` block containing at minimum `versionId` and `lastUpdated`, and typed data fields. Inter-resource links are reference strings, e.g. `"subject": {"reference": "Patient/abc123"}`. A search operation returns a `Bundle` with `type: "searchset"`, an `entry[]` array of matched resources, and a `link[]` array; pagination is driven by the entry whose `relation` is `"next"`. Canonical spec: <https://build.fhir.org/>. **\[spec\]**

**Versions — build to R4 (4.0.1) only.**

| Version | Number | Status for a US hospital ICU warehouse |
|------------------------|------------------------|------------------------|
| DSTU2 | 1.0.2 | Legacy / deprecated. Do not target. |
| STU3 | 3.0.x | Legacy / deprecated. Do not target. |
| **R4** | **4.0.1** | **Universal production standard. What every major US EHR serves for its certified API. Build to this.** |
| R4B | 4.3.0 | Maintenance branch. Not what US EHRs serve. |
| R5 | 5.0.0 (2023) | Not served by US hospital EHRs for production clinical data. Some greenfield servers (Medplum, HAPI, Google/Azure FHIR) support it. |
| R6 | — | In ballot as of 2026. Not in production. |

FHIR 4.0.1 is the version named in US certification, and Epic, Oracle Health/Cerner, athenahealth, and MEDITECH Expanse all serve R4 for the certified API. **\[spec\]** Epic developer reference: <https://open.epic.com/Interface/FHIR>.

**The US regulatory stack (what is legally in force as of this writing, 2026-07).**

- **Certification criterion:** **§170.315(g)(10)** "Standardized API for patient and population services." Requires certified health IT to serve, over FHIR 4.0.1 + US Core, both single-patient and **population** (multi-patient) access to all USCDI data. Authentication/authorization is **SMART App Launch**, including **SMART Backend Services** (OAuth 2.0 `client_credentials`) for the unattended server-to-server path. The population path must implement the **HL7 FHIR Bulk Data Access IG**. This criterion is the legal reason every certified hospital EHR exposes a Bulk Data `$export` operation. <https://onc-healthit.github.io/api-resource-guide/g10-criterion/> **\[spec\]**
- **Implementation Guide:** **US Core 6.1.0** is the version in force under the **HTI-1 Final Rule** as of **January 1, 2026**, paired with **USCDI v3** and **SMART App Launch 2.0.0**. US Core 3.1.1 and USCDI v1 were retired. <https://hl7.org/fhir/us/core/STU6.1/> and <https://www.healthit.gov/test-method/standardized-api-patient-and-population-services>. **\[spec\]**
- **Enforcement timeline:** ONC/ASTP granted enforcement discretion from **Jan 1, 2026 – Feb 28, 2026**; certified modules had until **March 1, 2026** to comply. As of July 2026 that window has closed and 6.1.0/USCDI v3 is fully enforced. <https://www.healthit.gov/topic/certification-criteria-compliance-dates-enforcement-discretion-notice>. **\[spec\]**
- **Voluntary advancement:** Later US Core versions (7.x / USCDI v4, 8.0.0, 9.0.0) may be adopted voluntarily via ONC's **SVAP** (Standards Version Advancement Process). **6.1.0 / USCDI v3 is the mandated floor**; Epic runs ahead of it via SVAP. **\[spec\]**
- **Conformance testing:** the ONC (g)(10) **Standardized API Test Kit**, known as **Inferno**, is the reference validator. <https://github.com/onc-healthit/onc-certification-g10-test-kit>. **\[spec\]**

**USCDI (the data-content standard).** USCDI = United States Core Data for Interoperability, maintained at <https://isp.healthit.gov/united-states-core-data-interoperability-uscdi>. **\[spec\]**

| USCDI version | Scale / date |
|------------------------------------|------------------------------------|
| v3 | \~16 data classes / \~94 data elements. **The mandated floor.** |
| v4 (2023) | Added \~20 elements. |
| v5 (Jul 2024) | Added \~16 elements + 2 classes. Approved for voluntary SVAP \~Aug 2025. |
| v6 | Published Jul 2025. |
| v7 (draft) | Published Jan 29, 2026. |

USCDI data classes relevant to ICU work: Patient Demographics, Encounter Information, Vital Signs, Laboratory, Diagnostic Imaging, Clinical Notes, Medications, Immunizations, Allergies & Intolerances, Problems, Procedures, Health Status/Assessments, Care Team, Provenance.

**Critical USCDI gaps for ICU.** The following are **not** standardized USCDI data elements: high-frequency device/waveform data, ventilator settings, continuous infusion rates, and RASS/GCS (Richmond Agitation-Sedation Scale / Glasgow Coma Scale) as standardized elements. These are exactly the fields CLIF's ICU-specific tables need, and they are precisely the data the bulk floor does not guarantee — which is why REST enrichment ([doc 03](03-fhir-resource-types.md)) is not optional for us.

**Scope note — CMS rules do not apply here.** The CMS Interoperability rules (Patient Access API, CMS-0057 Prior Authorization) bind **payers** (insurers), not hospital clinical servers. For a hospital-side ICU warehouse the operative mandate is **ONC §170.315(g)(10) + US Core**, not the CMS payer APIs. Do not design against CMS-0057.

**Floor-vs-ceiling, stated precisely.** Certification obliges a *minimum* conformant surface (US Core profiles over USCDI elements). Servers routinely exceed it: Epic exposes \~55 resource types across \~450 endpoints. Therefore server capability must be discovered empirically, not inferred from US Core. Crucially the obligation is *not uniform across access paths*: Epic's REST surface is wide, but its Bulk Export is constrained roughly to the US Core resource set. Architecture consequence: **bulk to find the cohort, REST to enrich it** — detailed in [doc 05](05-bulk-export-deep-dive.md) and [doc 08](08-what-hospitals-actually-provide.md).

## Edge cases and how it breaks

- **Version/format negotiation.** Always send `Accept: application/fhir+json` (and `Content-Type: application/fhir+json` on writes, which we do not do). Omitting it can yield XML, an HTML error page, or a legacy media type depending on server defaults. R4 servers may also honor a version parameter on the JSON media type; do not rely on it — pin behavior by testing the real endpoint. **\[community\]**
- **"R4" alone is under-specified.** Knowing a server is FHIR 4.0.1 tells you *nothing* about which US Core version it serves. R4 is the wire format; US Core 6.1.0 (or 7.x, 8.0.0, 9.0.0 via SVAP) is the content contract layered on top. You must determine the US Core version separately, per endpoint. **\[spec\]**
- **SVAP divergence across "identical" systems.** Because SVAP is voluntary and per-deployment, two hospitals both running Epic may serve *different* US Core / USCDI versions. One might be on the 6.1.0 floor and another ahead on 7.x or 9.0.0. Do not assume "it's Epic" implies a single behavior; profile each site. **\[spec\]**
- **The CapabilityStatement can lie in both directions.** The server's `/metadata` endpoint returns a `CapabilityStatement` that is supposed to enumerate supported resources, search parameters, and operations. In practice it can **overstate** availability (lists a resource or search param that returns errors, empty results, or is access-blocked for your client) **or understate** it (omits things the server actually serves). Treat `/metadata` as a hint, then verify each resource and search parameter you depend on with a real query. **\[community\]**
- **`meta.lastUpdated` as an unreliable watermark.** The incremental-load design depends on `meta.lastUpdated` reflecting true last-modification time so we can pull only changed records. Epic reportedly does not populate `lastUpdated` reliably, which can silently miss updates or force full re-pulls. This is a first-order risk to freshness, tracked in [doc 09](09-data-freshness.md). **\[community\]**
- **"Must support" ≠ "data present."** A field marked must-support guarantees the API *exposes* the element when the system holds it — not that any given patient has a value. Absent fields are normal and must not be treated as errors. **\[spec\]**
- **Paging must be followed to exhaustion.** A `searchset` Bundle's first page is not the full result. Stop only when no `link` with `relation: "next"` remains. Assuming page one is complete silently truncates cohorts. **\[spec\]**
- **The bulk/REST floor asymmetry is a data-completeness trap.** A pipeline that trusts Bulk Export alone will look conformant yet omit ventilator settings, infusion rates, and sedation scores — the ICU signal — because those sit outside the bulk floor. The failure is silent: valid FHIR, passing conformance, missing clinical content. **\[community\]**

## What we still don't know

- **Exactly which US Core / USCDI version each target hospital serves**, and whether they have moved ahead of the 6.1.0 / USCDI v3 floor via SVAP. Resolve by reading each site's `CapabilityStatement` and published implementation notes, then confirming empirically. **\[unverified\]**
- **Whether `meta.lastUpdated` is trustworthy on our specific target endpoints.** The unreliability is community-reported and vendor-dependent; it must be measured against each real server before we commit to a watermark-based incremental design. Escalated to [doc 09](09-data-freshness.md). **\[unverified\]**
- **The precise resource set exposed by each hospital's Bulk Export vs. its REST API**, i.e. exactly where the floor/ceiling gap falls per site. General claim is community knowledge; the per-endpoint boundary must be tested. Tracked in [doc 05](05-bulk-export-deep-dive.md) and [doc 08](08-what-hospitals-actually-provide.md). **\[unverified\]**
- **How faithfully each server's `/metadata` reflects reality** (over/understatement, per bullet above) — determinable only by probing each declared capability. **\[unverified\]**
- **Which non-USCDI ICU elements are retrievable at all, and via which resources/extensions** (ventilator settings, infusion rates, RASS/GCS, waveforms). This is the crux question for CLIF feasibility and is worked in [doc 03](03-fhir-resource-types.md). **\[unverified\]**