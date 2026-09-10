# Scope, design and estimate notes

[← Proposal](../README.md)

**Decision draft · 2026-09-10 · Midnight 2.x.** The first usable alpha delivers wallet transaction discovery and a private API. An expanded release adds replay-derived public data and public interfaces. This page defines both delivery boundaries, their engineering assumptions and investment evidence. The reference is Midnight Indexer `4.4.0-rc.3`; package 4.x, node 2.x and ledger v9 are different version axes. Capabilities and acceptance criteria describe the target service; the evidence section identifies the available implementation and validation assets.

## Delivery priority: usable wallet alpha

**The first deliverable is a working private flow:** a user registers a supported viewing key, discovers relevant transactions from a declared history range, follows live finalized observations, retrieves results and scan coverage through an authorized API, and pauses or revokes access. One selected application completes this flow. B owns private processing and C owns the private API as separate development responsibilities.

### Alpha data dependency

The wallet flow needs a small independently operable part of A: Midnight-node-only finalized input, transaction payload extraction with protocol/network identification, stable observation IDs, durable ordered storage, declared source coverage, restart and resumable reads. A receives no viewing keys or private associations. Unsupported payloads or protocol versions create explicit gaps or stop processing; they cannot count as fully scanned history. B consumes this committed source contract, and C consumes B; neither grows a separate ingestion pipeline. Minimum A also persists the nullifiers and commitments it extracts from observed offers, marked with unknown applied status, so verified indexing in the expanded release does not re-ingest that history; see [public nullifiers and commitments](#public-nullifiers-and-commitments).

**Full ledger replay is not a dependency of the alpha relevance predicate.** The pinned WASM API exposes `EncryptionSecretKey.test(offer)` and key deserialization. Its implementation tests output/transient ciphertexts using the supplied key. The reference indexer’s predicate checks guaranteed and fallible transaction offers without taking global ledger state. This supports a source-based design inference that transaction observations can feed private matching before full public state projections exist. Key format, versioned transaction extraction and real fixture results require validation. [WASM key API and predicate](https://github.com/midnightntwrk/midnight-ledger/blob/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm/src/zswap_keys.rs#L289), [reference transaction relevance](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/src/domain/ledger/transaction.rs#L250).

Alpha records **observed transaction relevance**, with node-reported finalized inclusion, source/version provenance and scan coverage. Applied outcomes are explicitly `unknown`; a match does not imply successful execution, received funds, a balance or complete outgoing history. Full replay, root verification and applied outcome enrichment belong to the expanded release. If the selected consumer requires those guarantees, it does not fit the alpha contract.

### Alpha private API and acceptance

| Capability | Usable alpha behavior |
|---|---|
| Register | Authenticate the tenant, accept a supported viewing key and requested scan range, return a monitor handle, and keep the key out of responses/logs |
| Read status | Report monitor lifecycle, source coverage and scanned ranges/progress; distinguish no matches from unscanned or unsupported data |
| Read matches | Return authorized paginated transaction references with stable cursors, provenance and explicit unknown applied outcome; polling provides live updates |
| Pause/revoke | Stop or revoke processing and access with stale-worker checks; restart/restore respects lifecycle state |
| Recover | Commit private matches and scan progress together, deduplicate retries, and preserve authorization and confidentiality across interruption and supported restore |

The alpha uses PostgreSQL, one network/version range, bounded history/load and one selected deployment profile. Its allowance includes the supplied TEE option for B and private request handling; A can run conventionally. A conventional private profile explicitly trusts the runtime operator. Supporting every profile, public processing inside a TEE and PGlite development packaging are expanded-release work. The alpha does not require a public blockchain API or WebSocket service; private cursor polling supplies the first consumer’s integration.

Alpha acceptance includes real node→WASM matching→private-store→API evidence, positive/negative and fallible-offer fixtures, historical/live continuity, wrong-tenant rejection, interruption/retry, revoke-during-scan, supported restore and bounded-load behavior. A’s verification combines Lean 4 storage proofs with property-based and PostgreSQL crash/recovery tests. Key/association protection, declared retention and recovery are delivery requirements. Public availability requires an external audit of project-specific code in this path, including minimum A and the chosen profile. Unmodified shared libraries, including midnight-ledger, are trusted dependencies outside re-audit; their integration and local modifications are in scope.

### Delivery sequence

| Delivery | Dependencies and parallel development | Value |
|---|---|---|
| **1 — Wallet/private-API alpha** | Minimum A ingestion/storage and B’s data/security contract establish the live integration. B and private C develop concurrently against fixtures/contracts and integrate incrementally; full A and public C are not prerequisites | A real application registers a key and retrieves historical/live relevance with coverage and lifecycle controls |
| **2 — Expanded data and APIs** | Reuse the alpha’s ingestion, identities and private service. Add complete required replay/root checks, outcome enrichment, retained-state reads, unshielded/selected-contract projections, public C routes and streaming; broaden deployment/PGlite support | Public data consumers and richer private context use the same foundations |
| **D — Optional enrichment** | Depends on selected external-source coverage and C presentation; does not gate the alpha or expanded core | Pool metadata and staking analytics |

These are product delivery boundaries and dependencies, not an executable implementation plan. Internal IDs/contracts need versioning so replay enrichment preserves the alpha’s observation references or provides an explicit migration; it does not create a second wallet indexer. Concrete network, application, history/load targets and deployment profile are feasibility inputs.

## Core delivery and optional extensions

| Project | Expanded-release capability, including alpha | Independent acceptance |
|---|---|---|
| **A — Data foundation** | Node-fed finalized block/transaction evidence, execution outcomes and stable identities; unshielded created/spent history; public zswap/DUST nullifier and commitment records with commitment-tree indices; selected contract actions/state reads; persisted ledger state, bounded retention, coherent publication and durable changes | Operates with zero wallets and B/C/D stopped. Replays the selected network’s required history, matches roots and expected nonempty data, and survives interrupted writes, retry, restart and supported restore without exposing partial state |
| **B — Private data foundation** | Private data definition, retention and threat model; protected runtime registration, historical/live scanning, wallet↔transaction mappings and coverage; conventional and optional TEE deployment with database/key-management lifecycle | A trusted harness can register, read, pause/revoke and restore monitors without C’s public server. Tests cover tenant isolation, storage/log/backup exposure, stale/revoked work and empty versus unscanned history; the TEE profile also validates attestation and key release |
| **C — API** | A small public/private query surface and one resumable change-delivery protocol; authentication, authorization, admission limits and one selected application integration | The selected consumer completes its required flow against A/B. Reconnect and overload have explicit behavior; keys and private results stay within the authorized trust boundary |
| **D — Other functionalities (optional)** | Pool metadata and staking analytics using a Blockfrost or UTxORPC-backed adapter; additional features scoped individually | Supported fields have verified source/derivation, network, freshness and coverage. C can expose D’s results; disabling or losing D does not interrupt A/B or core C routes |

The alpha consumer fits private transaction discovery with unknown applied outcomes. The expanded release also supports a dApp querying selected contract/transaction data. Full existing wallet synchronization may require the deferred DUST/tree interfaces and therefore changes the scope. Select the consumer and concrete fields before implementation estimates become commitments.

A supplies every datum and state operation needed by the **selected core** endpoints. C consumes A/B contracts for ingestion, replay and private scanning. B and C can develop against versioned internal contracts; integration depends on A. In the TEE profile, C’s private handler executes within the protected boundary or forwards encrypted requests to a handler there; API development is a separate project. D owns optional external enrichment and its derived data; C owns its external presentation. D-related API work is additional to the core C estimate.

B owns private authorization rules, the trusted key-intake/read boundary and secure persistence; C owns protocol adapters, credential integration and public admission controls. C must invoke B’s authorization checks on every private operation. Shared security mechanisms are counted once in B, with their API integration counted in C. C’s public work can proceed against A; private integration depends on B’s data and security contracts.

## Ledger integration, database and deployment

**Core processing uses midnight-ledger WASM.** Alpha uses versioned decoding and viewing-key matching; the expanded release adds full required state transitions and replay. Midnight Indexer integrates that ledger through native Rust. UmbraDB owns node input handling, orchestration, projections, persistence and publication; ledger semantics stay in the ledger implementation. The upstream repository provides a ledger WASM API and the UmbraDB ledger-v8 prototype exercises replay. [Ledger WASM source](https://github.com/midnightntwrk/midnight-ledger/tree/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm), [prototype replay](https://github.com/acedward/UmbraDB/blob/a562f9cdde3bc8982f336f30b16b988c6e2a2b8c/chain-archive-sync/ledger-replay.ts#L157).

Alpha feasibility validates key decoding, offer extraction and matching exports on the selected 2.x data. Expanded-release feasibility inventories exports for block closure/root checks, applied facts, state reads and durable state integration. The prototype includes custom v8 binding extensions; equivalent v9 replay coverage and efficient storage access require validation. Missing exports count as binding/integration work with separate review of local modifications. WASM is the target processing boundary; its use alone does not establish complete replay coverage, performance or TEE isolation.

**Database architecture:** PostgreSQL is the production database for public projections and publication metadata, with a compatible durable ledger state store. B uses a logically separate protected private store. **PGlite is the local-development target**, providing embedded PostgreSQL through WASM and a JavaScript/TypeScript API. The development scope covers a limited adapter, schema/migration checks, representative queries and local persistence. [PGlite architecture and development use](https://pglite.dev/docs/about).

PGlite compatibility requires checking UmbraDB’s SQL client interface, required extensions, transaction behavior and lock usage. PGlite uses one database connection; its socket multiplexer differs from a PostgreSQL server. Production concurrency, writer fencing, durability and restore acceptance therefore use PostgreSQL. PGlite development fixtures do not establish private-data protection or replace the TEE validation environment. [Connection model](https://pglite.dev/docs/), [socket compatibility limits](https://pglite.dev/docs/pglite-socket).

**Deployment:** service visibility and confidential execution are separate choices. Public-facing infrastructure and restricted/private installations each support conventional hosting or the supplied TEE platform. A, B and C have independent process boundaries; a TEE can cover public processing, private processing or both. Private API operations require authorization in every deployment.

| Profile | Operating model and trust |
|---|---|
| Conventional public or private deployment | Operator-controlled processes; encryption, authorization, tenant separation and recovery controls apply. The runtime administrator is trusted with plaintext processed by B |
| TEE for B | Private scanning, key intake, private query execution and authorized results share the confidential boundary; external storage is protected. A and public C routes can run conventionally |
| TEE for A and/or C | Public-chain processing or API execution uses confidential compute. Published chain data is public, and B requires its own protected boundary to offer confidentiality from the host |
| Local development | PGlite with development fixtures and a conventional runtime; PostgreSQL and the selected TEE platform supply production acceptance environments |

The TEE profile protects keys and associations against external host access within the declared platform threat model. It includes attested key release and fails closed if that policy cannot be satisfied; it does not silently downgrade to conventional processing. The estimate assumes reuse of one supplied TEE platform, shared deployment packaging and a limited PGlite adapter. Platform-specific public/private workload fit and memory limits are feasibility checks; additional platforms, general browser packaging and full PGlite production parity are outside scope.

## Expanded-release boundaries

| Topic | Included | Deferred or excluded |
|---|---|---|
| Ledger correctness | Official midnight-ledger WASM computation and required bindings for the selected chain, including DUST/bridge effects and required historical version transitions | New consensus/cryptography, arbitrary version combinations and broad historical migration support |
| Public reads | Core block/transaction/outcome, unshielded and selected contract data; applied zswap/DUST nullifier and commitment records with global commitment-tree indices and per-block roots/end indices; retained state needed to serve those reads and B | Complete contract analytics, advanced DUST snapshots/deltas, full wallet tree-sync interfaces and bridge analytics; staking/pool enrichment belongs to optional D |
| Data sources | A uses Midnight node, with the historical inputs needed for declared coverage; official ledger libraries provide computation | Official indexer as a production source/fallback is excluded. Blockfrost or a UTxORPC-backed service is allowed only for separately scoped D enrichment |
| History | Explicit retained coverage, restart/restore and supported source-generation changes | Unlimited snapshots, accelerated arbitrary-wallet recovery, importing every experimental database or preserving arbitrary provider cursors |
| Private product | Transactions detectable by the supported encryption viewing capability, with provenance, outcome context and coverage | Complete outgoing history/balances, DUST ownership, arbitrary private contract history or a general decrypted-payload export |
| Interfaces/deployment | One network, PostgreSQL production storage, limited PGlite development support, conventional hosting plus optional use of one supplied TEE platform, one private persistence design and one consumer willing to adapt/resync | Full GraphQL/transport parity, every wallet unchanged, SQLite parity, PGlite production parity, general browser packaging, a separate best-head/pending index or multiple TEE implementations |

Data integrity, privacy and recovery are requirements across the service. Source classification must be explicit, and omitted data must not be represented as a complete empty result.

## B — Private data and security scope

**B carries more design uncertainty than the focused API.** Its scope includes the private schema, trusted application boundary, database integration and operating lifecycle for conventional and TEE deployments. The TEE profile protects data during execution; storage and transport need their own protections. [Confidential Computing Consortium threat model](https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/CCC-A-Technical-Analysis-of-Confidential-Computing-v1.3_unlocked.pdf#page=10).

### Determine the minimum private data

The following candidate logical records define the starting data model. The first deliverable is a field-level contract specifying purpose, owner, sensitivity, retention/deletion and permitted outputs for each field. Validation against the selected application and actual viewing-key fixtures determines the physical storage design.

| Record | Minimum candidate contents and purpose |
|---|---|
| Monitor/key registration | Tenant and monitor identity, network and supported viewing capability, protected key reference, key version and authorization policy; establishes whose history may be scanned and read |
| Private association | Monitor reference, stable transaction/observation references from A, match semantics/version and provenance; deduplicated wallet↔transaction links preserve all relevant occurrences. Alpha outcome is unknown; expanded replay enriches execution/outcome context |
| Scan coverage | Requested history range, completed ranges/cursor, source generation and scanner version; distinguishes unscanned history from completed scans with no matches |
| Lifecycle and recovery | Active/paused/revoked state, revocation generation, retention/deletion state and restore metadata; prevents stale workers and restored snapshots from silently reviving access |
| Security records | Minimal protected authorization and lifecycle events needed for investigation; operational telemetry exposed outside the boundary must not disclose keys or wallet↔transaction relationships |

A stores transaction bytes and provenance, with public outcomes supplied by expanded replay; B owns the sensitive links and private progress. The baseline does not persist decrypted notes, balances or arbitrary private payloads. Add those only when an explicit consumer requirement justifies their retention and additional review. A successful output match and a complete wallet history are different product claims.

### Secure the whole private path

**Proposed trust model:** both profiles exclude unauthorized tenants from plaintext keys and associations. Conventional hosting trusts the runtime administrator. The TEE profile additionally excludes the external host/storage operator within its declared boundary. Identify the trusted application, platform, release approvers and key-policy administrators explicitly. Document residual timing, traffic-volume and storage-access leakage; full access-pattern hiding and new hardware/side-channel research are outside this estimate.

The private boundary must cover key intake and authorized responses, the scanner, database queries and indexes, temporary files, logs, crash dumps, backups and migrations. In the TEE profile, an ordinary external SQL database exposes plaintext association joins despite encrypted viewing-key columns. Choose one supported design: a private database engine inside the supplied confidential environment with protected persistent storage, or a TEE record adapter using authenticated encrypted records in an external store with an explicit leakage policy. Do not assume A’s PostgreSQL deployment can also host B’s plaintext data. B consumes public batches without exposing match-dependent requests to A; the threat model identifies residual access-pattern leakage.

Platform capabilities affect that choice. For example, Nitro Enclaves provide no persistent storage or external networking; AWS documents a separate attested key-release and encrypted external-storage integration. This illustrates work B must account for, without selecting AWS for this proposal. Reuse the supplied platform’s established attestation/key-management facilities and cryptographic libraries. [Nitro concepts](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave-concepts.html), [attestation and key release](https://docs.aws.amazon.com/enclaves/latest/user/kms.html).

Recovery must preserve confidentiality, integrity and revocation across restart, backup restore and software/key rotation. A candidate design checks restored state against a trusted freshness/revocation record outside the snapshot’s rollback domain; if the check is unavailable, it must deny private processing until freshness is established. The concrete authority and failure behavior need validation on the supplied platform. Encryption alone does not meet this proposed restore contract. Deletion requirements must cover retained backups and key lifetime, with explicit limits on data returned to an authorized user.

Acceptance evidence must include wrong-tenant access, unauthorized key release, tampered and stale stored records, revoke-during-scan, backup/restore, release/key rotation, log/crash exposure and representative scan/storage load. These checks establish behavior against the declared threat model; an external security review is a separate release requirement.

## D — Other functionalities

**D is an optional extension category covering Cardano pool metadata and staking analytics.** Its candidate data interfaces are **Blockfrost and a UTxORPC-backed service**. The core A/B/C release operates with D disabled. D’s provider selection and adapter implementation are outside the core delivery.

The reference stake pool operator (SPO) service uses Blockfrost for Cardano pool names/tickers/homepages and live/active stake, delegators, saturation and pledge. The selected core block/transaction/contract/private-discovery flows do not require these inputs. [Provider calls](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/spo-indexer/src/infra/spo_client.rs#L358).

| Reference operations | Effect of omitting the external source |
|---|---|
| `poolMetadata`, `poolMetadataList`, `stakeDistribution` | Complete metadata/staking responses are unavailable |
| `spoByPoolId`, `spoCompositeByPoolId` | Node identity/performance portions are obtainable; metadata-enriched responses are incomplete |
| `spoCount`, `spoList`, `registeredTotalsSeries` | Exact reference population/count/list behavior is unavailable because these queries depend on externally refreshed stake snapshots |

This accounts for **eight of 53 reference root operations**. Twelve other SPO operations have node-obtainable underlying facts, including registrations, committees, epochs and performance. These counts describe data-source dependencies; first-release endpoint selection follows the core scope. [Count/list queries](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-api/src/infra/storage/spo.rs#L112).

### Candidate data interfaces

| OPTION | PROS | CONS |
|---|---|---|
| **Blockfrost adapter** | Pool metadata/stake endpoints closely match the reference’s inputs | External service trust, availability/rate limits and recurring dependency; field definitions, network support and historical coverage need validation |
| **UTxORPC-backed adapter**, hosted or self-operated | Standard chain-data interface with a choice of compatible servers; supports custom derived views | UTxORPC is a protocol, not a guarantee of precomputed staking analytics. Verify server/version coverage and implement missing aggregation, synchronization/rollback and off-chain metadata handling |

The UTxORPC specification describes chain synchronization and state queries, with Cardano transaction/certificate types. Backend selection requires mapping each D field to a supported method or an application-level derivation, with a pinned server/spec version and network. Backend equivalence and performance require validation against the selected fields and workload. [UTxORPC specification](https://github.com/utxorpc/spec), [query interface](https://utxorpc.org/query/spec/), [Cardano types](https://utxorpc.org/cardano/), [server implementations](https://utxorpc.org/servers/).

Pool names/tickers and other metadata contents are generally retrieved from external pool URLs; chain access alone does not guarantee those documents are available. A self-operated backend can include additional indexing/metadata components, but that deployment work must be included in D’s own scope. [Cardano metadata management](https://docs.cardano.org/stake-pool-operators/metadata-management).

**Ownership and availability:** D maintains a distinct public enrichment dataset with source/network, freshness and coverage, joins through verified pool identities and supplies results to C. D’s outage must not block A’s Midnight ingest or B’s private processing. An unavailable/stale source must not produce fabricated zeros or supposedly complete empty lists. D owns external enrichment sources; A uses Midnight node exclusively.

**Effort and review:** D, its additional C routes and any hosted/self-operated backend costs are outside the **58–87 engineer-week** A+B+C allowance including contingency. Estimate D after choosing required fields, backend coverage and application-level derivations. Its source adapters, metadata fetching, untrusted input handling, freshness/rollback behavior and public API additions must join the security review when enabled. D’s code size and audit effort are not included in the core estimates below.

## Finality

The reference follows finalized blocks, verifies ancestry and checks replay roots. It does not supply a general automatic unwind of conflicting finalized history in the inspected path. Its 400-block catch-up margin changes how history is fetched; it is not a live indexing delay. Alpha follows a configured node’s finalized history and verifies ancestry/continuity; independent replay-root verification belongs to the expanded release. Track produced, finalized, ingested and wallet-scanned positions separately. Finality gap and processing throughput require measurement on the selected deployment. [Finalized ingestion](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/chain-indexer/src/infra/subxt_node.rs#L387), [continuity/root checks](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/chain-indexer/src/application.rs#L303).

## Required design work

**Replay and identity.** The v8 experiment demonstrates useful archive/replay integration, but its replay result is too narrow for complete selected projections. Archival occurrences, ledger executions and emitted events are distinct; multiple occurrences can represent one execution. Reuse official ledger semantics and expose applied facts with stable identities rather than guessing outcomes from transaction bytes. A transaction’s private relevance can include a nonapplied segment, so a match is not by itself proof of received funds. [Experiment replay](https://github.com/acedward/UmbraDB/blob/a562f9cdde3bc8982f336f30b16b988c6e2a2b8c/chain-archive-sync/ledger-replay.ts#L157), [reference relevance predicate](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/src/domain/ledger/transaction.rs#L240).

**Publication and retained state.** The prototype commits each archive block bundle atomically, with replay checkpoints and the ingest watermark handled separately. The service needs a committed query view across selected facts and referenced ledger state, plus durable subscription resume. The [publication and recovery contract](#publication-and-recovery) defines the required behavior. [Archive bundle transaction](https://github.com/acedward/UmbraDB/blob/a562f9cdde3bc8982f336f30b16b988c6e2a2b8c/src/postgres/chain-archive-store.ts#L387), [ingest write sequence](https://github.com/acedward/UmbraDB/blob/a562f9cdde3bc8982f336f30b16b988c6e2a2b8c/chain-archive-sync/sync-service.ts#L746).

**Private persistence.** Umbra’s wallet-history table needs a separate protected relevance engine and lifecycle. The [B scope](#b--private-data-and-security-scope) defines the required data-definition, TEE/database, authorization and recovery work. [Wallet-history storage](https://github.com/acedward/UmbraDB/blob/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/src/postgres/transaction-history-storage.ts#L455).

The official indexer contains a relevant-transaction scanner, SQL associations, persisted state and garbage collection. UmbraDB’s proposed value is independent reusable foundations, explicit private-data controls and an optional TEE privacy boundary. [Reference wallet mapping](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/migrations/postgres/001_initial.sql#L135).

### Publication and recovery

**Publication is the point at which a complete indexed result becomes visible to readers. Recovery restores processing from a durable, consistent position after interruption.** These are storage guarantees for finalized data; they do not implement chain reorganization handling.

For example, A publishes through block 99 while preparing block 100. Processing block 100 produces transaction outcomes, public projections and ledger state. A publishes those required facts, references to durable state, the published position and an ordered change record as one logical update. Reads pin a published generation or database snapshot so a multi-query response cannot mix states from blocks 99 and 100.

This replay/state example describes the expanded release. Alpha applies the same visibility principle to node observations, payloads, source coverage and committed progress. It needs no queryable global ledger-state store to publish those observations. B’s protected association/progress transaction and C’s durable cursor pagination are alpha requirements; push notifications and the public replay-state store are expanded capabilities.

| Failure point | Required behavior |
|---|---|
| Before publication commits | Readers see block 99. Recovery retries block 100 without duplicate logical records; staged state is invisible and eligible for safe cleanup |
| After publication commits, before a notification | Readers see block 100. C resumes from durable change records; a lost wake-up does not lose the indexed change |
| A stale writer attempts publication | A generation/lease check rejects it; writer fencing protects the committed sequence |
| A wallet scan stops during a batch | B commits associations and that monitor’s scan progress together, including empty batches; retry neither skips history nor duplicates logical matches |

A candidate implementation persists verified content-addressed ledger state first, then uses one SQL transaction to publish its references, projections, progress and change records. A single transaction covering state and SQL is another option when the WASM storage interface supports it. Garbage collection preserves state referenced by published generations and active readers. Notifications act as wake-ups; durable changes provide resumption, with stable identities for deduplication and explicit retention limits.

**A and B have separate progress.** A can publish block 100 while a monitor’s completed scan reaches block 96. C reports both positions and coverage; it does not present blocks 97–100 as an empty completed scan or block A on private work. Within B, association writes, scan progress and lifecycle checks share a commit boundary. Backup restore additionally enforces the private freshness/revocation contract.

The main complexity is coordinating the WASM ledger state store, SQL visibility, multiple writers, private scan lifecycle and consumer resumption across crashes. PostgreSQL transactions supply part of that mechanism; the integration must establish the complete contract. A owns public publication/recovery, B owns private scan/revocation recovery, and C owns resumable delivery. Their estimates include this work.

### Viewing-key scanning and wallet SDK synchronization

**Midnight Indexer `4.4.0-rc.3` supports viewing-key submission** through `connect(viewingKey, options)`, `disconnect(sessionId)` and `shieldedTransactions(sessionId, index)`. The connect handler registers the key, and the wallet-indexer scans and persists relevant transaction associations. These capabilities describe the pinned source implementation; live deployment availability and correctness require validation. [Schema](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-api/graphql/schema-v4.graphql#L1163), [registration handler](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-api/src/infra/api/v4/mutation.rs#L44), [scanner](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/wallet-indexer/src/application.rs#L295).

**The wallet SDK at the pinned commit defaults to client-side matching.** Its default shielded sync subscribes to public `zswapLedgerEvents(id)` and replays those events locally with the wallet's secret keys; this flow uses no viewing-key submission to the indexer. [Default builder](https://github.com/midnightntwrk/midnight-wallet/blob/66dd2b1d4962400d422a97854628da928a137758/packages/shielded-wallet/src/v1/V1Builder.ts#L154), [event subscription and local replay](https://github.com/midnightntwrk/midnight-wallet/blob/66dd2b1d4962400d422a97854628da928a137758/packages/shielded-wallet/src/v1/Sync.ts#L243).

**B provides opt-in hosted private discovery in the proposed service:** users explicitly submit a supported viewing key to a process that maintains protected wallet↔transaction mappings. The deployment profile determines whether the runtime operator is trusted or the process uses a TEE boundary. B serves applications requiring hosted private discovery. Complete default wallet synchronization through A/C requires a separate scope for the wallet data interfaces. The [public nullifier and commitment records](#public-nullifiers-and-commitments) are the stored input for that client-side path and for the nullifier-prefix filter; serving them is expanded public C work.

### Public nullifiers and commitments

**A stores every zswap and DUST nullifier and commitment as an indexed public record.** A shielded transaction publishes a commitment for each created coin and a nullifier for each spent coin; a DUST spend publishes a nullifier with its paired commitment. These values are public chain data and disclose no wallet association without the holder’s keys. Indexing them serves spend detection and reduced-download synchronization for wallets that never submit a viewing key, gives B a stable public reference for its private matches, and supplies the stored input for later Merkle-tree gap fills. The reference indexer persists zswap and DUST nullifier tables, ledger events with prefix-filterable `nullifier`/`commitment` fields, per-transaction zswap/DUST start/end indices and per-block roots, and serves them through `zswapLedgerEvents` and the `shieldedNullifierTransactions`/`dustNullifierTransactions` prefix subscriptions. [Nullifier tables](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/migrations/postgres/001_initial.sql#L175), [ledger events](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/migrations/postgres/001_initial.sql#L106), [tree indices](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-common/migrations/postgres/001_initial.sql#L51), [prefix subscription](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-api/graphql/schema-v4.graphql#L1984), [event stream](https://github.com/midnightntwrk/midnight-indexer/blob/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c/indexer-api/graphql/schema-v4.graphql#L2482).

| Field | Content |
|---|---|
| Value and kind | 32-byte nullifier or commitment; zswap or DUST; a DUST nullifier keeps its paired commitment |
| Origin | Transaction/block coordinates, stable observation ID, event order, logical segment and optional contract address |
| Applied status | `unknown` for alpha observations; applied or not applied once expanded replay confirms the segment outcome |
| Tree position | Global commitment-tree index for commitments; per-block zswap/DUST roots and end indices for verification |
| Indexes | Exact nullifier, nullifier prefix with block range, exact commitment and tree-index range |

Alpha persists the observed zswap values from the offers minimum A already extracts for B, so that history is not re-ingested when indexing is verified. An observed nullifier in a failed fallible segment is not a spend; readers must not treat `unknown` records as applied. Expanded replay marks applied records, adds DUST records, assigns global indices from ledger state and checks them against each block’s roots and end indices; a mismatch stops publication rather than logging a warning. The pinned WASM exposes offer inputs, outputs and transients with `nullifier()`/`commitment()` accessors and the chain state’s `firstFree` index; efficient index assignment across contract-produced outputs is an expanded feasibility check. [Offer accessors](https://github.com/midnightntwrk/midnight-ledger/blob/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm/src/zswap_wasm.rs#L675), [input nullifier](https://github.com/midnightntwrk/midnight-ledger/blob/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm/src/zswap_wasm.rs#L483), [output commitment](https://github.com/midnightntwrk/midnight-ledger/blob/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm/src/zswap_wasm.rs#L337), [tree end index](https://github.com/midnightntwrk/midnight-ledger/blob/4823b5351b17cc49e30f19760dbd30a73cf95e22/ledger-wasm/src/zswap_state.rs#L473).

These records are public and stay in A’s conventional store. B may reference them by stable identity inside its protected store; A never learns which records a monitor matched. A nullifier-prefix query carries a probabilistic hint about the requester’s coins, so C treats prefix length, block range and request rate as admission-controlled parameters and B’s threat model records the residual leakage. Full wallet tree-sync interfaces and collapsed Merkle updates remain deferred; this item defines the stored records and indexes they would consume.

## A — Formal verification and implementation testing

**The repository calls this formal verification of its storage algebra.** It uses **Lean 4 with mathlib** under `Formal/Lean`, alongside property-based tests using `fast-check` and real PostgreSQL. The pinned README identifies abstract proofs for temporal projection/coherence, watermark last-write-wins and checkpoint chunk merging. The CI workflow includes a Lean trust gate and independent proof checking. These are source findings, not a claim that this proposal executes the proof/test suites. [Verification terminology and coverage](https://github.com/acedward/UmbraDB/blob/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/README.md#L252), [Lean proof tree](https://github.com/acedward/UmbraDB/tree/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/Formal/Lean), [CI trust gate](https://github.com/acedward/UmbraDB/blob/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/.github/workflows/lean.yml).

| Method | Role in the data foundation | Evidence boundary |
|---|---|---|
| **Formal proofs — Lean 4** | Preserve covered storage laws and extend abstract models/proofs when their contracts change; record the exact statement and assumptions for each guarantee | Proof applies to the abstract model. The repository has no mechanized proof connecting that model to SQL/TypeScript |
| **Property-based tests — fast-check** | Exercise generated storage operations against the actual adapter and compare with the expected model; cover retries, observation identities, progress and retention | Sampled implementation evidence, using real PostgreSQL |
| **Crash/recovery and concurrency tests** | Interrupt writes/commits and restart workers/PostgreSQL; verify coherent publication, durable cursors, deduplication and rejection of stale writers | Evidence for selected fault/interleaving cases in the deployment environment |
| **Model checking — Quint** | Candidate bounded exploration for concurrency/GC/lease behavior | Planned in the pinned source; not an existing verified capability or an alpha dependency |

For alpha A, the target invariants are: visible progress never exceeds durable source data; a publication exposes a coherent observation set; retries preserve logical identity; unsupported/unscanned ranges are distinguishable from completed empty ranges; stale writers cannot overwrite committed progress; retained coverage never advertises unavailable data. Expanded A adds durable ledger-state references and safe retention around active readers. B uses corresponding atomic association/scan-progress checks and lifecycle enforcement. Every guarantee needs a named proof or runtime test and explicit assumptions; the existing storage proofs do not automatically establish these complete service invariants.

The runtime suite includes watermark property tests and a `saveAndAdvance` crash test that interrupts a transaction between its checkpoint and cursor writes. Reuse that pattern for A’s publication boundary and B’s private progress. Run applicable Lean gates, property tests and PostgreSQL fault tests in delivery validation; PGlite cannot substitute for production concurrency/durability evidence. [Watermark property test](https://github.com/acedward/UmbraDB/blob/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/test/postgres/watermarks.property.test.ts#L28), [atomic save/progress crash test](https://github.com/acedward/UmbraDB/blob/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca/test/integration/crash/saveandadvance-cotx.crash.test.ts#L17).

The estimate includes reuse/maintenance of existing proof gates and focused property/fault tests in A’s validation allowance. A complete proof of the new distributed protocol, SQL refinement, eventual GC or a new Quint verification project requires separate scope and effort. Formal verification supplies evidence for the project-specific review; it does not replace private-data, API or deployment security assessment.

## Optimistic delivery effort

These judgment-based estimates cover the alpha and expanded release together. They assume an experienced team, reuse of the node-ingest prototype and official midnight-ledger WASM, available historical node access, conventional deployment packaging and optional use of one supplied TEE platform with usable attestation/key-management facilities. Alpha targets bounded history/load and one profile, with allowance for the private TEE option. Expanded A includes limited PGlite support and broader packaging; expanded B completes operational coverage and profile support. Binding, database and platform compatibility are feasibility conditions. Each row includes its focused implementation tests; integration/recovery/security/load evidence is counted once.

| Project | Usable alpha | Expanded release — additional | Combined base engineer-weeks |
|---|---:|---:|---:|
| A — Data foundation | 6–10 | 18–26 | 24–36 |
| B — Private foundation | 10–14 | 6–10 | 16–24 |
| C — API | 4–6 | 4–6 | 8–12 |
| **Total** | **20–30** | **28–42** | **48–72** |

**Alpha allowance: 20–30 base / 24–36 with 20% contingency engineer-weeks.** This is part of the combined budget. Minimum A covers finalized payload ingestion and durable scan input; alpha B covers real matching, protected persistence, tenant/lifecycle controls and safe supported restart/restore; private C covers registration, status, paginated polling and one consumer. Expanded B covers broader recovery/retention workload evidence, operational automation, migrations/rotation tooling and additional deployment-profile validation. None of these deferrals permits alpha to skip key protection, revocation or crash-safe coverage.

**B requires an estimated 16–24 engineer-weeks** for private data definition, scanning and secure application/storage integration around the supplied TEE. C requires 8–12 for a focused API consuming A/B contracts. B has lower estimate confidence because its data model, persistence design and platform integration require validation in the feasibility evaluation. New cryptography, TEE platform construction, custom client attestation UX and full access-pattern hiding are outside this scope.

| Work | A | B | C |
|---|---:|---:|---:|
| Target/fixtures, node integration and supported WASM ledger replay | 6–9 | — | — |
| Persisted state, publication, retention and recovery | 8–12 | — | — |
| Core transaction/unshielded/selected-contract projections and public nullifier/commitment records | 5–8 | — | — |
| Internal reads and durable changes | 2–3 | — | — |
| A verification (Lean/property/PostgreSQL faults), deployment and limited PGlite support | 3–4 | — | — |
| Private data requirements, field classification and threat model | — | 3–4 | — |
| Historical/live matching, identity and scan coverage | — | 3–5 | — |
| Protected persistence, TEE boundary and attested key-release integration | — | 4–6 | — |
| Tenant/key lifecycle, backup/restore, revocation and rotation | — | 3–5 | — |
| Security/recovery integration evidence and scan/storage benchmarks | — | 3–4 | — |
| Focused public/private query interface | — | — | 3–4 |
| One resumable change-delivery protocol | — | — | 1–2 |
| Protocol credential integration, admission and resource controls | — | — | 2–3 |
| Selected-consumer and reconnect/recovery integration | — | — | 2–3 |
| **Base engineer-weeks** | **24–36** | **16–24** | **8–12** |

**A+B+C total: 48–72 engineer-weeks; 58–87 with a 20% contingency applied to that total. D is separately scoped and excluded.** The contingency gives 57.6–86.4, rounded up at each end to whole engineer-weeks; no per-project rounding is used. The initial 4–6 engineer-week feasibility evaluation is included in these rows, not an additional charge. Staffing and serial dependencies determine calendar delivery; dividing by headcount is not a schedule.

The combined work table allocates scope by responsibility; it does not require completing A before B or private C. Alpha/expanded columns split those same activities without double counting. The alpha range depends on validating direct node-payload matching; a requirement for fully applied wallet outcomes or missing match inputs changes that range.

The estimate covers selected projections/state views, including the public nullifier/commitment records, one native client contract and client resynchronization. Complete wallet synchronization, provider-cursor migration and broad reference compatibility require separate scope and estimates. Engineering effort is a planning judgment; the design simulation provides no productivity measurement.

External auditor effort, infrastructure, provider charges and ongoing maintenance are excluded. Contingency may cover ordinary fixes; major audit findings or a failed integration assumption can require revising the range.

## Evidence that justifies proceeding

The **4–6 engineer-week feasibility evaluation**, included in the alpha’s 20–30 base effort, tests the direct private-discovery path through a small real integration and benchmark. It covers private data/security decisions and protected persistence. Commitment to the alpha budget depends on the following evidence:

- **Node/matching fit:** a supported finalized node fixture yields versioned transaction offers, their nullifiers and commitments, and the expected positive/negative WASM viewing-key matches, including fallible offers. No official indexer supplies production inputs. Durable observation publication and retry preserve identities and declared source coverage. Missing payloads or required exports must be explicit before accepting the alpha estimate.
- **Verification fit:** map the publication/progress/retention invariants to applicable Lean storage laws and implementation tests; exercise a PostgreSQL interruption/retry case. Record unproved model-to-implementation assumptions instead of describing the entire indexer as formally verified.
- **Private-data/security fit:** an initial field inventory and threat model define the trusted boundary and allowed leakage. A small real path exercises protected key intake/release, one private association write/read and restart/restore, including rejection of an unauthorized read and stale revoked state. Record any missing platform/key-store dependency before accepting the B estimate.
- **Private-processing fit:** known viewing-key fixtures produce expected associations in the selected alpha profile. A representative scan/storage benchmark reports throughput, backfill cost and memory within a declared bounded workload. TEE isolation, authorized results and the selected profile’s key-release behavior form part of the acceptance evidence when that profile is used.
- **Consumer fit:** one application registers a key, reads matches and coverage, resumes polling and revokes access through private C. Its requirements fit observation-based relevance with unknown applied outcomes and no public/full-wallet endpoints.

Expanded-release investment has a separate readiness check within its own allowance: real WASM replay/root verification, durable queryable state and applied projections, PostgreSQL recovery, PGlite compatibility and broader deployment/load validation. Those checks do not gate an alpha consumer that fits the private-discovery contract.

Set numerical workload/latency/retention targets before evaluating the benchmark. Scan work grows roughly with registered keys × candidate ciphertexts; small result tables do not make scanning cheap. Proceed when the evidence supports the intended workload and integration budget. A failed criterion triggers scope adjustment, re-estimation or evaluation of upstream component reuse. Synchronization performance and operating cost require measurements under equivalent workloads.

## Security review and code size

**Review assumption: unmodified shared libraries, including midnight-ledger, are secure and outside re-audit.** This is an explicit trust assumption for the proposal, not a security certification. The review covers UmbraDB indexer-specific code, configuration and the way it calls those libraries. Locally changed ledger/WASM bindings, copied-and-modified shared code, serialization/key handling, authorization and durability integration are in scope. Shared project-specific A/B/C helpers are reviewed once; being shared inside the service does not exclude new code.

| Project-specific surface | Required review |
|---|---|
| A — Ingestion and publication | Untrusted node-input handling, version/identity mapping, atomic visibility, progress, writer fencing and retention/recovery |
| B — Private processing | Viewing-key custody, tenant-scoped mappings, protected database/indexes, logs/backups, pause/revoke/restore and TEE key-release configuration |
| C — Private/public API | Authentication and authorization integration, private result isolation, queries/cursors, admission/resource limits and reconnect behavior |
| Integration and deployment | Calls into trusted ledger/storage libraries, WASM boundary handling, local modifications, secret transport, privileged configuration and upgrade/restore procedures |

Unmodified upstream ledger cryptography, proof verification and shared-library internals do not consume this review scope. Their pinned versions and caller assumptions are part of the review contract. TEE hardware construction is excluded; project-specific TEE configuration, key policy and application boundary are included.

For the combined A/B/C alpha and expanded scope, excluding D, the **directional, low-confidence implementation footprint** is **24–39k application lines**, plus **2–4k operations/configuration lines**, or **26–43k lines** including retained Umbra code. This is the delivered code footprint, not the direct-review count after shared-library exclusions:

| Area | Estimated application lines | Review focus |
|---|---:|---|
| A | 14–22k | Input/ledger boundary, execution identities, SQL/publication, retained state, fencing and restore |
| B | 6–10k | Private data contract, trusted intake/key release, database/index protection, tenant separation, coverage, revocation, rotation and backup rollback |
| C | 4–7k | Authorization, protected intake/results, queries, streams, quotas, cancellation and reconnect |
| **Application total** | **24–39k** | Shared code counted once; includes roughly 6–8k retained Umbra lines within A |

Counts mean nonblank hand-maintained source lines, including comments, SQL and interface declarations. The footprint includes reused/adapted application code, the limited PGlite adapter and deployment-profile configuration; tests, fixtures, generated bindings and external libraries such as midnight-ledger are excluded from the count. Excluding the estimated **6–8k retained Umbra lines**, if all are unchanged and accepted as shared-library code, gives a provisional direct-review range of **16–33k application lines plus 2–4k configuration lines: 18–37k lines**. This uses interval subtraction (24−8 to 39−6); changed shared code is included rather than subtracted. A file manifest must identify the actual exclusions and local changes before an audit quote.

The external review is staged by delivered surface. Before public alpha availability, it covers project-specific minimum A, B, private C and the chosen deployment/restore path under the shared-library trust assumption. The directional alpha implementation footprint is **12–20k application lines plus 1–2k configuration lines**, or **13–22k lines**, included within the combined footprint. Its direct-review subset excludes unchanged shared code actually used by alpha; the full-service 6–8k exclusion cannot be subtracted from alpha without identifying that overlap. These are code-size judgments, not auditor effort estimates.

Expanded-release review covers project-specific replay/state/projection/public API code, changes to the alpha integration and any enabled D extensions. The review includes threat/field inventories, database migrations, key policies, trusted request/response paths, logs/backups and privileged configuration. Unmodified shared-library internals are excluded from both stages. Auditor effort is separate from the engineering estimate. An external security audit is a release requirement outside the available research evidence.

## Evidence and limitations

| Reviewed source | Pinned commit |
|---|---|
| [UmbraDB storage/library baseline](https://github.com/acedward/UmbraDB/tree/3c0c68b3d0397ee2e8344b77e9ed715132fef6ca) | `3c0c68b3d0397ee2e8344b77e9ed715132fef6ca` |
| [UmbraDB node-ingest experiment](https://github.com/acedward/UmbraDB/tree/a562f9cdde3bc8982f336f30b16b988c6e2a2b8c) | `a562f9cdde3bc8982f336f30b16b988c6e2a2b8c` |
| [Midnight Indexer 4.4.0-rc.3](https://github.com/midnightntwrk/midnight-indexer/tree/56561b2f5cf5c6839f678257fc69bed1a8b9ba2c) | `56561b2f5cf5c6839f678257fc69bed1a8b9ba2c` |
| [Midnight wallet SDK](https://github.com/midnightntwrk/midnight-wallet/tree/66dd2b1d4962400d422a97854628da928a137758) | `66dd2b1d4962400d422a97854628da928a137758` |
| [Midnight ledger / WASM bindings](https://github.com/midnightntwrk/midnight-ledger/tree/4823b5351b17cc49e30f19760dbd30a73cf95e22) | `4823b5351b17cc49e30f19760dbd30a73cf95e22` |

The available evidence consists of source inspection, a ledger v8 archive/replay prototype and a 15-scenario in-memory design simulation. The simulation executes a synthetic model without production Umbra adapters, ledger code, PostgreSQL or cryptography. Its passing assertions support the design discussion. The stateless alpha-matching path is a source-supported design inference; real node/WASM/DB/API integration, TEE isolation and workload evidence are required. Expanded 2.x replay correctness and broader client compatibility require their own integration evidence. Both releases are proposed implementations.
