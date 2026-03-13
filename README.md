# Stellar Bin Packing

Educational deep-dive into Stellar's parallel transaction scheduling algorithm — how the network packs Soroban transactions into a multi-stage, multi-core execution plan that maximizes fee revenue while keeping wall-clock time bounded.

## Artifacts

### [ALGORITHM.md](ALGORITHM.md) — Pseudocode & Explanation

Complete pseudocode for the bin packing algorithm, derived from the `stellar-core` implementation (`ParallelTxSetBuilder.cpp`). Covers all six phases:

1. **Conflict Detection** — scan footprints, mark RW-RW and RO-RW conflicts via key hashing
2. **Fee-Ordered Greedy Packing** — iterate transactions by descending inclusion fee, try stages in order
3. **Stage Placement** — find conflicting clusters, merge transitively, check per-bin instruction limits
4. **Bin Packing** — first-fit (fast path) + first-fit-decreasing FFD (fallback, 11/9 approximation)
5. **Multi-Stage-Count Selection** — run 1–4 stages in parallel, pick fewest stages within 99.9% of max fee
6. **Output Assembly** — flatten to wire format with canonical ordering

Includes a glossary, terminology cross-reference across stellar-core / CAP-0063 / design PDF / spec, and annotated data structures.

### [visualization.html](visualization.html) — Interactive Visualization

A single-file, zero-dependency interactive visualization. Open it in any browser.

**Three tabs:**

- **Packing Simulation** — step-by-step replay of the algorithm with three synchronized panels:
  - *Transaction Queue* — fee-sorted list with instruction bars; shows read/write keys for the active tx
  - *Conflict Graph* — circular node layout with RW-RW (red) and RO-RW (orange) edges; nodes colored by cluster assignment with convex hulls around cluster members
  - *Stages & Bins* — grid showing clusters stacking in bins with utilization percentages

- **Bin Packing Comparison** — side-by-side First-Fit vs First-Fit-Decreasing with configurable cluster count, bins, and capacity

- **Stage Comparison** — bar chart comparing fee revenue across 1–4 stage counts on the same transaction set, showing the 99.9% threshold and winner selection

**Presets:** Oracle Update, All Independent, Transitive Chain, Near Capacity, Random

**Controls:** Play/Pause, Step Forward/Back, Speed slider, configurable bins/stages/budget/tx count. Keyboard: Space to play/pause, Arrow keys to step.

## Source Materials

Reference materials used to produce the algorithm writeup (see [MANIFEST.md](MANIFEST.md) for details):

| Directory | Contents |
|-----------|----------|
| `stellar-core-code/` | `ParallelTxSetBuilder.cpp/.h`, `TxSetFrame.cpp/.h` — the implementation |
| `specs/` | Excerpts from `stellar-spec`: herder, transaction, and ledger specs |
| `cap-0063.md` | CAP-0063: Parallelism-friendly Transaction Scheduling |
| `Building tx sets for parallel processing.pdf` | Original design document by Dmytro Kozhevin |

## Quick Start

```bash
open visualization.html
```

Or serve locally:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/visualization.html
```

## License

The source materials (`stellar-core-code/`, `specs/`, `cap-0063.md`, PDF) are from the [Stellar](https://github.com/stellar) project and subject to their respective licenses. The algorithm writeup and visualization are provided for educational purposes.
