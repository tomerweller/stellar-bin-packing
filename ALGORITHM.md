# Stellar Bin Packing Algorithm

Complete pseudocode derived from the stellar-core implementation (`ParallelTxSetBuilder.cpp`), with comparison notes against CAP-0063, the design PDF, and the stellar-spec.

---

## Table of Contents

0. [Semantics and Glossary](#0-semantics-and-glossary)
1. [Requirements](#1-requirements)
2. [Data Structures](#2-data-structures)
3. [Phase 1: Conflict Detection](#3-phase-1-conflict-detection)
4. [Phase 2: Fee-Ordered Greedy Packing](#4-phase-2-fee-ordered-greedy-packing)
5. [Phase 3: Adding a Transaction to a Stage](#5-phase-3-adding-a-transaction-to-a-stage)
6. [Phase 4: Bin Packing](#6-phase-4-bin-packing)
7. [Phase 5: Multi-Stage-Count Selection](#7-phase-5-multi-stage-count-selection)
8. [Phase 6: Output Assembly](#8-phase-6-output-assembly)


---

## 0. Semantics and Glossary

### The Problem

Stellar closes a new ledger every ~5 seconds. Each ledger includes a **transaction set** — the batch of transactions validators agree to execute. Soroban (smart contract) transactions declare upfront which ledger entries they will read and write (their **footprint**). This gives validators enough information to figure out which transactions can safely run at the same time and which cannot.

The bin packing algorithm's job is: given a pool of candidate Soroban transactions, produce a transaction set that (a) maximizes fee revenue, (b) can be executed in parallel across multiple CPU cores, and (c) has a bounded worst-case execution time.

### Conceptual Model

The output of the algorithm is a **parallel Soroban phase** with the following hierarchy:

```
Phase (Soroban)
 └─ Stage 1                          ← stages execute sequentially
     ├─ Bin 1 (DependentTxCluster)   ← bins execute in parallel (one per CPU core)
     │   ├─ tx_a ─ tx_b ─ tx_c      ← transactions within a bin execute sequentially
     │   └─ ...
     ├─ Bin 2 (DependentTxCluster)
     │   └─ ...
     └─ ...
 └─ Stage 2
     └─ ...
```

- **Stages** are sequential barriers. Stage 2 cannot begin until stage 1 finishes. Having multiple stages lets the algorithm work around conflicts — e.g., an oracle price update can go in stage 1, while all the transactions that *read* that price can be parallelized across bins in stage 2.
- **Bins** are the parallel execution units. Each bin runs on a separate CPU core. Within a bin, no two transactions from *different* bins may touch the same mutable ledger entry. The number of bins per stage is capped by the network setting `ledgerMaxDependentTxClusters` (e.g., 8).
- **Transactions** within a bin execute sequentially because they may have data dependencies between each other.

The total "wall-clock" cost of the phase (in modeled instructions) is:

```
total_cost = sum over all stages of: max(bin_instructions in that stage)
```

This must not exceed `ledgerMaxInstructions`.

### Glossary

| Term | Definition |
|------|------------|
| **Transaction Set** | The set of transactions included in a single ledger close. Organized into phases: Phase 0 (classic) and Phase 1 (Soroban). |
| **Phase** | A partition of the transaction set by type. This document concerns Phase 1 (Soroban), which uses the parallel structure. |
| **Stage** | A sequential step in the execution schedule. All bins in a stage run in parallel, but stage N+1 waits for stage N to complete. The algorithm tries 1–4 stages. |
| **Bin** | A group of (possibly dependent) transactions assigned to one CPU core. Called `DependentTxCluster` in the XDR wire format, "logical thread" in the design PDF, and simply "cluster" in CAP-0063 and the spec. Limited to `ledgerMaxDependentTxClusters` per stage. |
| **Cluster** | (Internal to the algorithm) A set of transitively dependent transactions. Clusters are the fine-grained dependency groups that get *packed into* bins. Multiple independent clusters may share a bin. Not visible in the wire format. |
| **I/O Conflict** | Two transactions conflict if they access the same ledger key and at least one of them writes it. Two types: **RW-RW** (both write the key) and **RO-RW** (one reads, the other writes). Two transactions that both only read the same key do **not** conflict. |
| **Footprint** | The declared set of ledger keys a Soroban transaction will access, split into a read-only set and a read-write set. Declared before execution; enforced by the runtime. |
| **Inclusion Fee** | The portion of a transaction's fee bid that participates in prioritization and surge pricing. Transactions are packed in descending inclusion fee order. |
| **Surge Pricing** | When more transactions compete for space than the ledger can hold, the lowest-fee included transaction's fee becomes the base fee. Triggered when any transaction is skipped during packing. |
| **Bin Packing** | The sub-problem of fitting variable-sized clusters into a fixed number of equal-capacity bins (one per core). The algorithm uses first-fit (fast path) and first-fit-decreasing (FFD, 11/9 approximation ratio) heuristics. |
| **`ledgerMaxDependentTxClusters`** | Network configuration setting. Maximum number of bins (parallel execution units) per stage. Sets the minimum hardware parallelism the network assumes (e.g., 8 cores). |
| **`ledgerMaxInstructions`** | Network configuration setting. Total instruction budget for the Soroban phase, measured as `sum(max(bin_instructions) per stage)`. |
| **`SOROBAN_PHASE_MIN/MAX_STAGE_COUNT`** | Node-local configuration. Range of stage counts to try during construction (default 1–4). Not a protocol parameter — different stage counts produce different valid transaction sets. |

---

## 1. Requirements

The algorithm must produce a valid parallel transaction set satisfying all of the following:

### R1. Conflict Safety

No two transactions in **different bins within the same stage** may access the same ledger key where at least one access is a write. Formally: if tx A is in bin X and tx B is in bin Y (X ≠ Y) of the same stage, then `footprint(A).readWrite ∩ footprint(B).readWrite = ∅` and `footprint(A).readWrite ∩ footprint(B).readOnly = ∅` (and vice versa). Transactions within the *same* bin may conflict freely — they execute sequentially.

### R2. Bounded Execution Time

The total wall-clock cost of the phase must not exceed the network's instruction budget:

```
sum over all stages of: max(bin_instructions in that stage) ≤ ledgerMaxInstructions
```

This is the "makespan" — the longest bin in each stage determines that stage's cost, and stages run sequentially. The per-bin limit is derived as `ledgerMaxInstructions / stage_count`.

### R3. Fee Revenue Maximization

Transactions are processed in **descending inclusion fee order**. The algorithm greedily includes the highest-paying transactions first. When multiple stage counts are tried, the result with the **fewest stages** that captures at least **99.9%** of the maximum total fee across all candidates is selected — preferring simplicity over marginal revenue.

### R4. Parallelism Budget

Each stage has at most `ledgerMaxDependentTxClusters` bins (one per CPU core). This is a network-wide parameter that defines the minimum hardware parallelism validators must support.

### R5. Resource Limits

Beyond instructions, each transaction consumes non-instruction resources (read bytes, write bytes, transaction size). These are checked against a **single shared lane limit** summed across the entire phase. A transaction that exceeds remaining lane capacity is skipped regardless of whether it would fit in a stage's instruction budget.

### R6. Surge Pricing

If any transaction is skipped during packing (either because it exceeds lane limits or because no stage can fit it), **surge pricing** is triggered. The lowest inclusion fee among all included transactions becomes the base fee for the entire transaction set.

### R7. Deterministic Validation

Different validators may construct different transaction sets (e.g., by trying different stage counts or having different mempools). However, any valid parallel transaction set must pass the **same validation rules**: conflict safety (R1), instruction bounds (R2), parallelism limits (R4), and resource limits (R5). Construction is a local heuristic; validation is a protocol invariant.

---

## 2. Data Structures

```
struct ParallelPartitionConfig:
    stage_count          : uint32     # Number of sequential stages (1..4)
    clusters_per_stage   : uint32     # = ledgerMaxDependentTxClusters (network config)
    insns_per_cluster    : uint64     # = ledgerMaxInstructions / stage_count

    insns_per_stage():
        return insns_per_cluster * clusters_per_stage

struct BuilderTx:
    id            : uint             # Index into the original tx list
    instructions  : uint32           # sorobanResources().instructions
    conflict_txs  : BitSet           # Set of tx ids that conflict with this tx

struct Cluster:
    instructions  : uint64           # Total instructions (sequential sum)
    conflict_txs  : BitSet           # Union of all member tx conflict sets
    tx_ids        : BitSet           # Set of tx ids in this cluster
    bin_id        : optional<uint>   # Which bin this cluster is packed into

struct Stage:
    clusters             : list<Cluster>
    bin_instructions     : array[clusters_per_stage] of uint64   # Per-bin totals
    total_instructions   : uint64                                # Sum across all clusters
    all_conflict_txs     : BitSet                                # Union of all cluster conflicts
    tx_to_cluster        : array[N] of Cluster*                  # Tx id -> owning cluster
    tried_compacting     : bool = false                          # Optimization flag
```

---

## 3. Phase 1: Conflict Detection

Before any packing happens, we need to know which transactions conflict with each other. This phase scans every transaction's declared footprint (read-only and read-write keys), groups entries by key hash, and marks pairs of transactions as conflicting if they share a key where at least one side is writing. The output is a lightweight `BuilderTx` per transaction, each carrying a BitSet of its conflict partners. This conflict information drives all downstream placement decisions.

**Source:** `prepareBuilderTxs()` (lines 567-699)

```
function detect_conflicts(tx_frames) -> list<BuilderTx>:
    # Create lightweight BuilderTx for each transaction
    builder_txs = []
    for i, tx in enumerate(tx_frames):
        builder_txs.append(BuilderTx(id=i, instructions=tx.sorobanResources().instructions))

    # Collect all footprint entries into a flat vector
    fp_entries = []
    for i, tx in enumerate(tx_frames):
        footprint = tx.sorobanResources().footprint
        for key in footprint.readOnly:
            fp_entries.append((hash(key), tx_id=i, is_rw=false))
        for key in footprint.readWrite:
            fp_entries.append((hash(key), tx_id=i, is_rw=true))

    # Sort by key hash for cache-friendly grouping
    sort fp_entries by key_hash ascending

    # Scan for groups sharing the same key hash
    group_start = 0
    while group_start < len(fp_entries):
        group_end = group_start + 1
        while group_end < len(fp_entries)
              and fp_entries[group_end].key_hash == fp_entries[group_start].key_hash:
            group_end++

        # Skip singletons — no possible conflicts
        if group_end - group_start < 2:
            group_start = group_end
            continue

        # Partition group into RO and RW sets
        ro_txs = [e.tx_id for e in fp_entries[group_start..group_end] if not e.is_rw]
        rw_txs = [e.tx_id for e in fp_entries[group_start..group_end] if e.is_rw]

        # Mark RW-RW conflicts (bidirectional)
        for i in 0..len(rw_txs):
            for j in (i+1)..len(rw_txs):
                if rw_txs[i] != rw_txs[j]:                    # Skip self (hash collision within tx)
                    builder_txs[rw_txs[i]].conflict_txs.set(rw_txs[j])
                    builder_txs[rw_txs[j]].conflict_txs.set(rw_txs[i])

        # Mark RO-RW conflicts (bidirectional)
        for i in 0..len(ro_txs):
            for j in 0..len(rw_txs):
                if ro_txs[i] != rw_txs[j]:                    # Skip self
                    builder_txs[ro_txs[i]].conflict_txs.set(rw_txs[j])
                    builder_txs[rw_txs[j]].conflict_txs.set(ro_txs[i])

        group_start = group_end

    return builder_txs
```

**Key design choice:** Conflicts are detected via 64-bit key hashes, not exact key comparison. Hash collisions (probability ~K^2/2^64) are conservatively treated as real conflicts. This is safe (may slightly over-constrain parallelism) and avoids expensive key equality checks.

**Note:** RO-RO overlaps are *not* conflicts. Two transactions that both only read the same key can safely execute in parallel.

---

## 4. Phase 2: Fee-Ordered Greedy Packing

This is the main loop. Transactions are sorted by descending inclusion fee (highest bidder first) and processed one at a time. For each transaction, we first check whether it fits within the remaining non-instruction resource budget (read bytes, write bytes, tx size). If it does, we try to place it into the first stage that can accommodate it. If no stage can fit it, the transaction is skipped and the `had_tx_not_fitting_lane` flag is set, which triggers surge pricing for the resulting transaction set.

**Source:** `buildSurgePricedParallelSorobanPhaseWithStageCount()` (lines 457-565)

```
function pack_with_stage_count(sorted_tx_order, tx_resources, lane_limit,
                                builder_txs, tx_frames,
                                stage_count, soroban_cfg) -> BuildResult:

    config = ParallelPartitionConfig(stage_count, soroban_cfg)
    # config.insns_per_cluster = ledgerMaxInstructions / stage_count
    # config.clusters_per_stage = ledgerMaxDependentTxClusters

    # Initialize empty stages
    stages = [Stage(config, len(tx_frames)) for _ in 0..stage_count]

    lane_left = lane_limit          # Remaining non-instruction resource budget
    had_tx_not_fitting_lane = false  # Triggers surge pricing if true

    # Iterate transactions in DESCENDING inclusion fee order
    for tx_idx in sorted_tx_order:
        tx_res = tx_resources[tx_idx]

        # Check non-instruction resource limits (read bytes, write bytes, tx size, etc.)
        if any_dimension_exceeds(tx_res, lane_left):
            had_tx_not_fitting_lane = true
            continue

        # Try to add to the FIRST stage that can accommodate it
        added = false
        for stage in stages:
            if stage.try_add(builder_txs[tx_idx]):
                added = true
                break

        if added:
            lane_left -= tx_res        # Deduct non-instruction resources
        else:
            had_tx_not_fitting_lane = true   # Tx skipped → surge pricing triggered

    # Assemble output: map (stage, bin_id, tx_id) → TxStageFrameList
    result = assemble_stages(stages, tx_frames)   # See Phase 6
    result.had_tx_not_fitting_lane = [had_tx_not_fitting_lane]
    result.total_inclusion_fee = sum of inclusion fees of all included txs

    # Remove trailing empty stages
    while result.stages is not empty and result.stages.last is empty:
        result.stages.pop_last()

    return result
```

**Note on fee sorting:** Ties in inclusion fee are broken by a random tiebreaker (seeded with `rand_uniform`), ensuring no deterministic bias for equal-fee transactions.

**Note on lane resources:** For parallel Soroban phases, the instruction dimension is excluded from the lane resource check (set to max value). Instructions are instead governed by the per-stage/per-cluster limits within `try_add`. All other resource dimensions (read bytes, write bytes, tx size) are checked against a single shared lane limit summed across the entire phase.

---

## 5. Phase 3: Adding a Transaction to a Stage

This is where the core constraint logic lives. When a transaction is offered to a stage, we first fast-fail if the stage's total instructions are already at capacity. Then we find every existing cluster that conflicts with the new transaction, merge them all (plus the new tx) into a single cluster, and check whether the merged cluster's instructions exceed the per-bin limit. If it fits, we try to pack it into a bin — first with a cheap in-place first-fit, and if that fails, with a full first-fit-decreasing rebuild. If neither works, we roll back and the transaction is rejected from this stage.

**Source:** `Stage::tryAdd()` (lines 111-218)

```
function Stage.try_add(tx: BuilderTx) -> bool:

    # ── Fast-fail: would total stage instructions exceed the theoretical maximum? ──
    if self.total_instructions + tx.instructions > config.insns_per_stage():
        return false

    # ── Find all clusters that conflict with the new transaction ──
    conflicting_clusters = self.get_conflicting_clusters(tx)

    # ── Check if the merged cluster would exceed per-cluster instruction limit ──
    merged_insns = tx.instructions
    for cluster in conflicting_clusters:
        merged_insns += cluster.instructions
    if merged_insns > config.insns_per_cluster:
        return false

    # ── Create merged cluster ──
    new_cluster = Cluster(tx)
    for cluster in conflicting_clusters:
        new_cluster.merge(cluster)

    # ── Mutate stage: remove old conflicting clusters, append merged cluster ──
    saved_clusters = remove_conflicting_clusters(conflicting_clusters)
    self.clusters.append(new_cluster)

    # ── Attempt in-place bin packing (first-fit, unordered) ──
    if self.in_place_bin_packing(new_cluster, conflicting_clusters):
        self.total_instructions += tx.instructions
        self.all_conflict_txs |= tx.conflict_txs
        self.update_tx_to_cluster(new_cluster)
        return true

    # ── Optimization: skip full rebuild for independent txs after first failure ──
    if conflicting_clusters is empty:
        if self.tried_compacting:
            self.rollback(new_cluster, saved_clusters)
            return false
        self.tried_compacting = true

    # ── Full rebuild: first-fit-decreasing (FFD) bin packing ──
    new_bin_instructions = []
    if not self.ffd_bin_packing(self.clusters, new_bin_instructions):
        self.rollback(new_cluster, saved_clusters)
        return false

    self.total_instructions += tx.instructions
    self.bin_instructions = new_bin_instructions
    self.all_conflict_txs |= tx.conflict_txs
    self.update_tx_to_cluster(new_cluster)
    return true
```

### 5.1 Finding Conflicting Clusters

```
function Stage.get_conflicting_clusters(tx: BuilderTx) -> set<Cluster*>:
    # Fast path: if tx has no conflicts with ANY cluster in stage, return empty
    if not self.all_conflict_txs.get(tx.id):
        return {}

    # O(K) lookup where K = number of tx's conflicts
    result = {}
    for conflict_tx_id in tx.conflict_txs:
        cluster = self.tx_to_cluster[conflict_tx_id]
        if cluster is not null:
            result.insert(cluster)
    return result
```

### 5.2 Cluster Merging

```
function Cluster.merge(other: Cluster):
    self.instructions += other.instructions
    self.conflict_txs |= other.conflict_txs
    self.tx_ids |= other.tx_ids
```

This creates a **transitive closure**: if tx A conflicts with tx B and tx B conflicts with tx C, they all end up in the same cluster even if A and C don't directly conflict.

---

## 6. Phase 4: Bin Packing

A stage may accumulate many fine-grained clusters (each a transitive dependency group), but the network only allows a fixed number of parallel execution units (bins) per stage. This phase solves the classic bin packing problem: fit all clusters into N equal-capacity bins, where N = `ledgerMaxDependentTxClusters` and each bin's capacity = `ledgerMaxInstructions / stage_count`. Two heuristics are used — a fast in-place first-fit for the common case, and a full first-fit-decreasing (FFD) rebuild as a fallback when the fast path can't find room.

### 6.1 In-Place First-Fit (Fast Path)

**Source:** `inPlaceBinPacking()` (lines 277-303)

```
function Stage.in_place_bin_packing(new_cluster, removed_clusters) -> bool:
    # Free up space from clusters that were merged away
    for cluster in removed_clusters:
        self.bin_instructions[cluster.bin_id] -= cluster.instructions

    # Try to fit new_cluster into the first bin with enough room
    for bin_id in 0..clusters_per_stage:
        if self.bin_instructions[bin_id] + new_cluster.instructions <= insns_per_cluster:
            self.bin_instructions[bin_id] += new_cluster.instructions
            new_cluster.bin_id = bin_id
            return true

    # Revert freed space if we couldn't fit
    for cluster in removed_clusters:
        self.bin_instructions[cluster.bin_id] += cluster.instructions
    return false
```

**Approximation ratio:** ~1.7x (unordered first-fit).

### 6.2 Full Rebuild: First-Fit-Decreasing (Fallback)

**Source:** `binPacking()` (lines 358-399)

```
function Stage.ffd_bin_packing(clusters, out bin_instructions) -> bool:
    # Sort clusters by instructions DESCENDING
    sort clusters by instructions descending

    bin_instructions = array[clusters_per_stage] initialized to 0

    for i, cluster in enumerate(clusters):
        packed = false
        for bin_id in 0..clusters_per_stage:
            if bin_instructions[bin_id] + cluster.instructions <= insns_per_cluster:
                bin_instructions[bin_id] += cluster.instructions
                cluster.bin_id = bin_id
                packed = true
                break
        if not packed:
            return false

    return true
```

**Approximation ratio:** 11/9 (~1.22x) — the classic FFD bin packing guarantee.

### 6.3 The Compacting Optimization

After the first time an independent (no-conflict) transaction fails in-place packing and also fails the full FFD rebuild, the stage sets `tried_compacting = true`. All subsequent independent transactions that fail in-place packing are immediately rejected without attempting a full rebuild.

**Rationale:** Once FFD has been run and the stage is near capacity, re-running FFD for each of the remaining (possibly many) independent transactions is wasteful — the packing is already as compact as the heuristic can achieve. This prevents O(N) full rebuilds when the stage is saturated with independent transactions.

This optimization is **not** applied to transactions that have conflicts, because the cluster merge changes the packing problem fundamentally each time.

---

## 7. Phase 5: Multi-Stage-Count Selection

The algorithm doesn't know upfront how many stages will yield the best result. More stages help work around I/O conflicts (e.g., separating a writer from its readers) but each stage has a smaller per-bin instruction budget (`ledgerMaxInstructions / stage_count`), which can waste capacity if transactions are large. So the algorithm runs Phase 2–4 in parallel threads for each candidate stage count (default 1 through 4) and picks the winner: the result with the **fewest stages** that still captures at least 99.9% of the maximum total inclusion fee across all candidates.

**Source:** `buildSurgePricedParallelSorobanPhase()` (lines 703-801)

```
function build_parallel_soroban_phase(tx_frames, cfg, soroban_cfg,
                                       lane_config, ledger_version)
                                       -> TxStageFrameList:

    TOLERANCE = 0.999   # Accept ≥99.9% of max fee revenue

    # Shared read-only state
    builder_txs = detect_conflicts(tx_frames)
    sorted_tx_order = argsort(tx_frames, by=inclusion_fee descending,
                              tiebreak=random)
    tx_resources = [tx.getResources() for tx in tx_frames]
    lane_limit = lane_config.getLaneLimits()[0]     # Single Soroban lane

    # Run packing for each candidate stage count IN PARALLEL THREADS
    results = []
    for stage_count in cfg.SOROBAN_PHASE_MIN_STAGE_COUNT
                       .. cfg.SOROBAN_PHASE_MAX_STAGE_COUNT:
        spawn thread:
            results[stage_count] = pack_with_stage_count(
                sorted_tx_order, tx_resources, lane_limit,
                builder_txs, tx_frames, stage_count, soroban_cfg)

    join all threads

    # Select result: fewest stages while ≥99.9% of maximum total fee
    max_fee = max(r.total_inclusion_fee for r in results)
    fee_threshold = max_fee * TOLERANCE

    best = null
    for result in results:
        if result.total_inclusion_fee < fee_threshold:
            continue
        if best is null or len(result.stages) < len(best.stages):
            best = result

    return best.stages
```

**Key insight:** More stages generally yield better utilization (especially under contention) but add sequential overhead. The algorithm explores all options in parallel and picks the most compact result that doesn't sacrifice significant fee revenue.

---

## 8. Phase 6: Output Assembly

The internal representation (stages containing clusters assigned to bins) is now flattened into the output format. Each bin becomes a list of transaction frames — the fine-grained cluster boundaries are erased. Trailing empty stages are trimmed. The output then goes through XDR serialization, which applies canonical ordering (sort by SHA-256 hash at each level) and attaches the base fee.

**Source:** Lines 519-562

```
function assemble_stages(stages, tx_frames) -> TxStageFrameList:
    result_stages = []

    for stage in stages:
        result_stage = []     # List of clusters (each cluster = list of TxFrame)
        bin_to_cluster_idx = array[clusters_per_stage] initialized to UNSET

        stage.visit_all_transactions(callback(bin_id, tx_id):
            if bin_to_cluster_idx[bin_id] == UNSET:
                bin_to_cluster_idx[bin_id] = len(result_stage)
                result_stage.append([])
            result_stage[bin_to_cluster_idx[bin_id]].append(tx_frames[tx_id])
        )

        result_stages.append(result_stage)

    # Remove trailing empty stages
    while result_stages is not empty and result_stages.last is empty:
        result_stages.pop_last()

    return result_stages
```

After assembly, the output undergoes XDR serialization where:
- Transactions within each cluster are sorted by SHA-256 hash (canonical ordering)
- Clusters within each stage are sorted by hash of first transaction
- Stages are sorted by hash of first transaction of first cluster
- All included transactions share a single base fee (Soroban uses one lane)

