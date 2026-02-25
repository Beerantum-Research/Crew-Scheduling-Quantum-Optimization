# BVG Crew Scheduling — Quantum-Enhanced Optimization Pipeline

## Architecture Overview

The pipeline solves a 30-day crew scheduling problem for BVG Berlin bus line M29 by compressing 30 independent optimization days into ~4 representative archetype days using unsupervised learning, solving each archetype with a quantum-classical hybrid optimizer, and replicating solutions across the full month via a segment-matching adapter.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FULL PIPELINE FLOW                          │
│                                                                     │
│  30 days of raw data                                                │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────┐                                                   │
│  │  STAGE A     │  DBSCAN Day Clustering                            │
│  │  ML Layer    │  16 features → ~4 archetypes                      │
│  └──────┬───────┘                                                   │
│         │  4 representative days                                    │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  STAGE B     │  3-Level Hierarchical Decomposition               │
│  │  Decompose   │  Temporal → Spatial → Meta-Route                  │
│  └──────┬───────┘                                                   │
│         │  ~8 clusters per archetype, each ≤25 segments             │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  STAGE C     │  QUBO Encoding + Compression + Quantum Solve      │
│  │  Quantum     │  TN(10) → Krylov(8) → BF-DCQO on hardware        │
│  └──────┬───────┘                                                   │
│         │  Optimized driver-segment assignments per archetype       │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  STAGE D     │  Archetype → 30-Day Adapter + Post-Processing     │
│  │  Replicate   │  Template matching + coverage repair + overlap    │
│  └──────────────┘   resolution + Legalizer validation               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Stage A — DBSCAN Day Clustering

### Objective

Reduce 30 independent quantum optimization problems to ~4 by identifying days with statistically similar operational profiles and solving only one representative per cluster.

### Feature Engineering

Each of the 30 calendar days is characterized by a 16-dimensional feature vector extracted from the raw `Block Segment` Excel sheet:

| # | Feature | Description |
|---|---------|-------------|
| 1 | `total_routes` | Number of route segments operating on this day |
| 2 | `total_blocks` | Number of distinct vehicle blocks (shift containers) |
| 3 | `routes_per_block` | Average segment density per block |
| 4 | `earliest_start` | First departure of the day (minutes from midnight) |
| 5 | `latest_end` | Last arrival of the day (minutes from midnight) |
| 6 | `total_span` | Operating window duration (latest_end − earliest_start) |
| 7 | `mean_duration` | Average segment duration in minutes |
| 8 | `std_duration` | Duration variance — captures heterogeneity of segments |
| 9 | `m29_count` | Number of M29 line segments |
| 10 | `m41_count` | Number of M41 line segments |
| 11 | `m29_m41_ratio` | Line balance ratio |
| 12 | `from_depot_count` | Segments originating from Betriebshof (depot) |
| 13 | `to_depot_count` | Segments terminating at Betriebshof |
| 14 | `morning_peak` | Segments starting 07:00–10:00 (rush hour density) |
| 15 | `evening_peak` | Segments starting 17:00–20:00 (rush hour density) |
| 16 | `is_weekend` | Binary: Samstag/Sonntag indicator |

An additional `weekday_num` feature encodes the day-of-week ordinally (Montag=0, ..., Sonntag=6).

### Clustering Algorithm

1. **StandardScaler** normalizes all 16 features to zero mean, unit variance.
2. **DBSCAN** with epsilon sweep `[0.5, 1.0, 1.5, 2.0, 2.5]`, `min_samples=2`.
3. For each epsilon, the **silhouette score** is computed; the configuration with the highest silhouette and fewer than 5 noise points is selected.
4. **Noise reassignment**: any day labeled as noise (cluster = −1) is reassigned to the nearest valid cluster centroid in the scaled feature space via Euclidean distance.

### Representative Selection

For each cluster, the **representative day** is the one closest (L2 norm) to the cluster centroid in the 16-dimensional scaled space. This day's schedule is solved quantum-mechanically; all other days in the cluster inherit its solution through the Stage D adapter.

