# Development and architecture review — 2026-09-17

## Baseline and verdict

**Reviewed HEAD:** `8923ea2102bf88fe33afbf0f76e6c1c9957e154b` (`master`). The commit after the 2026-09-16 reviewed source HEAD `0009d6dc76e086fdbb2893eccaf6406d6c090527` is that review's documentation publication. A path-limited diff confirms no change to `src/`, `package.json`, `package-lock.json` or `nxs_package.json`; `src/` remains tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`.

**Verdict: unchanged prototype; not payment-release-ready.** No release blocker exited and no new runtime regression was introduced. Verification was offline: no wallet/core, profile/session, invoice, payment, installation or production request was made.

## Fresh offline verification

| Gate | Result |
|---|---|
| Identity and source delta | **PASS** — HEAD `8923ea21…`; no runtime/manifest/lockfile delta from the 2026-09-16 reviewed source |
| Git integrity / whitespace / scope | **PASS** — `git fsck --no-dangling --no-progress`, `git diff --check`; clean baseline and only the three intended review documents changed afterward |
| Production build | **PASS with warnings** — Webpack `5.105.1`; 872 KiB `app.js`; seven PNG assets / 9.05 KiB; three performance warnings; 3.124 s |
| Production package closure | **FAIL** — all seven emitted hashed PNG files under `dist/js/` are absent from the three-entry `nxs_package.json.files` allowlist |
| Project Markdown links | **PASS** — 10 files under `docs/` after this review was written; 0 missing local targets |
| Test and lint commands | **ABSENT / FAIL** — `npm test` and `npm run lint` both report missing scripts; no tracked CI workflow |
| Offline adapter diagnostic | **FAIL as a contract gate** — the production normalization closures reduced a nested current-style invoice to amount `0`, blank account/recipient and no items, and accepted payload `passenger-genesis` as ride owner when canonical owner was absent |
| Supported core / wallet install / payment acceptance | **NOT RUN** by scope |

The adapter diagnostic compiled the unchanged `src/api/nexusAPI.js` closure in memory with `apiCall` and `secureApiCall` replaced by network-denying stubs. It exercised normalization only and is not a checked-in test or supported-core acceptance result.

## Findings

1. **P1 — canonical contract ingestion still fails open.** `normalizeRideAsset` substitutes passenger-controlled `passenger-genesis` for absent canonical owner. `normalizeInvoice` reads top-level fields while the probed current-style envelope carries terms in `invoice.json`; it silently fabricates zero/empty terms instead of rejecting. Today's execution reconfirms both unchanged defects.
2. **P1 — payment mutations remain duplicate-prone and unreconciled.** `Driver.handleCreateInvoice` creates an invoice and then changes taxi projection inside one view-level try/catch; `Passenger.handlePayInvoice` pays and then updates the ride similarly. Neither retains intent, returned identity, unknown outcome or exact read-back evidence across restart.
3. **P1 — the production module package remains incomplete.** The build emitted seven hashed Leaflet/routing PNGs, but `nxs_package.json.files` still lists only `dist/index.html`, `dist/nexgo-logo.svg` and `dist/js/app.js`.
4. **P2 — bounded discovery, privacy and v0.2.0 migration remain open.** Fixed-limit reads collapse failures into empty values; precise pickup/destination data remains immutable history; the writer still emits the legacy raw `version: 1` ride shape.

## Prioritized coding batches

### Batch 1A — strict versioned adapters and one reproducible gate (P1)

**Targets:** split pure contracts from `src/api/nexusAPI.js` into `src/api/contracts.js`; keep transport wrappers in `src/api/nexusAPI.js`; add `test/contracts.test.js`, `test/api-envelopes.test.js`, `test/network-deny.js`; update `package.json`; add `.github/workflows/ci.yml`.

**Acceptance:** supported invoice envelopes preserve canonical address/owner/account/recipient/token, exact integer/base-unit amount, items, status and ride reference; rides require canonical address and owner, and any redundant passenger claim must match; missing/malformed/unknown-version/conflicting/unsafe-precision inputs reject rather than normalize to zero/empty; one offline command runs these fixtures, package inventory and build with all external access denied.

### Batch 1B — durable issue/pay reconciliation (P1)

**Targets:** add `src/services/rideSettlement.js` and `src/store/rideIntents.js`; reduce `src/App/Driver.js` and `src/App/Passenger.js` to UI dispatch/rendering; add `test/ride-settlement.test.js`.

**Acceptance:** issue/pay intent is persisted before PIN-confirmed submission and scoped to profile/network/ride/invoice; returned invoice address/txid is retained before projection; timeout/empty/malformed result becomes `submission_unknown`; exact invoice and payment evidence are read back before terminal state; repeated click, restart, copied descriptions, wrong terms, and either projection failure produce at most one issue and one pay call, never blind retry.

### Batch 1C — production package closure (P1)

**Targets:** `webpack.config.babel.js`, `nxs_package.json`, and `test/package-inventory.test.js`.

**Acceptance:** a clean build has zero JS/CSS-referenced runtime assets outside `nxs_package.json.files`; folder and zip installation in the supported NexusInterface version render marker, layer and routing icons with zero denied requests.

### Batch 2 — complete discovery and privacy-preserving migration (P2)

**Targets:** paginated readers in `src/api/nexusAPI.js`; migration adapters in `src/api/contracts.js`; passenger/driver status handling; joint fixtures for `Distordia_Standards` v0.2.0 request/offer/agreement.

**Acceptance:** multi-page reads expose completeness separately from records and later-page failure remains partial/error; oversized UTF-8 and invalid identity/coordinates reject before PIN; legacy and v0.2.0 records are explicitly distinguished; precise coordinates move to a consented off-chain handoff before public release.
