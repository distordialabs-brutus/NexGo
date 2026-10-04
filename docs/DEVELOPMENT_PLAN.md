# NexGo development plan

## Governing vision and portfolio traceability

Read [the repository vision](../vision.md) and [Distordia alignment/dependency map](DISTORDIA_ALIGNMENT.md) before assigning work. Authority is master Distordia strategy/customer evidence → portfolio roadmap/strategy decisions → repository vision → this development plan → tasks/code/tests/external evidence and human release.

**Portfolio purpose:** O1 open mobility coordination; O2 accountable identity; O4 exact settlement; O5 validated reliance. Role-owned interoperable ride agreements and privacy-preserving settlement/reputation evidence, usable by human and autonomous providers without a privileged NexGo execution path. The alignment map supplies customer-evidence qualification, batch ownership, upstream prerequisites and human gates. Each material task must name those fields alongside its exact production paths and collected acceptance tests. This documentation alignment changes no runtime, test result or release status; dated evidence below remains evidence for its stated snapshot only.

Status re-reviewed 2026-10-02 against supplied baseline and HEAD `b775fd572b64b5b8ee246be87f899a72613d7d9e`; runtime remains `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`. There are zero commits after the baseline. Historical 2026-09-30 evidence records a clean offline install/build at the same exact head; a fresh unattended clean-build attempt was approval-blocked by incomplete package threat-intelligence checks and was not rerouted. Fresh safe probes reproduce the blockers and add four source-backed constraints: cancel needs exact VOID/readback evidence; `-autotx` may omit txid; wallet session/readiness state must gate writes; and pinned module storage is not an acknowledged durable journal. Architecture authority: [ARCHITECTURE.md](ARCHITECTURE.md). Implementation contract: [NEXT_CODING_CONTRACTS.md](NEXT_CODING_CONTRACTS.md). Evidence summary: [DEVELOPMENT_REVIEW_2026-10-02.md](DEVELOPMENT_REVIEW_2026-10-02.md).

**No real invoice issue, payment, cancellation, profile write, production installation, or release until Batch 0 and Batches 1A–1D exit. Next implementation slice: Batch 0 followed by Batch 1A.**

## Batch 0 — Containment and reproducible offline gate (P0)

**Targets:** `package.json`, `test/`, `scripts/`, `.github/workflows/ci.yml`; no application mutation behavior yet.

- Keep payment/issue actions visibly disabled behind one release-safety gate until the durable protocol exits. Do not rely on documentation warnings alone.
- Add deterministic `test`, `lint`, `test:package`, and aggregate `verify` scripts. Prefer the existing toolchain or Node's built-in test runner; dependency changes require separate approval.
- Make tests deny all unmocked network and wallet access. Put pinned LLL-TAO/NexusInterface fixture provenance and source blob IDs next to fixtures.
- Add the currently executed contract-defect, host-result-ambiguity, cancel/session, `-autotx`, and package-inventory probes as collected regressions. A diagnostic that merely reproduces a defect is not a green product test and must fail until the repair lands.
- Run build, tests, static/lint checks, manifest inventory, Markdown links, and Git whitespace/conflict checks in CI.

**Executable exit:** from a clean clone, one offline command installs from the unchanged lockfile and runs every gate. The initial gate is allowed to be red only on explicitly enumerated Batch 1 defects; no test is hidden outside default collection. Exact-head CI uses the same command. An unattended approval block is reported as blocked, not converted to a pass from older output.

## Batch 1A — Pinned current-core adapter and exact money contract (P0)

**Targets:** `src/api/contracts.js`, `src/api/nexusAPI.js`, `test/contracts.test.js`, `test/api-envelopes.test.js`, pinned source fixtures.