### Quantitative Impact

With ~4 archetypes, the pipeline reduces quantum API calls from 30×(clusters/day) to 4×(clusters/day) — a **~87% reduction** in hardware utilization.

---

## Stage B — 3-Level Hierarchical Decomposition

Each archetype's representative day contains ~200 route segments that must be assigned to drivers. Solving a single 200-segment QUBO directly would require ~600 binary variables (200 segments × 3 drivers), far beyond current hardware capacity. The decomposition strategy breaks this into tractable subproblems.

### Level 1: Temporal Decomposition

Segments are sorted by `start_time` and partitioned into non-overlapping time windows of configurable width (default: 4 hours). This exploits the temporal locality of crew scheduling — a driver's assignment at 06:00 has no interaction with a 22:00 assignment.

The algorithm achieves O(n log n + n + w) complexity via pre-sorting, where w is the number of windows.

### Level 2: Spatial Clustering

Within each time window, segments are clustered by geographic and operational features using **Agglomerative Clustering** (Ward linkage) on a 4-dimensional feature vector per segment:

- `from_location` (encoded as integer index)
- `to_location` (encoded as integer index)
- `start_time` (minutes)
- `duration_min`

Features are z-normalized before clustering. The target is `MAX_CLUSTER_SIZE` segments per cluster (default: 25). Two additional decomposition strategies are available and selected adaptively:

- **Topological decompose**: greedy time-based splitting with a 2-hour gap threshold for natural break detection.
- **Hierarchical decompose**: two-pass coarse-to-fine clustering (coarse_size=50, fine_size=15).

### Level 3: Meta-Route Aggregation

Within each spatial cluster, segments sharing the same route signature `(bus_line, from_location, to_location)` are compressed into **meta-routes**. Each meta-route stores:

- The earliest segment as the representative
- Average start/end times and duration across all constituent segments
- Count of underlying segments (used for coverage-weighted selection)

Only the top `MAX_META_ROUTES` meta-routes (ranked by segment count) are retained. This compression reduces the QUBO variable count from `segments × drivers` to `meta_routes × drivers` while preserving the dominant route patterns.

A **variable budget** enforces a floor of 4 and ceiling of 7 meta-routes per cluster, yielding 12–21 QUBO variables (with 3 drivers) — the optimal range for the TN→Krylov compression chain.

---

## Stage C — QUBO Encoding, Compression, and Quantum Solve

### QUBO Formulation

Each cluster becomes a binary optimization problem: assign each meta-route to exactly one of `NUM_DRIVERS_PER_CLUSTER` drivers, minimizing penalty cost while satisfying hard and soft constraints.

**Binary variables**: `x_{r,d} ∈ {0,1}` — driver `d` is assigned to meta-route `r`.

#### Hard Constraints (large penalty weights)

| Constraint | Weight | Description |
|---|---|---|
| **Coverage** | 200.0 | Each meta-route must be assigned to exactly one driver. Implemented as a one-hot penalty: `−P·x_{r,d} + 2P·x_{r,d}·x_{r,d'}` for all driver pairs. |
| **Temporal Overlap** | 150.0 | No driver can serve two meta-routes whose time windows overlap. Checked pairwise within each driver. |
| **Turnaround** | 100.0 | Travel time between consecutive route endpoints must be physically feasible, using actual BVG travel time data from the dataset. |
| **Second Break** | 180.0 | At most one regular break (gap > 30 min) per shift. Penalizes the outer pair of routes that span two break gaps. |

#### Soft Constraints (reward/penalty bonuses)

