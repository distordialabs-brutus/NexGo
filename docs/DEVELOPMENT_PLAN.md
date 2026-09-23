# NexGo development plan

Status re-reviewed 2026-09-23 at `master` HEAD `78435a5d4f97deb27770051a8aacbbc3360b9355`; runtime remains `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`. A clean offline install/build passes, but current Nexus invoice creation is called with the wrong destination parameter, current invoice envelopes normalize to empty terms, canonical ride ownership/revision checks fail open, settlement has no durable reconciliation, and seven referenced PNGs remain outside the serving manifest. Authority: [architecture](ARCHITECTURE.md) and [executed review](DEVELOPMENT_REVIEW_2026-09-23.md).

**No real invoice, payment, profile write or production release until Batches 1A–1D exit. Next repair: Batch 1A.**

## Batch 1A — Pinned current-core API adapter (P0)

**Targets:** `src/api/contracts.js`, `src/api/nexusAPI.js`, `test/contracts.test.js`, `test/api-envelopes.test.js`, pinned current-core fixtures.

- Pin and document supported Nexus core and NexusInterface versions. Treat the reviewed LLL-TAO stable `11851455…` and wallet `1e923d46…` semantics as the initial contract unless maintainers choose a different release explicitly.
- Send invoice destination as `to` (or an explicitly tested `address_to`/`name_to`), never the stale vendored-doc `account` input. Continue reading the canonical resulting destination as `json.account`.
- Decode invoice terms from `invoice.json` while preserving top-level `address`, `owner`, created/modified metadata. Require address, canonical owner/issuer, account, recipient, token, status, non-empty items and exact amount. Convert decimal strings to integer base units using the token's decimals; do not pass a parsed JavaScript float through business state.
- Require ride `address` and `owner`; require payload `passenger-genesis` to match canonical owner if present. Preserve schema version and register metadata. Reject unknown/missing/conflicting/malformed values rather than defaulting to zero, blank, `requesting` or `(0,0)`.
- Add explicit API-envelope error handling and prove supported wallet return values. No current/legacy shape guessing.

**Executable exit:** one offline command runs fixtures copied from the pinned source contract with all wallet/network functions set to throw. It must prove: current nested invoice terms round-trip exactly; writer emits `to` and never `account`; missing canonical ride owner rejects; mismatched owner claim rejects; unknown version/status/token and unsafe precision reject; an API error/undefined result never becomes success.

## Batch 1B — Canonical ownership and revision-safe transitions (P0)

**Targets:** `src/api/contracts.js`, `src/services/rideTransitions.js`, `src/store/rideIntents.js`, `test/ride-transitions.test.js`.

- Replace every stale `currentData` merge with exact-address reread → validation → transition construction → submission → readback. Keep canonical payload digests and modified metadata in intent records.
- Scope intent and one-pending-write locks to wallet/profile, network, register address and operation. Persist only public identities/state; never PIN, password, Basic auth or session ID.
- Mutate taxis by canonical address and verify the active profile owns the address. Reject mutable `driver` mismatch; do not select `assets[0]` or rebuild mutation targets from mutable vehicle settings.
- Encode an allowed legacy state transition table. A changed prestate is a conflict, not an automatic merge. Document that current public asset updates expose no CAS; do not claim cross-client serializability for legacy whole-blob writes.
- Validate UTF-8 bytes, bounded labels, finite/ranged coordinates, identifiers and schema before opening the PIN dialog.

**Executable exit:** collected tests cover missing/conflicting owners, stale digest/modified state, duplicate click, profile/network switch, concurrent changed prestate, invalid transition, oversize/multibyte payload, ambiguous return and exact readback. Assert zero secure calls on every rejection and at most one secure call per durable intent.

## Batch 1C — Durable invoice issuance and payment reconciliation (P0)

**Targets:** `src/services/rideSettlement.js`, `src/store/rideIntents.js`, `src/App/Driver.js`, `src/App/Passenger.js`, `test/ride-settlement.test.js`.

