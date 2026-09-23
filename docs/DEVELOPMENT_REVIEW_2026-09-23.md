# Development and architecture review — 2026-09-23

## Baseline and verdict

**Reviewed repository:** `/home/brutus/github/NexGo`, branch `master`, HEAD `78435a5d4f97deb27770051a8aacbbc3360b9355`, matching `origin/master` at the clean baseline. HEAD is the 2026-09-17 documentation commit; no runtime change followed upstream feature `7a9c481`. The reviewed runtime identity is `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`.

**Verdict: prototype; not invoice-creation-compatible with current Nexus core and not payment-release-ready.** The production bundle builds reproducibly offline, but the invoice writer uses an obsolete/wrong request key, the current invoice response shape is normalized to empty terms, ride ownership and revision handling fail open, issue/pay effects are not reconciled, browser privacy boundaries are incomplete, and the package omits seven runtime assets. No wallet/core mutation, profile write, invoice or payment was executed.

## Exact source evidence

### NexGo

| Object | Git object |
|---|---|
| Repository HEAD | `78435a5d4f97deb27770051a8aacbbc3360b9355` |
| HEAD tree | `a029248c60f568aaec7f3fd385e4566ca8a6f562` |
| `src/` tree | `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f` |
| `src/api/nexusAPI.js` | `3ef4357a3e16d98867e806a0ec7ae9f243d3ba6f` |
| `src/App/Passenger.js` | `951337e6d839c33a3a588bb00f67e9a78cc2f892` |
| `src/App/Driver.js` | `dd4564c23718353b402409ec84b26f0ce809f531` |
| `src/configureStore.js` | `b60ccc2c5bc97bc39fe0d1386f9c40e81104fbdc` |
| `package.json` / lockfile | `0d7de8c13d8fb457a97bc20720e2a23a29228682` / `81e2beeb4b52330f1983d5ed318b3277e32e286a` |
| wallet manifest / Webpack | `600d0ce357c6ac7957f64ffc6c869e70f03813e0` / `d07dbe5404b19a471fada926a1b4c90017f2c2da` |

### Current external contracts (read-only source snapshots)

- LLL-TAO stable `master`: `1185145534a20ed4d2288e4513c505f271be536d` (commit message `Tag for release 5.1.6`, 2026-04-10). Invoice create/json blobs: `ebd87badd6075db45719d30f740c7cf9b55f94bc`, `3a24e1fa9b2a2542bc46b35ee9f381ee7cb29b10`; address extraction `aeca4c437abc68116175f51ceee2278ab2ed48f9`; generic update `14d796e0171cf860ceacdbbd797be928953c7f28`.
- LLL-TAO development `merging`: `8af9c3387244b4d396c0e00ee81cea78bb9c0177` (2026-09-03). Its invoice create/json blobs are identical to stable; command registration, pay and update paths were also inspected.
- NexusInterface `master`: `1e923d46a9cde1cf22b8608257c92da184aece16` (2026-04-22). Module WebView `e905ee956df9f51fdd5189f6066d8dba0a2ecd08`; API transport `b1239074ae85e9d79d771d98fadb0c65aa0b5ca1`; preload bridge `26d73318e40f1fa715464f67c1bd0b2ecdb44167`.
- Distordia Standards `main`: `83b9f0902a062d9a97089c2729889ea565d7af82` (2026-08-02). Ride/taxi/rating v0.2 draft blobs: `0df8f039ec831a047ee23e3c1970cb73438438d3`, `d7c0003b756f00b1920fa6463f9a47dcf33ea9bb`, `68fa21a0fe14bb77865f77297732972e7fe617bc`.

These snapshots establish source semantics only. No live node version, wallet install or network response was asserted.

## Executed verification

All application diagnostics were offline. Probe scripts lived only under the active profile scratch directory and replaced wallet calls with stubs; they are not repository tests.

