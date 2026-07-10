# Middleware architecture

*A systemd service, a state machine, and a web console. Designed so the data source is pluggable, every stage is resumable, and the expensive stage runs on the smallest possible cohort.*

---

## In plain words

The middleware is a program that runs continuously on a Linux server inside the hospital's network. Nobody logs into it to make it work. It wakes up on a schedule, talks to the hospital's records system, pulls whatever is new, files it away, and goes back to sleep. `systemd` is the part of Linux that keeps such programs running, restarts them if they crash, and hands them their secrets safely.

Alongside it runs a small website, reachable only from inside the hospital, which does three jobs: it lets an administrator configure and watch the service, it shows a clinician the mapping decisions that need review, and it reports how much of the data the middleware successfully understood.

The work happens in stages, and the ordering of those stages is the most important design decision in the whole system.

First the service **asks the hospital's server what it can do**, rather than assuming. Then it **proves who it is** using a cryptographic key. Then it runs a **bulk download** — cheap, wide, once a day at most — which gets demographic, encounter, lab, and diagnosis data for everyone in the requested group. Then it **filters** that population down to just the patients who were admitted as inpatients or placed under observation, which typically removes the great majority of them. Only then, on that much smaller group, does it make the **expensive per-patient requests** for the ICU-specific data that the bulk download refuses to give.

That order is not stylistic. Making the expensive calls before the filter would cost ten to fifty times as much and would take correspondingly longer, against a server that throttles you. Filtering first is what makes the design viable at all.

Everything the service receives lands, unmodified, in an append-only log. Nothing is thrown away and nothing is edited. The CLIF tables that researchers actually use are *derived* from that log by a purely mechanical process. If a translation turns out to be wrong, we fix the translation and rebuild the tables — we never have to go back and ask the hospital for the data again. That matters more than it sounds, because the hospital's server will only give us a full download about once a day.

Finally, every stage can be interrupted and resumed. Networks fail, tokens expire mid-download, servers restart. A pipeline that must start over from the beginning after any failure is a pipeline that never finishes.

---

## Technical detail

### Process model

A single long-lived service under systemd, with an internal scheduler, plus a separate web process.

```
clif-middleware.service      # the daemon: discover, auth, extract, land, transform
clif-middleware-web.service  # the console: config, mapping review, monitoring
clif-middleware.timer        # optional: nightly kickoff, if not using an internal scheduler
```

The daemon and the web process share the database and nothing else. The web process must never hold the private key; it cannot call the FHIR server. That separation means a web vulnerability does not become a PHI extraction.

Key material is delivered by systemd, not read from disk by the application:

```ini
[Service]
LoadCredentialEncrypted=fhir-private-key:/etc/clif/keys/fhir.key.cred
```

The key is decrypted only into the unit's private `/run/credentials/<unit>` tmpfs, is not world-readable, and is optionally TPM-bound via `systemd-creds`. See [doc 06](06-authentication.md) and [doc 16](16-security-and-governance.md).

### The pipeline as a state machine

Every run is a row in a `run` table. Every stage transition is durable. The service can be killed at any point and resumed from the last completed stage.

