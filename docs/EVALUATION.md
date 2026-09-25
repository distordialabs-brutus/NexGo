# NexGo current evaluation

Current as of 2026-09-25 for repository commit `7da750f1f674b9af839c650f6a47bc4a2af15729`. This is the rolling issue register; dated evidence is preserved in the [2026-09-25 development review](DEVELOPMENT_REVIEW_2026-09-25.md). Architecture and repair order are defined in [ARCHITECTURE.md](ARCHITECTURE.md) and [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md).

## Verdict

**Prototype; not payment-release-ready, package-ready, privacy-ready, or production-ready.**

There has been no application development since the prior reviewed runtime. Commit `7da750f…` changed only `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT_PLAN.md`, and added the 2026-09-23 review; `src/` remains tree `b0a45a0c3c4c7f38ac72f6dab134d2403da6f14f`. The 2026-09-25 offline rebuild reproduces the same bundle and the same release blockers.

## Active issue register

| Priority | Issue | Executed/source evidence | Current status | Exit evidence |
|---|---|---|---|---|
| P0 | Invoice creation uses the wrong destination key | `src/api/nexusAPI.js` sends `account`; LLL-TAO `Invoices::Create` calls `ExtractAddress(..., "to")`, whose accepted forms are `to`, `name_to`, `address_to` | Open | Production request test asserts `to` present/`account` absent; isolated target-core create/readback records address and txid |
| P0 | Current invoice terms are discarded | `normalizeInvoice` reads top-level fields; source `InvoiceToJSON` nests terms/status under `json`; offline probe returns amount `0`, blank account/recipient, zero items, no token | Open | Strict fixture preserves canonical envelope plus nested account/recipient/token/status/items and exact supported amount; malformed variants reject |
| P0 | Invoice owner is mis-modeled as issuer | Core `outstanding` standard is system-owned and `paid` is recipient-owned; current owner is lifecycle state | Open | Issuance tx/genesis is durably bound as issuer evidence; tests cover outstanding → paid/cancelled owner transitions and reject owner-as-issuer shortcuts |
| P0 | Ride/taxi authority fails open | Missing ride owner falls back to payload `passenger-genesis`; mutable taxi `driver` and settings-derived name control selection/mutation; first result is selected | Open | Canonical address/owner required; payload claims must match; active profile/network ownership verified; exact-address mutation only |
| P0 | No transition concurrency or prestate protection | `updateRideRequestAsset` merges stale `currentData`, performs zero reads, then calls `secureApiCall`; core update has no public CAS | Open | Exact pre-read/digest/state check, durable one-pending lock, allowed transition table, post-readback; changed prestate rejects before PIN |
| P0 | Invoice issue/pay are not durable | Driver/passenger view code stores no intent, remote identity, unknown outcome, or restart recovery; projection errors are reported as mutation failures | Open | Intent-first state machines survive every crash boundary; address/txid retained; exact readback resolves uncertainty; issue/pay each occur at most once |
| P0 | Money uses JavaScript floating point | Driver uses `parseFloat`; core source serializes invoice amounts as doubles and pays via `double * figures` | Open | Decimal-string input and integer base-unit policy; safe supported range; line-item/total equality; target-core transaction proves exact debited units at boundaries |
| P0 | No engineering gate | `npm test` and `npm run lint` are absent; no checked-in CI workflow or package test | Open | One clean-clone offline command runs all collected tests, lint/static checks, build, inventory, links, and whitespace; exact-head CI runs it |
| P0 | Production package omits runtime files | Clean build emits/references seven PNGs; manifest lists none | Open | Zero referenced/unlisted assets; clean folder and zip installation on pinned NexusInterface renders markers/layers/routes without denied requests |
| P1 | GPS denial triggers undisclosed IP lookup; exact trip data is public | `Main.js` calls `ipapi.co` after geolocation failure; map/geocoder/router calls are direct; ride raw payload stores exact coordinates/labels | Open | Provider-specific opt-in/revocation tests; no external request before consent; precise trip/location data removed from public records |
| P1 | Enumeration and failure semantics are incomplete | Fixed limits 100/500; several catches return `[]`/`{}` | Open | Multi-page complete/partial/error tests; deduplication; failures preserve known state and never display verified-empty claims |
| P1 | Legacy records do not implement the stated open protocol | Current `nexgo-ride` v1 combines intent/projection; ratings are mutable maps; autonomous service is a string claim | Open | Explicit legacy adapter plus shared request/offer/agreement and rating fixtures; role ownership, delegation, invoice binding, and private handoff proven |

## Verified positive controls

- The application uses NexusInterface `secureApiCall` for reviewed mutations; NexGo does not directly collect wallet PINs, profile passwords, API Basic credentials, or sessions.
- Current core invoice payment builds DEBIT and CLAIM in one transaction and rejects already-paid, cancelled, and wrong-token payment attempts.
- The lockfile supports a clean `npm ci --offline --ignore-scripts`; the production Webpack build succeeds reproducibly.
- The target commit and `origin/master` matched at review start, and the untracked `vision.md` remained outside Git and unchanged.

These controls do not close the active issues. A green build alone is not payment or package acceptance.

## Release decision

No real invoice, payment, profile write, production wallet installation, or production-readiness claim is approved. Implement [Batch 0 and Batches 1A–1D](DEVELOPMENT_PLAN.md), then run the isolated non-production live gates. The untracked vision is design context only and must not be staged implicitly with review documents.