| Exact command | Result |
|---|---|
| `git rev-parse --show-toplevel && git branch --show-current && git rev-parse HEAD && git show -s --format='%H%n%cI%n%s' HEAD && git status --short --branch` | **PASS** — expected path, `master`, HEAD `78435a5…`, clean and tracking `origin/master` |
| `git fsck --no-dangling --no-progress && git diff --check && git status --porcelain=v1` | **PASS** — no output at baseline |
| `npm ci --offline --ignore-scripts` in a clean clone at `78435a5…` | **PASS** — 2,112 packages added from cache in 4 s, 2,113 audited, 0 vulnerabilities; deprecation warnings present |
| `npm ls --depth=0` in that clean clone | **PASS WITH HYGIENE WARNING** — exit 0 and expected direct dependencies resolved; `bindings@1.5.0`, `file-uri-to-path@1.0.0` and optional `nan@2.25.0` were reported extraneous |
| `npm run build` in that clean clone | **PASS WITH WARNINGS** — Webpack `5.105.1`; 872 KiB `app.js`; seven PNGs / 9.05 KiB; three performance warnings; 3.083 s |
| `sha256sum dist/js/app.js dist/js/*.png` in clean clone and original worktree | **PASS** — identical artifacts; `app.js` SHA-256 `dc8da220d937ba3c38a6349dba1f5ea11a30b0086bac82043f36ed4ec830043d` |
| `npm test` | **FAIL / ABSENT** — `Missing script: "test"` |
| `npm run lint` | **FAIL / ABSENT** — `Missing script: "lint"` |
| `node /home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-23/offline-contract-probe.cjs` | **DIAGNOSTIC PASS, CONTRACT FAIL** — reproduced all asserted defects without network or real wallet calls |
| `python3 /home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-23/package-inventory-probe.py` | **FAIL (exit 1)** — seven emitted PNGs are referenced by `app.js`; zero are listed in the manifest |
| `python3 /home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-23/current-api-boundary-probe.py` | **PASS (exit 0)** — confirmed current-core `to` input/nested `json` output/no update CAS and current wallet PIN/session/Basic-auth boundaries |

The clean build reproduced these seven unlisted referenced files: `19680205f5c5bd08dcaa.png`, `2b3e1faf89f94a483539.png`, `416d91365b44e4b4f477.png`, `6569279e9204af87ffa9.png`, `680f69f3c2e6b90c1812.png`, `8f2c4d11474275fbc161.png`, `a0c6cc1401c107b501ef.png`.

The contract diagnostic produced:

- nested current invoice → amount `0`, blank account/recipient, zero items, token dropped;
- ride with absent canonical owner → accepted with payload `passenger-genesis` as owner; register version dropped;
- stale raw ride update → zero reads, one secure-call stub, old payload merged and submitted;
- long-label ride request → 2,544 UTF-8 bytes and one secure-call stub instead of pre-PIN rejection.

## Findings

### P0 — Current-core invoice creation request is invalid

`createRideInvoice` sends `account: paymentAccount`. Both reviewed current stable and development `Invoices::Create` implementations call `ExtractAddress(jParams, "to")`; the generic extractor accepts `to`, `name_to` or `address_to`, not `account`. The current core therefore rejects this request before invoice creation. The repository's vendored invoice docs describe `account` and have drifted from source.

**Acceptance:** the writer uses one pinned, tested current input shape; an offline request-construction test asserts `to` is present and `account` absent; an isolated current-core test creates an invoice, records address/txid and reads the exact address back.

### P0 — Current invoice reads fail closed only by accident, not design

Current core's `InvoiceToJSON` places terms and status under `json`, with canonical register metadata outside. `normalizeInvoice` reads top-level terms and fabricates defaults. As a result, description correlation cannot find the ride and the passenger normally cannot see the invoice on its ride; if a shape happens to map, displayed amount/account can be false.

**Acceptance:** current nested fixtures preserve canonical address/owner plus exact account, recipient, token, amount, items and status; malformed/unknown/legacy variants reject explicitly. Description remains a lookup hint and cannot authorize payment.

### P0 — Canonical ownership and revision semantics fail open

`normalizeRideAsset` substitutes payload `passenger-genesis` when canonical `asset.owner` is missing and does not preserve/validate register metadata or payload version. Taxi selection similarly mixes canonical owner with mutable `driver`; driver setup prefers `driverAsset.driver` over active profile genesis. Taxi updates are addressed by a name rebuilt from mutable vehicle settings, and `listMyTaxiAssets` silently selects the first result.

`updateRideRequestAsset` merges component-held `currentData` and performs no pre-read. Current public core `assets/update` has no optimistic-revision argument and raw writes replace the whole payload, so the app cannot describe this as strict compare-and-set.

**Acceptance:** exact canonical address/owner and explicit schema are mandatory; redundant claims must match; mutations use the read-back address and active canonical owner; stale snapshots never enter a write; exact pre-read/digest/state checks and post-read evidence are collected. Cross-client CAS is either enforced by a proven core condition or avoided with append-only owner-specific records.

