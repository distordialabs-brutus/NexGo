# NexGo architecture and acceptance boundaries

Reviewed 2026-09-25 against repository commit `7da750f1f674b9af839c650f6a47bc4a2af15729` (`master`, matching `origin/master` at review start). The commit changed documentation only relative to `78435a5d4f97deb27770051a8aacbbc3360b9355`; the application remains `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`. See the current [evaluation](EVALUATION.md), [development plan](DEVELOPMENT_PLAN.md), and [2026-09-25 executed review](DEVELOPMENT_REVIEW_2026-09-25.md).

**Status: prototype. Invoice creation/decoding, canonical authority, durable settlement, package closure, privacy, and engineering gates remain release blockers. No real payment, invoice, profile mutation, or production release is approved.**

## Authority and reviewed sources

| Source | Exact identity | Architectural use |
|---|---|---|
| NexGo | `7da750f1f674b9af839c650f6a47bc4a2af15729` | Reviewed application and tracked documentation |
| NexGo runtime | `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f` | Actual module behavior |
| LLL-TAO stable | `master` `1185145534a20ed4d2288e4513c505f271be536d` (5.1.6 release commit) | Supported initial invoice/register contract |
| LLL-TAO development | `merging` `8af9c3387244b4d396c0e00ee81cea78bb9c0177` | Drift check; reviewed invoice create/JSON behavior matches stable |
| NexusInterface | `master` `1e923d46a9cde1cf22b8608257c92da184aece16` | Module bridge and wallet trust boundary |
| Design context | untracked `vision.md`, SHA-256 `5829f1f5e8eb5a6a0fb748c4f031a29da2296b24a527c309fd741ddf411c6bfb` | Intended open mobility protocol; not implementation evidence and not part of the reviewed commit |

LLL-TAO source, not the vendored API prose, is authoritative when they disagree. Relevant stable blobs are invoice create `ebd87badd6075db45719d30f740c7cf9b55f94bc`, invoice JSON `3a24e1fa9b2a2542bc46b35ee9f381ee7cb29b10`, invoice pay `fd253638fadb2336430615ae57d2cd3f489a294c`, invoice registration `initialize.cpp`, address extraction `aeca4c437abc68116175f51ceee2278ab2ed48f9`, and generic update `14d796e0171cf860ceacdbbd797be928953c7f28`. These snapshots establish source semantics only; no live node or wallet acceptance was run.

## Product boundary from the design context

The intended product is an open coordination protocol, not a closed taxi marketplace. Independent wallets, fleet systems, accessibility clients, human providers, and autonomous agents should be able to use the same versioned records. Namespace/accountability claims, role-owned request/offer/agreement records, settlement evidence, and portable reputation are target protocol concepts.

The current code does not implement that target. It publishes legacy taxi and ride data, including precise coordinates, and uses one wallet module UI as the orchestrator. Human and autonomous labels are payload strings rather than verified delegation/capability evidence. Ratings are mutable per-passenger maps rather than agreement-bound evidence. Architecture and UI must describe these as prototype records, not Distordia v0.2 compliance.

## Runtime and trust boundaries

```text
NexGo React module (Electron WebView)
  - Redux UI and non-secret module settings
  - component-local ride/invoice orchestration
  - direct browser requests to location/map/routing providers
             |
             | nexus-module IPC: apiCall / secureApiCall / storage
             v
NexusInterface main process
  - owns core transport and HTTP Basic credentials
  - injects active multiuser session
  - displays endpoint/parameters and requests PIN for secureApiCall
  - reviewed source does not enforce a secure endpoint allowlist
             |
             | POST JSON over wallet-selected HTTP(S)
             v
Trusted Nexus core
  - sigchain authorization and canonical register state
  - invoice conditional transfer and atomic pay DEBIT + CLAIM
```

NexGo has no application backend. Passwords, PINs, API Basic credentials, and session IDs must never enter module state, storage, logs, or protocol records. Current writes use `secureApiCall`, so the wallet owns PIN/session handling. That is a useful boundary, but a PIN prompt is not application-level authorization: mutating endpoint identifiers need to be fixed internal values, every displayed parameter needs validation before the prompt, and writes need gates for active profile, intended network, synchronization, compatible core/wallet, and supported mode.

## Current records and canonical identity

- Taxi assets are typed JSON registers. Current code stores mutable `driver`, vehicle data, exact coordinates, and timestamps.
- Ride requests and ratings are raw JSON registers. Ride payload `version: 1` is not the Distordia v0.2 request/offer/agreement model.
- Core invoice registers are readonly typed payloads. `InvoiceToJSON` returns register metadata at top level and invoice terms/status under `json`.
- Core invoice payment is one atomic DEBIT + CLAIM transaction. Taxi occupancy and passenger ride projections are separate application writes and are not atomic with payment.

For taxi and ride assets, canonical `address` and `owner` are mandatory. Payload identity fields are assertions only and must match canonical ownership or an independently verified namespace/delegation binding. Missing owner/address, unknown schema/state, malformed coordinates, unsafe number conversion, or contradictory identity must reject rather than default to empty values, `(0,0)`, `requesting`, or a payload-supplied owner.

