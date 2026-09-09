# NexGo architecture and acceptance boundaries

Reviewed 2026-09-09 against `master` at `d2f9ac053eba610e15c113db1584896c9701cf76`. The only commit after runtime baseline `7a9c4816852d3e0a7b500f64bfac035715502f86` is the 2026-09-08 documentation baseline; `git diff --name-status 7a9c481..d2f9ac0` contains only README and documentation paths. Runtime behavior and release status are unchanged.
This is a NexusInterface wallet module, not a public-browser backend or an implemented escrow service. See [review evidence](DEVELOPMENT_REVIEW_2026-09-09.md) and [development plan](DEVELOPMENT_PLAN.md).

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

1. **One versioned API adapter.** Pin the supported core and wallet versions. Keep register metadata (`address`, `owner`) separate from invoice payload `json` fields. Reject malformed data and explicit API errors; distinguish unavailable discovery from a verified empty result. Gate mutating UI on intended network, synchronized core, compatible mode and active profile. Credentials and session persistence remain the wallet's responsibility, not module storage.
2. **Verified payment workflow.** Before a PIN prompt, bind the selected invoice to the canonical ride, authorized provider/account, passenger, token address and integer amount accepted by that passenger. A description's `ride=` substring is only a lookup hint, not authorization. Persist public workflow identity before submission. Retain `submission_unknown` after an ambiguous response; read the exact invoice and transaction before `payment_verified`. The subsequent passenger-owned ride update is a separate transition (`ride_projection_pending` -> `complete`) and can be retried only after rereading current state. Never repay to repair that projection.
3. **Explicit ownership.** Only the passenger updates its ride; a provider's own signed offer/invoice signals acceptance. Never equate a mutable claimed `driver` field with register ownership. Keep unknown/expired/conflicting offers unavailable for payment.
4. **Privacy and bounded data.** Exact pickup/destination coordinates and labels are currently public immutable transaction history, even if a register is later updated. Future discovery should use coarse locations and an authenticated private handoff after agreement, with a documented retention model. Validate raw UTF-8 payload size, coordinates and identity before requesting PIN. Limit rating growth by a specified versioned record model.
5. **Complete discovery or explicit partial state.** Centralize paginated enumeration for taxis/rides/ratings/invoices; return completeness separately from records. A first page of 100/500 records must not be presented as the whole network. Do not clear known state on transport failure.
6. **Reproducible wallet package.** The manifest is the serving allowlist. Every emitted runtime image/chunk must be explicitly listed or deliberately inlined. Test production installation, not only the development server.

## Acceptance status

**Prototype; not payment-release-ready.** The last executed production Webpack build passed on 2026-09-08, but the current core invoice response shape is not decoded, payment read-back is absent, and emitted runtime images are missing from the package allowlist. The 2026-09-09 gate attempt was approval-denied before execution and does not replace that evidence. No test/lint command or checked-in CI workflow establishes the required transitions. The [historical/current-flow diagrams](state-machines.md) remain useful UI descriptions, not the target settlement protocol.
