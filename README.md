# UmbraDB Indexer Proposal

**Decision draft · Midnight 2.x**

## Highlights

- **Privacy-focused:** protection for viewing keys, wallet↔transaction associations and retention.
- **Optional TEE:** conventional or confidential hosting for public-facing and private deployments.
- **Core midnight-ledger WASM:** official decoding, matching and replay.

## Project scope

| Project | Responsibility |
|---|---|
| **A — Data foundation** | Midnight-node-only ingestion, transaction evidence, public nullifier/commitment records, ledger-derived state/projections, retention and publication/recovery. Operates without wallets. |
| **B — Private data foundation** | Runtime viewing-key intake and scanning over A; protected wallet↔transaction mappings, scan coverage, private-access rules and key lifecycle. Optional TEE. |
| **C — API** | Independent public/private interfaces over A/B and optional D; authentication, B’s access rules, pagination and subscriptions. |
| **D — Other functionalities** | Optional enrichment, including Cardano pool metadata/staking via Blockfrost or UTxORPC; source adapters and derived data. Separately scoped. |

**A verification:** reuse **Lean 4 formal storage proofs**; validate publication, progress and retention with property-based and PostgreSQL crash/recovery tests. Abstract proofs require implementation-level validation.

## Design comparison

| Aspect | Midnight Indexer 4.4.0-rc.3 | Proposed UmbraDB |
|---|---|---|
| Structure | Chain, wallet, SPO and API components | Independent A/B/C delivered in six steps; wallet/private API usable at step 2 |
| Chain sources | Midnight node; Blockfrost for Cardano staking | A: Midnight node only. D: Blockfrost or UTxORPC |
| Ledger integration | Native Rust replay and root checks | Core ledger WASM; alpha decoding/matching, expanded replay/root checks |
| Database architecture | PostgreSQL/cloud; SQLite/standalone; ledger state store | PostgreSQL/production; PGlite/development target; separate private storage |
| Chain progress | Finalized blocks | Finalized input; separate ingestion and wallet-scan coverage |
| Private data | Viewing-key scanning; SDK defaults to local matching | Opt-in viewing keys and protected relevance mappings |
| Shielded public data | Nullifier tables, ledger events, tree end indices and prefix subscriptions | A: indexed nullifier/commitment records with tree indices; C: lookups, prefix filters and event streams |
| Deployment | Standalone or separate services | Public-facing or private; optional TEE for public/private processing |
| Interfaces | GraphQL, subscriptions and wallet protocols | Private alpha API; expanded public interfaces |

## Delivery steps, effort and investment decision \*

| Step | Delivers | Share of effort |
|---|---|---:|
| **0 — Testing infrastructure** (A) | Fixtures, property/crash harness, Lean gates and CI; the stack runs on PGlite for development and tests, PostgreSQL for acceptance | 7.5% |
| **1 — Common/public data store** (A) | Midnight-node-only finalized ingestion, versioned transaction evidence, stable observation IDs, publication/recovery contract and internal reads; operates without wallets | 18.3% |
| **2 — Wallet data store availability** (B, private C) | Viewing-key registration, historical/live matching, protected associations and coverage, lifecycle/revocation and cursor polling for one application; conventional profile, applied outcomes unknown | 31.7% |
| **3 — TEE execution for data availability** (B, private C) | Private scanning, key intake and private query execution inside the supplied confidential boundary, attested key release and rollback-safe restore | 10.8% |
| **4 — Extended public data** (A) | Ledger WASM replay and root checks, applied outcomes, retained state, unshielded/selected-contract projections and indexed nullifier/commitment records | 24.2% |
| **5 — Expanded public API** (C) | Public query routes, streaming and resumable delivery, admission/resource controls and reconnect behavior | 7.5% |
| **Steps 0–5 combined / with 20% contingency** | | **100% / 120%** |
| **6 — Other functionalities** (D) | Cardano pool metadata and staking enrichment via Blockfrost or UTxORPC | Separately scoped |

Shares are each step’s midpoint over the combined 0–5 midpoint. The wallet flow is usable after step 2 (steps 0–2: 57.5%); step 3 adds the confidential profile (steps 0–3: 68.3%). Steps 4–5 reuse the step 1 store and can proceed in parallel with steps 2–3.

**Included feasibility gate: 8.3% of the combined effort, within steps 0–3.** Tests node→WASM→protected storage→private API and recovery. Assumes one network, bounded history/load and one profile, including the supplied private TEE option. Conventional hosting trusts the operator.

\* AI speedup is not factored into these estimates.

## Required security review

Audit project-specific ingestion/publication, key custody, private database/TEE boundaries, tenant authorization, API limits and backup/revocation recovery. **Unmodified shared libraries, including midnight-ledger, are assumed secure and excluded from re-audit; integration and local modifications are in scope.** Review the delivered surface before public availability. **Provisional combined audit surface: 18–37k lines; exclusions require a file manifest.**

External audit, infrastructure and maintenance are additional. [Delivery, verification, estimates and audit details →](docs/design-and-effort.md)
