# Authentication: how a headless service proves who it is

*There is no password — the service signs a five-minute note with a private key, and the hospital's hardest part is the paperwork, not the crypto.*

## In plain words

Our middleware is a program, not a person. It never types a username, never sees a login page, never touches a patient's credentials. So how does a hospital's FHIR server know it is really us and not an impostor?

It works like a wax seal. Long before any data moves, we generate a matched pair of keys — a **private key** we guard and a **public key** we hand to the hospital during onboarding. When the service wants data, it writes a short note that says, in effect: *"I am client X. I am talking to token endpoint Y. This note is void in five minutes. Here is a one-time serial number so you can tell if someone photocopies it and tries to reuse it."* It signs that note with the private key. The hospital's server checks the signature against the public key it was given, and if it matches, hands back a temporary **access token** — a wristband, good for about five minutes, that opens the data door. The private key itself never travels across the network; only signatures made with it do.

That is the whole cryptographic idea, and it is the easy part. **The hard part is the paperwork.** Before any of this works, a human being — an Epic or Oracle analyst employed by the hospital — has to actively install our app into their environment, load our public key, and switch on the specific permissions for bulk export. That is a ticket-and-email process measured in weeks, sometimes months, and it is the true critical path for the whole project. The cryptography is a weekend; the onboarding is a quarter.

The standard that describes all of this is **SMART on FHIR Backend Services Authorization**. It is the machine-to-machine sibling of the "log in with your hospital account" flow, with the human removed. Every major US EHR that we care about — Epic, Oracle Health/Cerner — implements it because federal certification (US Core 6.1.0 / HTI-1, paired with SMART App Launch 2.0.0) requires it.

## Technical detail

