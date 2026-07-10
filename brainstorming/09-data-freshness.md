# Data Freshness, Pull Cadence, and Why "Real-Time CLIF over FHIR Bulk Export" Cannot Be Built as Stated

*Bulk export is a once-a-day snapshot on a ~24-hour leash, and asking "what changed since yesterday?" silently misses changes the server forgot to flag — so the README's aspiration to "make CLIF RealTime" is achievable only by moving latency into the source adapter, where the true real-time path is HL7 v2, not FHIR.*

## In plain words

Think of two ways a hospital can hand you data.

**The nightly delivery truck.** Bulk export (`$export`, see [doc 05](05-bulk-export-deep-dive.md)) is a truck that pulls up once a day with a stack of boxes containing everything about your patient cohort *as of the moment the truck was loaded*. It is efficient and complete. But it comes once a day and **it will not come twice** — if you phone the depot an hour later and ask for another truck, they refuse: "you already asked today." Everything about your freshness ceiling follows from that one fact.

**The phone call.** If you need to know something *right now* — a patient just got admitted to the ICU, a critical potassium just resulted, a ventilator setting just changed — you do not wait for tomorrow's truck. You need a live line. And in a hospital, that live line has been **HL7 v2** for thirty years. It is not FHIR, it is not pretty, and it is exactly what the monitors, the ADT system, and the lab analyzers already speak.

Now the dangerous part, the thing that will quietly corrupt a research warehouse if nobody names it. The obvious way to keep a daily truck cheap is to say "just send me what changed since yesterday." That only works **if the system reliably raises its hand every single time something changes.** It doesn't. A lab result gets amended, a status flips, a linked record updates — and the system fails to re-flag the parent record. So when you ask "what's new since yesterday?", that corrected result is simply *not in the truck*. You get no error. You get no warning. **You just quietly keep yesterday's wrong value, and the system never tells you it forgot.** We call this the missed-bump trap, and it is the single most dangerous fact in this entire project, precisely because it fails silently — a physician reading a dashboard cannot tell fresh-and-correct from stale-and-wrong.

The verdict this document has to deliver plainly: **"Real-time CLIF" via FHIR bulk export is not achievable.** What *is* achievable comes in tiers, and before you pay for a faster tier you must name the use case. A retrospective cohort study — "did patients on drug X have worse outcomes last year?" — does not care whether the data is a day old. A sepsis early-warning dashboard cares about seconds. CLIF's own foundational paper lists real-time quality dashboards as a *future* application, not a current one ([CLIF consortium paper, PMC11398431](https://pmc.ncbi.nlm.nih.gov/articles/PMC11398431/)). So: **name the use case before you pay for latency.** Most CLIF research does not need the phone; it needs the truck to arrive reliably and to be honest about what it forgot.

## Technical detail

### The re-export cadence ceiling

You cannot pull bulk exports as often as you like. The server throttles repeat kickoffs for the same cohort.