- Persist issue intent before PIN; reread ride/taxi and freeze passenger, provider, account, token and base-unit terms. Persist returned invoice address/txid before the independent taxi projection.
- Reread the exact created invoice and verify nested canonical terms. A taxi projection failure reports `invoice created; taxi status reconciliation required` and cannot recreate the invoice.
- Persist payment intent before PIN; reread the exact invoice immediately before payment and bind canonical issuer, account, recipient, token, amount/items/status to the accepted agreement. Description matching is discovery only.
- Persist payment txid/unknown state; verify exact invoice plus transaction/history evidence. Perform the passenger-owned ride projection separately and recover it by exact invoice address after paid invoices leave `list/outstanding`.
- Never retry issue/pay after timeout, undefined/empty response or restart until exact readback proves no effect. Never imply a projection failure means the payment failed.

**Executable exit:** tests cover wrong destination parameter, wrong issuer/account/token/recipient, copied ride description, duplicate invoices, fare change, paid/cancelled status, timeout before/after acceptance, empty response, restart at every state, projection failure and repeated click. Mutation counters remain `issue <= 1`, `pay <= 1`; no mutation occurs while outcome is unknown. Then two isolated non-production profiles execute request/read/issue/read/pay/read/project/read through the pinned wallet/core boundary with synthetic funds only.

## Batch 1D — Reproducible quality and package gate (P0)

**Targets:** `package.json`, `test/`, `.github/workflows/ci.yml`, `webpack.config.babel.js`, `nxs_package.json`.

- Add deterministic `test`, `lint` and `test:package` scripts; collect all contract/state tests by default. Add a network-denial test that fails on any unstubbed external access.
- Keep current compatible dependency versions unless a separately approved dependency change is required. A clean `npm ci` must reproduce the lockfile.
- Inline emitted map images or list every hashed runtime asset explicitly. Do not use unsupported wildcards. Assert every JS/CSS-referenced emitted file is served by the manifest.
- Run build, test, lint, package inventory, Markdown links and whitespace in CI. Test the production folder and zip under the pinned wallet, not only the dev server.

**Executable exit:** clean install/build produces no unlisted referenced asset; all seven current PNG cases are regressed; production folder and zip render markers/layers/routes with zero denied requests; checked-in test/lint/CI gates pass from a clean clone.

## Batch 2A — Browser privacy and integration (P1; before public use)

- Remove automatic IP-geolocation fallback after GPS denial. Require separate explicit consent and show provider/retention implications.
- Centralize external browser requests. URL-encode, debounce and abort Nominatim searches; make tile/geocoder/router providers configurable; replace the OSRM public demo service for production.
- Disclose that search text, viewed areas, IP and route coordinates leave the WebView. Keep exact pickup/destination and live location out of chain and durable module storage.
- Gate all Nexus writes on wallet initialization, active user, intended network, sync completion and supported core/wallet mode. Keep mutating endpoint constants closed and test that reads cannot trigger mutation.

**Executable exit:** browser tests prove no third-party call before consent, no call after denial/revocation, cancellation of stale search/route requests, redacted diagnostics and fail-closed write gates for logged-out/wrong-network/syncing/unsupported states.

## Batch 2B — Complete discovery and Distordia v0.2 migration (P1)

- Implement paginated, cancellable, deduplicated reads returning records plus completeness/error. Preserve known state on later-page failure.
- Design explicit adapters for legacy v1 and Distordia draft v0.2 request/offer/agreement, taxi and per-agreement rating assets. Never relabel legacy records as v0.2.
- Verify canonical owner-to-role mapping for passenger request/agreement, driver offer/taxi and rating rater. Bind fare and invoice to the accepted offer/agreement.
- Resolve current draft gaps before claiming compliance: cross-client mutation concurrency/CAS, namespace-to-genesis verification, self-address stamping recovery, agreement/invoice linkage, and privacy-preserving precise-location handoff.

**Executable exit:** multi-page and partial-page tests pass; unknown versions/owners fail closed; v0.2 conformance fixtures are shared with the standards repository; two-party ownership and fare/invoice bindings are demonstrated without precise on-chain coordinates.

## Remaining live gates

After all offline batches pass: supported-wallet production installation, isolated current-core integration, wrong-network/sync/restart tests, exact mutation readback, profile/session lock cleanup, synthetic two-profile settlement, and exact-head CI. Real funds and production profiles remain out of scope until maintainers explicitly approve a separate acceptance plan.