| Constraint | Weight | Description |
|---|---|---|
| **Line/Depot Preference** | −15.0 | Bonus when driver is assigned to their preferred bus line or home depot. |
| **Shift Type Match** | −20.0 | Bonus for matching Früh/Spät/Nacht (early/late/night) preference. |
| **Continuity** | −10.0 | Bonus for consecutive routes at the same depot (reduces deadheading). |
| **Day-Off Violation** | +50.0 | Penalty for assigning work on a driver's requested day off. |
| **Early Start** | +10.0/hr | Penalty proportional to how far before the driver's earliest allowed start time. |
| **Split Shift Rejection** | +15.0×level | Base penalty scaled by driver's rejection level (0–3). |
| **1/6 Duty Rejection** | +40.0 | Penalty for assigning short-duty (≤120 min) to a driver who rejects it. |
| **Rotation Match** | −10.0 | Bonus for matching preferred rotation direction (multi-day). |
| **Weekly Day-Off** | +100.0 | Penalty for working on preferred weekly rest day. |
| **Daily Hours** | +5.0/hr | Penalty per hour deviation from daily target. |
| **Weekly Hours** | +10.0/hr | Penalty per hour deviation from weekly contract. |
| **4-Week Rolling Average** | +15.0/hr | Penalty per hour exceeding 8h/day rolling 4-week average. |
| **Overnight Rest** | +80.0 | Penalty for less than 11 hours rest between consecutive shifts. |

### Compression Chain: QUBO → TN → Krylov → Quantum Hardware

The raw QUBO (typically 15–21 variables) is too large for direct hardware execution with high fidelity. A two-stage compression pipeline reduces dimensionality while preserving the energy landscape structure.

#### Stage 1: Tensor Network Mean-Field Reduction (TN)

The `RealTNQUBOReducer` uses an MPS-inspired rank-1 tensor network (product state / mean-field approximation with bond dimension χ=1):

1. **Self-consistent mean-field iteration**: computes the magnetization ⟨σ_i⟩ for each variable via iterative tanh updates with damped convergence.
2. **Variable classification**: variables with |⟨σ_i⟩| close to ±1 are "stable" (low uncertainty) and fixed classically. Variables with |⟨σ_i⟩| near 0 are "uncertain" and forwarded to quantum.
3. **QUBO reduction**: the fixed variables' contributions are absorbed into the constant offset and the reduced linear/quadratic coefficients.

Typical compression: 21 variables → 10 active variables (fixing ~11 stable variables).

#### Stage 2: Krylov Subspace Compression

The `KrylovReducer` projects the reduced QUBO into a Krylov subspace via the Lanczos algorithm:

1. **Lanczos iteration**: builds an orthonormal basis V spanning the Krylov subspace K_m(H, v₀) = span{v₀, Hv₀, H²v₀, ...} where H is the symmetrized QUBO Hamiltonian.
2. **Spectral projection**: the QUBO is projected as H_red = V^T H V, yielding a `KRYLOV_DIM`-dimensional problem.
3. **Solution expansion**: after quantum solving in the reduced space, solutions are mapped back via V-projection and threshold binarization.

Typical compression: 10 variables → 8 qubits (the final dimension sent to hardware).

### Quantum Solve: BF-DCQO on IBM Hardware

The compressed Ising Hamiltonian is submitted to Kipu Quantum's **Bias-Field Digitized Counterdiabatic Quantum Optimization (BF-DCQO)** solver via the PlanQK/Kipu Quantum Hub API. The solver constructs a variational quantum circuit with counterdiabatic driving terms that accelerate convergence toward the ground state.

Parameters:
- **Shots**: 100 measurement samples per circuit
- **Iterations**: 5 variational optimization loops
- **Greedy passes**: 1 classical post-processing pass inside Miray
- **Backend**: IBM Quantum hardware (ibm_fez, Heron r2, 156 qubits)

### Pre-Expansion Refinement

After the quantum solver returns raw solutions (in the compressed Krylov space), two classical refinement techniques are applied before expanding back to the original variable space:

1. **CVaR Filtering** (α=0.3): retains only the top 30% of solutions by energy, discarding noisy/suboptimal measurements.
2. **Probabilistic Bit Flipping**: performs local search by randomly flipping bits with probability 0.15 across 10 trials, keeping improvements. This escapes shallow local minima.

### Solution Expansion

The refined compressed solution is expanded back through the chain in reverse order:

