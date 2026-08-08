# Prior Auth Automation POC — Project Context

## Goal
Building a proof-of-concept for a Workato-powered prior authorization automation
workflow, to demo as part of a healthcare IT go-to-market pitch for a Workato
specialized partnership (SurfStack). The POC covers: a synthetic patient with a
stroke diagnosis needing a brain MRI, a FHIR data source, and (next) a Workato
recipe that extracts clinical info, checks medical necessity, and routes for
approval or human review.

## Decisions Made (and why)

- **Scenario: Stroke, not Bell's Palsy.** Originally scoped around Bell's Palsy,
  but switched to stroke because Synthea has a built-in cardiovascular/stroke
  module — no need to hand-author an entire clinical history.
- **Patient: Lavern240_Zieme486** (bundle file:
  `output/fhir/Lavern240_Zieme486_fde24445-81d4-346e-9842-cc65b9b2870d.json`).
  Chosen over a second stroke-positive candidate (Andre610) because Lavern had
  23 conditions vs. Andre's 41 — a cleaner, less cluttered clinical picture for
  the demo.
- **Patient UUID:** `fde24445-81d4-346e-9842-cc65b9b2870d`
- **Stroke diagnosis location:** NOT a standalone `Condition` resource — it's on
  `Encounter.reasonCode` (SNOMED `230690007`, Cerebrovascular accident).
  Encounter ID: `fde24445-81d4-346e-02cb-d768293b7544`, dated 2015-05-30.
- **Custom resources added** (not from Synthea, authored by hand, appended via `jq`):
  - `ServiceRequest` id `mri-order-stroke-01` — brain MRI order, CPT `70551`,
    references the Encounter via `encounter` field and stroke SNOMED code via
    `reasonCode` (NOT `reasonReference` — FHIR spec doesn't allow ServiceRequest
    to reasonReference an Encounter directly).
  - `DocumentReference` id `clinical-note-stroke-01` — base64-encoded clinical
    note describing the stroke presentation and rationale for MRI (right-sided
    weakness, expressive aphasia, CT showed no hemorrhage, need to characterize
    infarct extent before treatment planning).
  - Final merged bundle: `output/fhir/Lavern240_Zieme486_updated2.json`

## Environment Setup

- **Java:** Had to install Temurin 21 specifically (`brew install --cask temurin@21`)
  because the default Homebrew cask installs Java 26, which Gradle doesn't yet
  support. Set `JAVA_HOME=$(/usr/libexec/java_home -v 21)` before building.
- **Synthea:** Cloned to `~/synthea`, built with `./gradlew build check test -x test`.
  Generated with `-p 100 -a 45-75 Massachusetts` to get enough population to find
  a stroke-positive patient (stroke incidence is probabilistic, not directly
  requestable).
- **HAPI FHIR:** Running locally via Docker:
  `docker run -d -p 8080:8080 --name hapi-fhir hapiproject/hapi:latest`
  Endpoint: `http://localhost:8080/fhir`
- **Load order matters:** Practitioner/organization data must load BEFORE the
  patient bundle, or conditional references (e.g. `Practitioner?identifier=...`)
  fail to resolve. Load order used:
  1. `output/fhir/hospitalInformation*.json`
  2. `output/fhir/practitionerInformation*.json`
  3. `output/fhir/Lavern240_Zieme486_updated2.json`
- **Bundle gotcha:** Synthea's FHIR bundles are `type: transaction`, so every
  entry needs a `request: {method, url}` block or HAPI rejects the whole bundle.
- **Load result:** All 468 entries loaded successfully. Our custom ServiceRequest
  landed as `ServiceRequest/3679`, DocumentReference as `DocumentReference/3680`
  (verified 2026-08-08 by querying HAPI directly — IDs are auto-assigned
  sequentially on insert, not identical, so don't assume matching numbers).

## Azure Deployment (for Workato access)

Deployed 2026-08-08 since Workato is cloud-hosted and can't reach `localhost`.
- **Resource group:** `rg-hapi-fhir-poc` (westus) — kept separate from
  `rg-surfstack-prod` so it's easy to tear down after the demo.
- **Container instance:** `hapi-fhir-poc`, image `hapiproject/hapi:latest`,
  1 vCPU / 2GB RAM, port 8080, restart policy `OnFailure`.
- **Endpoint:** `http://hapi-fhir-poc-surfstack.westus.azurecontainer.io:8080/fhir`
  — use this in the Workato HTTP connector instead of an ngrok tunnel.
- **Cost:** ~$0.05–0.06/hr while running. Stop between sessions with
  `az container stop -g rg-hapi-fhir-poc -n hapi-fhir-poc`; restart with
  `az container start -g rg-hapi-fhir-poc -n hapi-fhir-poc` (takes ~1 min to
  boot — HAPI is a Spring Boot app).
- **No auth, no persistent volume.** Endpoint is open to the internet (fine —
  synthetic data only), but a container restart wipes all loaded data. Re-run
  the load steps below after any stop/start or redeploy.
- **Data loaded:** same three bundles, same order, as the local load above.
  All 468 entries loaded with zero failures. **IDs differ from the local
  instance** (separate database, sequential auto-assignment) — don't reuse
  local IDs against this endpoint:
  - Patient (Lavern240 Zieme486): `Patient/2277`
  - Encounter (stroke, 2015-05-30): `Encounter/2321`
  - ServiceRequest (MRI order, CPT 70551): `ServiceRequest/2743`
  - DocumentReference (clinical note): `DocumentReference/2744`

## Next Step: Workato Recipe

Design (not yet built):
```
TRIGGER: New/Updated ServiceRequest (poll HAPI endpoint)
  -> Fetch related Encounter + DocumentReference (clinical note)
  -> AI extraction: pull diagnosis, procedure, clinical justification from note text
  -> Medical necessity rules check (confidence score)
  -> IF high confidence: auto-approve, log outcome
  -> IF low confidence: post Teams adaptive card for human review, log outcome
```

Connectors needed: generic HTTP connector (HAPI isn't a native Workato app),
AI connector or direct LLM API call, Microsoft Teams connector, Google
Sheets/database for outcome logging.

Endpoint reachability is resolved — see "Azure Deployment" section above.

## Business Context (for the pitch)

This POC supports a Workato go-to-market pitch for healthcare IT and finance
data verticals. Prior auth automation is timely because CMS-0057-F
(Interoperability and Prior Authorization Final Rule) took effect Jan 1, 2026,
requiring payers to respond to standard PA requests within 7 days / expedited
within 72 hours, with FHIR Prior Authorization API requirements landing
Jan 1, 2027. This gives the demo a real regulatory hook.
