# Next coding contracts

Reviewed 2026-10-02 against NexGo `b775fd572b64b5b8ee246be87f899a72613d7d9e`, LLL-TAO `1185145534a20ed4d2288e4513c505f271be536d`, and NexusInterface `1e923d46a9cde1cf22b8608257c92da184aece16`. These are implementation contracts for Batch 0 and Batches 1A–1D, not claims that the current runtime satisfies them. The [2026-10-02 source review](DEVELOPMENT_REVIEW_2026-10-02.md) expands the historical 2026-09-30 contract with invoice cancellation, `-autotx`, session injection/readiness, and durable-storage requirements.

## 1. Closed mutation outcome

Every consequential adapter must return one discriminated outcome. Components must not consume a raw `secureApiCall` value.

```js
// Public, non-secret fields only.
{
  kind: 'accepted',
  operation: 'invoice_issue' | 'invoice_pay' | 'invoice_cancel' | 'projection',
  requestDigest: '<canonical SHA-256>',
  remote: { address: '<register address?>', txid: '<transaction id>' }
}

{
  kind: 'cancelled_before_submit',
  operation: '...',
  requestDigest: '<canonical SHA-256>',
  hostEvidence: { callId: '<host call id>', endpoint: '<fixed endpoint>' }
}

{
  kind: 'submission_unknown',
  operation: '...',
  requestDigest: '<canonical SHA-256>',
  reason: 'empty_result' | 'malformed_result' | 'transport_error' |
          'interrupted' | 'host_outcome_ambiguous'
}

{
  kind: 'rejected_before_submit' | 'remote_rejected',
  operation: '...',
  requestDigest: '<canonical SHA-256>',
  reason: '<closed reason code>'
}
```

Rules:

- Invoice issue, pay, and cancel accept only a non-empty canonical invoice `address` and exactly one non-empty `txid` in the pinned command-specific result. Pay/cancel response address must equal the frozen invoice target.
- A `success`/`address` response without `txid` is not accepted. Pinned core `-autotx` can produce this queue-only shape; it remains unresolved unless a separately reviewed durable queue protocol supplies an attributable identity and recovery contract.
- `undefined`, `null`, a scalar, an array, an error member, an empty object, or a missing identity is never `accepted`.
- A transport rejection after invocation is `submission_unknown` unless authoritative remote evidence proves the request was rejected before acceptance.
- `submission_unknown` prohibits projection, success UI, deletion, compensation, and automatic resubmission.
- UI handlers switch exhaustively on `kind`. Only `accepted` can advance to evidence verification; no component infers success from promise fulfillment.

## 2. Authoritative cancellation evidence

The pinned wallet does **not** expose authoritative cancellation. `webview.js` maps `confirmPin()` returning `undefined` to a fulfilled `undefined`; `api.js` also resolves `undefined` for an HTTP 2xx response with an empty, malformed, or missing `result`; `module_preload.js` resolves both identically.

Therefore:

1. Current-host `undefined` maps to `submission_unknown` with `host_outcome_ambiguous`, even when the user reports cancelling.
2. It is not retryable cancellation evidence.
3. `cancelled_before_submit` is legal only after a supported NexusInterface bridge emits a distinct trusted IPC result before `callAPI` is invoked, binding call ID, fixed endpoint, and request digest.
4. If the host cannot be upgraded, NexGo must keep real-value issue/pay/cancel disabled; application heuristics, timers, an empty outstanding list, or user recollection cannot substitute.
5. Tests must inject both explicit host cancellation and accepted-but-empty response. Both cause zero projections; only the explicit host result may close the intent as non-submitted.

## 3. Wallet session and readiness contract

NexusInterface owns session selection and PIN collection. In multiuser mode its API transport injects the active session; in single-user mode it sends no session. NexGo must never request, persist, log, or reconstruct a session or PIN. It must consume the wallet-supplied `state.nexus.coreInfo` and `state.nexus.userStatus` boundary instead.

Before intent commit and again immediately before any secure call, require initialized wallet state, unchanged active genesis, session indexing complete, intended network, synchronized/non-syncing core, supported lite/client mode, and compatible core/wallet version. Missing fields do not default healthy. The reviewed secure path supplies a PIN per call; do not incorrectly require cached `unlocked.transactions === true`. If a future pinless path is introduced, its unlock scope becomes a separate explicit prerequisite. A profile/network/readiness change after intent commit holds the operation for explicit reconciliation; it never silently rebinds the intent.

Real-value issue/pay/cancel additionally require a proven direct-transaction configuration with core `-autotx` disabled. Supporting `-autotx` is a separate protocol requiring durable queue identity, restart behavior, and exact eventual transaction reconciliation.

## 4. Strict invoice adapter

`decodeInvoice(value)` must return either a complete typed invoice or a typed rejection; it must not fabricate defaults.

Required canonical envelope:

- non-empty `address`;
- lifecycle `owner`;
- exact `created` and `modified` metadata in their source types;
- nested object `json`.

Required nested terms:

- non-empty `account`, `recipient`, and `token`;
- status in the explicitly supported lifecycle set;
- non-empty `items`;
- each item has strict decimal amount text at the NexGo boundary and an integer `units` value in range;
- nested total equals the integer-base-unit sum of all items.

The top-level owner is lifecycle evidence: the pinned core defines outstanding as system-owned and paid as recipient-owned. Issuer authority is the retained creation transaction/genesis bound to the exact invoice address. Description text such as `ride=...` is discovery-only.

