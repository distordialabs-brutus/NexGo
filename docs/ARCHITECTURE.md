# NexGo architecture and acceptance boundaries

Reviewed 2026-10-02 against repository commit and supplied baseline `b775fd572b64b5b8ee246be87f899a72613d7d9e`. There are zero commits after the baseline. Commits since the last runtime change remain documentation-only; the application is still `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`. See the current [evaluation](EVALUATION.md), [development plan](DEVELOPMENT_PLAN.md), [next coding contracts](NEXT_CODING_CONTRACTS.md), and [2026-10-02 executed review](DEVELOPMENT_REVIEW_2026-10-02.md).

**Status: prototype. Invoice creation/decoding/cancellation, canonical authority, durable settlement, package closure, privacy, and engineering gates remain release blockers. No real payment, invoice issue/cancel, profile mutation, or production release is approved.**

## Authority and reviewed sources

| Source | Exact identity | Architectural use |
|---|---|---|
| NexGo | `b775fd572b64b5b8ee246be87f899a72613d7d9e` | Reviewed application and tracked documentation |
| NexGo runtime | `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f` | Actual module behavior |
| LLL-TAO stable | `master` `1185145534a20ed4d2288e4513c505f271be536d` (5.1.6 release commit) | Supported initial invoice/register contract |
| LLL-TAO development | `merging` `8af9c3387244b4d396c0e00ee81cea78bb9c0177` | Drift check; reviewed invoice create/JSON behavior matches stable |
| NexusInterface | `master` `1e923d46a9cde1cf22b8608257c92da184aece16` | Module bridge and wallet trust boundary |
| Design context | untracked `vision.md`, SHA-256 `5829f1f5e8eb5a6a0fb748c4f031a29da2296b24a527c309fd741ddf411c6bfb` | Intended open mobility protocol; not implementation evidence and not part of the reviewed commit |

LLL-TAO source, not the vendored API prose, is authoritative when they disagree. The 2026-10-02 review re-fetched the pinned stable snapshots and recorded SHA-256 for invoice create `ffec21ac…`, JSON `91c5e740…`, pay `46ce222b…`, cancel `1260a5cd…`, address extraction `274ed9b9…`, generic transaction build `72a1cd63…`, session create `fca095d3…`, session status `dc093742…`, and NexusInterface API/session/bridge sources. These snapshots establish source semantics only; no live node or wallet acceptance was run.

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

NexGo has no application backend. Passwords, PINs, API Basic credentials, and session IDs must never enter module state, storage, logs, or protocol records. Current writes use `secureApiCall`, so the wallet owns PIN/session handling. In multiuser mode, pinned `callAPI` injects the wallet-selected active session; in single-user mode it sends no session. The module receives `coreInfo` and `userStatus`, not the session secret, through `state.nexus`. The reviewed per-call secure path supplies PIN to core, so `userStatus.unlocked.transactions === false` is not itself a rejection condition; it would matter for a pinless cached-unlock path. This is a useful boundary, but a PIN prompt is not application-level authorization: mutation endpoint identifiers must be fixed internal values, every displayed parameter needs validation before the prompt, and writes need gates for initialized wallet state, stable profile genesis, indexing complete, intended network, synchronization, compatible core/wallet, and supported mode. Current settlement handlers do not use those available gates.

The pinned wallet resolves a cancelled PIN prompt as a fulfilled `secureApiCall` with `undefined`, not as a rejected promise. Its HTTP client also resolves an empty, malformed, or missing-result HTTP 2xx response as `undefined`, and the module preload exposes both identically. Every NexGo mutation adapter must therefore require a strict command-specific success envelope and expected remote identities. For pinned invoice issue, pay, and cancel, direct `BuildResponse` includes the exact invoice `address` and transaction `txid`; both are required and the returned address must match the frozen target for pay/cancel. `undefined`, an error member, a malformed object, or a missing/mismatched address/transaction ID is non-success. More importantly, bare `undefined` is **ambiguous**, not authoritative authorization cancellation: it becomes `submission_unknown`, prohibits projection and automatic retry, and cannot close an intent as non-submitted. Only a future trusted host result that explicitly proves cancellation before `callAPI` may produce `cancelled_before_submit`. Current issue, pay, and invoice-cancel handlers violate this boundary: issue/pay advance projections, while cancel shows success and reloads, without validating a result or exact evidence.

Pinned core `BuildResponse` has an additional compatibility boundary: with `-autotx`, contracts may be queued and the response can contain `success`/`address` without any `txid`. This is not durable settlement evidence. NexGo must either prove `-autotx` disabled before enabling real-value actions or implement a separately reviewed durable queue identity and recovery protocol. A missing `txid` remains unresolved and never authorizes projection, success UI, deletion, compensation, or retry.

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
6. `invoices/cancel/invoice` reads nested recipient/status, rejects paid or already-cancelled invoices, locates the original conditional transfer, and builds a VOID. Direct-mode `BuildResponse` returns the invoice address and cancel txid; finality still requires exact cancelled-state and void/history readback.
7. Core invoice source serializes amount values through floating-point JSON and reconstructs payment amount from `double * token figures`. NexGo must submit decimal strings, retain expected integer base units, constrain the supported amount domain, and verify the actual target-core transaction amount. JavaScript `parseFloat` is not acceptable money state.

