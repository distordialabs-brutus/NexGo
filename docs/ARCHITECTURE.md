# NexGo architecture and acceptance boundaries

Reviewed 2026-09-23 at `master` HEAD `78435a5d4f97deb27770051a8aacbbc3360b9355` with a clean baseline. Runtime remains the `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`; the commits after upstream feature `7a9c481` are documentation-only. This document is the current architecture authority; see the [2026-09-23 executed review](DEVELOPMENT_REVIEW_2026-09-23.md) and [development plan](DEVELOPMENT_PLAN.md).

**Status: prototype; invoice creation, invoice ingestion, settlement recovery and package closure are release blockers. No real payment or profile mutation is approved.**

## Source contract reviewed

| Source | Exact identity | Relevant evidence |
|---|---|---|
| NexGo | `78435a5d4f97deb27770051a8aacbbc3360b9355` | `src/` tree `b0a45a0…`; `src/api/nexusAPI.js` blob `3ef4357…`; `package-lock.json` blob `81e2beeb…` |
| LLL-TAO stable | `master` `1185145534a20ed4d2288e4513c505f271be536d` (5.1.6 tag commit) | invoice `create.cpp` `ebd87ba…`, invoice `json.cpp` `3a24e1f…`, `extract.cpp` `aeca4c4…`, template `update.cpp` `14d796e…` |
| LLL-TAO development | `merging` `8af9c3387244b4d396c0e00ee81cea78bb9c0177` | the same invoice create/json blobs; command registration and current update semantics checked |
| NexusInterface | `master` `1e923d46a9cde1cf22b8608257c92da184aece16` | module `webview.js` `e905ee9…`, wallet API `api.js` `b123907…`, preload `26d7331…` |
| Distordia Standards | `main` `83b9f0902a062d9a97089c2729889ea565d7af82` | ride v0.2 `0df8f03…`, taxi v0.2 `d7c0003…`, rating v0.2 `68fa21a…` |

These are source snapshots, not a live node/wallet acceptance result. The repository's vendored invoice documentation is stale where it says creation uses `account`; current stable and development core source both resolve the destination from `to`, `name_to` or `address_to`.

## Runtime and trust boundaries

```text
NexGo React module (Electron WebView / browser context)
  - Redux UI and persisted non-secret settings
  - direct third-party map/geocode/location requests
  - nexus-module bridge calls
            |
            | IPC: apiCall / secureApiCall / updateState / updateStorage
            v
NexusInterface main process
  - owns active core configuration and HTTP Basic credentials
  - injects active session only when core reports multiuser
  - secureApiCall shows endpoint + params, prompts for PIN, injects PIN
  - current source has no secure endpoint allowlist
            |
            | POST JSON over wallet-selected HTTP/HTTPS transport
            v
Trusted Nexus core
  - profile/sigchain authorization and register ownership
  - invoice conditional transfer and atomic pay DEBIT + CLAIM
```

NexGo has no application backend. It must not collect, persist or log the profile password, PIN, API Basic credentials or Nexus session ID. Current code respects that boundary: writes use `secureApiCall`; the wallet owns the PIN and session; persisted module settings contain vehicle and public payment-account identifiers only. Future durable settlement storage may contain public profile/network/register/tx identities and intent state, never secrets. Wallet HTTPS currently sets `rejectUnauthorized: false`; deployment trust therefore depends on a controlled local node or separately secured transport.

A PIN dialog is not business authorization. NexusInterface currently accepts an arbitrary endpoint string from a module, displays it, and invokes it after PIN entry. NexGo must keep mutating endpoints as fixed internal constants, validate every displayed parameter before requesting the PIN, and gate writes on active profile, intended network, compatible core/wallet, non-client capability where required, and a synchronized node.

## Current records and authoritative identity

- Taxi assets are typed JSON registers. Current code also stores mutable claimed `driver`, mutable vehicle ID, exact coordinates and timestamp.
- Ride requests and ratings are legacy raw JSON blobs. The module writes ride payload `version: 1`; this is not the Distordia v0.2 request/offer/agreement protocol.
- Invoices are current-core readonly invoice registers. Canonical metadata is top-level (`address`, `owner`, timestamps); invoice terms are under `json` (`account`, `recipient`, `items`, `amount`, `token`, `status`).
- Core payment is atomic for the invoice's DEBIT + CLAIM. Taxi and ride projections are separate, non-atomic application writes.

Every accepted register must preserve the canonical envelope: `address`, `owner`, type/form, created/modified metadata and the supported payload schema. Payload identity claims are redundant assertions, not ownership. Require them to match canonical ownership or a separately verified namespace binding. Reject absent/invalid canonical ownership, unknown schema versions, invalid states, malformed addresses, unsafe numeric precision and contradictory claims. Never replace a missing owner with `passenger-genesis`, `driver`, or another payload field.

