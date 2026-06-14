# Block Replacement Policy for Hybrid Cache (MacSim)

A write-aware, partitioned cache replacement policy built on the MacSim architectural simulator. It extends the lifetime of Non-Volatile Memory (NVM / STT-RAM) in a hybrid L3 cache by ensuring write misses are never placed directly into the STT-RAM region — steering them exclusively to the faster, endurance-unlimited SRAM partition instead.

---

## Table of Contents

- [Motivation](#motivation)
- [Background: Hybrid Cache Architecture](#background-hybrid-cache-architecture)
- [What This Project Changes](#what-this-project-changes)
- [How It Works — Technical Deep Dive](#how-it-works--technical-deep-dive)
  - [1. Miss Classification (memory.cc)](#1-miss-classification-memorycc)
  - [2. Partitioned Replacement (cache.cc)](#2-partitioned-replacement-cachecc)
  - [3. Access Pattern Counter (cache.cc)](#3-access-pattern-counter-cachecc)
- [Cache Configuration](#cache-configuration)
- [Simulation Setup](#simulation-setup)
- [Build Instructions](#build-instructions)
- [Run Instructions](#run-instructions)
- [Output Files and How to Read Them](#output-files-and-how-to-read-them)
- [Known Limitations and Future Work](#known-limitations-and-future-work)

---

## Motivation

Modern processors increasingly pair **STT-RAM (Spin-Transfer Torque RAM)** with **SRAM** inside the same LLC (Last Level Cache). This hybrid design offers:

- STT-RAM: higher density → more capacity at the same area budget, non-volatile, fast reads
- SRAM: lower density, no endurance limit, fast for both reads and writes

The critical problem with STT-RAM is **write endurance**. Every write physically stresses its magnetic tunnel junction (MTJ) cell. Typical STT-RAM cells tolerate 10⁸–10¹² writes before failure — a limit that real workloads can approach when writes are not managed carefully.

The **default LRU replacement policy in MacSim is completely write-agnostic**: it will happily evict an SRAM line to make room for a write miss, pushing the write into STT-RAM. Over millions of accesses, this causes unnecessary wear on the NVM region. This project fixes that by making the replacement policy **aware of what type of miss is being served**.

---

## Background: Hybrid Cache Architecture

The L3 cache in this simulation is **16-way set-associative**. The physical ways are logically split into two regions:

```
Way:    [ 0  1  2  3  4  5  6  7  8  9  10  11 | 12  13  14  15 ]
Region: [         STT-RAM  (¾ of assoc = 12 ways)          | SRAM (¼ = 4 ways) ]
```

| Property           | STT-RAM Region (ways 0–11) | SRAM Region (ways 12–15) |
|--------------------|---------------------------|--------------------------|
| Capacity share     | 75%                        | 25%                      |
| Write endurance    | Limited (~10⁸–10¹²)       | Unlimited                |
| Read performance   | Fast                       | Fast                     |
| Write performance  | Slower (write disturb)     | Fast                     |
| Target traffic     | Read misses only           | Write misses only        |

The 3:1 ratio is based on the empirical observation that most workloads are **read-dominant**, so the larger partition can safely serve reads, and the smaller SRAM partition absorbs the less frequent write traffic.

---

## What This Project Changes

Three focused changes were made to the MacSim codebase:

| File | Change | Purpose |
|---|---|---|
| `src/memory.cc` | Classify every L3 miss as read (`writemiss=0`) or write (`writemiss=1`) before calling `insert_cache()` | Propagate miss type down to the replacement layer |
| `src/cache.cc` | Added `find_replacement_line_with_partition()` | Core partitioned LRU logic that restricts eviction search to the correct region |
| `src/cache.cc` | Added `access_cache_count()` | Instrumentation to count read/write hits in each region (STT-RAM vs SRAM) for analysis |

The existing `find_replacement_line()` (vanilla LRU across all ways) is retained and is still used for any access that doesn't originate from an L3 miss (e.g., line refills tagged with `writemiss=2`).

---

## How It Works — Technical Deep Dive

### 1. Miss Classification (`memory.cc`)

When the L3 fill queue processes an incoming miss, the handler now checks the request type before inserting into the cache:

```cpp
// READ MISS: instruction or data fetch
if ((req->m_type == MRT_DFETCH || req->m_type == MRT_IFETCH) && m_level == MEM_L3) {
    int writemiss = 0;  // signals: insert into STT-RAM region
    data = m_cache->insert_cache(req->m_addr, &line_addr, &victim_line_addr,
                                  req->m_appl_id, req->m_ptx, false, writemiss);
}

// WRITE MISS: data store or writeback
else if ((req->m_type == MRT_DSTORE && req->m_type == MRT_WB) && m_level == MEM_L3) {
    int writemiss = 1;  // signals: insert into SRAM region
    data = m_cache->insert_cache(req->m_addr, &line_addr, &victim_line_addr,
                                  req->m_appl_id, req->m_ptx, false, writemiss);
}

// All other cases (non-L3 levels, prefetches, etc.)
else {
    data = m_cache->insert_cache(...);  // writemiss defaults to 2 → vanilla LRU
}
```

The `writemiss` integer (`0`, `1`, or `2`) is then threaded through `insert_cache()` all the way down to the replacement function.

> **⚠️ Known Bug:** The write miss condition uses `&&` instead of `||`:
> `req->m_type == MRT_DSTORE && req->m_type == MRT_WB`
> A single request can never have two types simultaneously, so this branch is **never taken**. The correct condition is `req->m_type == MRT_DSTORE || req->m_type == MRT_WB`. As a result, only read miss partitioning (`writemiss=0`) is active in the current build. This is a pending fix.

---

### 2. Partitioned Replacement (`cache.cc`)

`insert_cache()` checks the `m_enable_hybrid` flag (hardcoded `true`) and dispatches to the new function:

```cpp
bool m_enable_hybrid = true;

if (!m_enable_hybrid) {
    // Original paths (GPU-aware partition, or vanilla LRU)
    ...
} else {
    if (writemiss != 2)
        ins_line = find_replacement_line_with_partition(set, appl_id, writemiss);
    else
        ins_line = find_replacement_line(set, appl_id);  // fallback
}
```

The key function `find_replacement_line_with_partition()`:

```cpp
cache_entry_c* cache_c::find_replacement_line_with_partition(int set, int appl_id, int writemiss)
{
    int m_assoc_start, m_assoc_end;

    if (writemiss == 1) {
        // Write miss → search only SRAM ways (12–15)
        m_assoc_start = (m_assoc * 3) / 4;   // = 12
        m_assoc_end   = m_assoc;              // = 16
    }
    else if (writemiss == 0) {
        // Read miss → search only STT-RAM ways (0–11)
        m_assoc_start = 0;
        m_assoc_end   = m_assoc - (m_assoc / 4);  // = 12
    }

    // Then run LRU (or Pseudo-LRU) within [m_assoc_start, m_assoc_end)
    ...
}
```

**LRU within the region:** Scans only the ways in the assigned range. Returns the first invalid (empty) slot if one exists, otherwise evicts the way with the smallest `m_last_access_time` — the Least Recently Used line within that partition.

**Pseudo-LRU path:** If `KNOB_CACHE_USE_PSEUDO_LRU` is set, uses a clock-like approximation: scan for any line with `m_last_access_time == 0`; if none found, reset all timestamps in the region to 0 and repeat. This is faster than tracking exact timestamps.

**Why this works:** Evictions within the STT-RAM region are never triggered by write misses, and evictions within the SRAM region are never triggered by read misses. The two traffic classes are fully isolated from each other within each cache set.

---

### 3. Access Pattern Counter (`cache.cc`)

`access_cache_count()` was added as a diagnostic function. Every time a cache lookup hits, it increments one of four counters depending on which region the hit landed in and whether it was a read or write:

```
count_write_STT  — write hits in ways 0–11
count_read_STT   — read hits in ways 0–11
count_write_SRAM — write hits in ways 12–15
count_read_SRAM  — read hits in ways 12–15
```

These are printed to stdout on every hit. In a future version, this should be redirected to the stats framework rather than `cout`. The ideal outcome is:
- `count_write_STT → 0` (no writes landing in NVM region)
- `count_read_SRAM → 0` (no reads evicted from fast SRAM unnecessarily)

---

## Cache Configuration

All parameters are set in `bin/params.in`. The relevant L3 configuration:

| Parameter | Value | Meaning |
|---|---|---|
| `l3_num_set` | 512 | Number of sets in L3 |
| `l3_assoc` | 16 | Ways per set (16-way set-associative) |
| `l3_line_size` | 64 | Cache line size in bytes |
| `l3_num_bank` | 8 | Number of L3 banks |
| `l3_latency` | 30 | L3 access latency in cycles |
| STT-RAM ways | 0–11 | 75% of associativity (hardcoded as `3/4 * assoc`) |
| SRAM ways | 12–15 | 25% of associativity (hardcoded as `assoc/4`) |

**Total L3 size:** 512 sets × 16 ways × 64 bytes = **512 KB**

The simulation models 4 large x86 OoO cores with the following hierarchy per core:
- L1 I-cache: 64 KB, 8-way
- L1 D-cache: 64 KB, 8-way, 3-cycle latency
- L2: 256 KB, 8-way, 8-cycle latency
- **L3 (shared, modified by this project): 512 KB, 16-way, 30-cycle latency**
- DRAM: FR-FCFS scheduled, 2 channels, 8 banks, ~99-cycle average latency

---

## Simulation Setup

**Simulator:** [MacSim](https://github.com/gthparch/macsim) — a cycle-accurate x86/PTX architectural simulator from Georgia Tech.

**Benchmark:** `xalancbmk` (SPEC CPU2006 — XML-to-text transformation workload). This is a memory-intensive x86 workload, making it a representative stress test for the cache replacement policy.

**Trace format:** Pin-based execution traces stored as `.txt` files containing instruction addresses, memory access addresses, and types.

**Simulation limit:** 200 million instructions (`max_insts 200000000`)

---

## Build Instructions

**Requirements:**
- Linux (Ubuntu recommended)
- GCC 4.4+ with C++11 support
- `scons` build system
- `zlib` development headers
- Python (for the build script)

```bash
# Install dependencies
sudo apt-get install scons zlib1g-dev python3

# Clone the repository
git clone https://github.com/Zeekersky/Block-Replacement-MacSIM
cd Block-Replacement-MacSIM

# Build
./build.py
```

A successful build produces the `bin/macsim` binary.

---

## Run Instructions

```bash
cd bin

# Verify these files exist before running:
# - params.in       (simulation configuration)
# - trace_file_list (path to benchmark trace)

./macsim
```

The simulator prints L3 cache dimensions at startup:
```
L3 cache size: 512 sets, 16 assoc: 64 line size: 8 banks
```

All output is written to `bin/*.stat.out` files upon completion.

---

## Output Files and How to Read Them

| File | Contents |
|---|---|
| `memory.stat.out` | Hit/miss rates per cache level, DRAM traffic, writebacks |
| `dram.stat.out` | DRAM row buffer activity, average latency, bandwidth |
| `general.stat.out` | Total instructions, cycles, execution time, IPC |
| `core.stat.out` | Per-core pipeline statistics |
| `bp.stat.out` | Branch predictor accuracy |
| `pref.stat.out` | Hardware prefetcher statistics |
| `power_units.stat.out` | Power model estimates |

**Key metrics to check for this project (from `memory.stat.out`):**

```
L3_HIT_CPU     — how often the modified replacement policy serves hits
L3_MISS_CPU    — L3 misses that go to DRAM (lower is better)
TOTAL_WB       — total writebacks (a proxy for write pressure on STT-RAM)
L3_WB          — writebacks originating from L3 evictions
TOTAL_DRAM     — total DRAM accesses (lower = better cache utilization)
AVG_MEMORY_LATENCY — average memory access latency in cycles
```

**Simulation results (xalancbmk, 20,000 cycles):**

| Metric | Value |
|---|---|
| L1 hit rate | ~83% (1.87M hits / 375K misses) |
| L2 hit rate | ~16% (18K hits / 96K misses) |
| L3 hit rate | ~3.2% (3.1K hits / 92.9K misses) |
| Total DRAM accesses | 95,302 |
| Avg. memory latency | 132.6 cycles |
| L3 writebacks | 2,349 |

The low L3 hit rate confirms xalancbmk's memory-intensive nature — most accesses miss all the way to DRAM, which makes every L3 insertion decision critical for endurance management.

---

## Known Limitations and Future Work

**Bug: Write miss branch is unreachable**
The condition `req->m_type == MRT_DSTORE && req->m_type == MRT_WB` is always false (a request cannot have two types at once). The write miss path (`writemiss=1`) never executes. Fix: change `&&` to `||`.

**Fixed partition ratio**
The 3:1 STT-RAM to SRAM split is hardcoded via `(m_assoc * 3)/4`. A future version should expose this as a `params.in` knob so it can be tuned per workload without recompiling.

**No GPU-aware partition**
A comment in `insert_cache()` notes that `find_replacement_line_from_same_type` (the existing GPU/CPU partition) has no equivalent with the new hybrid logic. Heterogeneous CPU+GPU workloads will bypass the partition entirely.

**Diagnostic counter uses stdout**
`access_cache_count()` prints via `cout` on every hit, which is disabled in the current build to avoid flooding output. The correct approach is to integrate these counters into MacSim's stats framework so they appear in `memory.stat.out`.

**Pseudo-LRU within a partition can thrash**
If write traffic exceeds the capacity of the 4-way SRAM partition, those 4 ways will evict frequently among themselves. A more adaptive policy (e.g., dynamically resizing the partition based on recent write pressure) would handle write-heavy workloads better.

**Single benchmark**
Results are only validated on `xalancbmk`. Testing on additional SPEC CPU2006/2017 workloads with varying write intensities (e.g., `mcf`, `omnetpp`, `lbm`) would give a complete picture of the policy's effectiveness.