Taxi mutation must target the exact canonical address selected from a verified owned register. The current name rebuilt from mutable vehicle settings, first-result selection, and preference for mutable `driver` over the active profile genesis are not authority.

## Invoice contract and ownership lifecycle

The reviewed stable core establishes these semantics:

1. `invoices/create/invoice` calls `ExtractAddress(jParams, "to")`. Accepted destination forms are `to`, `name_to`, and `address_to`; `account` is not a destination input. The resulting canonical destination is stored as `json.account`.
2. Creation requires `recipient` and non-empty `items`, derives token/decimals from the destination account, calculates the total, and returns an invoice address and transaction ID.
3. `InvoiceToJSON` starts with canonical register JSON, then adds invoice status inside `json`. Terms such as `account`, `recipient`, `items`, `amount`, and `token` are nested under `json`.
4. Invoice ownership is lifecycle state, not immutable issuer identity. The registered `outstanding` standard requires a system-owned conditional register; `paid` requires current owner equal to `json.recipient`. The create transaction genesis/retained issue evidence, not the current top-level `owner`, must establish issuer authority.
5. `invoices/pay/invoice` reads the exact invoice, rejects paid/cancelled status and wrong-token source accounts, and emits one DEBIT plus CLAIM transaction.
6. Core invoice source serializes amount values through floating-point JSON and reconstructs payment amount from `double * token figures`. NexGo must submit decimal strings, retain expected integer base units, constrain the supported amount domain, and verify the actual target-core transaction amount. JavaScript `parseFloat` is not acceptable money state.

Current NexGo violates the first two adapter boundaries: `createRideInvoice` sends `account`, while `normalizeInvoice` reads terms at the top level and fabricates empty/zero defaults. The UI therefore cannot safely issue, identify, display, authorize, or reconcile current-core invoices.

## Revision and transition strictness

Current public `assets/update/*` source exposes no compare-and-set parameter. Raw updates replace the allocated payload. The core-returned register version is not an application CAS token. Therefore:

1. Never merge component-held `currentData` into a write.
2. Persist a public intent snapshot before a consequential call.
3. Immediately before the PIN prompt, read the exact address and compare canonical owner, schema, state, modified metadata, and canonical payload digest.
4. Permit only a declared transition from that snapshot; serialize one pending operation per wallet/profile/network/register.
5. Persist returned remote identity or `submission_unknown` before any projection.
6. Read back the exact register/history/transaction before finalizing.
7. Treat changed prestate as conflict requiring a fresh decision.
8. Do not claim cross-client serializability unless a core-enforced condition is demonstrated. Prefer append-only, role-owned records where stale whole-blob replacement would control money or authority.

## Required settlement protocol

```text
issue_intent_persisted
  -> issue_submitted
  -> invoice_identity_known | submission_unknown
  -> invoice_readback_verified
  -> taxi_projection_pending
  -> issue_complete

payment_intent_persisted
  -> payment_submitted
  -> payment_tx_known | submission_unknown
  -> invoice_and_transaction_verified
  -> ride_projection_pending
  -> complete
```

Issuance intent freezes ride/taxi canonical addresses, passenger, provider, destination account, token, item decimals, integer base units, protocol version, and intended network. The returned invoice address and creation txid must be retained before taxi projection. A timeout, undefined response, or projection failure never authorizes invoice recreation.

Payment intent freezes the retained issuance identity and exact invoice terms. Description text such as `ride=...` is only a discovery hint. Immediately before payment, reread the exact address and verify nested terms, lifecycle owner/status, creation identity, payer account token, and expected base units. After payment, verify the returned transaction and exact invoice transition. A projection failure is `ride_projection_pending`, not payment failure and never permission to repay.

Recovery uses only registered/source-confirmed reads: exact `invoices/get/invoice`, invoice `history`/`transactions`, and ledger transaction lookup where required. It must not infer absence from `invoices/list/outstanding`, because a paid invoice leaves that subset.

## Browser privacy and external services

The WebView currently contacts `ipapi.co`, Nominatim, OpenStreetMap tiles, and Leaflet Routing Machine's default router. GPS failure or denial automatically triggers IP geolocation. Search text, viewed map areas, and exact route coordinates leave the wallet boundary. Exact pickup/destination labels and coordinates are then written on-chain.

Production architecture must require provider-specific consent, never reinterpret GPS denial as IP-location consent, encode/debounce/cancel searches, use configurable production-suitable providers, and keep precise trip/location data off-chain. The target protocol may publish coarse expiring discovery and integrity commitments, with private capability-controlled handoff between authorized parties.

## Discovery, packaging, and quality boundary

All global enumeration must paginate, deduplicate, and return `{records, complete, error}`. A bounded page or transport failure is not proof of an empty market. Known state must remain visible as partial/stale when refresh fails.

The production manifest is the NexusInterface file-serving allowlist. The clean reviewed build emits seven PNGs referenced by `app.js`; none is listed in `nxs_package.json.files`. A successful Webpack compilation is therefore not evidence of an installable production module.

No checked-in `test`, `lint`, package-inventory, or CI gate exists. Release acceptance requires collected adapter, authority, transition, durability, privacy, pagination, package, and browser tests; a clean production folder/zip install on the pinned wallet; and isolated non-production current-core acceptance before any real-value use.
