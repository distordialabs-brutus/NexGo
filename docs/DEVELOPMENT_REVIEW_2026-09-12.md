# Development and architecture review — 2026-09-12

## Scope and continuity

**Reviewed pre-publication HEAD:** `2effd33df1a11dceb9831774a694c95cbce0cb89` (`master`), equal to `origin/master` at review start.

**Prior 2026-09-10 review source HEAD:** `0109a57dec8a2f69e6f179f9c0d20084910bee67`. **Runtime baseline:** `7a9c4816852d3e0a7b500f64bfac035715502f86`.

No commit follows the 2026-09-10 review publication. `git diff 2effd33..HEAD` and the path-limited runtime diff are empty; the worktree was clean. Findings below are pre-existing at the unchanged runtime baseline, not post-review regressions. No wallet/core call, profile/session operation, financial mutation, dependency change or runtime repair was performed.

## Verdict

**Prototype; not payment-release-ready.** The 2026-09-10 invoice, payment, package, discovery, privacy and automation blockers remain. This review additionally closes a coverage gap in the prior plan: driver-side invoice creation has the same unsafe mutation/projection coupling as passenger payment and can invite duplicate invoices after a partial success.

## Findings and repair exits

1. **P1 — current invoice payloads are normalized incorrectly (retained).** `src/api/nexusAPI.js:259-273` reads only top-level fields while the supported nested register payload stores invoice data under `json`; `:529-540` maps outstanding invoices through that lossy adapter.
   - **Exit:** one versioned adapter preserves canonical register metadata separately from nested payload, rejects malformed/error envelopes, and proves exact address, owner, recipient, account/token, integer amount, status, items and ride reference against a supported-core fixture.
2. **P1 — invoice issuance can duplicate after partial success (newly documented, pre-existing).** `src/App/Driver.js:334-359` ignores the invoice creation result, then performs a separate taxi update inside the same `try`. If invoice creation succeeds and the taxi update fails, the UI reports “Failed to create invoice”; no invoice address/txid is persisted or read back, so retry can create another invoice. The same view later selects the first description match at `:390-396`.
   - **Exit:** persist a ride/profile/network-scoped issuance intent before mutation; capture and read back the returned address/txid; project taxi state separately; reconcile ambiguous/partial outcomes without reissuing; reject ineligible ride state and duplicate durable ride identity.
3. **P1 — payment success is conflated with ride projection (retained).** `src/App/Passenger.js:272-308` pays and immediately writes `paid`, with no durable intent, returned transaction identity or authoritative read-back. Selection at `:344-350` trusts the first `ride=` description substring without issuer/account/token/recipient/amount authorization.
   - **Exit:** freeze and reread exact terms; submit at most once; retain `submission_unknown` across restart; verify invoice and transaction evidence; then read/write/read the passenger-owned projection without repaying on projection failure.
4. **P1 — production package allowlist omits emitted images (retained).** `webpack.config.babel.js:29-31` emits image resources while `nxs_package.json:10` lists only HTML, logo and `app.js`; the latest actual inventory found seven omitted PNGs.
   - **Exit:** make a clean build assert that every runtime-referenced output is explicitly allowlisted or deliberately inlined, then install the production package in a supported Nexus Interface and verify all map/routing assets.
5. **P2 — complete discovery, validation, privacy and automation gates remain open.** Fixed-limit reads and failure-to-empty fallbacks remain in `src/api/nexusAPI.js:163-210` and `:369-540`; legacy ride creation at `:446-479` publishes precise coordinates and unbounded labels. `package.json` still has no test/lint command and no workflow is checked in.
   - **Exit:** return explicit complete/partial/error pagination state; validate identities, coordinates and UTF-8 bytes before PIN; migrate deliberately to the Standards v0.2.0 request/offer/agreement model with private precise-location handoff; add a clean test/lint/build/package gate.

## Source identity

Commit tree: `715f893517b4aabb62fbeda141709c8db3f57740`.

| Runtime file | SHA-256 |
|---|---|
| `package.json` | `3ed1903e47218a1c38f3851120d9b4b8efd16e819c04d5d15fb292c5a907c408` |
| `package-lock.json` | `db5aa094b6f5641d224944f97170dad4587e7940c83a6fdce2a2b5941827e09f` |
| `nxs_package.json` | `ce239047566e97b05adbb20528bebcc528ccc4a91eac8e002b20944ac814e819` |
| `webpack.config.babel.js` | `53f7f6009763617cdb9acb1a7bb5785c156f5fed0c4fa8e90c73a0abf8426edf` |
| `src/api/nexusAPI.js` | `54cea675cb0de1c8c281e6ad451a86e70b65cf3f6628f0318234ce7b6e6ebd2b` |
| `src/App/Passenger.js` | `b925f41ba3a852fd3816a6fb040fcdfbed2553002191a9302959295c7c97809d` |
| `src/App/Driver.js` | `10ca8502316d2d18418d7c59812df49a69c05a72f27810e12db3f4560aa138b3` |
| `src/configureStore.js` | `e4dd496fc70f14b99af9227ec36acccc0189a8f3b87757f639ae756b0f0357ca` |

## Executed evidence

| Gate | 2026-09-12 result |
|---|---|
| Branch/remote identity and clean start | **PASS** — local and remote `2effd33df1a11dceb9831774a694c95cbce0cb89`; `0 0` ahead/behind; clean worktree |
| Delta since prior publication and runtime continuity | **PASS** — no commit or file delta; runtime hashes match the prior baseline |
| Source reinspection | **PASS (static)** — issuance, normalization, payment, discovery and packaging paths inspected; new issuance finding mapped to exact source |
| Git object and whitespace checks | **PASS** — `git fsck --no-dangling --no-progress` and pre-edit `git diff --check` exited 0 |
| Fresh build/package/link compound gate | **NOT RUN** — the 2026-09-10 request was approval-denied and was not rerouted; latest actual build/package evidence remains historical below |
| Repository test/lint/CI contract | **ABSENT** — no scripts or checked-in workflow |
| Live supported-core / Nexus Interface / payment acceptance | **NOT RUN** by safety scope |

Historical evidence remains: the 2026-09-08 production Webpack build passed (872 KiB app, seven emitted PNGs, three performance warnings), while package inventory failed because all seven PNGs were absent from the manifest. Those are not fresh 2026-09-12 results.

## Repair handoff

Implement in this order: (1) invoice adapter plus deterministic tests and CI, (2) durable issue/read/project and pay/read/project state machines with fault injection, (3) package inventory and production-wallet installation, then (4) paginated discovery and privacy/schema migration coordinated with `Distordia_Standards`. Do not enable real payment until every P1 exit and isolated two-profile acceptance pass.