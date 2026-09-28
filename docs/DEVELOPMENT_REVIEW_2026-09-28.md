# Development and architecture review — 2026-09-28

## Baseline and verdict

**Reviewed repository:** `/home/brutus/github/NexGo`, branch `master`, target/HEAD `019e1fd79643e57d075b8efbd16778f0027d9ae2`, matching `origin/master` at review start. The only baseline worktree path was untracked `vision.md`; SHA-256 remained `5829f1f5e8eb5a6a0fb748c4f031a29da2296b24a527c309fd741ddf411c6bfb`. It was read as design context and not modified or staged.

Commit `019e1fd…` is documentation-only relative to `7da750f…`. Application code remains `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`.

**Verdict: prototype; not payment-release-ready, package-ready, privacy-ready, or production-ready.** The isolated exact-head lockfile install and production build pass. Current invoice issue/payment still violate the pinned core contract, settlement has no durable identity/recovery, money is parsed through JavaScript floating point, canonical authority fails open, package files are incomplete, and no repository test/lint/CI gate exists. This review additionally demonstrates a critical false-success boundary: cancelling the NexusInterface PIN prompt resolves `secureApiCall` with `undefined`, while current issue and pay handlers continue into taxi/ride projections.

No live Nexus call, wallet mutation, profile operation, transaction, dependency upgrade, lockfile change, runtime edit, commit, or push was performed.

## Authority and exact external source

Official source heads were read back on 2026-09-28:

| Source | Branch/head | Use |
|---|---|---|
| NexGo | `distordialabs-brutus/NexGo` `master` `019e1fd79643e57d075b8efbd16778f0027d9ae2` | Reviewed target |
| LLL-TAO stable | `Nexusoft/LLL-TAO` `master` `1185145534a20ed4d2288e4513c505f271be536d` | Initial supported invoice/register contract |
| LLL-TAO development | `Nexusoft/LLL-TAO` `merging` `8af9c3387244b4d396c0e00ee81cea78bb9c0177` | Drift check; heads unchanged from prior review |
| NexusInterface | `Nexusoft/NexusInterface` `master` `1e923d46a9cde1cf22b8608257c92da184aece16` | Module bridge, PIN, and package boundary |

The review fetched the pinned official implementations of invoice create/JSON/pay/registration, address extraction, generic update, wallet WebView handling, and module preload. Source truth overrides the stale vendored `Nexus API docs/COMMANDS/INVOICES.MD` examples.

The pinned implementation establishes:

- `Invoices::Create` calls `ExtractAddress(jParams, "to")`; accepted forms are `to`, `name_to`, and `address_to`.
- Invoice create retains canonical destination as nested `json.account` and returns invoice address plus creation txid through the command response.
- `InvoiceToJSON` preserves canonical register metadata at top level and places terms/status under `json`.
- `outstanding` is system-owned; `paid` is owned by `json.recipient`. Current owner is lifecycle state, not immutable issuer.
- Issuer authority therefore requires the exact invoice address plus retained creation transaction/genesis evidence.
- Create serializes item and total values as doubles; pay reconstructs units from `double * token figures`.
- Pay builds DEBIT and CLAIM in one transaction and checks paid/cancelled/token state.
- Generic raw update writes a replacement payload and exposes no compare-and-set parameter.
- NexusInterface has no endpoint allowlist in `secureApiCall`.
- On PIN cancellation, NexusInterface sends `undefined` as a non-error result, and module preload resolves the promise.

These are source semantics, not live-node or production-wallet acceptance.

## Executed offline verification

All build/probe execution occurred in an isolated detached clone at:

```text
/home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-28-head
```

| Command/probe | Result |
|---|---|
| Exact target checkout and baseline status | **PASS** — detached `019e1fd…`; source fixture clean before scratch evidence/probes |
| Official branch/API readback | **PASS** — heads above unchanged; source blobs fetched and hashed |
| `npm ci --offline --ignore-scripts` | **PASS** — 2,112 packages, 0 reported vulnerabilities; deprecation warnings only |
| `npm run build` | **PASS WITH WARNINGS** — Webpack 5.105.1; 872 KiB `app.js`; seven PNG assets; three performance warnings; exit 0 |
| `node offline-contract-probe.cjs` | **DIAGNOSTIC PASS / PRODUCT CONTRACT FAIL** — no live network/wallet calls; reproduced adapter, authority, size, money, failure-collapse, stale-write, and PIN-cancellation defects |
| `python3 source-contract-probe.py` | **PASS** — all pinned source-contract assertions observed |
| `python3 package-inventory-probe.py` | **FAIL AS EXPECTED** — exit 1; seven bundle-referenced PNGs absent from manifest allowlist |
| `npm test` | **FAIL / ABSENT** — missing script |
| `npm run lint` | **FAIL / ABSENT** — missing script |
| `npm ls --depth=0` | **PASS WITH HYGIENE WARNING** — direct dependencies resolve; three extraneous transitive native helpers in installed tree |
| Exact-head `git diff --check` in fixture | **PASS** |