- Pin LLL-TAO stable `1185145534a20ed4d2288e4513c505f271be536d` and NexusInterface `1e923d46a9cde1cf22b8608257c92da184aece16` as the initial supported contract unless maintainers explicitly select another version.
- Consume wallet-provided `state.nexus.coreInfo/userStatus`; keep session and PIN entirely inside NexusInterface. Gate every mutation on initialized wallet, stable genesis/network, indexing complete, synchronization, mode, and pinned core/wallet compatibility. The reviewed secure path supplies PIN per call, so do not require cached `unlocked.transactions === true`.
- Require direct transaction mode with core `-autotx` disabled for the first release. A queue-only `success`/`address` response without txid is unresolved, not accepted.
- Emit invoice destination as `to`. If aliases are supported, test `name_to` and `address_to` independently; never send the stale vendored-doc `account` input. Read the resulting destination from `json.account`.
- Split canonical envelope from typed payload. Require invoice address, lifecycle owner, created/modified metadata, nested account/recipient/token/status, non-empty items, and supported schema. No blank/zero/default success values.
- Model invoice current owner as lifecycle evidence, not issuer. Bind issuer to retained creation tx/genesis evidence. Verify expected system-owned outstanding and recipient-owned paid states from the pinned core source.
- Accept decimal strings at the UI/adapter boundary and convert with a strict decimal-to-integer function parameterized by token decimals. Reject exponent notation, negatives, excess scale, overflow, unsafe integer conversion, non-finite values, and item/total disagreement.
- Because reviewed core invoice JSON and pay code use doubles, define a conservative supported amount range and require isolated target-core tests that compare requested base units with the actual DEBIT contract. Do not claim arbitrary-token exactness from offline JavaScript fixtures.
- Require ride canonical `address` and `owner`, explicit legacy version/state, finite/ranged coordinates, and matching `passenger-genesis` if present. Preserve register metadata; reject unknown shapes.
- Handle wallet/API response envelopes with the closed discriminated result in [NEXT_CODING_CONTRACTS.md](NEXT_CODING_CONTRACTS.md). Require matching invoice `address` plus exactly one `txid` for issue, pay, and cancel. An error member, `undefined`, malformed success, address mismatch, txid array, or absent identity is non-success and cannot authorize projection or success UI. On the pinned host, `undefined` is `submission_unknown`, not authorization cancellation and not retry authority.

**Executable exit:** default offline tests prove writer uses `to` and never `account`; current nested invoice terms round-trip; current owner is not accepted as immutable issuer; missing/conflicting ride ownership rejects; malformed precision/status/token/version rejects; every readiness failure rejects before a secure call; and rejected, explicit-host-cancelled, undefined, error, address mismatch, queue-only, or malformed results produce no projection or success state. Accepted-but-empty and current-host PIN cancellation both become `submission_unknown`; only a distinct trusted pre-submit host result may establish non-submission. Target-core acceptance proves direct mode and exact supported base units.

## Batch 1B — Canonical authority and revision-safe transitions (P0)

**Targets:** `src/services/rideTransitions.js`, `src/store/rideIntents.js`, taxi/ride adapters, collected transition tests.

- Replace all stale `currentData` merges with exact-address reread → strict decode → authority/state/digest comparison → transition construction → submission → readback.
- Mutate taxis by canonical address. Require active-profile ownership and matching network; reject mutable `driver` conflicts. Never choose `assets[0]`, rebuild a target from current settings, or collapse a read error into “no asset.”
- Encode allowed legacy transitions separately for passenger-owned ride, driver-owned taxi, and invoice lifecycle. One role cannot rewrite another role's intent.
- Scope one-pending-operation locks and intents to wallet instance, profile genesis, network, register address, and operation. Storage contains public identities/digests/state only—never PIN, password, Basic auth, or session ID.
- Validate UTF-8 bytes, identifier/label bounds, coordinate ranges, and complete canonical schema before opening the wallet PIN dialog.
- State the concurrency limit honestly: pre-read plus local serialization detects many stale writes but is not cross-client CAS. Do not let legacy whole-blob writes authorize settlement.

**Executable exit:** tests cover missing/conflicting owner, wrong active profile/network, stale modified/digest, concurrent changed prestate, duplicate click, invalid transition, oversized/multibyte payload, API ambiguity, restart, and exact readback. Every rejection has zero secure calls; each committed intent has at most one consequential call.

## Batch 1C — Durable invoice issue/payment/cancel and projection recovery (P0)

**Targets:** `src/services/rideSettlement.js`, durable wallet-owned intent storage, `Driver.js`, `Passenger.js`, `test/ride-settlement.test.js`.

- First provide a pinned host storage interface with acknowledged commit and revision/CAS semantics across independent WebViews and restarts. The current fire-and-forget `updateStorage` path is not acceptable for a settlement journal. Keep real-value controls disabled until this prerequisite exits.
- Persist issuance intent before PIN with ride/taxi addresses, passenger, provider/issuer, destination account, token, items, decimal strings, integer base units, network, and prestate digest. A trusted explicit host pre-submit authorization cancellation may record a non-submitted disposition. The pinned host's bare `undefined` records `submission_unknown`; neither path triggers taxi projection or success UI.
- Immediately reread ride and taxi before issue. Persist returned invoice address and creation txid before taxi occupancy projection. Read back the exact invoice and creation transaction; verify terms and issuer evidence.
- If the issue response is missing/malformed or times out, retain `submission_unknown`. Reconcile against exact deterministic identity/evidence; never recreate from an empty or bounded list result.
- Persist payment intent before PIN. Immediately reread the exact invoice; verify retained issuance tx/issuer, lifecycle owner/status, recipient, destination account, token, items, and supported base-unit total. Description `ride=...` remains discovery-only. Every non-success outcome performs no ride projection or success UI; current-host `undefined` remains unresolved and blocks resubmission.
- Persist returned payment txid or unknown outcome before passenger ride projection. Verify exact invoice transition and actual DEBIT/CLAIM evidence. Taxi/ride projection failures have distinct pending states and cannot trigger repay/reissue.
- Persist cancel intent before PIN with exact invoice address, retained creation transfer/issuer, frozen outstanding envelope, network/profile, and intended projection. Require matching response address plus cancel txid. Verify exact VOID/history and `CANCELLED` state before success or projection; outstanding-list disappearance is never evidence. Unknown outcomes block recancel.
- Exercise fault injection before intent, after intent, after remote acceptance/before response, after response/before identity persistence, after persistence/before projection, during restart, and on duplicate invocation.