Taxi mutations must use the canonical address returned/read back for the selected owned taxi. Do not update a name rebuilt from mutable settings, choose the first local taxi silently, or prefer a mutable `driver` claim over the active profile's canonical genesis. A transport or authentication failure is not an empty asset list and must not invite duplicate creation.

## Revision and transition strictness

The core's returned `version` is not an application compare-and-set revision, and current public `assets/update/*` source exposes no `expected_version`/`If-Match` parameter. A raw update replaces the whole allocated payload. Therefore:

1. Never merge an old component snapshot (`currentData`) into a write.
2. Before a PIN prompt, read the exact address and compare owner, schema, state, modified metadata and a canonical payload digest with the intent snapshot.
3. Permit only a declared transition from that snapshot; serialize one pending mutation per profile/network/register in durable module state.
4. After submission, retain txid/unknown outcome and read the exact register/history back before projecting success.
5. Treat any changed prestate as a conflict requiring a fresh user decision.
6. Do not claim strict cross-client compare-and-set for mutable raw blobs. A release design must either prove an enforceable core condition or use append-only, owner-specific typed records so stale whole-blob overwrites cannot control settlement.

Distordia v0.2's passenger request, driver-owned offer and passenger agreement improve ownership separation and fare binding, but remain drafts and do not by themselves provide CAS. Legacy and v0.2 records must be explicit separate adapters and UI states.

## Current Nexus invoice semantics

For the reviewed current core:

- `invoices/create/invoice` resolves the payment destination through `to`, `name_to` or `address_to`, requires `recipient` and non-empty `items`, derives token and total from the destination account, and returns `success`, `address`, `txid`.
- Invoice reads encode canonical register metadata at top level and terms/status under `json`.
- `invoices/pay/invoice` takes invoice `address` and payer `from`, rejects paid/cancelled invoices and wrong-token accounts, then builds one atomic DEBIT + CLAIM transaction.
- Asset raw updates have size/allocation and ownership enforcement in core, but no public optimistic-revision argument.

NexGo currently sends `account` rather than `to`, so invoice creation is incompatible with both reviewed current core branches. Its normalizer reads terms at top level, so a current invoice becomes zero/empty and loses token/items. Both directions must fail closed until fixed and tested against pinned response fixtures.

## Required settlement state machine

```text
issue_intent_persisted
  -> issue_submitted
  -> invoice_identity_known | submission_unknown
  -> invoice_readback_verified
  -> taxi_projection_pending

payment_intent_persisted
  -> payment_submitted
  -> payment_tx_known | submission_unknown
  -> invoice_payment_verified
  -> ride_projection_pending
  -> complete
```

Before issuance, reread the ride and selected taxi; verify canonical owners, supported state/version, target provider and passenger, and freeze exact terms in integer base units. Persist intent before invoking `secureApiCall`. Consume and persist returned invoice address/txid before any taxi projection; timeout or empty/malformed success stays `submission_unknown` and forbids blind recreation.

Before payment, reread the exact invoice and bind canonical issuer/owner, recipient, destination account, token, integer/base-unit amount, items and status to the accepted ride agreement. A description `ride=...` is only a discovery hint. Persist payment intent before PIN. After submission, verify exact invoice and transaction/history evidence. A ride update failure is `ride_projection_pending`, never a payment failure and never permission to repay. Paid invoices disappear from the outstanding list, so recovery must query the retained exact address and paid/history endpoints.

## Browser and external-service boundary

The WebView directly contacts:

- `ipapi.co` automatically after GPS failure/denial;
- Nominatim with user-entered address text;
- OpenStreetMap tile servers with viewed map areas;
- the default public OSRM demo router through Leaflet Routing Machine with exact route coordinates.

These requests do not pass through an application backend or Nexus wallet proxy. GPS denial must not silently become consent to IP geolocation. Production must disclose each recipient, require explicit opt-in for location/geocoding/routing, URL-encode/debounce/cancel searches, use a production-suitable configurable router, and prevent exact coordinates/searches from entering logs or durable state. Precise pickup/destination and live location must move off-chain under the v0.2 privacy model before public deployment.

## Discovery, packaging and quality boundary

All global reads must paginate, deduplicate and expose `{records, complete, error}`. Later-page failure is partial/error, not an empty network. Validate UTF-8 byte size before PIN; the 2026-09-23 offline probe reached a 2,544-byte ride payload and still invoked the secure-call stub.

The production manifest is the wallet file-serving allowlist. A clean build emits `app.js` plus seven referenced PNGs; all seven are absent from `nxs_package.json.files`. The module is not installable as a complete production package until a clean folder/zip install on the pinned NexusInterface version renders every marker/layer/routing asset with zero denied requests.

No checked-in test, lint or CI command exists. Release acceptance requires collected contract, envelope, transition, storage/redaction, network-denial, package-inventory and browser-integration tests plus the clean production build. Live acceptance uses isolated non-production profiles and funds only after all offline gates pass.
