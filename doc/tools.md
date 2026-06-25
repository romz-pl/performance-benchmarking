# System Performance Analysis and Optimization Tools

## A Technical Reference for Computer Science Experts

---

## Table of Contents

1. [Conceptual Framework](#1-conceptual-framework)
2. [Tool Ecosystem Overview](#2-tool-ecosystem-overview)
3. [AMD uProf](#3-amd-uprof)
4. [Intel VTune Profiler](#4-intel-vtune-profiler)
5. [Windows Performance Analyzer (WPA) and ETW](#5-windows-performance-analyzer-wpa-and-etw)
6. [Supplementary Tools](#6-supplementary-tools)
7. [Cross-Layer Analysis Methodology](#7-cross-layer-analysis-methodology)
8. [CPU Bottleneck Analysis](#8-cpu-bottleneck-analysis)
9. [GPU Bottleneck Analysis](#9-gpu-bottleneck-analysis)
10. [Memory Subsystem Analysis](#10-memory-subsystem-analysis)
11. [Storage I/O Analysis](#11-storage-io-analysis)
12. [Software-Layer Profiling](#12-software-layer-profiling)
13. [Optimization Strategy Synthesis](#13-optimization-strategy-synthesis)
14. [Advanced Patterns and Anti-Patterns](#14-advanced-patterns-and-anti-patterns)
15. [Reference Metrics Cheat Sheet](#15-reference-metrics-cheat-sheet)

---

## 1. Conceptual Framework

### The Performance Analysis Hierarchy

System performance is best understood as a hierarchy of resource consumers and providers. Bottlenecks propagate upward from hardware to software, but the *root cause* is often at a level well below where symptoms manifest.

```
┌─────────────────────────────────────────────────┐
│              Application / OS Layer             │  ← Symptoms visible here
├─────────────────────────────────────────────────┤
│         Runtime / Compiler / Libraries          │
├──────────────────┬──────────────────────────────┤
│    CPU (cores,   │   GPU (SMs, caches,          │
│    caches, TLB,  │   VRAM, PCIe BW)             │
│    branch pred.) │                              │
├──────────────────┴──────────────────────────────┤
│          Memory Subsystem (DRAM, IMC)           │
├─────────────────────────────────────────────────┤
│        Storage & I/O (NVMe, SAN, net)           │  ← Root causes often here
└─────────────────────────────────────────────────┘
```

### Measurement Philosophy

Performance instrumentation falls into three categories, each with distinct overheads and fidelity trade-offs:

| Method | Overhead | Temporal Resolution | Intrusiveness |
|---|---|---|---|
| **Sampling (statistical)** | < 1–2 % | ~ms | Low: no source modification |
| **Instrumentation (tracing)** | 5–30 % | ~µs–ns | High: binary or source rewrite |
| **Hardware PMU counters** | Near-zero | cycle-accurate | None: hardware registers |
| **OS kernel tracing (ETW/eBPF)** | 1–10 % | ~100 ns | None: kernel ring buffers |

The professional workflow layers all four: PMU counters and OS tracing first (low overhead, system-wide), then targeted instrumentation on confirmed hot paths.

### Amdahl's Law and Roofline Model

Before instrumenting, frame expected gains mathematically:

**Amdahl's Law** bounds the speedup from optimizing a fraction *p* of execution time:

```
Speedup = 1 / ((1 - p) + p/s)
```

where *s* is the speedup factor applied to the parallel/optimized fraction. If only 80 % of the workload is parallelizable, the theoretical maximum speedup is 5×, regardless of core count.

**The Roofline Model** frames compute vs. memory-bandwidth limitations:

```
Attainable Performance = min(Peak FLOPS, Peak BW × Arithmetic Intensity)
```

Arithmetic Intensity (AI) = FLOP / byte transferred. Kernels below the ridge point are *memory-bound*; above it, *compute-bound*. All profiling tools discussed here produce data mappable onto this model.

---

## 2. Tool Ecosystem Overview

### Platform Coverage Matrix

| Tool | Vendor | OS | CPU | GPU | Memory | Storage | Trace |
|---|---|---|---|---|---|---|---|
| **uProf** | AMD | Win/Linux | AMD Zen 1–5 | AMD RDNA/CDNA | ✓ | – | – |
| **VTune** | Intel | Win/Linux | Intel + AMD | Intel Xe | ✓ | ✓ | ✓ |
| **WPA / ETW** | Microsoft | Windows | x86/ARM | Any | ✓ | ✓ | ✓ |
| **Perf + Linux PMU** | Linux kernel | Linux | Any | – | ✓ | ✓ | ✓ |
| **ROCm Profiler** | AMD | Linux | – | AMD CDNA | ✓ | – | ✓ |
| **NSight Systems** | NVIDIA | Win/Linux | Any | NVIDIA | ✓ | – | ✓ |
| **NSight Compute** | NVIDIA | Win/Linux | – | NVIDIA | ✓ | – | – |

### Taxonomy of Performance Events

Hardware Performance Monitoring Units (PMUs) expose *events* in four families:

1. **Core events** — cycles, instructions retired, branches mispredicted, micro-ops dispatched/retired
2. **Cache events** — L1/L2/L3 hits, misses, evictions, prefetch effectiveness
3. **Memory events** — DRAM reads/writes, memory controller occupancy, page faults (hardware)
4. **Uncore/offcore events** — IMC (Integrated Memory Controller) bandwidth, PCIe transactions, QPI/UPI link utilization

PMU multiplexing occurs when the number of requested events exceeds available hardware counter registers (typically 4–8 programmable + fixed). Tools time-slice counter groups, introducing sampling variability. For high-fidelity micro-architectural studies, limit event groups to ≤ 4 per collection pass.

---

## 3. AMD uProf

### Architecture and Data Collection Engine

AMD uProf (MicroProfiler) is AMD's unified profiling suite for Zen-architecture CPUs and RDNA/CDNA GPUs. It exposes the full AMD Family 17h–1Ah PMU event hierarchy, including Zen 4's expanded L3 cache PMU and DDR5 memory controller counters.

uProf operates through three collection drivers:
- **AMDuProfPcm** — kernel-mode driver for PMU event access
- **AMDuProfCLU** (Cache Line Utilization) — instruction-level cache access recorder
- **AMDuProfSys** — system-level power and thermal event collector

### Installation and Configuration

```bash
# Linux installation (RPM-based)
sudo rpm -ivh AMDuProf-<version>.x86_64.rpm
# Load kernel module
sudo modprobe msr
sudo modprobe cpuid
# Verify PMU access
AMDuProfCLI info --system
```

On Linux, `perf_event_paranoid` must be ≤ 1 for user-space profiling without root:

```bash
echo 1 | sudo tee /proc/sys/kernel/perf_event_paranoid
```

### CPU Profiling Modes

#### Time-Based Sampling (TBS)

Samples the instruction pointer (IP) and call stack at a configurable rate (default: 1 ms). Produces a call-graph with self-time and cumulative-time attribution.

```bash
AMDuProfCLI collect --config tbs --duration 30 \
    --call-stack --interval 1 \
    -o ./profile_results \
    -- ./my_application --workload heavy
```

Key output metrics:
- `Samples %` — fraction of samples hitting each function/line
- `CPU Cycles` — estimated cycles (samples × interval × frequency)
- `IPC` — derived from samples vs. instruction-retired event correlation

#### Event-Based Sampling (EBS)

Fires a PMU interrupt every *N* occurrences of a hardware event. Ideal for locating instructions causing specific micro-architectural stalls.

```bash
# Sample on L2 data cache misses (DC_REFILL_FROM_L3 + DC_REFILL_FROM_DRAM)
AMDuProfCLI collect --config ebs \
    --event "l2_cache_miss_from_l3:100003,l2_cache_miss_from_dram:100003" \
    -o ./cache_miss_profile \
    -- ./my_application
```

#### Assess (Top-Down Microarchitecture Analysis)

uProf's Assess mode implements a Zen-adapted top-down methodology. The root node partitions pipeline slots into:

```
Pipeline Slots
├── Retiring (useful work)
├── Bad Speculation (branch misprediction flush)
├── Frontend Bound
│   ├── Branch Resteers
│   ├── ICache Miss
│   └── Fetch Latency
└── Backend Bound
    ├── Memory Bound
    │   ├── L1 Bound
    │   ├── L2 Bound
    │   ├── L3 Bound
    │   └── DRAM Bound
    └── Core Bound
        ├── Divider
        ├── Serializing Operation
        └── Ports Utilization
```

```bash
AMDuProfCLI collect --config assess -o ./tda_result -- ./my_application
AMDuProfCLI report -i ./tda_result --report-type assess
```

The report shows slot utilization percentages. A `Backend Bound > Memory Bound > DRAM Bound` value above 30 % signals a memory-bandwidth bottleneck; proceed to memory subsystem analysis (see §10).

#### Cache Line Utilization (CLU)

CLU instruments load/store instructions at the binary level to record which bytes within each cache line are actually accessed. It identifies *cache line fragmentation* (false sharing) and *spatial locality failures* where only a fraction of each 64-byte line is used.

```bash
AMDuProfCLI collect --config clu -o ./clu_result \
    -- ./my_application
AMDuProfCLI report -i ./clu_result --report-type clu \
    --output-format csv
```

Output columns:
- `CL Utilization %` — bytes used / 64 bytes per cache line fetch
- `Access Count` — number of cache line fetches to this structure
- `Source Location` — file:line of the allocating/accessing code

A `CL Utilization < 50 %` on a hot structure indicates poor spatial locality; consider structure packing, array-of-structs-to-struct-of-arrays transformation, or access reordering.

#### NUMA and Memory Topology Analysis

For multi-socket and chiplet architectures (Zen 3/4 have multiple CCDs sharing an I/O die):

```bash
AMDuProfCLI collect --config memory_access \
    --event "umc_data_slot_reads:10000,umc_data_slot_writes:10000" \
    --numa-node-map \
    -o ./numa_profile -- ./my_application
```

The NUMA affinity view shows per-NUMA-node access counts, exposing cross-NUMA-node traffic that inflates latency by 2–3× compared to local access.

### Power and Thermal Profiling

uProf's Energy Profiler hooks into AMD RAPL (Running Average Power Limit) via MSR registers:

```bash
AMDuProfCLI collect --config energy \
    --affinity-mask "0-7" \
    -o ./energy_profile -- ./my_application
```

Output includes:
- Per-package and per-CCD power consumption (Watts, time-series)
- `Package Power` vs. `TDP` ratio — values >90 % indicate thermal throttling risk
- `Core Frequency` time-series — frequency drops below base clock confirm P-state intervention

---

## 4. Intel VTune Profiler

### Architecture Overview

VTune (Intel VTune Profiler, formerly VTune Amplifier) is the most comprehensive single-vendor profiler for Intel platforms. It combines hardware PMU access, OS kernel trace correlation, JIT profiler APIs, and GPU EU (Execution Unit) metrics into a unified analysis framework.

VTune's collection backend operates at three levels:
1. **Hardware event-based sampling** via Intel PEBS (Precise Event-Based Sampling) — eliminates skid in IP attribution
2. **PTI (Processor Trace Instrumentation)** — cycle-accurate instruction-level trace replay
3. **SEP (Sampling Enabling Product)** — kernel driver for ring-0 PMU access

### Top-Down Microarchitecture Analysis (TMA)

VTune's TMA implementation follows Intel's documented methodology precisely. The four top-level buckets and their sub-nodes:

```
Pipeline Slots
├── Retiring
│   ├── Light Operations
│   └── Heavy Operations (FP, SIMD, complex)
├── Bad Speculation
│   ├── Branch Mispredicts
│   └── Machine Clears
├── Frontend Bound
│   ├── Fetch Latency
│   │   ├── ICache Misses
│   │   ├── ITLB Overhead
│   │   └── Branch Resteers
│   └── Fetch Bandwidth
│       ├── MITE (legacy decode)
│       └── DSB (Decode Stream Buffer — uop cache)
└── Backend Bound
    ├── Memory Bound
    │   ├── L1 Bound (DTLB overhead, store fwd blocked)
    │   ├── L2 Bound
    │   ├── L3 Bound (L3 hit latency stall)
    │   └── DRAM Bound
    │       ├── MEM BW (bandwidth saturation)
    │       └── MEM Latency (high-latency accesses)
    └── Core Bound
        ├── Divider
        ├── Ports Utilization
        │   ├── Port 0/1 (ALU/SIMD)
        │   ├── Port 2/3 (loads)
        │   └── Port 4/7 (stores)
        └── Serializing Operations
```

```bash
# Command-line collection
vtune -collect performance-snapshot -result-dir ./vtune_ps -- ./my_app
vtune -collect uarch-exploration -knob enable-stack-collection=true \
      -result-dir ./vtune_uarch -- ./my_app
# Report generation
vtune -report summary -result-dir ./vtune_uarch
vtune -report hotspots -result-dir ./vtune_uarch -format csv
```

### Precise Event-Based Sampling (PEBS)

Standard interrupt-based sampling has *skid*: the PMU interrupt fires up to ~100 instructions after the faulting instruction, misattributing the event. PEBS hardware-records the exact IP and register state at the event, eliminating skid for precise hot-instruction attribution.

Key PEBS events:
- `MEM_LOAD_RETIRED.L3_MISS` — precise instruction triggering an L3 miss
- `MEM_TRANS_RETIRED.LOAD_LATENCY` — per-load latency histogram (requires `ldlat` threshold: typical 50–100 cycles)
- `BR_MISP_RETIRED.ALL_BRANCHES` — exact mispredicting branch instruction

```bash
vtune -collect memory-access \
      -knob analyze-mem-objects=true \
      -knob mem-object-size-min-thres=1024 \
      -result-dir ./mem_access -- ./my_app
```

The Memory Access analysis produces a *memory object* view attributing cache misses to specific C++ data structures (requires debug symbols + debug info).

### Intel Processor Trace (PT) and Instruction-Level Tracing

Intel PT records the complete control flow of a process with cycle-accurate timestamps, at hardware cost of ~5 % overhead and ~100 MB/s trace bandwidth:

```bash
vtune -collect itrace -knob enable-stack-collection=true \
      -result-dir ./pt_trace -- ./my_app
# View instruction-level timeline
vtune -report callstacks -result-dir ./pt_trace
```

PT enables:
- **Exact branch target recording** — reconstruct the complete execution path
- **Cycle-accurate timing** — identify latency between any two points
- **Speculation window visualization** — see wrong-path execution

This is invaluable for diagnosing intermittent latency spikes (jitter) in latency-sensitive systems (HFT, real-time control).

### Threading and Concurrency Analysis

```bash
vtune -collect threading -knob sampling-and-wakeup-interval=1 \
      -result-dir ./threading_result -- ./my_app
```

The Threading analysis exposes:
- **CPU utilization timeline** — per-thread utilization across all cores
- **Wait time breakdown** — time spent in `pthread_mutex_lock`, `futex`, `sleep`, I/O wait
- **Lock contention graph** — mutex/rwlock pairs with waiter counts and hold times
- **Spin time** — time in busy-wait loops (identified via `PAUSE` instruction sampling)

Critical metrics for concurrent C++ programs:
- `Effective CPU Utilization` > 85 % × core count = healthy scaling
- `Lock and Sync Functions` > 10 % = synchronization bottleneck
- `Overhead` (OS scheduler, context switches) > 5 % = excessive threading overhead

### GPU Compute Offload Analysis (Intel Xe / oneAPI)

For Intel GPU workloads via OpenCL or SYCL (oneAPI):

```bash
vtune -collect gpu-offload -result-dir ./gpu_offload -- ./my_gpu_app
vtune -collect gpu-hotspots -knob profiling-mode=source-analysis \
      -result-dir ./gpu_hotspots -- ./my_gpu_app
```

GPU analysis provides:
- **EU (Execution Unit) utilization** — active vs. stall vs. idle cycles per EU
- **L3 Cache Hit Rate** for GPU L3 (SLM — Shared Local Memory in Intel nomenclature)
- **PCIe Transfer Bandwidth** — host-to-device and device-to-host throughput
- **Kernel occupancy** — active warps/threads vs. theoretical maximum

### I/O and Storage Analysis

VTune's I/O analysis correlates OS I/O calls with CPU activity:

```bash
vtune -collect io -result-dir ./io_result -- ./my_app
```

Output includes per-call latency distributions for `read`, `write`, `pread`, `pwrite`, `fsync`, and async I/O operations, enabling identification of pathological synchronous-I/O patterns in otherwise asynchronous code.

---

## 5. Windows Performance Analyzer (WPA) and ETW

### ETW Architecture

Event Tracing for Windows (ETW) is a kernel-level, lock-free ring-buffer tracing infrastructure built into the Windows NT kernel since Windows XP. It is the foundation for WPA, WPR (Windows Performance Recorder), xperf, and third-party tools like ProcMon, PerfView, and UIforETW.

ETW operates through a producer–consumer model:

```
Kernel / Drivers / User-mode Providers
          │  (ETW Events via EtwWrite)
          ▼
     ETW Session (ring buffer in kernel)
          │
          ▼
    Consumer (WPR, xperf, custom)
          │  (Writes .etl file)
          ▼
    WPA / xperf / PerfView  (Analysis)
```

Key provider GUIDs of interest:
- `Microsoft-Windows-Kernel-Process` — process/thread creation, exit, priority changes
- `Microsoft-Windows-Kernel-Memory` — page faults, working set changes, pool allocations
- `Microsoft-Windows-Kernel-Disk` — disk I/O requests, completions, latency
- `Microsoft-Windows-Kernel-Network` — TCP/UDP send/receive events
- `Microsoft-Windows-Kernel-Power` — C-state transitions, P-state changes, thermal events
- `Microsoft-Windows-DxgKrnl` — GPU scheduling, context switches, VSync events

### Capturing ETW Traces with WPR

Windows Performance Recorder (WPR) is the preferred collection interface, using XML profile definitions:

```xml
<!-- custom_profile.wprp -->
<WindowsPerformanceRecorder Version="1.0">
  <Profiles>
    <SystemCollector Id="SystemCollector" Name="NT Kernel Logger">
      <BufferSize Value="1024"/>
      <Buffers Value="128"/>
    </SystemCollector>
    <EventCollector Id="EventCollector_CPU" Name="CPU Analysis">
      <BufferSize Value="256"/>
      <Buffers Value="64"/>
    </EventCollector>
    <Profile Id="CpuMemoryIO.Verbose" Name="CPU_Memory_IO"
             Description="CPU scheduling, memory, and I/O analysis"
             DetailLevel="Verbose" LoggingMode="File">
      <Collectors>
        <SystemCollectorId Value="SystemCollector">
          <Keywords>
            <Keyword Value="CpuConfig"/>
            <Keyword Value="CSwitch"/>       <!-- Context switches -->
            <Keyword Value="DiskIO"/>        <!-- Disk I/O -->
            <Keyword Value="DPC"/>           <!-- Deferred Procedure Calls -->
            <Keyword Value="FileIO"/>        <!-- File I/O operations -->
            <Keyword Value="HardFaults"/>    <!-- Page faults requiring disk -->
            <Keyword Value="Interrupt"/>
            <Keyword Value="MemoryInfo"/>
            <Keyword Value="NetworkTrace"/>
            <Keyword Value="Power"/>
            <Keyword Value="ProcessThread"/>
            <Keyword Value="ReadyThread"/>   <!-- Scheduler readyqueue events -->
            <Keyword Value="SampledProfile"/> <!-- CPU sampling -->
            <Keyword Value="SoftFaults"/>
            <Keyword Value="VirtualAlloc"/>
          </Keywords>
        </SystemCollectorId>
      </Collectors>
    </Profile>
  </Profiles>
</WindowsPerformanceRecorder>
```

```powershell
# Start trace
wpr -start custom_profile.wprp -filemode
# Run workload
.\my_application.exe --workload
# Stop and save
wpr -stop output.etl
```

For command-line xperf (legacy but powerful):

```batch
xperf -on PROC_THREAD+LOADER+DISK_IO+HARD_FAULTS+CSWITCH+PROFILE ^
      -stackwalk CSwitch+ReadyThread+SampledProfile ^
      -BufferSize 1024 -MaxBuffers 512 -MaxFile 2048 -FileMode Circular
xperf -stop -d merged.etl
```

### WPA Analysis Panels

WPA's power lies in its composable, zoomable panels:

#### CPU Usage (Sampled) — Flame Graph / Call Tree

The sampled CPU panel shows a statistical call graph built from `SampledProfile` events (default: 1000 Hz).

Essential workflow:
1. Open `CPU Usage (Sampled)` graph
2. Group by `Stack` → `Thread Name` → `Module`
3. Apply "Flame by Weight" view for visual hot-path identification
4. Filter to a specific process; zoom the time axis to an anomalous region
5. Examine `% Weight` column — functions above 5 % warrant investigation

#### CPU Usage (Precise) — Scheduling Analysis

The `CSwitch` events give exact timestamps when each thread is switched off and back on a CPU core:

- `NewThreadWaitReason` — reason the waking thread was waiting:
  - `Executive` — blocked on kernel object (mutex, event, I/O)
  - `WrQueue` — waiting on a work queue
  - `WrUserRequest` — waiting on user-mode I/O
  - `Suspended` — debugger or manual suspend
- **Ready Thread Latency** — time from when a thread becomes ready (unblocked) to when it actually runs. Values > 1 ms on a lightly loaded system indicate scheduler pathology (priority inversion, CPU affinity conflict, NUMA migration).
- **Context Switch Rate** — > 50,000/sec system-wide suggests excessive thread contention or busy-polling inefficiency.

#### Disk I/O Analysis

The Disk I/O panel displays per-request I/O with:
- `Disk Service Time` — time at the storage device
- `I/O Init Time` — queuing time in the OS I/O stack
- `Path Name` — file being accessed
- `I/O Size` — request size in bytes

Red flags:
- `Service Time > 10 ms` on an NVMe drive = storage hardware issue or queue depth saturation
- Irregular I/O size distribution (mixing 512-byte and 64 KB requests) = misaligned access pattern

#### GPU Analysis via DxgKrnl

The `GPU Hardware Queue` and `GPU Software Queue` panels expose:
- Per-context GPU utilization (render, compute, copy engines)
- `DMA Packet Latency` — time from CPU submission to GPU start
- `Preemption events` — GPU context switches, critical for latency analysis
- `VSync events` — correlate with frame boundaries in graphics workloads

#### Memory Analysis

The `Memory` panel group covers:
- **Virtual Memory Snapshots** — working set, private bytes, commit over time
- **Hard Faults (Page-In)** — time-stamped events with latency (disk read required)
- **Soft Faults (Zero-Page/Copy-on-Write)** — cheaper but still zero-initialize cost
- **Pool Allocations** — paged/non-paged pool growth (kernel-mode memory pressure)

### Custom ETW Providers in C++

For application-level tracing integrated with WPA:

```cpp
// Manifest-based ETW provider (preferred over TraceLogging for WPA integration)
#include <evntprov.h>

// {GUID from mc.exe -um MyProvider.man}
REGHANDLE g_etw_handle = 0;
GUID provider_guid = { 0xdeadbeef, ... };

void initialize_etw() {
    EventRegister(&provider_guid, nullptr, nullptr, &g_etw_handle);
}

// Emit an event with custom payload
EVENT_DESCRIPTOR ev_compute_start;
EventDescCreate(&ev_compute_start, 1 /*Id*/, 0 /*Ver*/, 0 /*Ch*/,
                TRACE_LEVEL_INFORMATION, 0 /*Task*/, 0 /*Opcode*/, 0 /*Keyword*/);

void trace_compute_start(uint64_t job_id) {
    EVENT_DATA_DESCRIPTOR data[1];
    EventDataDescCreate(&data[0], &job_id, sizeof(job_id));
    EventWrite(g_etw_handle, &ev_compute_start, 1, data);
}
```

These custom events appear in WPA alongside kernel events, enabling correlation between application-level operations and system-level behavior (e.g., "this job_id=42 submission caused a subsequent burst of hard page faults").

### Windows Kernel Stack Walk

Stack-walking at collection time (the `-stackwalk` xperf flag) records call stacks for every `CSwitch`, `ReadyThread`, and `SampledProfile` event. This enables:
- Exact identification of which code path caused a blocking call
- Attribution of scheduler latency to specific synchronization primitives
- Correlation between CPU sampling and blocking events on the same timeline

Requirement: symbol files (PDB) must be accessible. Configure `_NT_SYMBOL_PATH`:

```batch
set _NT_SYMBOL_PATH=srv*C:\symbols*https://msdl.microsoft.com/download/symbols;C:\MyApp\build\Debug
```

---

## 6. Supplementary Tools

### Linux perf

The Linux `perf` subsystem provides PMU access, tracepoints, kprobes, uprobes, and BPF programs from a single interface:

```bash
# Top-down microarchitecture analysis (requires pmu-tools/toplev)
toplev.py --level 3 -o toplev_report.csv -- ./my_application

# Cache miss profiling with PEBS (Intel) or IBS (AMD)
perf record -e cache-misses:pp -c 1000003 --call-graph dwarf -- ./my_application
perf report --stdio --no-children

# Memory access latency (Intel PEBS load latency)
perf mem record -e ldlat-loads -- ./my_application
perf mem report --sort=mem

# Flame graph generation (via Brendan Gregg's scripts)
perf record -F 999 -g -- ./my_application
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
```

### Valgrind Callgrind + KCacheGrind

Callgrind instruments every instruction via dynamic binary translation, providing exact instruction and cache access counts (no statistical sampling):

```bash
valgrind --tool=callgrind --cache-sim=yes --branch-sim=yes \
         --callgrind-out-file=callgrind.out.%p \
         ./my_application

kcachegrind callgrind.out.<pid>  # GUI visualization
callgrind_annotate --auto=yes callgrind.out.<pid> > annotated.txt
```

Overhead is 10–50×, so use only on bounded, reproducible workloads. The exact counts make it ideal for micro-benchmark comparison where statistical noise would obscure differences.

### NVIDIA Nsight Systems and Nsight Compute

For CUDA workloads, the NVIDIA toolchain provides two complementary perspectives:

**Nsight Systems** — system-wide timeline:
```bash
nsys profile --trace=cuda,nvtx,osrt --sample=cpu \
             --output=profile.nsys-rep ./my_cuda_app
nsys stats profile.nsys-rep
```

**Nsight Compute** — kernel-level micro-architecture:
```bash
ncu --set full --target-processes all \
    --output kernel_report ./my_cuda_app
ncu-ui kernel_report.ncu-rep  # Interactive analysis
```

### AMD ROCm Profiler (rocprof / rocprofv2)

```bash
rocprof --stats --hip-trace \
        --roctx-trace \
        -o rocprof_output.csv \
        ./my_rocm_app

# ATT (Advanced Thread Trace) for GPU instruction-level analysis
rocprof --att kernel_name --att-shader compute \
        -o att_output ./my_rocm_app
```

### Intel Memory Latency Checker (MLC)

MLC measures actual DRAM bandwidth and latency for the specific NUMA topology of the target system:

```bash
mlc --latency_matrix        # NUMA node cross-latency (ns)
mlc --bandwidth_matrix      # NUMA node cross-bandwidth (GB/s)
mlc --peak_injection_bandwidth -b 256  # Peak BW at 256-byte stride
mlc --loaded_latency        # Latency vs. bandwidth trade-off curve
```

This produces system-specific Roofline Model parameters, more accurate than vendor TDP specifications.

---

## 7. Cross-Layer Analysis Methodology

### The Iterative Refinement Loop

```
1. Establish baseline measurement (wall time, throughput, latency)
2. System-wide coarse profiling (TMA level 1, WPA CPU/disk overview)
3. Identify dominant bottleneck category (CPU? Memory? I/O? Sync?)
4. Drill down with category-specific tool
5. Localize to data structure or code region
6. Formulate and implement hypothesis-driven optimization
7. Measure against baseline (A/B with controlled environment)
8. Repeat until target met or diminishing returns
```

### Establishing a Controlled Measurement Environment

Performance measurement noise invalidates comparisons. For reproducible results:

```bash
# Disable CPU frequency scaling (Linux)
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $cpu
done

# Disable TurboBoost (Intel)
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# Set CPU affinity for the workload and profiler
taskset -c 0-7 ./my_application

# Disable NUMA balancing (avoid OS-initiated page migration during measurement)
echo 0 | sudo tee /proc/sys/kernel/numa_balancing

# Flush filesystem page cache before storage benchmarks
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

On Windows, use Process Lasso or PowerShell to set `High Performance` power plan and disable CPU parking:

```powershell
powercfg -setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c  # High Performance GUID
```

### Correlation Across Tool Outputs

Tools produce data in different units and sampling domains. The key correlation technique is *timeline alignment*:

1. Emit custom markers in application code (ETW events, NVTX ranges, ITT tasks)
2. Collect all tools simultaneously with synchronized start timestamps
3. Align timelines in WPA or a custom visualization layer by marker timestamps

Example: correlating a CPU stall spike (WPA `CPU Usage (Sampled)`) with a DRAM bandwidth surge (uProf UMC counters) and an L3 miss rate increase (VTune) identifies a data structure with poor locality that is accessed under a specific code path.

---

## 8. CPU Bottleneck Analysis

### Instruction-Level Parallelism and Execution Port Saturation

Modern out-of-order CPUs issue up to 4–6 µops per cycle. Port saturation occurs when too many µops contend for the same execution port:

```
Intel Golden Cove (Alder Lake P-Core) execution ports:
Port 0: ALU, SIMD FP Mul, AES, Div
Port 1: ALU, SIMD FP Add, Shuffle
Port 2: Load + AGU
Port 3: Load + AGU
Port 4: Store Data
Port 5: ALU, SIMD Shuffle, Branch
Port 6: ALU, Branch, Predicted Jump
Port 7: Store AGU
```

VTune's `Ports Utilization` metric shows the ratio of cycles with each port at 100 % utilization. A `Port 0` bound workload may be alleviated by rewriting vectorized code to balance FP multiplies and adds, or by using `vfmadd` (which uses the FMA pipe rather than consuming both multiply and add port capacity separately).

### Branch Misprediction Analysis

Misprediction cost is 10–20 cycles per mispredict on modern architectures. Identifying the culprit:

```bash
# VTune: identify mispredicting branches
vtune -collect uarch-exploration -result-dir ./branch_analysis \
      -knob enable-stack-collection=true -- ./my_app
vtune -report hotspots -result-dir ./branch_analysis \
      -group-by source-line \
      -filter "Retiring:Bad Speculation/Branch Mispredicts > 0.05"
```

Remediation strategies:
- **Branchless code**: replace predictable-but-mispredicted branches with conditional moves (`cmov`, `blendvps`)
- **Data sorting**: sort input data to create predictable branch patterns
- **Branch target hints**: `__builtin_expect` / `[[likely]]` / `[[unlikely]]` for profile-guided layout
- **Loop unrolling**: reduce branch count in hot loops

### Vectorization and SIMD Efficiency

```bash
# Verify vectorization with compiler reports
g++ -O3 -march=native -fopt-info-vec-optimized -fopt-info-vec-missed \
    my_code.cpp -o my_app 2>&1 | grep "vectorized\|missed"

# Intel ISPC compiler: explicit SIMD programming
ispc kernel.ispc -o kernel.o --target=avx2-i32x8
```

VTune's `SIMD Width Usage` histogram shows the fraction of SIMD instructions using full 512-bit (AVX-512), 256-bit (AVX2), or 128-bit width. A bimodal distribution with high 128-bit usage in a nominally AVX-512 kernel indicates loop remainder handling or alignment penalties requiring explicit alignment attributes:

```cpp
alignas(64) float data[N];  // 64-byte aligned for AVX-512 aligned loads
```

---

## 9. GPU Bottleneck Analysis

### Occupancy vs. IPC Trade-off

GPU performance is governed by the interplay between thread occupancy (active warps/waves per compute unit) and per-instruction throughput. The optimal operating point balances:

- **High occupancy** → hides memory latency through warp switching
- **Low occupancy** → more registers per thread → better IPC → better for compute-bound kernels

Identifying the regime:

```
NSight Compute metrics:
sm__warps_active.avg.pct_of_peak_sustained_active  → Achieved Occupancy (%)
sm__inst_executed.sum / sm__cycles_active.sum       → IPC
l1tex__t_sector_hit_rate.pct                       → L1 Hit Rate (%)
lts__t_sector_hit_rate.pct                         → L2 Hit Rate (%)
dram__bytes.sum.per_second                         → DRAM Bandwidth (GB/s)
```

A kernel with 20 % achieved occupancy and high IPC is compute-bound; increasing occupancy (reducing register count with `--maxrregcount`) would decrease IPC without benefiting throughput. A kernel with 80 % occupancy and 50 % memory throughput is latency-bound; restructuring data access to coalesce reads resolves it.

### PCIe Bottleneck Analysis

Data transfer between CPU and GPU over PCIe is a common bottleneck:

```bash
# NVIDIA bandwidth test
bandwidthTest --memory=pinned --mode=range --start=1 --end=536870912 --increment=1
# AMD bandwidth test (ROCm)
rocm-bandwidth-test -t 0 -d 0 -b  # CPU→GPU
```

Mitigation: use pinned (page-locked) host memory for transfers; overlap transfers with computation using CUDA streams or HIP streams; use NVLink or AMD Infinity Fabric for multi-GPU configurations.

### GPU Roofline Analysis

NSight Compute's Roofline chart plots each kernel as a point at (Arithmetic Intensity, Achieved Performance). Points below the memory-bandwidth roof are memory-bound; points below the compute roof are compute-bound; points at the peak are roofline-limited.

```bash
ncu --metrics sm__warps_active.avg.pct_of_peak_sustained_active,\
              l1tex__t_requests_pipe_lsu_mem_global_op_ld.sum,\
              sm__sass_thread_inst_executed_op_ffma_pred_on.sum,\
              gpu__time_duration.sum \
    --output roofline_data ./my_cuda_app
```

---

## 10. Memory Subsystem Analysis

### Cache Hierarchy Profiling

The cache hierarchy introduces latency at each level:

| Level | Typical Latency | Typical Size |
|---|---|---|
| L1 D-Cache | 4–5 cycles | 32–64 KB |
| L2 Cache | 12–15 cycles | 256 KB – 1 MB |
| L3 / LLC | 30–50 cycles | 4–64 MB |
| DRAM (local NUMA) | 60–80 ns | GB–TB |
| DRAM (remote NUMA) | 120–200 ns | |

Working set analysis to identify which cache level is under pressure:

```bash
# Perf stat sweep across stride sizes to map cache hierarchy
for stride in 64 128 256 512 1024 2048 4096 8192 16384 32768 65536 131072; do
    perf stat -e cache-references,cache-misses,L1-dcache-load-misses,\
              LLC-load-misses,LLC-loads \
    ./cache_sweep --stride $stride --size 256M 2>&1 | \
    awk -v s=$stride '/cache-misses/{printf "stride=%d misses=%s\n", s, $1}'
done
```

The miss rate vs. stride curve reveals the cache size boundary as a sharp inflection point.

### False Sharing Detection

False sharing occurs when two threads write to different variables that reside in the same cache line (64 bytes on x86). The cache coherence protocol bounces the line between cores, creating contention invisible at the software level.

Detection:
```bash
# Linux perf c2c (Cache-to-Cache) tool
perf c2c record -g -F 1000 -- ./my_threaded_app
perf c2c report --stdio --details
```

Output highlights *hitm* (Hit Modified) events — cache line loads that hit a line in Modified state in another core's L1. High HITM count on a specific line address traces back to a hot shared structure.

Mitigation: pad structures to cache line boundaries:

```cpp
struct alignas(64) PerThreadCounter {
    std::atomic<uint64_t> value;
    char padding[64 - sizeof(std::atomic<uint64_t>)];
};
```

### DRAM Bandwidth Saturation

DRAM bandwidth is a finite resource shared across all cores. On modern systems (DDR5, 8-channel EPYC), theoretical peak is ~300–600 GB/s, but sustained read bandwidth is typically 70–80 % of peak.

Measuring with uProf (AMD UMC counters):

```bash
AMDuProfCLI collect \
    --event "umc_data_slot_reads:1000,umc_data_slot_writes:1000" \
    --duration 10 \
    -o ./bw_profile -- ./my_application

# Convert counts to GB/s:
# BW = (reads + writes) * 64 bytes * counter_period / elapsed_time
```

When DRAM bandwidth is saturated, optimization strategies include:
1. **Cache blocking (tiling)** — restructure loops so the working set fits in L2/L3
2. **Data compression** — store floats as half-precision (FP16) where precision allows
3. **Streaming stores** — bypass cache for write-only data: `_mm_stream_si128`
4. **Prefetching** — software prefetch hints 200–400 cycles ahead of access

### TLB Pressure Analysis

The Translation Lookaside Buffer (TLB) caches virtual-to-physical page mappings. With 4 KB pages, a 1 GB working set requires 262,144 TLB entries — far exceeding L2 STLB capacity (~2048 entries on Intel). Each TLB miss triggers a page walk (4 memory accesses on x86-64, longer with 5-level paging).

Mitigation:
```cpp
// Use transparent huge pages (2 MB pages) to reduce TLB pressure by 512×
void* buf = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
// Or advise for THP
madvise(buf, size, MADV_HUGEPAGE);
```

VTune's `DTLB Load/Store Miss` metrics and the `Memory Bound > L1 Bound > DTLB Overhead` TMA node quantify the impact before and after.

---

## 11. Storage I/O Analysis

### I/O Stack Layers

```
Application (POSIX read/write, Win32 ReadFile)
        │
OS Page Cache (vfs_read → pagecache_read_folio)
        │
Block Layer (I/O Scheduler: none/mq-deadline/kyber)
        │
NVMe Driver (nvme_queue_rq)
        │
NVMe Controller (PCIe DMA)
        │
NAND Flash (read latency: ~80 µs; write: ~100–500 µs)
```

Each layer adds latency and queuing effects. WPA and perf can instrument every layer.

### Queue Depth and Latency Analysis

NVMe SSDs achieve maximum throughput at Queue Depth (QD) 32–128. Sequential read benchmarking:

```bash
# fio: establish baseline I/O characteristics
fio --name=seq_read --ioengine=libaio --direct=1 \
    --rw=read --bs=128k --numjobs=1 \
    --iodepth=1 --iodepth=32 --iodepth=128 \
    --filename=/dev/nvme0n1 --size=10G \
    --runtime=30 --time_based --output-format=json
```

The latency vs. QD curve reveals whether the storage device or the I/O stack is the bottleneck. A flat latency up to QD=32 followed by sharp rise at QD=64 indicates device saturation.

### blktrace + blkparse (Linux)

For kernel-level I/O tracing:

```bash
blktrace -d /dev/nvme0n1 -o trace -- sleep 30 &
./my_application
wait
blkparse trace -o trace.txt
btt -i trace.blktrace.0 -l trace_btt.txt  # Latency breakdown
```

`btt` output sections:
- **Q2G** — time from request queued to I/O dispatched to driver
- **G2I** — time in I/O scheduler
- **I2D** — time from driver to device activation
- **D2C** — device service time (actual hardware latency)
- **Q2C** — total I/O latency

### io_uring vs. epoll vs. Synchronous I/O

For applications where storage latency is in the critical path, the I/O submission mechanism matters:

```cpp
// io_uring: zero-syscall I/O batching
io_uring_queue_init(QUEUE_DEPTH, &ring, IORING_SETUP_SQPOLL);

// Submit batch of reads without syscall (via SQ polling)
for (int i = 0; i < batch; i++) {
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf[i], BUF_SIZE, offset[i]);
    sqe->user_data = i;
}
io_uring_submit(&ring);  // Single syscall for entire batch
```

Profiling shows `io_uring` SQPOLL mode (kernel thread polls SQ ring) reduces per-operation syscall overhead to zero, achieving near-NVMe-hardware latency.

---

## 12. Software-Layer Profiling

### Allocator Performance

Dynamic memory allocation is a frequent hidden bottleneck in systems code. `malloc`/`free` under thread contention involves:
- **Lock contention** on arena locks (glibc ptmalloc)
- **TLB shootdowns** from `munmap` on large frees
- **Fragmentation** inflating working set beyond cache capacity

Analysis:
```bash
# Heap profiling with Google gperftools
LD_PRELOAD=/usr/lib/libprofiler.so HEAPPROFILE=/tmp/heap_prof ./my_app
pprof --pdf ./my_app /tmp/heap_prof.0001.heap > heap.pdf

# Allocation count and size distribution (custom instrumentation)
valgrind --tool=massif --pages-as-heap=yes --detailed-freq=1 ./my_app
ms_print massif.out.<pid> | head -100
```

Mitigation strategies: tcmalloc or jemalloc (lower lock contention, better NUMA awareness), memory pool allocators for hot paths, `std::pmr::monotonic_buffer_resource` for scoped allocation.

### Lock Contention and Synchronization Analysis

```bash
# Mutrace: mutex contention analysis
LD_PRELOAD=/usr/lib/libmutrace.so ./my_app 2>&1 | sort -k3 -n -r | head -20

# Output format:
# Mutex 0x7f3a4c000b80: 42873 lock() calls, 8931 contentions (20.8%), avg wait 14.3 µs
```

High contention (> 5 %) on hot locks suggests:
- **Lock granularity reduction**: split one coarse lock into per-item locks
- **Lock-free data structures**: `std::atomic` / `std::atomic<std::shared_ptr>`
- **Read-copy-update (RCU)**: for read-heavy, write-rare patterns
- **Seqlock**: ultra-low-reader-overhead for small frequently-updated data

### Compiler Optimization Verification

Profiling without verifying that the compiler actually applied expected optimizations wastes time:

```bash
# Check assembly output for vectorization
objdump -d --disassembler-color=on ./my_app | grep -A5 "vmovaps\|vfmadd\|vpgather"

# BOLT: post-link binary optimization using profile data
perf record -e cycles:u -j any,u -- ./my_app
perf2bolt ./my_app -p perf.data -o bolt.fdata
llvm-bolt ./my_app -o ./my_app.bolt \
    -data=bolt.fdata \
    -reorder-blocks=ext-tsp \
    -reorder-functions=hfsort \
    -split-functions \
    -split-all-cold \
    -dyno-stats
```

BOLT reorders basic blocks and functions based on execution frequency, improving instruction cache hit rates by 5–15 % on large binaries.

---

## 13. Optimization Strategy Synthesis

### Decision Tree: Bottleneck → Strategy

```
TMA shows Frontend Bound > 20 %?
  ├─ ICache Miss > 10 % → Function reordering (BOLT), hot/cold splitting
  ├─ DSB Miss > 10 %   → Reduce code size in hot loops, avoid complex decode
  └─ Branch Resteers   → Indirect call devirtualization, branch prediction hints

TMA shows Bad Speculation > 10 %?
  ├─ Machine Clears    → Avoid store-load forwarding stalls, aliasing
  └─ Mispredictions    → Branchless code, sorted inputs, PGO

TMA shows Backend Bound > Memory Bound?
  ├─ DRAM Bound > 15 % → Cache blocking, data layout (SoA), prefetch
  ├─ L3 Bound > 15 %   → Reduce working set, better locality
  ├─ L1 Bound          → DTLB: huge pages; Store Fwd: alignment
  └─ Bandwidth         → Non-temporal stores, compression

TMA shows Backend Bound > Core Bound?
  ├─ Port 0/1 Sat.     → Rebalance SIMD ops, FMADD fusion
  ├─ Divider > 5 %     → Replace division with reciprocal multiply
  └─ Serializing       → Reduce CPUID, MFENCE, LFENCE usage

WPA shows Lock Contention > 10 %?
  ├─ Single hot mutex  → Shard / lock-free replacement
  ├─ Many mutexes      → Audit lock hierarchy, avoid lock nesting
  └─ Reader-biased     → RWLock or RCU

WPA shows Disk I/O Latency > 1 ms on hot path?
  ├─ Random small I/O  → Application-level read-ahead, larger I/O sizes
  ├─ Sync on write     → WAL pattern, async fsync
  └─ Cache miss        → Explicit file cache warming at startup
```

### Quantifying Optimization Impact

Always measure speedup relative to a pinned baseline. Use statistical significance testing for micro-benchmarks:

```cpp
// Google Benchmark: statistically rigorous micro-benchmarking
#include <benchmark/benchmark.h>

static void BM_MyKernel(benchmark::State& state) {
    std::vector<float> data(state.range(0));
    // Setup...
    for (auto _ : state) {
        my_kernel(data.data(), data.size());
        benchmark::ClobberMemory();
    }
    state.SetBytesProcessed(int64_t(state.iterations()) *
                            int64_t(state.range(0)) * sizeof(float));
}
BENCHMARK(BM_MyKernel)->RangeMultiplier(4)->Range(1<<10, 1<<24)
                       ->MinTime(5.0)  // 5s minimum for statistical stability
                       ->UseRealTime();
BENCHMARK_MAIN();
```

`SetBytesProcessed` enables automatic GB/s reporting for comparison against the Roofline bound.

---

## 14. Advanced Patterns and Anti-Patterns

### Hardware Prefetcher Interaction

Hardware prefetchers (stride, stream, next-line) work best on regular, strided access patterns. Profiling must distinguish hardware prefetch effectiveness from software access efficiency:

```
VTune metric: L3_HIT_LATENCY vs. L3_MISS_LATENCY
uProf metric: l2_pf_hit_in_l3 (prefetch hit in L3) vs. l2_pf_miss (prefetch miss → DRAM)
```

When the prefetcher is ineffective (pointer chasing, random access), software prefetch with carefully chosen distance:

```cpp
// Prefetch distance: (memory_latency_cycles / iteration_cycles) iterations ahead
const int PREFETCH_DIST = 20;
for (int i = 0; i < n; i++) {
    __builtin_prefetch(&data[idx[i + PREFETCH_DIST]], 0, 1);  // read, L2 locality
    process(data[idx[i]]);
}
```

### Profile-Guided Optimization (PGO)

PGO uses collected execution profiles to guide compiler decisions about inlining, branch layout, and function ordering:

```bash
# Stage 1: Instrumented build
clang++ -O2 -fprofile-instr-generate my_app.cpp -o my_app_inst

# Stage 2: Profile collection with representative workload
./my_app_inst --representative-workload
llvm-profdata merge default.profraw -o my_app.profdata

# Stage 3: Optimized build using profile
clang++ -O3 -fprofile-instr-use=my_app.profdata \
        -fprofile-use my_app.cpp -o my_app_pgo
```

Typical PGO gain: 5–15 % for CPU-bound workloads with heterogeneous code paths.

### Speculative Execution Mitigations Overhead

Post-Spectre/Meltdown kernel patches (`KPTI`, `Retpoline`, `IBRS`, `STIBP`) add measurable overhead:

- `KPTI` — kernel page-table isolation: ~10–30 % overhead on syscall-heavy workloads
- `Retpoline` — indirect branch replacement: ~2–5 % on indirect-call-heavy code
- `IBRS` — Indirect Branch Restricted Speculation: significant on context-switch-heavy loads

VTune's `System Overview` analysis separates `Kernel Time` from `User Time`. Abnormally high `Kernel Time` (> 20 % on a compute workload) with many context switches warrants measuring mitigation overhead:

```bash
# Temporarily disable mitigations for baseline measurement (NEVER in production)
echo "mitigations=off" | sudo tee /sys/kernel/security/lsm

# Quantify mitigation overhead
perf stat -e cpu-clock,context-switches,cs,migrations,page-faults,\
          cycles,instructions,branches,branch-misses \
          ./my_app
```

---

## 15. Reference Metrics Cheat Sheet

### CPU Health Indicators

| Metric | Healthy | Investigate | Critical |
|---|---|---|---|
| IPC (modern OoO core) | > 2.5 | 1.0–2.5 | < 1.0 |
| Branch Mispredict Rate | < 1 % | 1–5 % | > 5 % |
| L1 Hit Rate | > 95 % | 85–95 % | < 85 % |
| L3 Hit Rate | > 80 % | 60–80 % | < 60 % |
| Frontend Stall % | < 10 % | 10–20 % | > 20 % |
| Memory Bound % (TMA) | < 15 % | 15–30 % | > 30 % |

### Memory Subsystem Indicators

| Metric | Healthy | Investigate | Critical |
|---|---|---|---|
| DRAM BW Utilization | < 50 % peak | 50–80 % | > 80 % |
| DTLB Miss Rate | < 0.1 % | 0.1–1 % | > 1 % |
| NUMA Cross-Traffic | < 10 % | 10–30 % | > 30 % |
| Cache Line Utilization | > 80 % | 50–80 % | < 50 % |

### GPU Indicators

| Metric | Healthy | Investigate | Critical |
|---|---|---|---|
| SM / EU Utilization | > 70 % | 40–70 % | < 40 % |
| L1 Hit Rate | > 80 % | 50–80 % | < 50 % |
| Achieved Occupancy | > 50 % | 25–50 % | < 25 % |
| PCIe BW Utilization | < 60 % | 60–85 % | > 85 % |

### Storage I/O Indicators

| Metric | Healthy | Investigate | Critical |
|---|---|---|---|
| NVMe Read Latency (P99) | < 200 µs | 200–1000 µs | > 1 ms |
| NVMe Queue Depth Avg | 1–32 | 32–64 | > 64 (or < 1) |
| Page Cache Hit Rate | > 95 % | 80–95 % | < 80 % |
| Hard Fault Rate | < 100/sec | 100–1000/sec | > 1000/sec |

---

*Document prepared for expert reference. All tool versions and metrics reflect tool capabilities as of 2024–2025. Hardware PMU event names vary between CPU microarchitecture generations; consult vendor optimization manuals (Intel® 64 and IA-32 Architectures Optimization Reference Manual; AMD Processor Programming Reference for Family 19h) for generation-specific event definitions.*