### P0 — Payment effects and projections are not recoverable

Driver issue and passenger pay remain view-level sequences. No intent, returned invoice identity, txid, unknown outcome or exact evidence survives restart. A taxi projection failure is reported as invoice creation failure; a ride projection failure is reported as payment failure. The UI only lists outstanding invoices, so an actually paid invoice disappears precisely when reconciliation is needed. Repeated action can duplicate invoice/payment effects.

The core invoice payment itself is atomic DEBIT + CLAIM and validates status/token; that does not make the later NexGo projection atomic.

**Acceptance:** durable issue/payment state machines persist public intent before PIN, retain address/txid/unknown outcome, reconcile by exact invoice and transaction/history, and allow projection-only recovery. Fault tests assert no blind repeat and at most one issue/pay mutation.

### P1 — Secret boundary is mostly correct; authorization gating is incomplete

Current NexGo does not store PIN/password/session/API Basic credentials. NexusInterface owns Basic auth, injects session only in multiuser mode, and `secureApiCall` displays parameters, prompts, then injects PIN. However current wallet source has no secure endpoint allowlist, and NexGo does not gate mutation on initialized wallet, active profile, intended network, sync status or supported mode/version. Persisted account names are public identifiers but still privacy-sensitive.

**Acceptance:** fixed endpoint constants, schema-validated displayed params, write gates for user/network/sync/version/mode, storage/redaction tests, and explicit proof that no secret enters storage/logs.

### P1 — Browser integrations leak data outside the wallet boundary

The WebView directly contacts `ipapi.co`, Nominatim, OpenStreetMap tiles and Leaflet Routing Machine's default public OSRM demo endpoint. GPS failure/denial automatically triggers IP geolocation without separate consent. Search text and exact route coordinates leave the wallet; the OSRM package itself warns its demo server is not suitable for production. Calls do not use a NexGo backend or the wallet proxy.

**Acceptance:** explicit provider-specific consent, no fallback after denial, configurable production endpoints, encoded/debounced/abortable searches, redacted telemetry, and browser tests proving no third-party request before consent.

### P1 — Production package is incomplete

The manifest permits only `dist/index.html`, `dist/nexgo-logo.svg` and `dist/js/app.js`. All seven emitted PNGs are referenced by `app.js` and omitted. A successful Webpack build is not an installable wallet package.

**Acceptance:** package inventory has zero referenced/unlisted runtime files and folder/zip installation on the pinned wallet renders all map/routing assets without denied requests.

### P2 — Discovery, payload bounds, privacy and standards migration remain open

Fixed limits (100/500) are not paginated; errors collapse to empty lists/maps. Existing precise ride coordinates/labels and taxi coordinates create permanent public history. Rating state is one growing raw blob and is not ride-gated. Current writes are legacy v1, while Distordia v0.2 drafts define typed passenger request, driver offer, passenger agreement and per-agreement rating records with coarse geohashes and off-chain live location.

**Acceptance:** complete/partial/error pagination semantics, pre-PIN UTF-8 byte validation, bounded data, explicit legacy/v0.2 adapters, owner-role checks, fare/invoice binding and no precise public coordinates. Draft compliance is not claimed until shared executable fixtures and unresolved CAS/namespace/self-address recovery questions are closed.

## Required repair order and remaining gates

1. **Batch 1A:** correct current-core invoice input and strict current envelopes.
2. **Batch 1B:** canonical ownership/address plus honest revision/conflict handling.
3. **Batch 1C:** durable issue/pay reconciliation and projection recovery.
4. **Batch 1D:** checked-in tests/lint/CI and complete production package.
5. **Batch 2:** consented browser integrations, complete discovery and explicit Distordia v0.2 migration.

Still not run and still required: supported-wallet folder/zip installation, live/current-core compatibility, isolated two-profile synthetic settlement, wrong-network/sync/client-mode gates, restart/crash recovery, exact transaction/history readback, profile/session cleanup and exact-head CI. Real funds, production profiles and production writes remain prohibited for this acceptance scope.

## Scope and denied operations

Only `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT_PLAN.md` and this dated review were intentionally changed. Build output is ignored and scratch probes are outside the repository. No dependency version or lockfile changed. An archive-extraction preparation command and a piped source-filter command were denied by unattended approval policy; they were not treated as evidence. Independently permitted clean-clone, direct file-read and source-hash checks are reported above.