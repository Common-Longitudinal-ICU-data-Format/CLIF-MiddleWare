# FHIR Bulk Data Access `$export` — Complete Protocol Walkthrough and Vendor Deviations

*The asynchronous "give me everything on this cohort" job: how the four-step order-and-collect protocol works on paper, where Epic and Oracle Health each bend it, and the two silent-correctness traps (`error[]` and `handling=lenient`) that a naive client will walk straight into.*

## In plain words

Bulk export is like asking the hospital records department for **"everything on this list of patients."** It is not a live question you get an instant answer to — it is an **order** you place and come back for. The interaction has four beats:

1. **You place the order** (the *kickoff*). You name the patient group and, optionally, which kinds of records you want and how far back. The server says "Accepted — come back later" and hands you a **claim ticket** (a URL to check on).
2. **You check back periodically** (*polling*). Each time you ask the claim-ticket URL "is it ready?" The server either says "still working, roughly half done, check again in 120 seconds" or "done — here is your pickup slip."
3. **You pick up the boxes** (*download*). The pickup slip lists a set of files to fetch. Each file is a stack of records of one kind. A single kind (say, all lab results) may be split across many boxes — never assume one box per record type.
4. **You read the note they stapled to the boxes** (*the error list*). Alongside the boxes, the slip has a short section listing **what they could not give you** — "patient 7's medications were not accessible," and so on. This note is the single most-ignored part of the whole exchange, and it is exactly where "we quietly got less than we asked for" hides.

Two plain-language warnings up front, because they cause wrong data rather than obvious failure:

- **The badge that lets you into the building expires after about five minutes** (the access token — see [doc 06](06-authentication.md)). A big export can take much longer than five minutes to build. So you must keep renewing your badge *while you are still waiting in the lobby*, or you will be locked out mid-pickup.
- **"Be lenient" politeness can silently drop your filter.** There is a way to tell the server "if you don't understand one of my requests, just ignore it rather than rejecting the whole order." That sounds friendly, but it means you can ask for *only laboratory results*, have the server silently ignore that narrowing, and hand you *everything* — and you will believe you filtered when you did not. This is not a crash; it is bad data that looks fine. Treat it as a first-class hazard.

Which of the three ways to place an order actually works depends entirely on the vendor, and at Epic the cohort itself is built for you, by a hospital analyst, out of band — you cannot invent an arbitrary patient list. That constraint shapes our whole cohort strategy; see [doc 10](10-cohort-and-encounter-class.md) and the access-mode comparison in [doc 04](04-api-access-modes.md).

## Technical detail

The governing standard is the **HL7 FHIR Bulk Data Access Implementation Guide**. **v2.0.0** is what US vendors target today; Oracle Health/Cerner's Group export still conforms to **v1.0.1** with experimental v2.0.0 parameters bolted on. Canonical spec: [hl7.org/fhir/uv/bulkdata](https://hl7.org/fhir/uv/bulkdata/) and the [export operation](https://hl7.org/fhir/uv/bulkdata/export.html); continuous build at [build.fhir.org export](https://build.fhir.org/ig/HL7/bulk-data/export.html); the SMART-backsystem-authorization flow at [the authorization guide](https://hl7.org/fhir/uv/bulkdata/authorization/index.html). **[spec]**

### 1. Kickoff

Three kickoff endpoints exist in the spec **[spec]**:

| Level | Request | Notes |
|---|---|---|
| System | `GET [base]/$export` | Everything the client is authorized for. Effectively not offered by Epic or Oracle. |
| Patient | `GET [base]/Patient/$export`, or `POST` with a `patient` parameter list | All-patient GET is blocked at both major vendors; the POST-with-explicit-ID-list form is Oracle's supported path. |
| Group | `GET [base]/Group/[id]/$export` | **The only mode that works at Epic.** |

**Required kickoff headers** **[spec]**:

- `Accept: application/fhir+json`
- `Prefer: respond-async` — **REQUIRED**; this is what makes it an async job rather than a synchronous read.
- optional `Prefer: handling=lenient` — server *ignores* unsupported parameters instead of erroring. Oracle honors this; **without it, an unsupported parameter returns `422 Unprocessable Entity`.** (See the hazard note in Edge cases — lenient handling turns a loud 422 into a silent dropped filter.)

**Kickoff query parameters** **[spec]**:

| Param | Meaning |
|---|---|
| `_outputFormat` | NDJSON media type. `application/fhir+ndjson` (default), `application/ndjson`, or bare `ndjson`. |
| `_since` | An `instant`; return only resources whose `meta.lastUpdated` is after this. |
| `_type` | Comma-delimited resource types, e.g. `_type=Patient,Encounter,Observation`. Omit to get all supported types. |
| `_typeFilter` | Per-type FHIR search filter, URL-encoded; the search context is a *single* resource type. E.g. `_typeFilter=Observation%3Fcategory=laboratory` or `_typeFilter=Encounter%3Fdate=ge2026-01-01`. A server that cannot honor a given `_typeFilter` *SHOULD* return an `OperationOutcome`. |
| `_elements` | Element subset, comma-delimited, optionally `[type].[element]`. Support is uneven across vendors. |
| `includeAssociatedData` | v2.0.0, **experimental**; e.g. `LatestProvenanceResources`, `RelevantProvenanceResources`. [ValueSet](https://build.fhir.org/ig/HL7/bulk-data/ValueSet-include-associated-data.html). |
| `organizeOutputBy` | v2.0.0, optional; rarely supported by EHRs. |
| `patient` | Patient-level POST only — the explicit ID list. |

A minimal Epic-shaped kickoff, using a `_typeFilter` date range because Epic lacks `_since`:

```
GET [base]/Group/systemlist|1234/$export?_type=Encounter&_typeFilter=Encounter%3Fdate=ge2026-01-01T00:00:00Z%26date=lt2026-02-01T00:00:00Z
Accept: application/fhir+json
Prefer: respond-async
Authorization: Bearer <token>
```

**Kickoff response** **[spec]**: `202 Accepted`, with a `Content-Location:` header carrying the **absolute polling URL**. There is no meaningful body (possibly an `OperationOutcome`).

### 2. Polling

`GET` the `Content-Location` URL with `Accept: application/json` **[spec]**:

- **In progress** → `202 Accepted`. Headers: `X-Progress:` (free text, e.g. `"50% complete"` — **advisory only, unparseable**) and `Retry-After:` (seconds or an HTTP-date — **honor it**; fall back to exponential backoff if absent).
- **Complete** → `200 OK` plus the **completion manifest** (JSON):

| Field | Meaning |
|---|---|
| `transactionTime` | An `instant`: the server's snapshot time. **This is your next watermark — never wall-clock time.** |
| `request` | The original kickoff URL, echoed. |
| `requiresAccessToken` | Boolean. If `true`, send the bearer token when downloading NDJSON. |
| `output[]` | Array of `{ type, url, count? }`. The boxes. |
| `error[]` | Array of `{ type: "OperationOutcome", url }`. **Download and inspect every one** — this is where "resource X not accessible" hides. |
| `deleted[]` | v2.0.0: resources deleted since `_since`. Thin EHR support. |

- **Error** → 4xx/5xx with an `OperationOutcome` body.

### 3. Download

`GET` each `output[].url` with `Accept: application/fhir+ndjson` (plus bearer if `requiresAccessToken`) **[spec]**. **NDJSON** = one FHIR resource per line, newline-delimited. A single logical type may be split across many files, so iterate `output[]` — do not key downloads by type name.

### 4. Cancel

`DELETE` the status URL → `202 Accepted` **[spec]** (Oracle returns `204`). Use this to abandon a job you no longer want the server building.

### Sequence walkthrough

```
CLIENT                                   SERVER
  |  GET .../Group/[id]/$export            |
  |  Prefer: respond-async                 |
  |--------------------------------------->|
  |          202 Accepted                  |
  |   Content-Location: <status-url>       |
  |<---------------------------------------|
  |                                        |
  |  GET <status-url>                      |
  |--------------------------------------->|
  |   202 Accepted                         |
  |   X-Progress: "50% complete"           |
  |   Retry-After: 120                     |
  |<---------------------------------------|
  |        ...wait Retry-After, re-mint token if near expiry...
  |  GET <status-url>                      |
  |--------------------------------------->|
  |   200 OK  + completion manifest        |
  |   { transactionTime, output[], error[] }
  |<---------------------------------------|
  |                                        |
  |  GET output[0].url  (NDJSON)           |
  |  GET output[1].url  ...  (N files)     |
  |--------------------------------------->|
  |   200 OK  application/fhir+ndjson      |
  |<---------------------------------------|
  |                                        |
  |  GET error[i].url   <-- DON'T SKIP     |
  |--------------------------------------->|
  |   200 OK  OperationOutcome NDJSON      |
  |<---------------------------------------|
  |                                        |
  |  DELETE <status-url>  (release job)    |
  |--------------------------------------->|
  |   202 (Epic) / 204 (Oracle)            |
  |<---------------------------------------|
```

### Epic deviations

All **[community]** unless noted. Sources: [SMART/Cumulus discussion](https://github.com/smart-on-fhir/cumulus/discussions/5), [FHIR chat archive](https://chat-archive.fhir.org/stream/179166-implementers/topic/Epic.20Bulk.20Data.20Export.20questions.3F.html), [interopiO article](https://support.interopio.com/hc/en-us/articles/4419488428052), [fhir.epic.com bulk docs](https://fhir.epic.com/Documentation?docId=fhir_bulk_data).

- **Group-level `$export` only.** No system-wide, no all-patient. Launched in the Aug 2021 release.
- **The Group is created out-of-band by a hospital Epic analyst** (a Registry or patient list); they email you the Group ID. IDs look like `mylist|<id>` (My List) or `systemlist|<id>` (system/registry list). You cannot create or query arbitrary cohorts. This is the hinge of our cohort design — [doc 10](10-cohort-and-encounter-class.md).
- **`_since` is NOT implemented.** Use date ranges inside `_typeFilter` instead: `?_type=Encounter&_typeFilter=Encounter?date=ge2026-01-01T00:00:00Z&date=lt2026-02-01T00:00:00Z`.
- **`meta.lastUpdated` is not reliably populated** in bulk output → unusable as a watermark. See [doc 09](09-data-freshness.md).
- **Kickoff throttle:** same Group + same client roughly **once per 24 hours**. Error text: *"Request not allowed: The Client requested this Group too recently."* Governed by a tunable server-side window (`FHIR_BULK_CLIENT_REQUEST_WINDOW_TBL`).
- **~1000 patients per export** recommended.
- **Hidden nested Observations:** resources referenced by `DiagnosticReport.result` and `Observation.hasMember` are **not returned by default**, and Epic supports **neither `_include:iterate` nor `_revinclude`**. Workaround: add `&_include=Observation:hasMember:Observation`, export `Observation` alongside `DiagnosticReport`, and **stitch by reference** yourself.
- **Epic FHIR ids can exceed the FHIR 64-char `id` limit** (there is an Epic toggle). Size warehouse keys accordingly.
- Returns HTTP **429** under load.
- **Exported types ≈ 20**, essentially the US Core set (see [doc 03](03-fhir-resource-types.md)). Notably, **`MedicationAdministration` is NOT among them** — a major gap for an ICU dataset, and the reason bulk must be paired with REST enrichment ([doc 04](04-api-access-modes.md)).
- **App permissions are four separate Bulk Data operations** — *Kick-off, Status, File retrieval, Delete*. Register all four or the job stalls partway.

### Oracle Health / Cerner deviations

**[spec]** — [Oracle Millennium bulk export API](https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfbda/api-bulk-export.html).

- **Group export** = Bulk IG **v1.0.1** + experimental v2.0.0 params. Groups are provisioned via **Ignite Management Tooling**.
- **Patient export** = v2.0.0, `POST /Patient/$export`. **All-patient export is NOT allowed**; an explicit patient-ID list is required. **Max 20,000 patients per request** (≤10,000 recommended).
- **System-level `$export`** effectively not offered.
- **`_since` IS supported** (unlike Epic).
- Status URL shape: `/bulk-export/jobs/{jobId}`. Cancel `DELETE` → `204`.
- **Retention: 30 days** for completed job output.
- Error codes: `400` invalid, `401` login/expired, `403` forbidden, `404` not-found, `406` (bad `Accept`), `415` (bad `Content-Type` on Patient export), `422` (unsupported search-param values), `429` throttled, `500` exception.

## Edge cases and how it breaks

- **Token expiry mid-export (the recurring one).** Epic/SMART access tokens live roughly **5 minutes**; an export can take far longer to build and download. You must **re-mint the token while polling** and before each download, keyed off actual expiry, not a fixed schedule. A job that kicks off fine and then 401s on file retrieval is almost always this. Full mechanics in [doc 06](06-authentication.md). **[community]**
- **`Prefer: handling=lenient` silently drops your filter — a correctness bug, not an error.** With lenient handling, a `_typeFilter` or `_type` the server doesn't understand is *ignored*, and you get a `200` with *more* data than you asked for. You will believe you filtered to laboratory results and you did not. **Prefer strict handling (or omit `lenient`) so an unsupported param throws `422`, and always reconcile the returned `output[]` types against what you requested.** Do not trust that a successful job honored your narrowing. **[spec]** / **[unverified]** (that vendors uniformly implement lenient this way).
- **`error[]` is silently ignored by naive clients.** A client that reads only `output[]` will happily ingest a partial dataset and never learn that, say, one department's medications were withheld. **Always download every `error[].url`, parse the `OperationOutcome` lines, and surface them.** This is the records-department note nobody reads. **[spec]**
- **A type spans many NDJSON files.** `output[]` may list ten `Observation` entries. Iterate the array; never assume one file per type, and never dedupe by filename. **[spec]**
- **`count` is optional.** `output[].count` may be absent, so you cannot rely on it for a completeness check or a progress bar — you must count lines as you download. **[spec]**
- **`X-Progress` is free text.** `"50% complete"`, `"halfway"`, or anything else. Advisory only; never parse it or branch on it. Drive your loop off `Retry-After` and the `202` vs `200` status alone. **[spec]**
- **Files expire, and the 24h throttle makes a lost download expensive.** Manifest URLs are retained for a limited window (Oracle: **30 days**; Epic unspecified). Combined with Epic's **once-per-24h** kickoff throttle, a corrupted or half-finished download is costly to redo — so **download and checksum every file immediately** on manifest receipt, before releasing the job, so you can retry from the still-valid manifest rather than re-kicking off. **[community]**
- **`transactionTime` vs wall-clock watermark.** The next incremental run's lower bound must be the manifest's `transactionTime`, **not** the clock time you started or finished. Wall-clock creates gaps or overlaps against the server's actual snapshot boundary. At Epic, where `_since` doesn't exist *and* `meta.lastUpdated` is unreliable, you cannot even do true incremental bulk — you fall back to `_typeFilter` date-window slicing and accept re-pulls. See [doc 09](09-data-freshness.md). **[spec]** / **[community]** (the Epic caveat).
- **422 on kickoff without lenient handling.** Oracle returns `422` for unsupported `_typeFilter` search-param values; a param your code assumes is universal may reject the whole job on one vendor. Probe capabilities per vendor rather than assuming. **[spec]**
- **DELETE status varies.** `202` at Epic, `204` at Oracle. Don't hard-code the expected cancel status. **[spec]**

## What we still don't know

- **Epic file-retention window.** Oracle documents 30 days; Epic's manifest/NDJSON expiry is not clearly published. Needs empirical measurement against the target instance. **[unverified]**
- **Exact behavior of `Prefer: handling=lenient` per vendor** — whether every server truly drops-and-continues vs. partially honors, and whether the dropped param is ever echoed in an `OperationOutcome` warning. Assume the worst (silent) until verified on the live endpoint. **[unverified]**
- **Whether the `_include=Observation:hasMember:Observation` workaround captures *all* nested Observations** (e.g. multi-level `hasMember` nesting or `DiagnosticReport.result` chains that Epic's non-iterating `_include` won't traverse). Stitching completeness must be validated against known-multi-component panels. **[unverified]**
- **The real value of the Epic throttle window** (`FHIR_BULK_CLIENT_REQUEST_WINDOW_TBL`) at *our* hospital — "roughly 24h" is a default that a site admin can tune; confirm the actual configured window before designing the run cadence. **[community]**
- **Whether Epic's ~20 exported types at our site match the generic US Core list** and exactly which are present — must be read from the live `$export` output and the server's CapabilityStatement, not assumed (US Core is a floor; see [doc 03](03-fhir-resource-types.md)). **[unverified]**
- **Oracle Group-export param support** — which experimental v2.0.0 params (e.g. `includeAssociatedData`, `organizeOutputBy`) the v1.0.1-based Group endpoint actually accepts vs. rejects with `422`. **[unverified]**
