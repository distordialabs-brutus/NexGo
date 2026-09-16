# NexGo development plan

Status re-reviewed 2026-09-16 at freshly fetched local/remote HEAD `0009d6dc76e086fdbb2893eccaf6406d6c090527`; runtime is unchanged from `7a9c4816852d3e0a7b500f64bfac035715502f86`. A fresh production build passes, while the runtime package check still finds seven referenced emitted PNGs absent from the manifest. Authority: [architecture](ARCHITECTURE.md), [executed review](DEVELOPMENT_REVIEW_2026-09-16.md). This plan is work to implement, not completed functionality. **Next repair:** Batch 1 before any real invoice or payment testing.

## Batch 1 — Invoice adapter and reproducible gate (P1, release blocker)

- Add versioned normalized contracts in `src/api/nexusAPI.js`: decode the current invoice `json` payload, preserve canonical register metadata and token identity, and require valid amount/items/recipient/account/status. For rides, require canonical `address` and `owner`; reject instead of falling back from a missing `owner` to passenger-controlled `passenger-genesis`, and reject a redundant claim that differs from canonical ownership. Support another shape only through an explicit tested compatibility branch, never zero/empty success defaults.
- Add `test`, `lint` and a clean-lockfile CI command covering API payloads, data errors, amount precision, package inventory and build. Keep NexusInterface-compatible versions; do not blindly upgrade the dependency tree.
- Regress the current failures: an invoice `{address, json: {account, recipient, amount: 10, items: [{description: 'NexGo ride ride=abc123'}], status}}` must retain the amount, identity and ride reference; a ride with no canonical owner must reject even when its raw payload claims `passenger-genesis`. Reject missing/malformed/conflicting fields; test real supported core fixtures and wallet return envelopes.
- Exit: full gate passes and isolated supported-core invoice/ride reads traverse the production adapters without synthetic fallback identity or terms.

## Batch 2 — Verified invoice issuance and settlement (P1, release blocker)

- Move `Driver.handleCreateInvoice` orchestration out of the view. Persist a ride-scoped issuance intent, reject ineligible/cancelled/paid requests after an authoritative reread, and retain the returned invoice address/txid before any taxi-state projection.
- Read back and validate the exact new invoice. A failed taxi `occupied` projection must be recoverable without recreating the invoice, and the UI must report “invoice created; taxi status requires reconciliation” rather than “failed to create invoice.” Deduplicate by durable ride/invoice identity rather than description order.
- Move `Passenger.handlePayInvoice` orchestration out of the view. Persist public intent scoped to profile/network/ride/invoice, freeze accepted terms, validate issuer/account/token/recipient/amount and reread authoritative state immediately before payment.
- Store returned txid, then verify exact invoice state and transaction evidence. Unknown submission remains held across restart. Mark ride projection complete only after its own read-back. A projection failure must say “payment requires reconciliation,” not imply payment failed and invite repayment.
- Test issue-accepted/taxi-update-failed, issue timeout, accepted-but-timeout payment, empty response, wrong issuer, copied description, duplicate invoices, changed fare, repeated click, profile/network switch, crash after either mutation and before projection, and restart. Assert mutation counts: one issue and one pay maximum, with no mutation while evidence is unknown.
- Exit: two isolated test profiles complete request/read/issue/read/pay/read/project/read through the supported wallet boundary; no production credentials or funds.

## Batch 3 — Packaging (P1)

- Generate an explicit manifest from emitted runtime assets or inline map images deliberately. Never add unsupported wildcard entries.
- Assert every JS/CSS-referenced emitted image/chunk belongs to `nxs_package.json.files`, excluding deliberately non-runtime source maps/licenses.
- Exit: clean build plus production folder/zip installation displays all marker/layer/routing icons with no denied requests.

## Batch 4 — Bounded discovery and schema/privacy migration (P2; before public deployment)

- Implement complete paginated reads with cancellation, deduplication and `partial/error` state. Test more than one full page and error on subsequent page; never interpret failure as zero rides/invoices.
- Validate UTF-8 raw payload bytes (not character count), coordinates, identity and immutable request fields before calling `secureApiCall`. The actual source probe currently submits 2,335 bytes from a long pickup label. Confirm the supported core's register allocation/update rules before fixing a numerical protocol limit.
- Specify legacy-to-v0.2.0 request/offer/agreement adapters jointly with `Distordia_Standards`; do not relabel existing blobs as compliant. Move precise location exchange off-chain and require consent/retention controls.
- Exit: multibyte/oversized inputs reject before PIN or mutation; unknown versions and untrusted owners fail closed; draft schemas have executable conformance fixtures.
