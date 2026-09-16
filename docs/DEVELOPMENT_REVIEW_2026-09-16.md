# Development and architecture review — 2026-09-16

## Baseline and verdict

**Reviewed HEAD:** `0009d6dc76e086fdbb2893eccaf6406d6c090527` (`master`), equal to freshly fetched `origin/master` (`0 0` ahead/behind). The only commit after the 2026-09-15 reviewed HEAD `9c5369ff1ed4102b5b3cf94f75786ca6eb58d676` is that review's documentation publication. No runtime file changed; key runtime hashes remain `nexusAPI.js` `54cea675…`, `Passenger.js` `b925f41b…`, `Driver.js` `10ca8502…`, package manifest `ce239047…` and `package.json` `3ed1903e…`.

**Verdict: prototype; not payment-release-ready.** Fresh build evidence is green, but no release blocker exited. No wallet/core, profile/session, invoice/payment, install or production call was made.

## Fresh evidence

| Gate | Result |
|---|---|
| Fetch/identity | **PASS** — local and `origin/master` both `0009d6dc…`; clean start; `0 0` ahead/behind |
| Git integrity/whitespace | **PASS** — `git fsck --no-dangling --no-progress`; pre/post-build `git diff --check` |
| Production build | **PASS with warnings** — Webpack 5.105.1; 872 KiB `app.js`; 7 cached PNG assets; 3 performance warnings; worktree remained unchanged |
| Runtime package inventory | **FAIL** — 7 referenced emitted PNGs, 0 listed in `nxs_package.json.files`; all 7 are missing from the allowlist |
| Local Markdown links | **PASS** — 9 Markdown files checked with URL-decoded local targets; 0 missing |
| Tests/lint/CI | **ABSENT** — `package.json` still exposes only `build` and `dev`; no checked-in YAML workflow found |
| Supported-core, wallet install and payment acceptance | **NOT RUN** by safety scope |

## Findings and exits

1. **P1 — canonical ride ownership currently fails open.** `normalizeRideAsset` sets `owner` to `asset.owner || parsed['passenger-genesis']`. A passenger-controlled raw claim therefore becomes apparent canonical ownership when register metadata is absent. This compounds the already-known invoice normalization problem.
   - **Next repair:** make versioned invoice and ride adapters the only ingestion path; require canonical register `address` and `owner`, preserve metadata separately, and require any redundant passenger identity to match the canonical owner rather than replace it.
   - **Exit tests:** missing owner/address, forged passenger claim, malformed raw data, unknown version and conflicting compatibility shapes reject; real supported-core fixtures retain exact canonical metadata.
2. **P1 — invoice issuance/payment remain duplicate-prone after ambiguous or partial success.** View-local state still discards mutation identity, combines settlement with projection updates and authorizes a first description match rather than complete immutable terms.
   - **Exit tests:** issue-success/taxi-failure, accepted timeout, pay-success/ride-failure, restart, repeated click, copied description, wrong owner/issuer/account/token/recipient/amount and duplicates hold unknown evidence for reconciliation and permit at most one mutation.
3. **P1 — wallet package remains incomplete.** The clean build references seven hashed Leaflet/routing PNGs that are not in the manifest allowlist.
   - **Exit tests:** clean inventory has zero unlisted runtime references; production folder/zip installation renders all map/routing assets with zero denied requests.
4. **P2 — discovery, input/privacy boundaries and migration remain incomplete.** Reads are fixed-limit and collapse failures into empty data; precise location is still written to immutable history; no executable v0.2.0 adapter exists.
   - **Exit tests:** paginated later-page failures expose partial/error state; oversized UTF-8 and invalid coordinates/identity reject before PIN; unknown versions/owners/references fail closed; precise-location handoff is off-chain and consented.

## Handoff

Implement Batch 1 first: strict versioned invoice/ride adapters and deterministic tests, including canonical-owner mismatch cases. Then add durable issue/pay reconciliation, package closure, and finally paginated privacy-preserving schema migration. Do not enable real payment before all P1 exits and isolated two-profile acceptance pass.