Current NexGo violates all adapter boundaries: `createRideInvoice` sends `account`; `normalizeInvoice` reads terms at the top level and fabricates empty/zero defaults; mutation helpers/UI treat a fulfilled `undefined` result as success; cancel has no retained intent/result/evidence; and settlement ignores wallet readiness. The UI therefore cannot safely issue, identify, display, authorize, cancel, reconcile, or project current-core invoices.

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
  -> authorization_cancelled                         # only explicit host pre-submit evidence
  -> issue_submitted
  -> invoice_identity_known | submission_unknown
  -> invoice_and_creation_tx_verified
  -> taxi_projection_pending
  -> issue_complete

payment_intent_persisted
  -> authorization_cancelled                         # only explicit host pre-submit evidence
  -> payment_submitted
  -> payment_tx_known | submission_unknown
  -> invoice_and_transaction_verified
  -> ride_projection_pending
  -> complete

cancel_intent_persisted
  -> authorization_cancelled                         # only explicit host pre-submit evidence
  -> cancel_submitted
  -> cancel_tx_known | submission_unknown
  -> invoice_cancel_and_void_verified
  -> cancel_projection_pending
  -> cancel_complete
```

Issuance intent freezes ride/taxi canonical addresses, passenger, provider, destination account, token, item decimals, decimal wire strings, integer base units, protocol version, and intended network. Before submission, validate wallet readiness and exact prestates. An explicit trusted host cancellation result may close the intent without projection. On the pinned wallet, a bare cancellation/`undefined` result remains `submission_unknown` because it is indistinguishable from accepted-but-empty output. A successful response must contain the invoice address and creation txid; retain both before taxi projection. Read back the exact invoice and creation transaction, prove the expected provider genesis created/transferred that invoice address, and compare the nested terms. A timeout, undefined response, malformed envelope, or projection failure never authorizes invoice recreation.

Payment intent freezes the retained issuance identity and exact invoice terms. Description text such as `ride=...` is only a discovery hint. Immediately before payment, reread the exact address and verify nested terms, lifecycle owner/status, creation transaction/genesis evidence, payer account token, and expected base units. Any non-success result causes no projection; current-host `undefined` remains `submission_unknown`. After a successful response, persist the payment txid and verify its exact DEBIT/CLAIM contracts plus the invoice ownership/status transition before projecting the ride. A projection failure is `ride_projection_pending`, not payment failure and never permission to repay.

Invoice cancellation is a consequential mutation with the same journal boundary. Freeze the exact invoice address, creation transfer identity, issuer genesis, expected outstanding state, network/profile, and intended projection before authorization. Require a matching response address plus cancel txid, persist both, then verify the exact VOID/history and `CANCELLED` state before reporting success or changing ride/taxi state. A bounded outstanding-list refresh is discovery only: disappearance may mean paid, cancelled, filtered, paginated, or failed. Undefined, timeout, address mismatch, missing txid, or `-autotx` queue-only response remains unresolved and blocks a second cancel.

Recovery uses only registered/source-confirmed reads: exact `invoices/get/invoice`, invoice `history`/`transactions`, and ledger transaction lookup where required. It must not infer absence from `invoices/list/outstanding`, because a paid invoice leaves that subset.

`authorization_cancelled` requires authoritative host evidence that authorization ended before submission. A bare fulfilled `undefined` is sufficient to prohibit projections, but is not durable proof that no submission occurred: the pinned host uses the same module-visible value for PIN cancellation and an HTTP 2xx empty/malformed/missing result. Therefore the supported bridge cannot currently produce `authorization_cancelled`; it must retain `submission_unknown`. Test both a future explicit pre-submit cancellation result and accepted-but-empty response; neither projects success, and only the explicit discriminant may establish non-submission.

The pinned module `updateStorage` path is fire-and-forget: the WebView handler calls `writeModuleStorage` without returning an acknowledgement, revision, or CAS token to the module. It cannot prove an intent was durably committed before a secure call or serialize independent WebViews. Batch 1C therefore has a host prerequisite: add and pin an acknowledged authoritative journal interface with revision/CAS semantics, or keep issue/pay/cancel disabled. Redux persistence remains suitable only for non-authoritative preferences.

## Browser privacy and external services

The WebView currently contacts `ipapi.co`, Nominatim, OpenStreetMap tiles, and Leaflet Routing Machine's default router. GPS failure or denial automatically triggers IP geolocation. Search text, viewed map areas, and exact route coordinates leave the wallet boundary. Exact pickup/destination labels and coordinates are then written on-chain.

Production architecture must require provider-specific consent, never reinterpret GPS denial as IP-location consent, encode/debounce/cancel searches, use configurable production-suitable providers, and keep precise trip/location data off-chain. The target protocol may publish coarse expiring discovery and integrity commitments, with private capability-controlled handoff between authorized parties.

## Discovery, packaging, and quality boundary

All global enumeration must paginate, deduplicate, and return `{records, complete, error}`. A bounded page or transport failure is not proof of an empty market. Known state must remain visible as partial/stale when refresh fails.

The production manifest is the NexusInterface file-serving allowlist. The clean reviewed build emits seven PNGs referenced by `app.js`; none is listed in `nxs_package.json.files`. A successful Webpack compilation is therefore not evidence of an installable production module.

No checked-in `test`, `lint`, package-inventory, or CI gate exists. Release acceptance requires collected adapter, authority, transition, durability, privacy, pagination, package, and browser tests; a clean production folder/zip install on the pinned wallet; and isolated non-production current-core acceptance before any real-value use.