```
                 ┌──────────┐
                 │ DISCOVER │  GET /metadata, /.well-known/smart-configuration
                 └────┬─────┘  cache CapabilityStatement; diff against last run
                      ▼
                 ┌──────────┐
                 │   AUTH   │  sign client_assertion → client_credentials → token
                 └────┬─────┘  token cached, ~5 min TTL, re-minted on demand
                      ▼
                 ┌──────────┐
                 │ KICKOFF  │  GET Group/[id]/$export?_type=…&_typeFilter=…
                 └────┬─────┘  persist Content-Location BEFORE returning
                      ▼
                 ┌──────────┐
                 │   POLL   │  honor Retry-After; re-mint token; survive restart
                 └────┬─────┘  on 200 → persist manifest + transactionTime
                      ▼
                 ┌──────────┐
                 │ DOWNLOAD │  fetch output[] NDJSON; checksum; persist
                 └────┬─────┘  ALSO fetch and parse error[] — never skip this
                      ▼
                 ┌──────────┐
                 │  LAND    │  parse NDJSON → clif_events (upsert on id+versionId)
                 └────┬─────┘  update clif_signals catalog
                      ▼
                 ┌──────────┐
                 │  FILTER  │  Encounter.class ∈ {IMP, ACUTE, NONAC, OBSENC}
                 └────┬─────┘  → cohort table. Cohort rule is a versioned config value
                      ▼
                 ┌──────────┐
                 │  ENRICH  │  REST, cohort-scoped only:
                 └────┬─────┘  MedicationAdministration, Observation (SDE/LDA/Assessments),
                      │        Consent. Rate-limited, resumable per (patient, resource type)
                      ▼
                 ┌──────────┐
                 │   MAP    │  MapperEngine per table: propose → human approve
                 └────┬─────┘  BLOCKS on human input; not part of the nightly path
                      ▼
                 ┌──────────┐
                 │ TRANSFORM│  clif_events ⋈ signal_map → clif_*.parquet
                 └────┬─────┘  deterministic, offline, no model
                      ▼
                 ┌──────────┐
                 │ VALIDATE │  clifpy .validate() → errors, coverage report
                 └──────────┘
```

**`MAP` is not in the nightly path.** It blocks on a human, and humans are not available at 02:00. The nightly run lands events, updates the signal catalog with any newly-seen signals, transforms whatever is already mapped, and emits a coverage report that says how many new signals await review. Mapping review happens on human time. New unmapped signals do not stall the pipeline; they show up as a coverage gap.

### Why the filter sits between bulk and enrich

Bulk export gives roughly the US Core resource set for everyone in the hospital's Group ([doc 08](08-what-hospitals-actually-provide.md)). The Group is defined by a hospital analyst and will typically be broader than our cohort — we should in fact *ask* for it to be broader, so that our cohort definition stays client-side, versioned, and reproducible ([doc 10](10-cohort-and-encounter-class.md)).

Enrichment costs approximately one HTTP round trip per patient per resource type. On the full Group that is prohibitive and will trip rate limits. On inpatients and observation patients it is a few thousand calls, spread over hours, well within throttle budgets.

So: **bulk is O(1) requests for O(N) patients; REST is O(N) requests.** Shrink N before paying the O(N) cost. Concretely, if a Group contains all 40,000 encounters at a hospital over 90 days and 4,000 of them are inpatient or observation, the filter is a 10× cost reduction on the entire enrichment leg.

### Idempotency and resumability

Every stage is designed so that re-running it is safe.

- **Kickoff**: persist the `Content-Location` polling URL *before* the function returns. If the process dies between the server accepting the job and us recording the URL, the export runs to completion on the server and is unreachable — and under a 24-hour throttle, that costs a day. This is the single most important durability point in the system.
- **Poll**: the polling URL is durable state. A restart resumes polling. Re-mint the access token as needed; it will expire mid-poll (~5 min TTL) on any real export.
- **Download**: content-addressed storage. Checksum each NDJSON file on arrival and record the hash. A partial download is detectable and re-fetchable. Files expire (Oracle documents 30 days; treat Epic as *immediately*), so download promptly and never rely on being able to re-fetch.
- **Land**: upsert on `(source_resource_id, source_version_id)`. Because Epic incrementals use overlapping date windows ([doc 09](09-data-freshness.md)), the same resource arrives repeatedly by design. Landing must be idempotent or the log fills with duplicates.
- **Enrich**: checkpoint per `(patient_key, resource_type)`. A crash at patient 3,000 of 4,000 resumes at 3,000.
- **Transform**: fully rebuildable from `clif_events` + `signal_map`. Deleting the parquet output is always safe.

### Backpressure and rate limiting

Three distinct limits, three distinct handlers:

