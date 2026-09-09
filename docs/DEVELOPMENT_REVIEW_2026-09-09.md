# Development and architecture review — 2026-09-09

## Scope and continuity

**Reviewed HEAD:** `d2f9ac053eba610e15c113db1584896c9701cf76` (`master`, aligned with the locally recorded `origin/master` at review start).

**Runtime baseline:** `7a9c4816852d3e0a7b500f64bfac035715502f86`.

The worktree was clean at review start. The only commit after the runtime baseline is `d2f9ac0` (`docs: establish architecture and payment acceptance review`). Its delta contains only `README.md` and documentation. There is no source, test, manifest, dependency, lockfile, build-configuration, or workflow delta to classify as new runtime work. The known invoice, settlement, package and discovery findings below are unchanged defects, not regressions introduced since the 2026-09-08 review.

No live wallet/core call, financial mutation, dependency upgrade, lockfile edit, code modification, stage, commit, or push was performed.

## Code evidence re-inspected

### P1 — Nested invoice payload is still discarded

`src/api/nexusAPI.js:259-273` still reads `recipient`, `account`, `amount`, `status`, and `items` only from the invoice top level. `listOutstandingInvoices` at lines 529-540 still maps through that function. It does not decode the current-core `{address, owner, json: {...payload, status}}` shape established in the prior source-pinned review, so amount and ride-reference data are still lost. No runtime delta changes this conclusion.

**Repair exit:** introduce one versioned adapter that preserves top-level register metadata separately from nested payload, rejects malformed/error envelopes instead of manufacturing zero/empty defaults, and prove a supported-core fixture plus isolated read retains address, owner, recipient, account, token, amount, status, items, and ride reference.

### P1 — Payment success and ride projection remain conflated

`src/App/Passenger.js:272-308` still pays, immediately writes passenger ride status `paid`, and reports both as one success. It persists no intent or returned transaction identity and performs no invoice/transaction read-back. `payRideInvoice` at `src/api/nexusAPI.js:570-574` still accepts any resolved bridge value. Invoice selection at `Passenger.js:344-350` remains the first description containing `ride=<alphanumeric>`, without binding issuer/account/token/recipient/amount to accepted ride terms.

**Repair exit:** persist profile/network/ride/invoice-scoped public intent before payment; freeze and reread canonical terms; allow at most one pay; hold ambiguous responses as `submission_unknown`; verify exact invoice and transaction evidence; then separately read/write/read the passenger-owned ride projection. A projection failure must not invite repayment.

### P1 — Production package allowlist still omits emitted PNGs

`webpack.config.babel.js:29-31` still emits images as resources, while `nxs_package.json:10` lists only `dist/index.html`, `dist/nexgo-logo.svg`, and `dist/js/app.js`. No manifest or build-config delta repairs the seven PNG omissions found by the last executed production build.

**Repair exit:** after a clean production build, assert every runtime JS/CSS/image/chunk reference is explicitly listed or deliberately inlined, then install the production folder/zip into a supported Nexus Interface and prove map, marker, layer, and routing assets load without denied requests.

### P2 — Discovery, size, privacy and automation gates remain open

`createRideRequestAsset` at `src/api/nexusAPI.js:446-479` still publishes precise coordinates and unbounded labels before `secureApiCall`. Enumeration remains fixed-limit and several failures become empty arrays/objects (`369-418`, `482-513`, `529-540`). `package.json` still has only `build` and `dev`; no test/lint scripts or checked-in CI gate were introduced.

**Repair exit:** validate coordinates, identity and UTF-8 bytes before PIN; return explicit complete/partial/error discovery state with paginated cancellation/deduplication; define and test the legacy-to-v0.2.0 privacy/schema migration; and add Nexus Interface-compatible test/lint/package-inventory/clean-build CI without blind dependency upgrades.

## Verdict

Runtime architecture and payment readiness are **unchanged since 2026-09-08**. NexGo remains a useful prototype but is not payment-release-ready. PIN-confirmed writes are a positive wallet boundary; they do not supply invoice authorization, payment evidence, reconciliation, privacy, complete discovery, or package correctness.

## Reviewed snapshot hashes

Committed tree: `a65d9fb4f3ddf0e56ee580e12adeac0ec981de8d`.

| Reviewed runtime file | SHA-256 |
|---|---|
| `package.json` | `3ed1903e47218a1c38f3851120d9b4b8efd16e819c04d5d15fb292c5a907c408` |
| `package-lock.json` | `db5aa094b6f5641d224944f97170dad4587e7940c83a6fdce2a2b5941827e09f` |
| `nxs_package.json` | `ce239047566e97b05adbb20528bebcc528ccc4a91eac8e002b20944ac814e819` |
| `webpack.config.babel.js` | `53f7f6009763617cdb9acb1a7bb5785c156f5fed0c4fa8e90c73a0abf8426edf` |
| `src/api/nexusAPI.js` | `54cea675cb0de1c8c281e6ad451a86e70b65cf3f6628f0318234ce7b6e6ebd2b` |
| `src/App/Passenger.js` | `b925f41ba3a852fd3816a6fb040fcdfbed2553002191a9302959295c7c97809d` |

## Verification evidence

The configured commands were requested on 2026-09-09, but execution was approval-denied before any command ran. They are **not** reported as fresh results and were not retried through another mechanism. Latest actual evidence remains the 2026-09-08 run at the unchanged runtime baseline:

| Command | Latest actual result | 2026-09-09 status |
|---|---|---|
| `npm run build` | 2026-09-08 PASS after dependency install; Webpack 5.105.1, 872 KiB app, 7 emitted PNGs, 3 performance warnings | Not executed; approval denied |
| `npm test` | 2026-09-08 FAIL — script absent | Not executed; approval denied; script remains absent in `package.json` |
| `npm run lint` | 2026-09-08 FAIL — script absent | Not executed; approval denied; script remains absent in `package.json` |
| package inventory probe | 2026-09-08 FAIL — all 7 emitted PNGs absent from manifest | Not freshly executed; manifest/build config unchanged |

No checked-in test or lint gate, live supported-core fixture, Nexus Interface production installation, or payment/restart acceptance exists at this head.