1. **Krylov expand**: V-projection from `KRYLOV_DIM` back to TN-reduced dimension.
2. **TN expand**: reinserts the classically-fixed variables, reconstructing the full QUBO solution.
3. **Meta-route unpack**: each meta-route assignment is expanded to its constituent segments.

---

## Stage D — 30-Day Replication and Post-Processing

### Archetype-to-Day Adapter

The quantum solver produces an optimized schedule for each archetype's representative day. The `adapt_schedule_to_target_day` function maps this template schedule to every other day in the same cluster.

**Matching algorithm**:

1. Build a lookup index: `(from_location, to_location)` → sorted list of target-day segments.
2. For each segment in the representative schedule, find the closest match by:
   - **Route signature**: exact match on `(from_location, to_location)`
   - **Temporal proximity**: among signature matches, select the segment with the nearest `start_time`
3. Accept the match only if the start-time deviation is within `time_tol_min=10` minutes.
4. Mark matched segments as used (no double-assignment).
5. Segments with no valid match are counted as **ghost assignments** and dropped — these are routes that exist on the representative day but not on the target day.

### The Legalizer (Cluster-Level Validation)

Applied immediately after each cluster's quantum solution is decoded, the Legalizer enforces physical feasibility:

1. **Segment length check**: removes segments shorter than `MIN_SEGMENT_DURATION=5` minutes.
2. **Temporal overlap resolution**: for each driver's schedule, segments are sorted chronologically. If two consecutive segments overlap in time, the shorter one is removed.
3. **Travel time validation**: verifies that the gap between consecutive segments ≥ actual BVG travel time + 5-minute buffer. Infeasible transitions are broken by removing the offending segment.
4. **Greedy reassignment**: removed segments are re-tested against all drivers for feasible insertion, recovered if possible.

### Global Post-Processing (Cross-Cluster)

After cluster solutions are merged into a full-day schedule, global optimization handles inter-cluster interactions:

#### Stage 1 — Coverage Repair

Identifies unassigned segments (dropped during cluster-level solving or adapter ghosting) and attempts greedy insertion:

- For each orphan segment (sorted by start_time):
  - For each driver: check (a) 10-hour daily limit, (b) travel feasibility from previous segment, (c) travel feasibility to next segment.
  - Assign to the first feasible driver.
  - Maintain chronological order after insertion.

#### Stage 2 — Overlap Resolution

Final safety pass: for each driver, sort segments chronologically and greedily accept only non-overlapping segments. Any remaining overlaps (from cross-cluster merging) are removed.

### 30-Day Coverage Summary

After all archetypes are solved and replicated, the pipeline produces a `monthly_schedules` dictionary mapping `(config_label, day_label)` → `{driver_id: [segments]}` for all 30 days, enabling full-month visualization and KPI computation.

---

## QUBO Variable and Qubit Budget

| Pipeline Stage | Variables | Method |
|---|---|---|
| Raw segments per cluster | ~25 | After spatial decomposition |
| After meta-route aggregation | 7 meta-routes × 3 drivers = **21** | Level 3 compression |
| After TN mean-field | **10** active variables | ~11 variables fixed classically |
| After Krylov projection | **8** qubits | Lanczos subspace projection |
| Sent to IBM hardware | **8 qubits** per cluster | BF-DCQO circuit |

**Total quantum calls**: 4 archetypes × ~6 non-trivial clusters = **~24 Miray API calls** for the full 30-day schedule.

---

## Constraint Enforcement Summary

| Layer | Scope | Hard Constraints | Method |
|---|---|---|---|
| QUBO Encoding | Per cluster | Coverage, overlap, turnaround, second break | Penalty terms in Hamiltonian |
| QUBO Encoding | Per cluster | Driver preferences, shift types, labor law | Soft penalty/reward terms |
| Legalizer | Per cluster | Physical travel feasibility, segment duration | Greedy removal + reassignment |
| Global Post-Processing | Cross-cluster | 10-hour daily limit, full coverage | Greedy insertion + overlap removal |
| Adapter | Cross-day | Route signature matching, temporal proximity | Nearest-neighbor matching with tolerance |
