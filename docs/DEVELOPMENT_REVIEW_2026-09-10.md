# Development and architecture review — 2026-09-10

## Scope and continuity

**Reviewed pre-publication HEAD:** `0109a57dec8a2f69e6f179f9c0d20084910bee67` (`master`), equal to `origin/master` at review start.

**Prior dated review's source HEAD:** `d2f9ac053eba610e15c113db1584896c9701cf76`. **Runtime baseline:** `7a9c4816852d3e0a7b500f64bfac035715502f86`.

`git diff --name-status d2f9ac0..0109a57` contains only the 2026-09-09 architecture, plan and review documents. A path-limited diff over `src/`, package manifests and Webpack configuration is empty. The worktree was clean. There is no new source, test, dependency, package-manifest, build-configuration or workflow change to classify. Findings retained below are unchanged defects, not regressions introduced after the 2026-09-09 review.

No wallet/core call, profile/session action, financial mutation, dependency change or runtime repair was performed.

## Verdict

**Prototype; not payment-release-ready.** PIN-confirmed wallet writes remain a useful boundary, but the module still lacks canonical invoice decoding and authorization, payment intent/read-back/recovery, complete discovery, privacy-safe request handling, a correct package allowlist and an automated test/CI contract.

## Retained findings and repair exits

1. **P1 — current invoice payloads are normalized incorrectly.** `src/api/nexusAPI.js:259-273` reads only top-level fields, while outstanding invoices are mapped through this function at `:529-540`. The supported nested `{address, owner, json: {...}}` payload loses recipient, account, amount, status, items and ride reference by falling back to empty/zero values.
   - **Exit:** one versioned adapter preserves register metadata separately from nested payload, rejects malformed/error envelopes, and a supported-core fixture proves exact address, owner, recipient, account/token, integer amount, status, items and ride reference.
2. **P1 — payment success is still conflated with ride projection.** `src/App/Passenger.js:272-308` pays and immediately writes `paid`, persists no pre-submit intent or returned transaction identity, and performs no authoritative invoice/transaction read-back. Selection at `:344-350` trusts the first description containing a matching `ride=` substring without binding issuer/account/token/recipient/amount.
   - **Exit:** persist network/profile/ride/invoice-scoped intent; freeze and reread exact terms; submit at most once; hold ambiguous responses as `submission_unknown`; verify invoice and transaction evidence; then perform and read back the separate passenger-owned projection without repaying on projection failure.
3. **P1 — production package allowlist still omits emitted map images.** `webpack.config.babel.js:29-31` emits image resources, while `nxs_package.json:10` still lists only HTML, logo and `app.js`. The seven-PNG failure remains the latest actual inventory evidence.
   - **Exit:** a clean production build asserts that every runtime-referenced emitted asset is allowlisted or deliberately inlined, followed by installation in a supported Nexus Interface with no denied marker/layer/routing requests.
4. **P2 — discovery, size/privacy and automation gates remain open.** Ride creation still publishes precise coordinates and unbounded labels before the PIN call (`src/api/nexusAPI.js:446-479`); fixed-limit reads and error-to-empty fallbacks remain (`:369-418`, `:482-540`). `package.json` still defines only `build` and `dev`, with no checked-in CI workflow.
   - **Exit:** validate coordinates, identity and UTF-8 bytes before PIN; return explicit complete/partial/error pagination state; migrate deliberately to the Standards v0.2.0 request/offer/agreement model with private precise-location handoff; add a clean test/lint/build/package gate.

## Source identity

Commit tree: `fa716e9d25c97fa9cdc1f1e397a87867e0015e12`.

| Runtime file | SHA-256 |
|---|---|
| `package.json` | `3ed1903e47218a1c38f3851120d9b4b8efd16e819c04d5d15fb292c5a907c408` |
| `package-lock.json` | `db5aa094b6f5641d224944f97170dad4587e7940c83a6fdce2a2b5941827e09f` |
| `nxs_package.json` | `ce239047566e97b05adbb20528bebcc528ccc4a91eac8e002b20944ac814e819` |
| `webpack.config.babel.js` | `53f7f6009763617cdb9acb1a7bb5785c156f5fed0c4fa8e90c73a0abf8426edf` |
| `src/api/nexusAPI.js` | `54cea675cb0de1c8c281e6ad451a86e70b65cf3f6628f0318234ce7b6e6ebd2b` |
| `src/App/Passenger.js` | `b925f41ba3a852fd3816a6fb040fcdfbed2553002191a9302959295c7c97809d` |

These hashes match the 2026-09-09 review and substantiate runtime continuity.

## Verification evidence

| Gate | 2026-09-10 result |
|---|---|
| Branch/remote identity and clean start | **PASS** — local and remote `0109a57dec8a2f69e6f179f9c0d20084910bee67`; clean worktree |
| Prior-review delta and runtime path diff | **PASS** — three documentation paths only; runtime/configuration diff empty |
| Source reinspection and SHA-256 identity | **PASS** — invoice, payment, discovery and packaging paths/hashes unchanged |
| Fresh build, package inventory, Markdown links and whitespace compound gate | **BLOCKED** before execution by explicit approval denial; not retried or reformulated |
| Repository test/lint/CI contract | **ABSENT** in unchanged `package.json` / repository |
| Live supported-core / Nexus Interface / payment acceptance | **NOT RUN** by safety scope |

The latest actual build evidence remains the 2026-09-08 Webpack pass (872 KiB app, seven emitted PNGs, three performance warnings); the latest actual package inventory remains a failure because all seven emitted PNGs are absent from the manifest. No fresh 2026-09-10 result is claimed for those gates.

## Repair handoff

Owner should implement in this order: (1) nested invoice adapter plus deterministic tests and CI, (2) durable verified invoice-to-ride settlement with fault injection, (3) package inventory and production-wallet installation, then (4) paginated discovery and privacy/schema migration coordinated with `Distordia_Standards`. Do not enable real payment until all P1 exits and isolated two-profile acceptance pass.