- **Epic: roughly once per day per Group.** Epic enforces a server-side kickoff throttle that rejects a repeat `$export` for the same Group + client inside a rolling window — default about 24 hours, tunable via `FHIR_BULK_CLIENT_REQUEST_WINDOW_TBL`. The rejection reads: *"Request not allowed: The Client requested this Group too recently."* **[community]** — [smart-on-fhir/cumulus discussion #5](https://github.com/smart-on-fhir/cumulus/discussions/5)
- **Oracle Health / Cerner: bounded by job runtime and throttling; nightly batch is standard.** Excess requests return `429` throttled. **[spec]** — [Oracle Millennium bulk export API](https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfbda/api-bulk-export.html)

So the practical bulk-export floor is **one export per cohort per day** at Epic, and effectively nightly at Oracle. That is the ceiling on freshness for the truck.

### `_since` support diverges by vendor

The spec's mechanism for "only what changed" is `_since` (an `instant`; return resources whose `meta.lastUpdated` is later — see the kickoff parameter table in [doc 05](05-bulk-export-deep-dive.md)).

- **Oracle / Cerner: `_since` IS supported** → a true incremental Group export is possible in principle. **[spec]**
- **Epic: `_since` is NOT implemented for bulk.** You cannot ask Epic bulk export "since timestamp T." Instead you must encode the date window inside `_typeFilter`, e.g. `_typeFilter=Encounter?date=ge2026-01-01T00:00:00Z&date=lt2026-02-01T00:00:00Z`. **[community]**

### The missed-bump trap — the intellectual center of this document

Every timestamp-based incremental strategy — `_since`, `_lastUpdated`, `_typeFilter` on a date — rests on one load-bearing assumption: **that the server bumps `meta.lastUpdated` whenever the clinically meaningful content changes.** In real EHRs that assumption is false, and it fails in three common ways:

1. A **linked resource updates** but the parent's envelope is untouched — the parent's `lastUpdated` does not move.
2. A **flowsheet value is corrected** in place without re-stamping the resource.
3. A **status changes** (e.g. `preliminary` → `final`, or an amendment) without touching the fields the server watches for its watermark.

In each case the resource *did* change but its `meta.lastUpdated` did not advance, so a `_since` / `_lastUpdated` incremental pull **silently omits it.** No error, no gap indication — you simply retain stale or wrong data and believe you are current.

Epic makes this worse: it **under-populates `meta.lastUpdated` in bulk output**, so at Epic the field is not merely occasionally-unreliable, it is unusable as a watermark *at all*. **[community]**

**The consequence you must design around: you cannot rely on incremental deltas for correctness.** Deltas are a bandwidth optimization, never a correctness guarantee.

### The correct incremental strategy

1. **`transactionTime` is the only authoritative watermark.** Each completed export manifest carries a `transactionTime` — the server's own statement of the instant the snapshot represents. Use *that*, never your client wall-clock, to mark "this snapshot is current as of T." **[spec]**
2. **On Epic:** run **overlapping date-windowed `_typeFilter` exports** — e.g. re-pull a trailing 30–90 day window every night — combined with **idempotent upsert / dedup keyed on resource `id` + `meta.versionId`**. Then periodically run a **full historical backfill** to sweep up resources that were changed but never re-emitted (the missed bumps). The overlap catches most late edits; the periodic full re-pull catches the rest.
3. **On Oracle:** you *may* set `_since` = the previous export's `transactionTime` for efficiency, but you **must still run a periodic wider re-pull** to defeat missed bumps. `_since` alone is not safe for correctness even where it is supported.
4. **REST search with `_lastUpdated`** inherits exactly the same limitation (it reads the same field). Use it only for *targeted, known-cohort catch-up* — "re-check these 12 ICU patients now" — never as a warehouse-wide correctness mechanism.

The mental model: incremental pulls make the truck cheaper, but only a periodic full re-pull makes it *honest*. Budget for both.

### Near-real-time options, ranked by what actually works

1. **HL7 v2 feeds — the real low-latency path.** `ADT^A01/A02/A03/A08` messages carry admissions, discharges, transfers, and demographic/visit updates (census and bed movement); `ORU^R01` carries results, vitals, and device/monitor data. Delivered via **Epic Bridges / Cerner interfaces over MLLP**. This is how latency-sensitive integration is *actually* done in every hospital, and it is **the only practical route to continuous ICU device data** — which FHIR bulk export does not meaningfully deliver at all.
2. **FHIR Subscriptions — theoretically right, practically weak.** R4 `Subscription` (rest-hook), R5 topic-based subscriptions, or the [R4 Subscriptions Backport IG](https://argonautproject.github.io/subscription-backport-ig/). Support is **weak or absent at Epic and Oracle** **[community]**, but solid in HAPI and Medplum. So this is viable **only if you front the EHR with your own FHIR server that you feed from HL7 v2** — i.e., subscriptions become useful *downstream* of the v2 line, not against the EHR directly.
3. **Frequent windowed REST / bulk polling — minutes to hours, throttle-bound.** Tight `_lastUpdated` REST polling over a *small* ICU cohort can reach minutes; bulk cannot beat its ~24h throttle. Expensive, and still subject to the missed-bump trap for correctness.

### The achievable tiers, stated honestly

| Tier | Latency | Mechanism | Fit |
|---|---|---|---|
| **Daily batch** | ~24 h | Bulk `$export` (truck) | The realistic default. Correct for retrospective research. |
| **Hourly → minutes** | minutes–hours | Targeted REST `_lastUpdated` polling on a small ICU cohort | Possible, throttle-bound, expensive, correctness-limited by missed bumps. |
| **Seconds** | seconds | HL7 v2 `ORU`/`ADT` + device middleware over MLLP | The **only** true real-time path — and it is **not FHIR.** |

### The architectural consequence

Because these tiers use fundamentally different transports, the middleware must be built so that **latency is a property of the source adapter, not of the pipeline.** The `clif_events` table and the mapper should not know or care whether a given signal arrived as a line in a nightly NDJSON file or as a live MLLP message off a socket — they see a normalized event either way. That decoupling is what lets one CLIF warehouse serve both a retrospective cohort study and (eventually) a live dashboard without a rewrite. This is the crux of [doc 12](12-clif-events-and-signals.md); the access-mode trade-offs behind each transport are in [doc 04](04-api-access-modes.md), and what any given hospital will actually expose is [doc 08](08-what-hospitals-actually-provide.md).

## Edge cases and how it breaks

- **Overlapping windows manufacture duplicates.** The trailing-window strategy re-pulls resources you already have. **Dedup by `id` + `meta.versionId` is mandatory, not optional** — without it every nightly overlap inflates the warehouse and can double-count events. Upserts must be idempotent by construction.
- **`transactionTime` is the only safe watermark.** Client wall-clock, "now," or "when the download finished" all drift from what the snapshot actually represents. Persist the manifest's `transactionTime` alongside the data it describes.
- **A failed download after a successful export can cost you a full day.** The export succeeded, the throttle clock started — but if your download of the NDJSON files then fails, Epic's ~24h window blocks a re-kickoff. **Checksum and persist every file immediately** on receipt; treat the download as the fragile step, not the kickoff.
- **Export files expire.** Oracle retains output files ~30 days; Epic expires them **sooner** **[unverified]**. A slow or retried consumer can find the pickup slip's URLs already dead. Download promptly; do not assume files linger.
- **Backfill windows must overlap the throttle window.** If your trailing re-pull window is narrower than the cadence at which you actually manage to run it, you leave a gap through which missed-bump corrections escape forever. Make the overlap comfortably wider than one throttle period.
- **Clock skew between your box and the server** corrupts any date-windowed `_typeFilter` you construct from local time. Always derive window boundaries from server-supplied `transactionTime`, never from your host clock, and account for the server's timezone/UTC handling.
- **Late-arriving corrections are invisible.** An amended lab (a result finalized, then corrected hours later) will not appear in a `_since` pull **if `meta.lastUpdated` was not bumped** — the missed-bump trap in its most clinically dangerous form. Only a periodic full re-pull recovers it. For a research warehouse, an undetected wrong lab value is worse than a missing one.
- **Deleted / retracted resources.** The Bulk Data v2.0.0 manifest can list deletions in a `deleted[]` output array, but **EHR support for emitting deletions is thin.** If the source retracts a record (an erroneous encounter, a merged patient, a withdrawn result) and does not report it in `deleted[]`, **your warehouse retains data the EHR has retracted.** That is a direct IRB and data-integrity concern: the research copy asserts something the source of truth no longer stands behind. Plan for periodic reconciliation, and flag this explicitly to the IRB.

## What we still don't know

- **Exact Epic file-expiry window.** We know it is sooner than Oracle's ~30 days but not the precise value or whether it is version/config dependent. **[unverified]**
- **Whether any production Epic or Oracle deployment we can access actually exposes FHIR Subscriptions**, and if so with what latency and reliability. Current read is "weak/absent" **[community]** — needs on-site confirmation per hospital.
- **Precisely which change types fail to bump `meta.lastUpdated`** at a given Epic/Oracle version. We know the *class* of failure (linked-resource, in-place correction, envelope-untouched status change); we do not have a vendor-published enumeration, so the safe posture is "assume any change may be missed."
- **How reliably Oracle populates `deleted[]`**, and whether Epic emits deletions in bulk output at all — determines how bad the retracted-record problem is in practice.
- **The real minimum viable REST-polling latency** for a small ICU cohort under production throttling — the "minutes" figure is an estimate, not a measured floor.
- **Whether the target hospital's HL7 v2 interface engine (Epic Bridges / Cerner) can be tapped for a research feed at all**, organizationally and technically — this gates the entire seconds-tier and is answered in [doc 08](08-what-hospitals-actually-provide.md), not here.
