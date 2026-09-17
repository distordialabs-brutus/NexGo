# NexGo architecture and acceptance boundaries

Reviewed 2026-09-17 at `master` HEAD `8923ea2102bf88fe33afbf0f76e6c1c9957e154b`. The only commit after the 2026-09-16 reviewed source (`0009d6dc76e086fdbb2893eccaf6406d6c090527`) is that review's documentation publication; a path-limited diff confirms no runtime, package-manifest, lockfile or wallet-manifest change, and `src/` remains tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`.
This is a NexusInterface wallet module, not a public-browser backend or an implemented escrow service. See [review evidence](DEVELOPMENT_REVIEW_2026-09-17.md) and [development plan](DEVELOPMENT_PLAN.md).

## Current implementation

```text
Passenger / Driver React views
  -> Redux for taxi/settings/UI; component-local rides and invoices
  -> src/api/nexusAPI.js
  -> nexus-module apiCall (reads), secureApiCall (PIN-confirmed writes)
  -> trusted wallet-selected Nexus core
```

Taxi registers are typed JSON; passenger-owned rides and rating maps are raw JSON blobs. Provider invoices are the intended acceptance/payment signal. The current module writes legacy `nexgo-ride` records (`version: 1`), not the request/offer/agreement assets in the sibling Standards v0.2.0 draft. A diagrammed state transition is not evidence of atomicity or ownership authorization.

## Required architecture

1. **One versioned API adapter.** Pin the supported core and wallet versions. Keep register metadata (`address`, `owner`) separate from invoice payload `json` fields. Canonical register ownership is mandatory evidence: never substitute a payload claim such as `passenger-genesis` when `owner` is absent, and require any redundant identity claim to equal the canonical owner. Reject malformed data and explicit API errors; distinguish unavailable discovery from a verified empty result. Gate mutating UI on intended network, synchronized core, compatible mode and active profile. Credentials and session persistence remain the wallet's responsibility, not module storage.
2. **Verified invoice workflow.** On issuance, persist the ride-scoped intent, consume the returned invoice address/txid, read it back, and only then project taxi occupancy. A failed occupancy update must not be reported as failed invoice creation or permit a blind duplicate invoice. Before a payment PIN prompt, bind the selected invoice to the canonical ride, authorized provider/account, passenger, token address and integer amount accepted by that passenger. A description's `ride=` substring is only a lookup hint, not authorization. Persist public workflow identity before submission. Retain `submission_unknown` after an ambiguous response; read the exact invoice and transaction before `payment_verified`. The subsequent passenger-owned ride update is a separate transition (`ride_projection_pending` -> `complete`) and can be retried only after rereading current state. Never recreate or repay to repair a projection.
3. **Explicit ownership.** Only the passenger updates its ride; a provider's own signed offer/invoice signals acceptance. Never equate a mutable claimed `driver` field with register ownership. Keep unknown/expired/conflicting offers unavailable for payment.
4. **Privacy and bounded data.** Exact pickup/destination coordinates and labels are currently public immutable transaction history, even if a register is later updated. Future discovery should use coarse locations and an authenticated private handoff after agreement, with a documented retention model. Validate raw UTF-8 payload size, coordinates and identity before requesting PIN. Limit rating growth by a specified versioned record model.
5. **Complete discovery or explicit partial state.** Centralize paginated enumeration for taxis/rides/ratings/invoices; return completeness separately from records. A first page of 100/500 records must not be presented as the whole network. Do not clear known state on transport failure.
6. **Reproducible wallet package.** The manifest is the serving allowlist. Every emitted runtime image/chunk must be explicitly listed or deliberately inlined. Test production installation, not only the development server.

## Next implementation slice

First isolate pure versioned decoding in `src/api/contracts.js`, retain transport only in `src/api/nexusAPI.js`, and add network-denying contract/envelope tests. Supported invoices must preserve canonical metadata and exact terms from the real nested payload; supported rides must require canonical `address` and `owner` and reject conflicting payload identity. Then move invoice issue/pay transitions from `src/App/Driver.js` and `src/App/Passenger.js` into `src/services/rideSettlement.js` with persisted public intent in `src/store/rideIntents.js`. Packaging closure is a separate parallel-safe change to `webpack.config.babel.js`, `nxs_package.json` and a clean-build inventory test.

## Acceptance status

**Prototype; not payment-release-ready.** A fresh 2026-09-17 production Webpack build passes with three performance warnings, but all seven emitted PNGs remain outside the package allowlist. An offline execution of the production normalization closures again reduced a nested invoice to zero/empty terms and accepted a payload passenger claim as ownership when canonical owner was absent. Issuance/payment read-back and reconciliation are absent, and no test/lint command or checked-in CI workflow establishes the required transitions. Exit the next slice only when nested invoice payloads preserve canonical metadata and exact terms, ride records require canonical owner/address and matching redundant identity, malformed/error/precision cases fail closed, and isolated supported-core reads traverse the same adapters. The [historical/current-flow diagrams](state-machines.md) remain useful UI descriptions, not the target settlement protocol.
