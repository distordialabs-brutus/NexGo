# NexGo — Decentralized Mobility Vision

## Portfolio roadmap authority

The master Distordia project also owns `PORTFOLIO_DEVELOPMENT_PLAN.md` and its strategy-decision register. The full order is **master strategy/customer evidence → portfolio roadmap/decisions → this vision → architecture/development plan → tasks/code/tests/release evidence**. Read the [portable repository alignment](docs/DISTORDIA_ALIGNMENT.md) for objective, customer-evidence, ownership and dependency mapping. Master-source paths below are local workspace references, not promised GitHub links. This section adds portfolio sequencing; it does not certify the envisioned behavior or amend unresolved master strategy assumptions.

The canonical business DOCX governs the thesis and the Customer Problem Atlas classifies problem evidence; the Atlas does not select a mobility direction or prove willingness to pay. The maintained portfolio plan is the required layer between those originals and this repository: it records sequencing and unresolved decisions without silently amending the DOCX. Mobility remains an unvalidated non-Atlas hypothesis. No marine Class A evidence transfers to rider, provider, fleet, wallet-settlement, privacy, insurance, or regulatory demand.

## Accountability venture context — not canonical authority

Local-only, non-link venture sources `/home/brutus/projects/Distordia/staked-accountability-rails.md` and `/home/brutus/projects/Distordia/infrastructure-buildout.md` are venture hypotheses and dependency-design context. They do not amend canonical strategy or prove enforceable collateral/slashing, non-custody, regulatory status, reputation, or adoption. Interpret unresolved claims through the master portfolio decision register (SD-002–SD-008); feasibility, legal assessment and human decisions remain required.

NexGo is an open mobility coordination layer for people and autonomous agents. It should let riders discover and contract with human drivers, owner-operated vehicles, robotaxis, and fleet agents through the same public standards—without making one application the gatekeeper.

The long-term product is not a map screen or a taxi marketplace. Interfaces will be cheap and replaceable. Durable value lies in scarce coordination primitives: accountable identity, interoperable ride agreements, verifiable settlement, portable reputation, privacy-preserving evidence, and explicit allocation of risk. NexGo applies the Distordia thesis to mobility.

**Current status:** this repository is a prototype and is not payment-release-ready or production-ready. The vision describes the intended system, not capabilities already proven by the code.

## Strategic grounding

NexGo inherits its direction from the canonical Distordia sources:

- **Local-only, non-link canonical source:** `/home/brutus/projects/Distordia/Distordia_Labs_Business_Thesis_and_Strategy_v2.docx`
- **Local-only, non-link customer-evidence source:** `/home/brutus/projects/Distordia/Distordia_Customer_Problem_Atlas_v2.docx`

The canonical sources and recorded portfolio decisions establish the operating posture; the venture documents above remain hypothesis context:

- **Open edge, integrated trust path.** Public asset standards and APIs permit independent clients, providers, fleets, verifiers, and agents to participate. NexGo may provide a reference experience, but must not control execution.
- **Standard-setter, not gatekeeper.** The defensible layer is legitimate, reproducible coordination and evidence—not exclusive access to riders, providers, or maps.
- **Namespace-centric identity.** A namespace is the accountable issuer behind a person, organization, vehicle operator, or autonomous agent. Payload identity claims never override canonical ownership or verified namespace bindings.
- **Evidence before assertion.** Trust is earned through verifiable history: signed agreements, settlement evidence, credential attestations, completed rides, disputes, and verdicts. Unsupported claims fail closed.
- **Human authority for consequences.** Agents may discover, negotiate, recommend, monitor, and propose. A clearly identified human must authorize consequential payments and adverse verdict or slash execution unless an explicit, bounded delegation has been reviewed and proven safe.

## The mobility protocol

NexGo should standardize a small set of composable records rather than encode an entire trip on-chain:

1. **Identity and authority** — namespace-bound identities for riders, human providers, fleet operators, vehicles, and autonomous agents; scoped delegation proves which agent may quote, accept, dispatch, or attest for whom.
2. **Service availability** — signed, expiring capability announcements with service class, coarse operating area, availability, credential references, and supported protocol version.
3. **Ride formation** — separate, role-owned request, offer, and agreement records. The accepted agreement binds rider, provider, service terms, fare basis, expiry, and the exact settlement reference without allowing one party to rewrite another party's intent.
4. **Settlement** — payment uses the underlying chain's conditional invoice and atomic payment semantics. Application projections such as `paid`, `occupied`, or `complete` are not themselves proof of payment. NexGo retains exact invoice and transaction identities, reconciles uncertain outcomes by readback, and never blindly repeats a consequential mutation.
5. **Reputation and accountability** — reputation is derived from agreement-bound evidence, not an unbounded popularity score. Ratings identify the ride and rater authority; objective breaches use deterministic verification where possible. Challenges, disputes, and verdicts remain portable and reproducible. Staked claims may strengthen accountability, but collateral, slashing, and arbitration are later protocol layers—not present guarantees.