**Executable exit:** collected tests cover wrong issuer evidence/account/token/recipient, copied description, duplicate invoice, fare change, paid/cancelled state, exact cancel VOID evidence, explicit host authorization cancellation, current-host ambiguous PIN cancellation, timeout before/after acceptance, accepted-but-empty and queue-only responses, every restart boundary, projection failure, repeated click, independent-WebView CAS conflict, and profile/network switch. Every non-success envelope produces zero downstream projections/success UI; only explicit host pre-submit evidence is `cancelled_before_submit`; mutation counters remain `issue <= 1`, `pay <= 1`, and `cancel <= 1`; unknown outcomes block mutation. Then two isolated non-production profiles complete issue/pay and issue/cancel workflows with exact transaction/history/readback on the pinned direct-mode wallet/core using synthetic funds.

## Batch 1D — Package and production-wallet closure (P0)

**Targets:** `webpack.config.babel.js`, `nxs_package.json`, package tests, clean folder/zip fixture.

- Inline map/routing images deliberately or list every emitted runtime dependency explicitly. NexusInterface manifests do not support wildcard serving.
- Inventory references from emitted JS/CSS/chunks, not only file existence. Fail if a referenced runtime file is outside `nxs_package.json.files`.
- Build and inspect both production folder and release zip. Keep source maps/license files out of runtime manifest only when the bundle does not request them.
- Install on pinned NexusInterface with production module policy configured for the acceptance fixture; render Leaflet markers, layers, routing icons, main icon, and route UI with zero denied requests.

**Executable exit:** clean install/build has zero referenced/unlisted files; all seven current PNG cases are regressed; folder and zip installations render without missing-resource errors; aggregate verify and exact-head CI pass.

## Batch 2A — Consent, privacy, and write readiness (P1; before public use)

- Remove automatic IP geolocation after GPS denial. Add separate provider-specific opt-in, revocation, and disclosure for IP location, geocoder, tiles, and router.
- URL-encode, debounce, and abort Nominatim searches. Make providers configurable and replace the public OSRM demo boundary for production.
- Keep precise pickup/destination, route, and live position off-chain. Use coarse/expiring discovery plus private authorized handoff consistent with the design context.
- Gate writes on initialized wallet, active profile, intended network, sync completion, compatible core/wallet, and supported mode. Keep mutation endpoint constants closed.

**Executable exit:** browser tests prove no third-party call before consent or after denial/revocation, stale requests cancel, logs/storage contain no sensitive coordinates/secrets, and every unsupported wallet/core condition rejects before PIN.

## Batch 2B — Complete discovery and open protocol migration (P1)

- Implement cancellable pagination with deduplication and explicit complete/partial/error state for taxis, rides, ratings, and invoices. Preserve prior known records when refresh fails.
- Keep legacy v1 adapters explicit. Implement separate role-owned passenger request, provider offer, passenger agreement, and per-agreement rating records only from shared versioned fixtures.
- Bind human/autonomous actions to canonical namespace/operator/delegation evidence rather than a payload string. Use the same public record/API contract for both provider types.
- Bind agreement, fare terms, invoice creation identity, payment evidence, and rating authority. Resolve namespace/genesis mapping, self-address references, cross-client concurrency, and private location handoff before claiming conformance.

**Executable exit:** multi-page and later-page failure tests pass; legacy and new schemas never alias; shared conformance fixtures prove role ownership and invoice/rating binding without precise public trip data; independent clients can reproduce accepted evidence.

## Remaining live and release gates

After all offline batches pass: production wallet installation, isolated current-core API compatibility, wrong-network/sync/client-mode cases, restart/crash recovery, exact transaction/history readback, profile/session cleanup, two-profile synthetic settlement, and exact-head CI. Real funds and production profiles require a separately approved acceptance plan.
