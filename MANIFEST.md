# Stellar Bin Packing — Materials Manifest

Educational materials on Stellar's bin packing algorithm for parallel transaction scheduling (Protocol v23+, Soroban).

## Source Materials

### 1. Design Document (PDF)
- **`Building tx sets for parallel processing.pdf`** — Original design doc by Dmytro Kozhevin
  - Terminology (I/O conflicts, conflict clusters)
  - Requirements (fee ordering, bounded runtime, deterministic validation)
  - Apply schedule structure (stages > threads > clusters)
  - Greedy partitioning algorithm (fee-ordered iteration, per-stage placement, FFD bin packing)
  - Evaluation benchmarks (randomized traffic, oracle update, arbitrage scenarios)
  - Key finding: 3-4 stages achieve ~100% utilization in most conflict patterns

### 2. Protocol Specification (CAP)
- **`cap-0063.md`** (440 lines) — [CAP-0063: Parallelism-friendly Transaction Scheduling](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0063.md)
  - Protocol-level definition of stages, clusters, and validation rules
  - New network setting: `ledgerMaxDependentTxClusters`
  - TTL update semantics changes (deferred read-only TTL updates for commutativity)
  - Fee refund timing changes (post-phase instead of per-transaction)
  - Initial rollout: starts at parallelism=1, increased via governance vote

### 3. Stellar Spec (from tomerweller/stellar-spec)
Saved under **`specs/`**:

- **`specs/HERDER_SPEC.md`** (1973 lines) — Primary spec for tx set construction
  - Section 6: Transaction Set Construction (pool selection, phase building, surge pricing)
  - Section 7: Parallel Soroban Transaction Sets (stage/cluster model, footprint conflict rules, instruction budgets, cluster count limits, canonical ordering)
  - Section 8: Transaction Set Validation (parallel phase validation rules)
  - Section 12.6: Surge pricing interaction with parallel phases

- **`specs/TX_SPEC.md`** (3122 lines) — Transaction processing details
  - Section 2.4: Transaction Set Structure (phase/stage/cluster hierarchy)
  - Section 8.7: Parallel Soroban Execution (snapshots, thread-local state, deferred TTL bumps, state merging)

- **`specs/LEDGER_SPEC.md`** (1889 lines) — Ledger close pipeline
  - Section 5.4: Parallel Phase Application (orchestration, snapshots, deterministic commit)
  - Section 5.5: Transaction Application Ordering
  - Section 14: Threading Model
  - Appendix D: Mermaid diagram of parallel phase execution

### 4. stellar-core Implementation
Saved under **`stellar-core-code/`**:

- **`stellar-core-code/ParallelTxSetBuilder.h`** (29 lines) — Header declaring `buildSurgePricedParallelSorobanPhase()`
- **`stellar-core-code/ParallelTxSetBuilder.cpp`** (803 lines) — Core algorithm implementation
  - `prepareBuilderTxs()` — Conflict detection via sorted footprint analysis using BitSet masks
  - `Stage::tryAdd()` — Per-stage transaction placement with cluster merging
  - Bin packing: first-fit (fast path) + first-fit-decreasing (FFD, fallback full rebuild)
  - `buildSurgePricedParallelSorobanPhaseWithStageCount()` — Single stage-count packing
  - `buildSurgePricedParallelSorobanPhase()` — Multi-stage-count parallel search, picks fewest stages within 99.9% of max fee revenue
- **`stellar-core-code/TxSetFrame.h`** (592 lines) — Types: `TxClusterFrame`, `TxStageFrame`, `TxStageFrameList`, `ApplicableTxSetFrame`
- **`stellar-core-code/TxSetFrame.cpp`** (2454 lines) — Orchestration: `applySurgePricing()`, validation, XDR serialization

## Concept Map

```
Mempool Transactions
        │
        ▼
┌─────────────────────────────────────┐
│  1. Sort by descending inclusion fee │
│  2. For each tx, try stages in order │
│     ├─ Check ledger resource limits  │
│     ├─ Find conflicting clusters     │
│     ├─ Merge into single cluster     │
│     └─ Bin-pack clusters → threads   │
│  3. Try stage_count = 1..4 in parallel│
│  4. Pick fewest stages ≥ 99.9% fees  │
└─────────────────────────────────────┘
        │
        ▼
  Transaction Set Phase
  ┌─────────────────────────────┐
  │ Stage 1                     │
  │  Thread 1: [C1] [C2] ...   │
  │  Thread 2: [C1] [C2] ...   │
  │  ...                        │
  │  Thread M: [C1] [C2] ...   │ ← max_threads (e.g. 8)
  ├─────────────────────────────┤
  │ Stage 2                     │
  │  Thread 1: [C1] ...        │
  │  ...                        │
  ├─────────────────────────────┤
  │ ...                         │
  └─────────────────────────────┘
   Sum of max(thread_instructions)
   across stages ≤ ledgerMaxInstructions
```

## Key Parameters

| Parameter | Source | Description |
|-----------|--------|-------------|
| `ledgerMaxDependentTxClusters` | Network config | Max threads/bins per stage (e.g. 8) |
| `ledgerMaxInstructions` | Network config | Total instruction budget for the ledger |
| `SOROBAN_PHASE_MIN_STAGE_COUNT` | Node config | Min stages to try (default 1) |
| `SOROBAN_PHASE_MAX_STAGE_COUNT` | Node config | Max stages to try (default 4) |