Primary output files:

```text
outputs/npm-run-build.txt
outputs/offline-contract-probe.json
outputs/source-contract-probe.json
outputs/package-inventory-probe.json
outputs/npm-test.txt
outputs/npm-run-lint.txt
outputs/npm-ls-depth-0.txt
outputs/upstream-source-sha256.txt
outputs/reviewed-source-sha256.txt
outputs/dist-sha256.txt
```

### Contract probe result

The executable probe imported the real `src/api/nexusAPI.js` with only wallet/network boundaries stubbed. It demonstrated:

- Current-core nested invoice fixture normalized to blank recipient/account, amount `0`, zero items, and no token.
- Invoice create emitted keys `name`, `account`, `recipient`, `items`; `to` was absent.
- A ride missing canonical owner was accepted and inherited payload `passenger-genesis`; schema version was discarded.
- Stale ride update performed zero reads and one secure write.
- A 5,954-byte UTF-8 ride payload reached `secureApiCall` without local size rejection.
- Malformed taxi coordinates and price became numeric zero; owner became blank.
- Invoice transport failure collapsed to an empty list.
- PIN-cancelled issue and pay helpers both resolved `undefined`; static executable trace confirmed both UI handlers chain projections without result checks.
- Current UI `parseFloat` changed `9007199254740993` to `9007199254740992` and accepted exponent text `1e3` as `1000`.

### Package probe result

All three manifest-listed files exist, but the production bundle references these unlisted files:

```text
dist/js/19680205f5c5bd08dcaa.png
dist/js/2b3e1faf89f94a483539.png
dist/js/416d91365b44e4b4f477.png
dist/js/6569279e9204af87ffa9.png
dist/js/680f69f3c2e6b90c1812.png
dist/js/8f2c4d11474275fbc161.png
dist/js/a0c6cc1401c107b501ef.png
```

Because `nxs_package.json.files` is the NexusInterface serving allowlist, successful Webpack compilation does not prove an installable production package.

## Findings

### Critical fund-loss or false-settlement paths

#### P0 — PIN cancellation can project issue/payment success without a mutation result

The pinned wallet treats PIN cancellation as fulfilled `undefined`. `createRideInvoice` and `payRideInvoice` pass that value through. `Driver.handleCreateInvoice` then marks the taxi occupied; `Passenger.handlePayInvoice` then writes ride status `paid`. Neither handler validates address/txid or exact readback before projection and success UI.

**Required fix:** one strict mutation-result decoder per command. Cancellation, `undefined`, API error, malformed output, or missing remote identity must produce no downstream mutation or success UI. Persist intent before authorization, record a non-submitted cancellation disposition, and collect UI/service tests asserting zero projection calls.

#### P0 — Invoice creation targets the wrong parameter

Current adapter sends `account`; official core extracts `to`. The current issue call cannot satisfy the pinned core contract.

**Required fix:** emit `to` and assert `account` absent from the request. If aliases are supported, test `name_to` and `address_to` independently. Retain returned invoice address and creation txid.

#### P0 — Invoice decode erases payment-authorizing terms

Current normalization reads top-level account/recipient/amount/items/status, while official source nests them under `json`. It fabricates empty/zero/default values and drops token.

**Required fix:** strict versioned decoder preserving canonical envelope plus nested terms. Reject malformed/unsupported state; never create success defaults.

#### P0 — Issue and pay have no durable exactly-once recovery

Component-local `try` blocks issue/pay and then project. No intent, txid/address, unknown-outcome state, restart recovery, or projection-only retry is retained. Paid invoices leave the outstanding subset used by discovery.

**Required fix:** persist frozen intent, submit once, retain identity or `submission_unknown`, verify exact source evidence, and project idempotently. Never infer absence from a bounded/outstanding list and never repeat issue/pay because projection failed.

