# Development and architecture review — 2026-09-15

## Baseline and verdict

**Reviewed HEAD:** `9c5369ff1ed4102b5b3cf94f75786ca6eb58d676` (`master`), equal to freshly fetched `origin/master` (`0 0` ahead/behind). The only commit after prior reviewed baseline `2effd33df1a11dceb9831774a694c95cbce0cb89` is the 2026-09-12 review publication. Runtime paths are unchanged from `7a9c4816852d3e0a7b500f64bfac035715502f86`.

**Verdict: prototype; not payment-release-ready.** Fresh build evidence is green, but every release blocker remains. No wallet/core, profile/session, invoice/payment, install or production call was made.

## Fresh evidence

| Gate | Result |
|---|---|
| Fetch/identity | **PASS** — local and `origin/master` both `9c5369ff…`; clean start |
| Git integrity/whitespace | **PASS** — `git fsck --no-dangling --no-progress`; pre/post-build `git diff --check` |
| Production build | **PASS with warnings** — Webpack 5.105.1; 872 KiB `app.js`; 7 cached PNG assets; 3 performance warnings; worktree remained unchanged |
| Runtime package inventory | **FAIL** — all 7 emitted PNGs referenced by `app.js` are absent from `nxs_package.json.files` |
| Local Markdown links | **PASS** — 8 Markdown files checked; 0 missing local targets |
| Tests/lint/CI | **ABSENT** — `package.json` exposes only `build` and `dev`; no checked-in workflow found |
| Supported-core, wallet install and payment acceptance | **NOT RUN** by safety scope |

Source commit tree is `7f44465096360e85b905b99bb98d098ac9d630df`; runtime baseline tree is `c2964d495c2e3dd976de1e9ad3d9970bae3ce85f`. Runtime hashes remain identical to the 2026-09-12 evidence (`nexusAPI.js` `54cea6…`, `Passenger.js` `b925f4…`, `Driver.js` `10ca85…`, package manifest `ce2390…`).

## Unchanged blockers and exact exits

1. **P1 — invoice normalization is lossy.** `normalizeInvoice` reads top-level fields and silently defaults missing terms instead of decoding supported nested `json` payloads.
   - **Next repair:** create one versioned adapter that separates canonical register metadata from payload, rejects API errors/malformed data and preserves exact recipient, provider/owner, account/token, integer amount, status, items and ride reference.
   - **Exit tests:** nested supported-core fixture round-trips every term; malformed envelopes, absent identity, precision loss and conflicting compatibility shapes reject; isolated supported-core read uses the same adapter.
2. **P1 — invoice issuance and payment can duplicate after partial/ambiguous success.** Driver and passenger views discard mutation identity, couple projection updates to settlement, and select the first description match without full authorization.
   - **Exit tests:** issue-success/taxi-failure, accepted-timeout, pay-success/ride-failure, restart, repeated click, copied description, wrong issuer/account/token/recipient/amount and duplicate invoices all preserve one-mutation maximum and hold unknown evidence for reconciliation.
3. **P1 — package allowlist omits runtime assets.** Fresh build reproduced seven referenced PNGs not listed by the manifest.
   - **Exit tests:** clean build inventory has zero unlisted runtime references; supported production-folder/zip installation renders map and routing assets with zero denied requests.
4. **P2 — discovery, validation and privacy migration remain incomplete.** Fixed limits and failure-to-empty behavior persist; legacy raw rides publish precise coordinates/labels and are not Standards v0.2.0 handshakes.
   - **Exit tests:** multi-page and later-page failure fixtures expose complete/partial/error state; oversized/multibyte and invalid coordinate/identity inputs reject before PIN; unknown versions/owners/references fail closed.

## Handoff

Implement the adapter and deterministic gate first, then durable issue/pay reconciliation, package closure, and finally paginated privacy-preserving schema migration. Do not enable real payment before all P1 exits and isolated two-profile acceptance pass.