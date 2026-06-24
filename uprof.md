# AMD uProf: System Performance Analysis and Optimization Tool

## Overview

**AMD uProf** ("MICRO-prof") is a free, vendor-supplied performance analysis tool suite
targeting x86-64 applications on Windows, Linux, and FreeBSD. It is purpose-built for
AMD "Zen"-family processors (Zen through Zen 5) and AMD Instinct MI-series CDNA
accelerators (MI100, MI200, MI300). Its design philosophy mirrors that of Intel VTune
Profiler, but it surfaces AMD-specific microarchitectural events that generic tools such
as `perf` or `gprof` do not expose.

The current stable release is **uProf 5.3** (May 2026), which introduced DuckDB as the
default storage backend, replacing SQLite.

---

## Architecture: Two-Phase Workflow

uProf separates **data collection** from **analysis**, a standard approach in
production-grade profilers:

1. **Collection** — low-overhead instrumentation or sampling runs alongside or wraps the
   target process, writing raw event records to disk.
2. **Translation / Report generation** — raw records are decoded, aggregated, and stored
   in a database (DuckDB by default since 5.3) for querying and visualization.

Both phases are exposed via `AMDuProfCLI` (command-line) and the `AMDuProf` GUI. The
CLI is appropriate for headless HPC nodes and batch scripting; the GUI provides
flame graphs, timeline views, and source-level annotation.

---

## Profiling Mechanisms

### 1. Time-Based Profiling (TBP)

The simplest mode. The OS timer fires at a configurable rate and the kernel captures the
instruction pointer (IP) and call stack of the interrupted thread. This mode:

- Requires no special privileges on most Linux configurations.
- Is suitable for identifying call-graph hot spots in any x86-64 binary (C, C++, Java,
  Python).
- Supports **callstack stitching** for OpenMP runtimes and **mixed-mode callstacks**
  for CPython (native + interpreted frames interleaved).

### 2. Event-Based Sampling (EBS) via Core PMCs

uProf programs the on-chip **Performance Monitoring Counters (PMCs)** to overflow at
a configurable threshold and deliver a precise (or near-precise) interrupt. The set of
available events is architecture-dependent and includes:

| Category | Example Events |
|---|---|
| Pipeline | retired instructions, uops dispatched, pipeline stalls |
| Branch | branch mispredictions, indirect branch misses |
| Cache | L1/L2/L3 data/instruction cache misses |
| Memory | DRAM accesses, NUMA remote accesses |
| TLB | L1/L2 DTLB misses, page-walker cache misses |
| FPU/SIMD | FP/SSE/AVX dispatch events |

PMC multiplexing is used when the number of requested events exceeds the number of
physical PMC registers per core (typically 6 on Zen), at the cost of reduced statistical
accuracy per counter.

### 3. Instruction-Based Sampling (IBS)

IBS is AMD's solution to the **instruction attribution skid** problem inherent in
event-based sampling on out-of-order superscalar processors. Rather than triggering on
the Nth event (which may be speculative or reordered), IBS marks the Nth instruction
that **retires** and tags it as it flows through the entire pipeline. All microarchitectural
events attributed to that instruction are recorded at retirement with zero measurement
overhead (no pipeline stalls). Two IBS sub-modes exist:

- **IBS Fetch Sampling** — records events during the instruction fetch phase: fetch
  latency, iTLB misses, fetch block boundaries.
- **IBS Op Sampling** — records events during execution and memory access: effective
  address, load/store latency, dTLB misses, cache level where a miss was served,
  branch prediction outcome, dispatch port.

IBS enables workflows that event-based sampling cannot: precise false-sharing detection,
load/store latency attribution to exact source lines, and per-instruction NUMA locality
analysis. uProf 5.3 added the `IBS_[LD,ST]_L1_DTLB_REFILL_LAT` metric for Zen 4/5,
enabling fine-grained TLB refill latency breakdowns.

---

## Analysis Modes

### CPU Profiler

- **Hotspot Analysis** — inclusive/exclusive sample attribution per function and source
  line, with call-graph reconstruction.
- **Microarchitecture Analysis** — identifies pipeline bottlenecks: front-end bound
  (fetch/decode), back-end bound (execution unit pressure, memory bound), bad
  speculation.
- **False Cache-Line Sharing Detection** — uses IBS Op to identify cache lines
  simultaneously written by multiple threads on different cores, a common source of
  performance collapse in lock-free and parallel code.
- **NUMA Analysis** — attributes remote DRAM accesses to source locations, critical on
  multi-socket EPYC platforms with complex NUMA topologies.
- **Roofline Model** — places kernel performance in the context of the hardware's
  memory bandwidth and peak FLOP rate to distinguish memory-bound from compute-bound
  workloads.

### Threading Analysis

Timeline-based visualization of thread states (running, blocked, waiting on a
synchronization primitive) across all logical CPUs. Metrics include:

- Thread concurrency histograms
- Lock contention hot spots
- Idle/unused thread detection (new in 5.3: "Unused Threads" metric handles variable
  thread counts across profile runs)
- OpenMP fork-join overhead

### MPI / OpenMP Tracing

Trace-level analysis of **MPI** and **OpenMP** workloads, targeting compute and load
imbalance across ranks and worker threads. Per-rank timeline views allow identification
of stragglers, barrier wait times, and communication overhead. uProf 5.3 introduced
per-rank analysis improvements in HTML reports.

### GPU Profiler (AMD Instinct / CDNA)

For HPC workloads on MI-series accelerators:

- Kernel dispatch timelines
- GPU hardware component utilization (compute units, memory controllers, L2 cache)
- Roofline on device
- Heterogeneous profiling on MI300A (APU-class, unified CPU+GPU memory), with the
  Overview profile type showing the full CPU+GPU execution timeline.

The GPU profiling backend uses the `rocprofiler-sdk` API.

### Power and Thermal Profiling

The `timechart` command (and `AMDuProfPCM` utility) collects:

- Per-socket and per-core CPU frequency, TDP, and package power via AMD's RAPL/SMU
  interfaces
- Thermal throttling events
- PCIe metrics (Zen 3 server platforms and above)
- Memory bandwidth via uncore PMCs (IMC counters on EPYC)

A live timeline is displayed in the GUI. This mode is relevant for power-capping analysis
and distinguishing thermal throttling from algorithmic inefficiency.

### System-Level Analysis

`AMDSystemCheck` (Linux, new in 5.3) collects platform topology: NUMA node layout,
BIOS settings, OS configuration, and CPU feature flags. This is valuable in
virtualized and cloud environments where vIBS support (KVM) is available, as
hypervisor-level effects on performance are frequently non-obvious.

---

## Remote Profiling

A Windows host can drive data collection on a remote Linux target over SSH.
Collection and translation run on the remote node; results are displayed in the local
GUI. This is practical for profiling headless compute nodes in an HPC cluster without
requiring a display on the target.

---

## Backend and Storage

Since **uProf 5.3**, the default database backend is **DuckDB**, an in-process OLAP
engine optimized for analytical queries over columnar data. This is a significant
engineering choice: profiling sessions for long-running, many-threaded, or MPI
workloads generate large volumes of sample records; DuckDB's vectorized query execution
substantially reduces report generation time compared to the previous SQLite backend.
SQLite remains available for backward compatibility.

---

## Language and Runtime Support

| Language / Runtime | CPU Hotspot | Call Graph | IBS |
|---|---|---|---|
| C / C++ | ✓ | ✓ | ✓ |
| Fortran | ✓ | ✓ | ✓ |
| Java (JVM) | ✓ | ✓ | partial |
| Python (CPython) | ✓ | mixed-mode | — |
| OpenMP | ✓ | stitched | ✓ |
| MPI | ✓ (trace) | per-rank | ✓ |

---

## CLI Reference (Selected Commands)

```bash
# Time-based profiling
AMDuProfCLI collect --config tbp --output-dir ./out ./my_app

# Event-based profiling (L3 miss attribution)
AMDuProfCLI collect --config assess --output-dir ./out ./my_app

# False sharing detection (IBS Op)
AMDuProfCLI collect --config false-sharing --output-dir ./out ./my_app

# Power / thermal timeline
AMDuProfCLI timechart --output-dir ./out

# Generate CSV report from collected data
AMDuProfCLI report --input-dir ./out/AMDuProf-my_app-TBP_<timestamp>/

# System info
AMDuProfCLI info
```

---

## Positioning in the AMD Toolchain

| Tool | Primary Use |
|---|---|
| **uProf** | CPU + GPU profiling, power, threading, MPI — Windows & Linux |
| **rocprofv3** | GPU kernel profiling on Linux (ROCm) |
| **rocprof-compute** | Roofline and bottleneck analysis on AMD Instinct (Linux) |
| **rocprof-sys** | Full-system tracing (CPU + GPU) on Linux |
| **Radeon GPU Profiler** | Graphics pipeline (RDNA, rasterization/shaders) |

uProf is the only tool in AMD's suite that supports **Windows**, making it the default
choice for Windows-hosted development targeting Ryzen or EPYC platforms.

---

## Summary

AMD uProf is a comprehensive, production-quality profiler that leverages AMD-specific
hardware — particularly IBS — to deliver precise instruction-level attribution that
generic profilers cannot match on out-of-order cores. Its coverage spans CPU
microarchitecture, threading, MPI/OpenMP, power, and GPU workloads, under a unified
CLI/GUI interface. The 5.3 release's move to DuckDB signals a pragmatic acknowledgment
that profiling data volumes in modern HPC workloads have outgrown single-threaded
row-store databases.