### High money-contract and authority defects

#### P0 — Invoice owner is lifecycle state, not issuer

Outstanding invoices are system-owned and paid invoices are recipient-owned. Owner cannot prove the provider who issued the invoice.

**Required fix:** bind exact invoice address to creation transaction/genesis evidence, verify provider authority and creation contract, and test outstanding/paid/cancelled ownership transitions.

#### P0 — JavaScript float handling cannot establish exact money

UI uses `parseFloat`; core create/pay also cross double boundaries. Offline JavaScript round trips cannot prove exact target DEBIT units.

**Required fix:** strict decimal-string parser plus retained integer base units, token-decimal validation, no exponent notation, conservative supported range, item/total equality, and isolated target-core DEBIT inspection at below/exact/above boundaries.

#### P0 — Canonical ride/taxi authority fails open

Missing ride owner inherits a payload claim; taxi driver is mutable; taxi mutation rebuilds names from settings; first owned asset is selected; list errors become empty. Malformed data defaults to plausible state.

**Required fix:** require canonical address/owner, matching redundant claims, active profile/network ownership, exact-address mutation, explicit ambiguity/error state, and strict finite/bounded schema validation.

#### P0 — Raw transitions overwrite stale state without prestate protection

Ride updates merge component-held `currentData` and submit without reread. Official core exposes no public CAS argument.

**Required fix:** exact reread, canonical digest/modified/state comparison, role-specific transition table, durable local operation lock, strict response, and exact post-readback. Do not claim cross-client CAS and do not use legacy whole-blob state as settlement authority.

### Test, package, and live-evidence gaps

- No checked-in test, lint, package-inventory, aggregate verify, or CI gate.
- Build passes but package allowlist omits seven requested PNGs.
- Fixed limits and catch-to-empty results make discovery incomplete and ambiguous.
- No pinned-wallet folder/zip production install has run.
- No live target-core exact amount, creation-genesis, owner lifecycle, issue/pay recovery, wrong-network/sync, PIN-cancel UI, or two-profile synthetic settlement acceptance has run.
- The maintained `CLAUDE.md` still advertises stale `account` invoice input, top-level invoice shape, guaranteed payment language, and `npm install`; it must be repaired to follow the current architecture/source hierarchy. This review attempted the authorized documentation repair, but the protected-file write was blocked by the execution policy and was not retried.

### Privacy and operational hardening

- GPS failure or denial triggers undisclosed IP geolocation.
- Geocoder text, map view, and route coordinates leave the WebView directly.
- Exact pickup/destination coordinates and labels are written permanently on-chain.
- Writes lack wallet/profile/network/sync/version/mode readiness gates.
- Legacy records do not prove namespace delegation, autonomous authority, agreement-bound settlement, or agreement-bound reputation from the design context.

### Positive controls verified

- Reviewed mutations use wallet `secureApiCall`; NexGo does not directly collect or persist profile password, PIN, Basic auth, or session IDs.
- Official core pay builds DEBIT and CLAIM atomically and rejects paid/cancelled/wrong-token attempts.
- Exact-head lockfile installs offline and builds reproducibly.
- Target and origin matched at review start; untracked `vision.md` remained unchanged.

These controls do not close any release blocker.

## Repair order

1. **Batch 0:** fail-closed runtime containment for issue/pay/cancel, collected offline gate, package probe, static checks, and CI.
2. **Batch 1A:** pinned current-core adapter, strict response/envelope decoding, creation evidence model, and exact supported money domain.
3. **Batch 1B:** canonical address/owner selection and revision-safe local transitions.
4. **Batch 1C:** durable issue/pay intent, PIN-cancel handling, remote identity, unknown-outcome reconciliation, and projection recovery.
5. **Batch 1D:** manifest closure plus pinned-wallet folder/zip acceptance.
6. **Batch 2:** consent/privacy, complete discovery, and explicit open-protocol migration.

Every batch has executable exit criteria in [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md). No real funds or production profiles are approved before a separately reviewed live acceptance plan executes.

## Documentation scope

This review updates maintained architecture, evaluation, repair plan, state-machine, README, and this dated evidence record. It does not alter runtime, manifest, dependencies, lockfile, generated distribution, Git index, or untracked design context. `CLAUDE.md` remains an explicit unresolved documentation defect because the protected-file write was denied.
