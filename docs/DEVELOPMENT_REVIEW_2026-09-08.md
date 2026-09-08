# Development review — 2026-09-08

## Scope and continuity

Source: `7a9c4816852d3e0a7b500f64bfac035715502f86`, `master`; exact origin branch read-back matched before review. No commits since the previous job's supplied 2026-09-07 06:12:57 timestamp. There was no dated architecture review in this checkout; these are newly recorded baseline findings, not regressions introduced yesterday. Review changes are documentation only.

## Findings

### P1 — Current-core invoice shape is discarded

`src/api/nexusAPI.js:259-273,529-537` reads invoice fields at top level. Current Nexus `Invoices::InvoiceToJSON` preserves payload under `json` and inserts `json.status`. Registration uses that serializer for `invoice` and `outstanding`.

Source reference: Nexusoft/LLL-TAO `merging` resolved to `8af9c3387244b4d396c0e00ee81cea78bb9c0177`; [pinned serializer](https://github.com/Nexusoft/LLL-TAO/blob/8af9c3387244b4d396c0e00ee81cea78bb9c0177/src/TAO/API/commands/invoices/json.cpp), `commands/invoices/initialize.cpp`. This is source compatibility evidence, not a live installed-core observation.

An offline probe imported the unchanged production API module with only `nexus-module` transport replaced. A synthetic core-shaped invoice with amount 10, account, recipient and ride item became amount 0, empty account/recipient/items, and lost the ride reference. Driver/Passenger joins at `Driver.js:390-398` and `Passenger.js:344-350` consequently cannot discover the intended ride invoice with this shape. Do not “fix” creation `items.amount` to an old `unit_amount` example: inspected current core creation code uses `ExtractPrecision` and stores `amount`.

### P1 — Payment and ride projection are incorrectly treated as one success

`Passenger.js:272-308` calls payment then writes `paid` into passenger-owned raw data; it neither records txid nor reads invoice/transaction evidence before the second write. `nexusAPI.js:570-574` returns any resolved bridge value. The real helper accepted `{}` in the probe with exactly one mutation and zero reads. A ride-update failure is reported as “Failed to pay invoice” even if the payment completed. This does not prove core permits duplicate invoice payment; it proves the application lacks a safe reconciliation and truthful reporting boundary.

The invoice join trusts the first description containing `ride=<alphanumeric>` without validating expected issuer/account/token/amount against agreed terms. PIN confirmation is a positive control but does not establish this binding. Freeze and verify the payment contract before payment is enabled.

### P1 — Build output and wallet serving allowlist disagree

`webpack.config.babel.js:29-31` emits image resources. The production build emitted seven PNGs under `dist/js`; `nxs_package.json:10` lists only HTML, logo and `app.js`. Every emitted PNG was absent from the manifest. This is a static package coverage failure, not a claimed live screenshot of broken icons.

### P2 — Input/privacy/discovery boundaries remain prototype-level

`createRideRequestAsset:446-479` publishes precise location and arbitrary labels without a byte budget. The real function accepted a 2,000-character pickup label and submitted a **2,335-byte** raw blob to the stub. No real chain mutation was attempted; this demonstrates missing validation, not an experimentally established universal core size cap. Reads at `166-189,369-418,497-513,529-540` stop at fixed page limits; several catch errors and return empty success-like collections. The code implements legacy raw ride/rating records, not the sibling v0.2.0 handshake/privacy draft.

## Executed verification

- Node `v22.23.2`, npm `10.9.8`.
- Initial `npm run build`: failed (`cross-env` absent). `npm ci --ignore-scripts --no-audit --no-fund`: succeeded, installed 2,112 packages without lockfile changes. No lifecycle scripts or live financial operations run.
- Repeated `npm run build`: passed, Webpack 5.105.1, three bundle-performance warnings; app bundle 872 KiB and seven emitted PNGs. Browserslist data warning also present.
- `npm run test` and `npm run lint`: both failed because scripts are absent. `git ls-files .github` returned no workflow files. Build success is not a behavioral gate.
- `node /tmp/nexgo-review-20260908.mjs`: offline source probes passed assertions reproducing empty-payment acceptance, oversized submission and core-shape data loss. These temporary probes are **not** checked-in regression tests or live wallet acceptance. Reproduction fixtures and expected exit assertions are specified in [development plan](DEVELOPMENT_PLAN.md).
- `python3 /tmp/repo-review-20260908.py`: confirmed all seven emitted PNGs are absent from the manifest.

## Verdict

Useful prototype and correctly separated read/PIN-confirmed bridge calls, but **not sufficient for payment release**. Implement the ordered [development plan](DEVELOPMENT_PLAN.md) and enforce the [architecture](ARCHITECTURE.md). No new feature implementation since the last scheduled window was claimed; no live Nexus or wallet installation acceptance was performed.
