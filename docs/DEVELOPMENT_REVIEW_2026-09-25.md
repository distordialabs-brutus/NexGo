# Development and architecture review — 2026-09-25

## Baseline and verdict

**Reviewed repository:** `/home/brutus/github/NexGo`, branch `master`, target/HEAD `7da750f1f674b9af839c650f6a47bc4a2af15729`, matching `origin/master` at review start. The only working-tree path at baseline was untracked `vision.md`; it was read as design context, hash-checked, and not staged or modified.

Relative to `78435a5d4f97deb27770051a8aacbbc3360b9355`, target commit `7da750f…` changed only `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT_PLAN.md`, and added `docs/DEVELOPMENT_REVIEW_2026-09-23.md` (297 insertions, 42 deletions). There was no application development: `src/` remains tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`.

**Verdict: prototype; not payment-release-ready, package-ready, privacy-ready, or production-ready.** A clean offline install/build succeeds reproducibly, but current-core invoice creation uses the wrong request key, nested invoice terms are lost, invoice owner is incorrectly available to be treated as issuer, ride/taxi authority fails open, issue/pay effects have no durable recovery, JavaScript floats are used for money, seven runtime PNGs remain outside the wallet serving manifest, and no test/lint/CI gate exists. No live Nexus call, wallet mutation, profile operation, transaction, secret access, or dependency change was performed.

## Source contract reviewed

Read-only GitHub queries on 2026-09-25 confirmed these branch heads had not advanced from the 2026-09-23 review:

| Source | Branch/head | Relevant blobs |
|---|---|---|
| NexGo | `distordialabs-brutus/NexGo`, `master` `7da750f1f674b9af839c650f6a47bc4a2af15729` locally/remotely | `src/` tree `b0a45a0…`; API `3ef4357…` |
| LLL-TAO stable | `master` `1185145534a20ed4d2288e4513c505f271be536d` (2026-04-10, release 5.1.6) | create `ebd87badd6075db45719d30f740c7cf9b55f94bc`; JSON `3a24e1fa9b2a2542bc46b35ee9f381ee7cb29b10`; pay `fd253638fadb2336430615ae57d2cd3f489a294c`; extract `aeca4c437abc68116175f51ceee2278ab2ed48f9`; update `14d796e0171cf860ceacdbbd797be928953c7f28` |
| LLL-TAO development | `merging` `8af9c3387244b4d396c0e00ee81cea78bb9c0177` (2026-09-03) | Reviewed invoice create/JSON behavior remains the same as stable |
| NexusInterface | `master` `1e923d46a9cde1cf22b8608257c92da184aece16` (2026-04-22) | module WebView `e905ee956df9f51fdd5189f6066d8dba0a2ecd08` |

Authoritative source confirms:

- `Invoices::Create` invokes `ExtractAddress(jParams, "to")`; extractor forms are `to`, `name_to`, and `address_to`. The vendored document's creation `account` parameter is stale.
- `InvoiceToJSON` puts invoice terms and status in `json`, preserving register metadata outside.
- Invoice command registration defines outstanding as system-owned and paid as owned by `json.recipient`. Current top-level owner is lifecycle state, not stable issuer identity.
- `Invoices::Pay` checks status/token, then builds DEBIT and CLAIM together.
- Create serializes item/total values as doubles and pay reconstructs units from `double * figures`; exact application money claims require bounded target-core acceptance, not only JavaScript fixtures.
- Generic raw update replaces payload bytes and exposes no public compare-and-set argument.
- Reviewed wallet `secureApiCall` displays arbitrary endpoint/params, asks for PIN, then calls it; it has no endpoint allowlist.

These are source semantics, not live-node acceptance.

## Executed offline verification

| Command / probe | Result |
|---|---|
| `git rev-parse --show-toplevel && git rev-parse HEAD && git status --short --branch` plus target metadata/index checks | **PASS** — expected path/target; `master...origin/master`; only `?? vision.md`; index tree `3176b7a4d521255b42424345ec8e9727d624a4db` |
| `git show --stat 7da750f1f674b9af839c650f6a47bc4a2af15729` and `git diff --name-status 78435a5d4f97deb27770051a8aacbbc3360b9355..7da750f1f674b9af839c650f6a47bc4a2af15729` | **PASS** — exactly three documentation paths; no whitespace error |
| `git fsck --no-dangling --no-progress && git diff --check && git status --porcelain=v1` | **PASS WITH EXPECTED UNTRACKED CONTEXT** — only `?? vision.md` |
| `gh repo view distordialabs-brutus/NexGo ...` plus explicit `gh api` queries for LLL-TAO/NexusInterface | **PASS** — explicit origin identified; source heads/blobs above retrieved |
| `git clone --no-hardlinks --no-checkout .../NexGo .../nexgo-review-2026-09-25` then detached checkout `7da750f…` | **PASS** — isolated exact-target fixture |
| `npm ci --offline --ignore-scripts && npm run build` in isolated fixture | **PASS WITH WARNINGS** — 2,112 packages installed from cache, 0 vulnerabilities; Webpack 5.105.1; 872 KiB `app.js`; seven PNGs; three performance warnings; build 3.144 s |
| `npm ls --depth=0` in reviewed worktree | **PASS WITH HYGIENE WARNING** — direct dependencies resolve; `bindings@1.5.0`, `file-uri-to-path@1.0.0`, `nan@2.25.0` extraneous |
| `npm test` | **FAIL / ABSENT** — missing script `test` |
| `npm run lint` | **FAIL / ABSENT** — missing script `lint` |
| `node offline-contract-probe.cjs` in isolated fixture, with wallet calls stubbed and `fetch` denied | **DIAGNOSTIC PASS / CONTRACT FAIL** — defects reproduced; no external call |
| `python3 package-inventory-probe.py` in isolated fixture | **FAIL AS EXPECTED (exit 1)** — seven referenced PNGs absent from manifest |
| SHA-256 inventories over exact-target source/config and complete post-build `dist/` | **PASS** — inventories below |
| `python3 markdown-link-check.py` over the four edited documents | **PASS** — all relative links resolve |

Reproduction commands used for the isolated exact-target build and probes:

```bash
git clone --no-hardlinks --no-checkout /home/brutus/github/NexGo /home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-25
git -C /home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-25 checkout --detach 7da750f1f674b9af839c650f6a47bc4a2af15729
npm ci --offline --ignore-scripts
npm run build
node offline-contract-probe.cjs
python3 package-inventory-probe.py
python3 markdown-link-check.py
```

The last five commands ran with `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/nexgo-review-2026-09-25` as the working directory. Probe scripts were review-only scratch artifacts, not repository tests.

The contract probe returned:

```json
{
  "network": "denied by stubs",
  "invoice_normalization": {
    "amount": 0,
    "account": "",
    "recipient": "",
    "item_count": 0,
    "token_preserved": false
  },
  "ride_owner_without_canonical_owner": "PAYLOAD_CLAIM",
  "ride_version_preserved": false,
  "invoice_create_keys": ["name", "account", "recipient", "items"],
  "stale_update_reads": 0,
  "stale_update_secure_calls": 1,
  "oversized_ride_utf8_bytes": 2749,
  "oversized_ride_secure_calls": 1
}
```

The package probe found all manifest-listed files present, but all seven emitted/referenced PNGs unlisted:

```text
dist/js/19680205f5c5bd08dcaa.png
dist/js/2b3e1faf89f94a483539.png
dist/js/416d91365b44e4b4f477.png
dist/js/6569279e9204af87ffa9.png
dist/js/680f69f3c2e6b90c1812.png
dist/js/8f2c4d11474275fbc161.png
dist/js/a0c6cc1401c107b501ef.png
```

## Findings

### Critical fund-loss/duplicate-action paths

#### P0 — Invoice issuance cannot target the reviewed current core

`createRideInvoice` sends `account: paymentAccount`; stable source requires destination through `to`/aliases. The call is rejected before invoice creation. UI help also tells the user that `account` is the create input, reinforcing the stale contract.

**Fix:** emit `to`; treat aliases as separately tested compatibility paths; store canonical returned `json.account`; require isolated create/readback with retained address and creation txid.

#### P0 — Invoice reads erase the terms needed to authorize payment

Current source nests terms under `json`; `normalizeInvoice` reads top-level fields and fabricates zero/empty defaults. It drops token entirely. Description correlation consequently cannot safely identify, display, or authorize a ride invoice.

**Fix:** strict versioned decoder; preserve top-level register envelope and nested account/recipient/token/status/items/amount; reject unsupported/malformed data. Description is discovery-only.

#### P0 — Invoice owner cannot stand in for issuer

The prior architecture used “canonical issuer/owner” language. Core registration proves outstanding invoices are system-owned conditional registers and paid invoices are recipient-owned. Current owner changes with lifecycle and does not identify the creator.

**Fix:** persist creation txid/address and verify the creation transaction genesis/contract as issuer evidence. Validate owner only as expected lifecycle state. Test outstanding, paid, and cancelled transitions.

#### P0 — Issue/pay have no durable exactly-once protocol

Driver and passenger views call mutation then projection in component-local `try` blocks. They persist no intent, address, txid, unknown result, or recovery cursor. Projection failure is reported as issue/payment failure, inviting a repeat. Paid invoices leave the outstanding subset used by the UI.

**Fix:** intent-first durable issue/payment states; persist remote identity or unknown outcome before projection; recover by exact address and registered invoice get/history/transactions plus transaction evidence; projection-only retry; mutation counters at most one.

### High money-contract and authority defects

#### P0 — JavaScript float handling is not an exact money contract

Driver code uses `parseFloat`; adapter calls `.toString()`. Core accepts precise strings internally but serializes invoice amounts through doubles and pays from `double * figures`. Offline normalization alone cannot establish exact debit units for all values/tokens.

**Fix:** strict decimal strings and integer base units in application state, token-decimal validation, conservative supported range, item/total equality, and target-core boundary tests that inspect actual DEBIT units. Unsupported precision fails closed.

#### P0 — Canonical ride/taxi ownership and mutation target fail open

A ride without top-level owner inherits payload `passenger-genesis`; register version is lost. Taxi logic can prefer mutable `driver`, rebuild a mutation name from mutable settings, silently select `assets[0]`, and turn list errors into empty results. Payload authors can therefore influence authority/selection.

**Fix:** require exact canonical address/owner and active profile/network match; redundant claims must match; mutate exact address; represent read failure separately from empty; reject ambiguous multiple owned taxis.

#### P0 — Stale whole-payload updates lack prestate checks

`updateRideRequestAsset` merges `currentData`, performs no read, and submits. Core raw update has no public CAS. UI-local pending flags do not protect restart or another client.

**Fix:** exact pre-read plus canonical digest/modified/state comparison; allowed role-specific transitions; durable one-pending operation scope; exact post-read. Do not claim cross-client CAS; keep legacy whole-blob state out of settlement authority.

### Test, packaging, and live-evidence gaps

- No test/lint/CI command exists.
- Clean build succeeds but the production allowlist omits all seven referenced PNGs.
- Fixed list limits and catch-to-empty behavior make discovery incomplete and ambiguous.
- No supported-wallet production folder/zip install has run.
- No live target-core invoice amount, owner lifecycle, issue/pay recovery, wrong-network/sync, or two-profile settlement acceptance has run.

### Privacy and operational hardening

- GPS denial automatically triggers `ipapi.co` without separate consent.
- Nominatim search text, OpenStreetMap viewed areas, and router coordinates leave the WebView directly.
- Exact pickup/destination coordinates and labels are written to permanent chain history.
- Wallet writes are not gated on network, sync, version/mode, or active profile readiness.
- Current legacy records do not prove namespace delegation, autonomous authority, agreement-bound settlement, or agreement-bound reputation described in `vision.md`.

### Positive controls actually verified

- Reviewed mutations use wallet `secureApiCall`; NexGo does not directly persist profile password, PIN, API Basic credentials, or sessions.
- Core pay source makes DEBIT and CLAIM atomic and rejects paid/cancelled/wrong-token attempts.
- Exact-target lockfile installs offline and builds reproducibly.
- Target and origin matched; `vision.md` remained untracked and unchanged.

## Repair order

1. **Batch 0:** payment containment plus one collected offline/CI gate.
2. **Batch 1A:** current-core adapter, issuer lifecycle model, strict envelope, and exact supported money domain.
3. **Batch 1B:** canonical ownership/address and revision-safe local transitions.
4. **Batch 1C:** durable issue/pay identity, unknown-outcome reconciliation, and projection recovery.
5. **Batch 1D:** manifest closure and pinned-wallet folder/zip acceptance.
6. **Batch 2:** consent/privacy, complete discovery, and explicit open-protocol migration.

The executable criteria are maintained in [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md). Real funds and production profiles remain prohibited until a separately approved live acceptance plan executes.

## Complete exact-target source/config SHA-256 inventory

```text
82d8ba9ba9f9c0eb05529e44a372193f82968ca07c921016e49ecddd8993b17b  .prettierrc
0c387c0689281fc84f3ff6e0cb80bd82dd34bc6b8d4834a34313cd0e92d2e506  babel.config.js
25e18d3b7774709d7fa4ce997a3e8ab687ebc9d071049838feb3c72c8fc76d7a  nxs_package.dev.json
ce239047566e97b05adbb20528bebcc528ccc4a91eac8e002b20944ac814e819  nxs_package.json
db5aa094b6f5641d224944f97170dad4587e7940c83a6fdce2a2b5941827e09f  package-lock.json
3ed1903e47218a1c38f3851120d9b4b8efd16e819c04d5d15fb292c5a907c408  package.json
10ca8502316d2d18418d7c59812df49a69c05a72f27810e12db3f4560aa138b3  src/App/Driver.js
01c1d83c342631fe6b3588b254af9b7306511dc2b951730aa0073f846af25df9  src/App/Main.js
b925f41ba3a852fd3816a6fb040fcdfbed2553002191a9302959295c7c97809d  src/App/Passenger.js
402b1698eb6e413fdb432afa06be24c64e137620cab1f9a02eeb255fb76351a0  src/App/index.js
e1f3cfa2d9f5fd9bb6c8ae6959bded19ee8ce900511f01368dbfae9ec164b1e8  src/actions/actionCreators.js
574293cfe8219524f551923615d79c41d58fa714c33e78e7d4498e6c7941a875  src/actions/types.js
54cea675cb0de1c8c281e6ad451a86e70b65cf3f6628f0318234ce7b6e6ebd2b  src/api/nexusAPI.js
537f35dd76fe52d4a31914e0ee5cbb705da8e1143a67c99067693599031d769b  src/components/Map.css
2dc4bcedff6552fc3820e902617bc835b3a242496ed75dedc1495c42607aee3e  src/components/Map.js
126aea5b92d3ff06e16915d60af7c7e750a496b26bfa75c7044f356ac2878df8  src/components/RoutingMachine.js
e4dd496fc70f14b99af9227ec36acccc0189a8f3b87757f639ae756b0f0357ca  src/configureStore.js
becf637f4d551e5a0e0907f15c8b1f3832fe6965e65e56225574d72d23d18f33  src/index.js
c857d7943712b2468de0e23d81e7d9580a51f73ed64f089f64939ab9e11f8d90  src/reducers/index.js
41910c8b7388c1cb41ffebc29f00b2e8d091611b42f97174e44295d6683265c9  src/reducers/settings/driverPaymentAccount.js
e2a12ab356f6386563af0ab8508e7a1b8a8c577209e77288de7efe7440fbdec8  src/reducers/settings/index.js
5525332881581cbca812665d5b148f8603fa245cdf6e2bcba5aff61e2bf06201  src/reducers/settings/passengerPaymentAccount.js
cc67229784644805fedb337c9cbd92ddb0a42091f4b09d55ef0e8b72d9de013e  src/reducers/settings/showingConnections.js
e43f84bffd0a8aef210b19820b710538062a8a6d2810fd6f1356c43d404bbfe0  src/reducers/settings/vehicleId.js
1c81f812627a776fdae0e1721285a3124bf6b0e5bb1340edda53a0359d2b2661  src/reducers/settings/vehicleType.js
16943ec56631656e7a3ba1d7738fd5c9b565b795bee284f472411b673da09767  src/reducers/taxi.js
2f5e86d152f40c801c94df4101b9061ed4f6cb8e2459b3460f4c3ca9d0da10a2  src/reducers/ui/activeTab.js
0101afff1415b01e6b6aae0ae4e228716beae65f7db0162572a2a80c94e2a22e  src/reducers/ui/broadcasting.js
8a5d0c912c7b9e733d8247f31b3ce6f800440e98d72be2b1e6679af50839d2d2  src/reducers/ui/driverStatus.js
2f265ea7854e7099dbebecd6cf3592bb81c167054871d1e65646fb14cb51b134  src/reducers/ui/index.js
a3324e5b61ec8baac70be07ae1bfa225e8553874cc762f97ebe832918aa24ed6  src/reducers/ui/inputValue.js
f5c386a69969a57113877988b1b1301f045d56fac9e94cef5259cb4084edf3e2  src/reducers/ui/userPosition.js
c5416bea7f01d472ab71aff99ab97a3086fba136a5eeb90df71afa64ca66ec26  tsconfig.json
6ede2d86006d03a39b3529c7214878d550643b56ca241cfc4f3fae3ed10b6339  webpack-dev.config.babel.js
53f7f6009763617cdb9acb1a7bb5785c156f5fed0c4fa8e90c73a0abf8426edf  webpack.config.babel.js
```

Design context, outside the commit:

```text
5829f1f5e8eb5a6a0fb748c4f031a29da2296b24a527c309fd741ddf411c6bfb  vision.md
```

## Complete clean-build `dist/` SHA-256 inventory

```text
40c176cebac0c4da4b7769206b7266ae01af5644b22b66151ad6615721fbf82d  dist/dev.html
a586c6675f67e92f5b5396166d4be75fc4c0190539af336b8bf40500e47ce563  dist/distordia-logo.svg
75e1da9e1ca967dfbe46f1c2f4b5e34ebecebcb7ea0261328d11c1ca03a4a89f  dist/nexgo-logo.svg
e1a84a8a95c82bed3fd70239ccb1d1141a65ee71995d7e1ed5fa1380a75f6d56  dist/react.svg
fb2373c808aabcf7360e673798ea480d3114433e33e1a4065b2b406e4743b50b  dist/index.html
772d34938511bd98498938137dbc5d016ac48a408a259e25449cfb4bb98ea79d  dist/js/19680205f5c5bd08dcaa.png
574c3a5cca85f4114085b6841596d62f00d7c892c7b03f28cbfa301deb1dc437  dist/js/2b3e1faf89f94a483539.png
1dbbe9d028e292f36fcba8f8b3a28d5e8932754fc2215b9ac69e4cdecf5107c6  dist/js/416d91365b44e4b4f477.png
3cf31e141ce5b8dc4b5f94a25a9bc1673f45dde5d80b534d9c45ed746d531f0a  dist/js/6569279e9204af87ffa9.png
00179c4c1ee830d3a108412ae0d294f55776cfeb085c60129a39aa6fc4ae2528  dist/js/680f69f3c2e6b90c1812.png
066daca850d8ffbef007af00b06eac0015728dee279c51f3cb6c716df7c42edf  dist/js/8f2c4d11474275fbc161.png
264f5c640339f042dd729062cfc04c17f8ea0f29882b538e3848ed8f10edb4da  dist/js/a0c6cc1401c107b501ef.png
dc8da220d937ba3c38a6349dba1f5ea11a30b0086bac82043f36ed4ec830043d  dist/js/app.js
681349b67afce65b47cfbd602fa0f1a00bd6a8d2db56af4b44b38590f05c562f  dist/js/app.js.LICENSE.txt
09832f24e214645a66bf555471e26046e17b680b73ebd14243ed3ec34a3f59a4  dist/js/app.js.map
```

## Scope and encountered issues

Intentionally edited only `docs/ARCHITECTURE.md`, `docs/EVALUATION.md`, `docs/DEVELOPMENT_PLAN.md`, and this review. No runtime, manifest, dependency, lockfile, generated distribution, or Git index change was intended. Scratch probes live outside the repository.

One initial composite shell probe containing a heredoc was denied by unattended approval policy and was not used as evidence. A first clean-clone invocation also did not execute because its requested working directory did not yet exist; cloning from the scratch parent and then running the exact commands succeeded. No denied operation was bypassed. No commit or push was made.