`createRideInvoice` sends destination as `to`; it never sends destination as `account`. Supported aliases, if any, need independent tests. The canonical destination read back from the invoice is nested `json.account`.

Cancellation decodes the same strict invoice envelope and retains its creation transfer/issuer evidence. A cancel result is only provisional until the exact cancel tx/VOID evidence and `CANCELLED` readback match. Disappearance from `list/outstanding` is never cancel proof.

## 5. Exact money domain

- UI and service boundaries accept canonical decimal strings only: digits plus one optional decimal point; no sign, exponent, whitespace, separators, `Infinity`, or `NaN`.
- Convert once to integer base units using the verified token decimals. Reject excess scale, overflow, zero/negative values, and values outside the conservative target-core-tested range.
- Freeze the decimal wire string, token identity, token decimals, integer unit amount, units, and integer total in the intent before authorization.
- Never use `parseFloat`, `Number` multiplication, or a decoded core `double` as payment authority.
- Because pinned core create/pay crosses `double`, isolated target-core acceptance must compare the actual DEBIT integer against the frozen expected integer at below/exact/above supported boundaries.

## 6. Durable settlement journal

Persist public intent before opening wallet authorization. Scope keys to wallet installation, profile genesis, network identity, operation, and source register. Never persist PIN, password, Basic-auth credential, or session ID.

Minimum issuance fields:

- intent ID and request digest;
- ride and taxi canonical addresses, owners, prestates, and payload digests;
- provider genesis and destination account;
- token, decimals, item wire strings, integer units, integer total;
- invoice name/reference and protocol version;
- state, timestamps, invoice address, creation txid, and verification diagnostics.

Minimum payment fields:

- retained issuance identity and verified invoice terms;
- payer account, expected DEBIT source/destination/token/integer amount;
- state, timestamps, payment txid, and verification diagnostics.

Minimum cancellation fields:

- retained issuance address, creation transaction/contract and issuer genesis;
- frozen outstanding envelope/digest, network/profile, and intended projection;
- state, timestamps, cancel txid, exact VOID/history evidence, and verification diagnostics.

Allowed progress is intent → submitted → identity known or submission unknown → exact evidence verified → projection pending → complete. A projection failure retries only the idempotent projection. It never repeats issue/pay/cancel. Recovery uses exact invoice, transaction, and history reads; bounded list absence is not authoritative. A queue-only `success`/`address` result without txid follows the unresolved branch.

The storage API must provide an acknowledged authoritative commit plus revision/CAS semantics across independent WebViews and restarts. The pinned host's module `updateStorage` is fire-and-forget and provides no such acknowledgement. Redux component state, middleware invocation, or promise-shaped wrappers are insufficient evidence of durable global at-most-once behavior. Until a pinned host bridge supplies the contract, real-value issue/pay/cancel remain disabled.

## 7. Authority and transition contract

Before any mutation:

1. Read the exact canonical address.
2. Strictly decode owner, schema/version, lifecycle state, modified metadata, and payload digest.
3. Match active profile and intended network.
4. Compare against the frozen prestate and allow only one declared role-specific transition.
5. Commit the operation intent/lock before the secure call.
6. Validate the strict mutation outcome and read back the exact effect.

Never select `assets[0]`, rebuild a mutation target from mutable vehicle settings, prefer mutable payload `driver` over canonical owner, merge component-held `currentData`, or turn read/decode failure into an empty market/default record. This local protocol detects stale state but does not claim cross-client CAS where the core exposes none.

## 8. Package contract

`nxs_package.json.files` is the production serving allowlist. The package gate must:

- build from the unchanged lockfile;
- parse emitted JS/CSS/chunk references;
- require every referenced runtime file in the manifest and on disk;
- require declared entry and icon in the allowlist;
- build a clean folder and release zip from exactly the permitted production files;
- install both on the pinned wallet and assert zero denied/missing resource requests.

Current exact-head output references seven PNGs absent from the manifest. A successful Webpack compile is not a package pass.

## 9. Required collected tests

The default offline gate must include:

- issue/pay/cancel outcome table: explicit pre-submit authorization cancellation, undefined, null, empty object, malformed object, structured error, thrown transport error, address mismatch, `success`/address without txid, txid array, and valid direct-mode command identity;
- session/readiness table: single-user and multiuser injection, indexing, locked versus cached-unlocked transaction scope with per-call PIN semantics, missing status fields, wrong network, syncing, unsupported mode/version, profile/network switch, and proof that no PIN/session enters module storage;
- accepted-but-empty versus explicit host cancellation, asserting zero projections for both and retry permission for neither unless cancellation is authoritative;
- current nested invoice fixture, owner lifecycle, issuer transaction evidence, wrong account/token/recipient, copied description, and item/total mismatch;
- decimal below/exact/above scale/range and unsafe integer/exponent cases;
- crash/restart at every issue, pay, and cancel intent, submission, identity, verification, and projection boundary;
- cancel exact-state and VOID/history proof; paid/already-cancelled/wrong-issuer cases; bounded outstanding-list disappearance is never accepted;
- two independent WebViews/controllers prove acknowledged journal commit and CAS exclusion before any secure call;
- duplicate click and independent-controller stale snapshot;
- exact-address authority/prestate/readback cases;
- seven emitted PNG package cases plus folder/zip installation;
- no unmocked network or wallet access.

A diagnostic probe that successfully reproduces a defect is evidence, not a green release test. Batch status remains open until the repaired assertions are collected by one clean-clone command and exact-head CI.