1. **Bulk kickoff throttle** — roughly one export per Group per 24 hours at Epic **[community]**. This is not a retryable error. A `429` or "requested this Group too recently" must set a cooldown timestamp and **stop**, not enter a retry loop. Retrying into a throttle is how you get an integration disabled by a hospital's IT department.
2. **REST rate limits** — Epic and Oracle both return `429`. Use a token-bucket limiter with a conservative default (single-digit requests per second), honor `Retry-After`, and back off exponentially with jitter. Make the rate a config value the hospital can lower.
3. **Internal backpressure** — landing a billion-row NDJSON file must not hold the whole file in memory. Stream line by line, batch inserts, and let the database's write throughput set the pace.

### Storage

A split, because the two workloads are genuinely different:

| Store | Engine | Why |
|---|---|---|
| `clif_events` (large, append-only, columnar scans) | **Parquet partitioned by month + source, queried via DuckDB** | clifpy already speaks parquet; no server to operate; transform is a columnar join |
| `clif_signals`, `signal_map`, `run`, `cohort`, config, audit | **Postgres** | transactional, small, mutable, backs the review UI |
| downloaded NDJSON + checksums | filesystem, content-addressed | replay and provenance |
| `clif_*.parquet` | filesystem, in clifpy's expected layout | the deliverable |

This is a recommendation with a real cost: two engines to operate. A single-Postgres deployment is simpler and probably fine up to a few hundred million events. Decide with a real volume estimate from the pilot site, not in the abstract.

The CLIF output directory follows clifpy's convention exactly — flat files named `clif_<table>.parquet` alongside a `clif_config.json` naming the `data_directory`, `filetype`, and `timezone` ([doc 01](01-what-is-clif.md)).

### Source adapters

The pluggability that [doc 08](08-what-hospitals-actually-provide.md) argues for lives here. An adapter's only contract is: *produce `Event` rows and register `Signal` rows.*

```
FhirBulkAdapter    # NDJSON files → events
FhirRestAdapter    # cohort-scoped searches → events
Hl7v2Adapter       # MLLP listener or file drop; ORU^R01, ADT^A01/A02/A03/A08 → events
ClarityAdapter     # SQL over IP_FLWSHT_MEAS, MAR_ADMIN_INFO → events
```

Nothing downstream of `clif_events` knows which adapter produced a row, beyond the `source_system` column. The mapper, the transform, and the validator are adapter-blind. **This is what lets the FHIR-versus-Clarity question be a deployment decision instead of a rewrite.**

The HL7 v2 adapter is where real-time would live, if real-time is ever needed. It changes the latency of the pipeline without changing its shape ([doc 09](09-data-freshness.md)).

### The web console

Three surfaces, one process, read-mostly:

- **Configure** — FHIR base URL, client id, Group id, cohort rule, `_type` and `_typeFilter` sets, schedule, rate limits, timezone. Every value versioned; changes audit-logged with an actor.
- **Review** — the mapping dashboard described in [doc 13](13-mapper-engine.md). Sorted by `event_count`. Shows raw string, code, unit, value distribution, proposal, rationale. Approve / correct / reject / ignore.
- **Monitor** — run history, stage timings, throttle cooldowns, the contents of `error[]` from each manifest, coverage per CLIF table, `clifpy` validation results, and unmapped-signal counts trending over time.

The console must never be reachable from outside the hospital network, must authenticate its users against hospital identity, and must not have access to the FHIR private key.

### Discovery, not assumption

The `DISCOVER` stage exists because nearly every fact in these documents is site-dependent. On each run:

- Fetch `/metadata` and `/.well-known/smart-configuration`; store them.
- **Diff against the previous run and alert on change.** A hospital upgrading Epic can silently change which resources are available. Discovering that from a coverage drop three weeks later is worse than being told on the night it happened.
- Record the exact `output[]` resource types the manifest actually returned. That empirical list, not the spec, is the truth about this server ([doc 17](17-open-questions-and-roadmap.md), question 1).

The CapabilityStatement can be incomplete or overstated **[community]**, so treat it as a hint and treat the manifest as evidence.

---

## Edge cases and how it breaks

**Losing the `Content-Location` URL.** A crash between kickoff and persistence orphans a running export and burns the 24-hour throttle window. Write it to durable storage inside the same transaction that records the kickoff, before returning. Consider recording the kickoff *intent* before issuing the request, so a crash mid-request is detectable.