Standards should remain compact, typed, versioned, bounded, and agent-readable. On-chain records contain only what independent parties need to establish authority, commitments, integrity, state transitions, and settlement evidence. Large artifacts and sensitive operational data are referenced by hashes or capability-controlled locators rather than copied into permanent public history.

## Privacy and safety boundary

Precise live location, pickup and destination coordinates, route searches, passenger counts, and travel history are sensitive. The target architecture keeps them off-chain and shares them privately only with the parties and services needed for an active ride. Public records should use coarse or expiring discovery data where required and commit to private trip artifacts with minimal hashes or attestations.

Location access and third-party geocoding, tiles, routing, or IP-derived positioning require provider-specific disclosure and affirmative consent. Denial must not trigger a hidden fallback. Secrets—including wallet PINs, passwords, sessions, and API credentials—remain inside the wallet boundary and must never enter NexGo storage, logs, or protocol records.

Safety and regulatory credentials should be represented as scoped, renewable attestations from accountable issuers, not as self-declared strings. Privacy does not remove accountability: authorized parties must be able to produce evidence for a dispute without publishing every rider's movements by default.

## Human and autonomous interoperability

Human providers and autonomous providers use the same ride and settlement standards. Differences belong in identity, delegation, capability, credential, and risk attestations—not in a closed parallel marketplace. Autonomous systems should interoperate through stable APIs and common agent protocols while namespaces provide the trust overlay: who operates the agent, what authority it has, what evidence it can produce, and who answers when it fails.

No participant should need the NexGo UI. A wallet module, fleet system, accessibility client, municipal service, or autonomous agent can discover and execute the same protocol independently. Any reputation algorithm presented as authoritative must be documented and reproducible from accepted evidence.

## Decision hierarchy

When sources conflict, apply this order:

1. **Canonical master strategy and customer evidence** — local-only, non-link sources `/home/brutus/projects/Distordia/Distordia_Labs_Business_Thesis_and_Strategy_v2.docx` and `/home/brutus/projects/Distordia/Distordia_Customer_Problem_Atlas_v2.docx`. The master local-only, non-link `/home/brutus/projects/Distordia/PORTFOLIO_DEVELOPMENT_PLAN.md` records portfolio sequencing and explicit strategy decisions before this repository vision.
2. **This vision** defines the durable mobility outcome and product boundaries.
3. **[Architecture](docs/ARCHITECTURE.md) and [development plan](docs/DEVELOPMENT_PLAN.md)** define current technical constraints, sequencing, and acceptance gates.
4. **Code, tests, executed reviews, and releases** demonstrate what is actually implemented and verified.

Lower layers may refine implementation but must not silently redefine higher-level intent. Conversely, vision language is not implementation evidence: only tested code and executed acceptance results justify capability or readiness claims.

## Development checklist

A change advances this vision only when it can answer **yes** to the applicable checks:

- [ ] Uses a published, versioned asset or message standard with explicit role ownership and canonical identifiers.
- [ ] Supports both human and autonomous providers without granting either a privileged private path.
- [ ] Binds every agent action to a namespace, operator, and narrowly scoped delegation.
- [ ] Stores the minimum necessary data on-chain; keeps precise location and trip details private/off-chain by design.
- [ ] Validates schema, ownership, state, bounds, and UTF-8 size before any wallet authorization prompt.
- [ ] Represents money in exact integer base units and binds agreement terms to the exact invoice.
- [ ] Persists intent before consequential submission, records returned identities, handles unknown outcomes, and verifies effects by exact readback.
- [ ] Distinguishes atomic chain settlement from non-atomic application projections and provides idempotent recovery for each.
- [ ] Makes reputation agreement-bound, Sybil-resistant where practical, explainable, and reproducible from evidence.
- [ ] Keeps payment approval and adverse verdict/slash execution under clear human authority unless an explicit bounded delegation is proven.
- [ ] Requires informed consent for every location or third-party data disclosure and emits no secrets or sensitive coordinates to logs.
- [ ] Paginates and reports partial discovery honestly; network or decoding failures never become an empty-market claim.
- [ ] Includes contract, transition, privacy, reconciliation, packaging, and adverse-path tests with pinned external interfaces.
- [ ] Preserves open APIs and exportable evidence so independent clients can interoperate and verify outcomes.
- [ ] Avoids production-readiness claims until wallet packaging, isolated end-to-end settlement, recovery, security, and privacy gates have executed successfully.