The spec: **SMART Backend Services** ([hl7.org backend-services](https://hl7.org/fhir/smart-app-launch/backend-services.html), [build](https://build.fhir.org/ig/HL7/smart-app-launch/backend-services.html)) resting on **asymmetric ("private_key_jwt") client authentication** ([client-confidential-asymmetric](https://build.fhir.org/ig/HL7/smart-app-launch/client-confidential-asymmetric.html)), with [scopes](https://build.fhir.org/ig/HL7/smart-app-launch/scopes-and-launch-context.html) and the Bulk Data [authorization](https://hl7.org/fhir/uv/bulkdata/authorization/index.html) profile layered on top. **[spec]**

**Step 0 — registration (out of band, done once, the slow part).** Register a **confidential client** with the hospital's authorization server and give it your **public** key. Two delivery methods: a **JWKS URL** — a TLS-protected endpoint you host that serves your JWK Set, *preferred* because you can rotate keys without re-registering — or supplying the JWK Set inline at registration, which is *discouraged* because rotation then means going back through onboarding. The keypair is **RSA (RS384)** or **EC P-384 (ES384)**; a conformant client SHALL support both. **[spec]**

**Step 1 — build a one-time `client_assertion` JWT**, signed with your private key. The claims people get wrong are marked:

```
{
  "iss": "<your client_id>",
  "sub": "<your client_id>",        // SAME as iss — NOT a user
  "aud": "https://.../oauth2/token", // the TOKEN endpoint URL, NOT the FHIR base
  "exp": 1780000300,                 // epoch seconds, ≤ 5 minutes from now
  "jti": "a-unique-nonce-per-JWT"    // single-use; server uses it for replay defense
}
```

JWT header: `alg` = `RS384` or `ES384` (**not** RS256 — see edge cases), `kid` = the key id, unique in your JWK Set and matching a key served at your JWKS URL, `typ` = `JWT`. **[spec]**

**Step 2 — `POST` to the token endpoint** as `application/x-www-form-urlencoded`:

```
grant_type=client_credentials
client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
client_assertion=<the signed JWT from Step 1>
scope=system/Observation.rs system/Patient.rs
```

**Step 3 — token response** (JSON): `access_token` (required), `token_type: "bearer"`, `expires_in` (seconds; recommended **300**, SHOULD NOT exceed 300), and `scope` — **which may be narrower than you asked for; check it.** Then call FHIR with `Authorization: Bearer <access_token>`. **[spec]**

**Scopes.** v1: `system/Patient.read`, `system/*.read` (only `.read`/`.write` suffixes exist). v2: the suffix is an in-order subset of **`.cruds`** (Create, Read, Update, Delete, Search); read+search = **`.rs`**, and v1 `.read` maps to v2 `.rs`. **Granular/category scopes (v2.2)** let a scope carry a search-parameter filter, e.g. `system/Observation.rs?category=laboratory`; support is uneven, and a server that does not implement it falls back to the resource-level scope. Whether Epic honors granular category scopes is **[unverified]**. For HIPAA minimum-necessary, request the narrowest scopes and stack v2 `.rs` + `?category=` + `_type`/`_typeFilter` — see [doc 16](16-security-and-governance.md). **[spec]**

**Discovery.** `GET {fhirBase}/.well-known/smart-configuration` is public JSON and should advertise `token_endpoint`, `scopes_supported`, `token_endpoint_auth_methods_supported` (must include **`private_key_jwt`**), `token_endpoint_auth_signing_alg_values_supported` (must include RS384 and/or ES384), and `capabilities`. `GET {fhirBase}/metadata` (the CapabilityStatement) carries the same OAuth endpoints under `rest.security.extension` (`http://fhir-registry.smarthealthit.org/StructureDefinition/oauth-uris`). Both should agree, but Epic's CapabilityStatement can be incomplete or overstate availability — trust `.well-known` and, ultimately, an empirical token request. **[community]**

**Epic** ([oauth2 docs](https://fhir.epic.com/Documentation?docId=oauth2), [JWKS URLs](https://fhir.epic.com/documentation?docId=oauth2&section=jwks-urls), [OAuth tutorial](https://open.epic.com/Tutorial/OAuth), [endpoints](https://open.epic.com/MyApps/Endpoints)). Register a **"Backend Systems"** app at fhir.epic.com → `client_credentials` grant. Epic accepts **both** a JWKS URL and an uploaded base64-encoded X.509 public key (`.pem`); its guidance is PEM upload for the **sandbox**, but for **production** generate and publicly host a **JWKS URL**. Use **separate keys for non-production and production, and a different key per customer/environment.** Each app has **two client IDs** — a Non-Production (sandbox) and a Production Client ID. New sandbox keys can take **~60 minutes** to propagate **[community]**. `aud` = the token endpoint URL; assertion `exp` ≤ ~5 minutes. Critically, **the hospital must "download"/activate your app** into their Epic environment: you give the org your Production Client ID, an Epic analyst links it, loads your customer-specific public key, and grants API + Bulk Data permissions. App Orchard was retired/renamed — the current surfaces are **Vendor Services / Showroom / Connection Hub**, and the org uses **"Review & Manage Downloads."** For bulk you must add **Patient.Search (R4)** plus the four Bulk Data permissions — **Kick-off, Status, File, Delete**. Find a site's base URL via Epic's endpoint bundles ([open.epic.com/MyApps/Endpoints](https://open.epic.com/MyApps/Endpoints)) or cross-vendor via ONC **Lantern** ([lantern.healthit.gov](https://lantern.healthit.gov), org API `GET https://lantern.healthit.gov/api/organizations/v1`). **[community]**

**Oracle Health / Cerner** ([fhir.cerner.com](https://fhir.cerner.com/), [code Console](https://code-console.cerner.com/console), [FHIR authorization framework](https://docs.oracle.com/en/industries/health/millennium-platform-apis/fhir-authorization-framework/)). Register in **code Console**, application type **System**, privacy **Confidential**. A **system account** is auto-generated in Cerner Central; manage its secret and **JWKS setup for bulk apps** on the app details page. Scopes are `system/*.read` etc. **Base URLs are tenant-scoped:** `https://fhir-ehr-code.cerner.com/r4/{tenantId}/...` (secured) and `https://fhir-open.cerner.com/r4/{tenantId}/...` (open); develop against the `cernerdemo` sandbox tenant. **[spec]**

**mTLS / UDAP / TEFCA.** **UDAP** ([udap.org](https://www.udap.org/), [HL7 UDAP Security IG v2.0.0](https://hl7.org/fhir/us/udap-security/index.html)) extends OAuth/OIDC with trusted X.509 certificates plus dynamic client registration. It is **not required** for a single hospital with static registration — SMART Backend Services suffices. It becomes relevant for **TEFCA**: as of **Jan 1, 2026**, TEFCA "Facilitated FHIR" auth SHALL use the FAST/UDAP SSRAA IG — but that only matters for cross-organization exchange via QHINs, not a service running inside one hospital. And **mTLS is not generally required** by SMART Backend Services or by Epic/Cerner backend flows: they authenticate via the signed JWT, not a client TLS certificate. Some enterprise gateways bolt mTLS on anyway — do not assume it, but be ready to supply a client cert if a specific site's edge demands one. **[spec]**

**Key custody for a hospital-deployed daemon.** Prefer systemd **`LoadCredential=` / `ImportCredential=`** with `systemd-creds` (optionally TPM-bound) so the private key is decrypted only into the unit's private `/run/credentials/<unit>` and never sits world-readable on disk. Alternatives: HashiCorp Vault (short-lived token / Vault Agent), `sops`+`age` for encrypted-at-rest config, or an HSM/TPM/PKCS#11 module holding a non-exportable key. **Never** bake the key into a container image or a git repo. **Rotation:** host a JWKS with multiple keys, each with a distinct `kid`; add the new key, start signing with the new `kid`, retire the old one after propagation — Epic/Cerner require per-customer, per-environment keys, so track them in a matrix. **Audit-log** every token acquisition (`jti`, scope, timestamp) and every export kickoff/poll/download with file hashes. This connects directly to [doc 16](16-security-and-governance.md), and the tokens minted here are what authorize the flows in [doc 05](05-bulk-export-deep-dive.md) and [doc 11](11-middleware-architecture.md).

## Edge cases and how it breaks

- **`sub` ≠ `iss`.** The assertion's `sub` must equal `iss` (both your `client_id`). People instinctively put a "user" in `sub`; there is no user. Wrong `sub` → token request rejected.
- **`aud` points at the FHIR base instead of the token endpoint.** `aud` must be the exact token-endpoint URL you are POSTing to, not `{fhirBase}`. A very common first-day failure.
- **`RS256` instead of `RS384`.** Most JWT libraries default to RS256 or HS256. The spec requires **RS384 or ES384**. The signature "works" locally and the server rejects it — one of the most common bugs in this flow.
- **Reused `jti`.** The nonce must be fresh for every single JWT. Caching or reusing one trips the server's replay defense and the second call fails even though the first succeeded.
- **`exp` too far out (or clock skew).** `exp` must be ≤ 5 minutes ahead. A drifting server clock also breaks `exp`/`nbf` validation — keep the host on NTP.
- **Access token outlives nothing; the export outlives the token.** Tokens last ~5 minutes (SHOULD NOT exceed 300s), but a bulk export can poll for hours. You must **re-mint a fresh assertion and token mid-poll** and mid-download, not hold one token for the job.
- **Granted `scope` narrower than requested.** The token response's `scope` can be a subset of what you asked. A client that never reads it back sails on and then fails confusingly at the FHIR call, far from the real cause.
- **JWKS propagation delay.** A newly loaded key (~60 min at Epic sandbox) makes the very first attempt fail spuriously. Do not treat the first 401 after onboarding as a code bug — wait and retry.
- **Key-matrix drift.** A distinct key per customer per environment means a matrix that silently rots: a rotated prod key at one site, a stale sandbox key at another. Track it explicitly or lose hours to "works here, 401 there."
- **`.well-known/smart-configuration` and `/metadata` disagree.** They should match but can diverge (Epic especially). Prefer `.well-known`, then confirm empirically.
- **401 vs 403 mean different things.** **401** = the token is bad/expired/malformed (an *authentication* problem — re-mint). **403** = the token is valid but lacks the scope/permission for what you asked (an *authorization* problem — fix registration/scopes, re-minting won't help). Conflating them sends you debugging the wrong layer.

## What we still don't know

- **Whether Epic honors granular v2.2 category scopes** (`system/Observation.rs?category=laboratory`) or silently falls back to resource-level — decides how much filtering we push into the token vs. into `_typeFilter`. **[unverified]**
- **Real onboarding wall-clock per target site** — how many weeks from handing over a Production Client ID to an analyst actually loading the key and granting Bulk Data permissions. This is the project's critical path and it is site-specific.
- **Whether any target site's enterprise gateway adds mTLS** on top of the JWT flow, which would mean provisioning and rotating a client TLS certificate we don't otherwise need.
- **Exact per-client throttle limits** the hospital binds to our registration — these shape both the REST enrichment leg in [doc 04](04-api-access-modes.md) and how aggressively we can poll in [doc 05](05-bulk-export-deep-dive.md).
- **Oracle's precise JWKS-for-bulk activation steps** on the app details page, and whether the auto-generated system account needs additional per-tenant grants beyond the Console defaults.
- **Which free/test servers** ([doc 07](07-free-fhir-servers.md)) implement the full asymmetric `private_key_jwt` flow faithfully enough to rehearse against before we ever touch a hospital.
