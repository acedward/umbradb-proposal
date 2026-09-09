# UmbraDB Indexer Proposal

**Decision draft · Midnight 2.x**

## Highlights

- **Privacy-focused:** protection for viewing keys, wallet↔transaction associations and retention.
- **Optional TEE:** conventional or confidential hosting for public-facing and private deployments.
- **Core midnight-ledger WASM:** official decoding, matching and replay.

## Project scope

| Project | Responsibility |
|---|---|
| **A — Data foundation** | Midnight-node-only ingestion, transaction evidence, ledger-derived state/projections, retention and publication/recovery. Operates without wallets. |
| **B — Private data foundation** | Runtime viewing-key intake and scanning over A; protected wallet↔transaction mappings, scan coverage, private-access rules and key lifecycle. Optional TEE. |
| **C — API** | Independent public/private interfaces over A/B and optional D; authentication, B’s access rules, pagination and subscriptions. |
| **D — Other functionalities** | Optional enrichment, including Cardano pool metadata/staking via Blockfrost or UTxORPC; source adapters and derived data. Separately scoped. |

**A verification:** reuse **Lean 4 formal storage proofs**; validate publication, progress and retention with property-based and PostgreSQL crash/recovery tests. Abstract proofs require implementation-level validation.

## Design comparison

| Aspect | Midnight Indexer 4.4.0-rc.3 | Proposed UmbraDB |
|---|---|---|
| Structure | Chain, wallet, SPO and API components | Independent A/B/C; wallet/private-API alpha first |
| Chain sources | Midnight node; Blockfrost for Cardano staking | A: Midnight node only. D: Blockfrost or UTxORPC |
| Ledger integration | Native Rust replay and root checks | Core ledger WASM; alpha decoding/matching, expanded replay/root checks |
| Database architecture | PostgreSQL/cloud; SQLite/standalone; ledger state store | PostgreSQL/production; PGlite/development target; separate private storage |
| Chain progress | Finalized blocks | Finalized input; separate ingestion and wallet-scan coverage |
| Private data | Viewing-key scanning; SDK defaults to local matching | Opt-in viewing keys and protected relevance mappings |
| Deployment | Standalone or separate services | Public-facing or private; optional TEE for public/private processing |
| Interfaces | GraphQL, subscriptions and wallet protocols | Private alpha API; expanded public interfaces |

## Delivery order

1. **Wallet alpha — minimal A + B + private C.** Register keys, scan history/live transactions, retrieve matches/coverage and pause/revoke access. One application uses authenticated cursor polling. Applied outcomes are unknown; complete wallet accounting/sync is outside scope.
2. **Expanded data/APIs.** Replay/root checks, applied outcomes, retained state, unshielded/selected-contract projections, public APIs, streaming and PGlite development.
3. **Optional D.** Enrichment follows separate scope/source validation.

## Effort and investment decision \*

| Delivery | Engineer-weeks |
|---|---:|
| **Usable wallet/private-API alpha** | **20–30** |
| Expanded data/public APIs — additional | 28–42 |
| **Combined A+B+C / with 20% contingency** | **48–72 / 58–87** |
| D — Other functionalities | Separately scoped |

Alpha with contingency: **24–36 engineer-weeks** within the combined total.

**Included feasibility gate: 4–6 engineer-weeks.** Tests node→WASM→protected storage→private API and recovery. Assumes one network, bounded history/load and one profile, including the supplied private TEE option. Conventional hosting trusts the operator.

\* AI speedup is not factored into these estimates.

## Required security review

Audit project-specific ingestion/publication, key custody, private database/TEE boundaries, tenant authorization, API limits and backup/revocation recovery. **Unmodified shared libraries, including midnight-ledger, are assumed secure and excluded from re-audit; integration and local modifications are in scope.** Review the delivered surface before public availability. **Provisional combined audit surface: 18–37k lines; exclusions require a file manifest.**

External audit, infrastructure and maintenance are additional. [Delivery, verification, estimates and audit details →](docs/design-and-effort.md)