**The token expires mid-poll.** Access tokens are ~5 minutes; a large export takes far longer. Any implementation that fetches one token at the start and holds it will fail on exactly the exports that matter most, and will pass every test against a small sandbox dataset.

**`Prefer: handling=lenient` turns a filter error into silent full data.** If the server does not support a `_typeFilter` you sent, lenient mode drops it and returns everything, with no error. You believe you filtered. You did not. **This is a silent correctness bug, and it is the most dangerous single line of configuration in the system.** Prefer strict handling, catch the `422`, and fail loudly. See [doc 05](05-bulk-export-deep-dive.md).

**Ignoring `error[]` in the manifest.** The completion manifest carries an `error[]` array of `OperationOutcome` files describing what the server could not give you. Naive clients download `output[]` and never look at `error[]`. Those files are where "you lack scope for resource X" lives. Parse them, store them, surface them in the console.

**Retrying into a throttle.** A retry loop against a 24-hour throttle produces hundreds of rejected requests per day and looks exactly like an attack. Cooldowns must be persistent, not in-memory, so a service restart does not reset them.

**The Group changes underneath you.** If the hospital defines the Group as "currently admitted patients," its membership differs every night, and consecutive exports are not comparable. Ask for a stable, rule-based Group with an explicit time window, and record the Group definition text in warehouse metadata ([doc 10](10-cohort-and-encounter-class.md)).

**Enrichment on a stale cohort.** The filter runs on the Encounters from tonight's bulk export. If enrichment runs hours later, patients have been admitted since. That is fine — they arrive in tomorrow's run — but the cohort must be snapshotted with the run, not recomputed live, or the two legs disagree about who they are talking about.

**Web console with FHIR credentials.** If the review UI can reach the FHIR server, a session hijack becomes a bulk PHI export. Enforce the split at the process and network level, not by convention.

**Clock skew.** The `client_assertion` carries `exp` and the server checks it. A daemon whose clock drifts by minutes fails authentication with an unhelpful error. Require NTP; alert on drift.

**Timezone handling at the boundary.** Store UTC everywhere in `clif_events`. Convert exactly once, at parquet-write time, to the single timezone clifpy will apply on load. Two conversions is one too many, and the bug surfaces as a one-hour shift twice a year.

**Storage growth outruns estimates.** ICU flowsheet data is dense. If `raw_payload` retention is on and the site is larger than expected, disk fills silently and the nightly run fails at `LAND`, after a successful export that cannot be repeated for 24 hours. Monitor free space as a first-class health check, and check it *before* kickoff, not after download.

---

## What we still don't know

- **Whether a single Postgres deployment is sufficient**, avoiding the DuckDB/parquet split. Depends entirely on real event volume at the pilot site, which we do not have. Estimate from the MIMIC demo, then re-estimate from the first real export.

- **Whether the enrichment leg is affordable in practice.** The cost model assumes one round trip per patient per resource type and a cohort in the low thousands. If Epic's REST rate limit at the site is aggressive, enrichment may not fit in a nightly window. Measure against the Epic sandbox before promising it.

- **How the mapping-review loop interacts with the nightly schedule** when a site upgrade introduces hundreds of new signals overnight. Coverage drops; the pipeline keeps running; someone must notice. The alerting design for this is unspecified.

- **Whether the HL7 v2 adapter belongs in this service at all**, or whether it should be a separate daemon feeding the same event log. An MLLP listener has very different operational characteristics — long-lived TCP, ordering guarantees, acknowledgement semantics — from a nightly batch job. Probably separate. Unresolved.

- **What "done" looks like for a run.** Is a run successful if it lands events but coverage is 60% because mappings are pending? The answer determines what the monitoring alerts on, and it should be decided before anyone is paged at 03:00.

- **Deployment shape.** Container versus native package. Containers complicate `LoadCredential` and TPM binding; native packages complicate distribution. Hospital IT will likely have an opinion, and it will not be the one we prefer.
