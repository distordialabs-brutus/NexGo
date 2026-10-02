# Development and architecture review — 2026-10-02

## Reviewed identities and publication scope

- NexGo baseline and reviewed detached HEAD: `b775fd572b64b5b8ee246be87f899a72613d7d9e` (local `refs/remotes/origin/master` at publication preparation).
- Runtime identity: `src/` tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`; no application commits follow the supplied baseline.
- Pinned LLL-TAO source: `1185145534a20ed4d2288e4513c505f271be536d`.
- Pinned NexusInterface source: `1e923d46a9cde1cf22b8608257c92da184aece16`.
- Published review set: this report plus `ARCHITECTURE.md`, `EVALUATION.md`, `DEVELOPMENT_PLAN.md`, and `NEXT_CODING_CONTRACTS.md`.

The pre-existing untracked `vision.md` was reviewed as design context at SHA-256 `5829f1f5e8eb5a6a0fb748c4f031a29da2296b24a527c309fd741ddf411c6bfb`; it is not implementation evidence and is not included. Pre-existing dirty `README.md` and `docs/state-machines.md`, the untracked 2026-09-30 report/evidence directory, and all runtime/package files are also excluded from this documentation publication.

## Executed evidence

| Check | Result |
|---|---|
| Pinned invoice/cancel/build/session/wallet/storage source assertions | PASS on 2026-10-02. |
| Offline current-runtime adapter probes | Diagnostic PASS / product FAIL: wrong invoice destination key, nested-term loss, missing authority, unsafe floating-point money, false-success issue/pay/cancel handlers, and absent readiness/durability gates were reproduced. |
| Offline package inventory against the preserved 2026-09-30 build output, rerun on 2026-10-02 | FAIL as expected: seven bundle-referenced PNG files are absent from `nxs_package.json.files`; the referenced files exist in that output. |
| Offline package inventory in the detached publication worktree without rebuilding | FAIL: tracked `dist/js/app.js` is absent. This is not a clean-build result. |
| Runtime SHA-256 verification | PASS against the historical 2026-09-30 evidence. |
| Live node, wallet, profile, invoice, payment, cancellation, or production installation | NOT RUN. |

A diagnostic probe is green when it reproduces a known defect; it is not a release test and closes no blocker.

## Build provenance and limits

Historical 2026-09-30 evidence records a clean `npm ci --offline --ignore-scripts` and successful production Webpack build at the same exact NexGo head. On 2026-10-02 a fresh unattended clean-install/build attempt was blocked before execution because package threat-intelligence checks were incomplete. The denied gate was not retried or rerouted, so there is no fresh 2026-10-02 clean-build pass. No exact-head CI workflow exists locally, and no fresh remote CI result is claimed.

## Verdict

The runtime remains a prototype and is not payment-, cancellation-, package-, privacy-, or production-release-ready. The current issue register is in [EVALUATION.md](EVALUATION.md); executable implementation contracts and repair order are in [NEXT_CODING_CONTRACTS.md](NEXT_CODING_CONTRACTS.md) and [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md).
