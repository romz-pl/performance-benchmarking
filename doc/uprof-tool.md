# AMD uProf — Performance Analysis Tool Suite

## Overview

AMD uProf (pronounced *MICRO-prof*) is a performance analysis tool suite for x86-based applications running on Windows, Linux, and FreeBSD operating systems. It provides performance metrics specifically tailored for AMD "Zen"-based processors and AMD Instinct™ MI Series accelerators, enabling developers to better understand performance bottlenecks, identify optimization scope, and evaluate the impact of improvements.

As a Lead Software Performance Engineer at AMD, uProf is your primary first-party instrument for the full performance analysis lifecycle across CPU, GPU, memory, storage, and software layers — precisely the scope described in the job responsibilities.

---

## Supported Languages and Environments

uProf supports a broad range of application environments and languages, including C, C++, Fortran, Assembly, Java, Python, and .NET. This makes it applicable across the heterogeneous workloads you will encounter — from native systems code and HPC kernels to AI/ML Python pipelines.

---

## Core Profiling Modes

### 1. CPU Profiler — Statistical Sampling

The AMD uProf CPU profiler follows a statistical sampling-based approach to collect profile data and identify performance bottlenecks in applications. Data collection can be triggered by OS timer, core PMC (Performance Monitoring Counter) events, and IBS (Instruction-Based Sampling). It provides a Hotspots profile type to identify the hottest inclusive and exclusive time-consuming functions, and performs callstack stitching of OpenMP applications to provide accurate total inclusive time for OpenMP compute functions. It also provides mixed-mode callstack analysis of Python applications.

#### Instruction-Based Sampling (IBS)

IBS is AMD's hardware-level precise sampling mechanism — conceptually analogous to Intel's PEBS, but architecturally distinct. Unlike PMU sampling, which attributes events to nearby instructions due to out-of-order execution skid, IBS attributes events to the exact instruction that caused them — with no skid. This is critical for accurate attribution of cache misses, TLB faults, and memory latency to specific lines of source code.

Recent versions added new IBS metrics, such as `IBS_[LD,ST]_L1_DTLB_REFILL_LAT` for Zen 4 and Zen 5 platforms, enabling analysis of TLB-related load and store bottlenecks.

#### PMC-Based Sampling and Multiplexing

Hardware Performance Monitoring Counters allow counting of specific micro-architectural events — cache hits/misses, branch mispredictions, instruction retirement, stall cycles, and more. uProf handles PMC multiplexing so that many more events than the physical counter count can be measured within a single profiling run, reducing the need for repeated experiments.

### 2. Threading Analysis

Threading analysis profile types help visualize thread state timelines and workload behavior, thread concurrency, and system-wide performance. This directly supports the job requirement around concurrent programming, multi-threaded systems, and performance validation — allowing you to detect lock contention, idle threads, synchronization overhead, and scheduling inefficiencies.

Recent releases introduced an "Unused Threads" metric in inefficiency metrics to handle profile runs with varying thread counts, and added Thread Name and TID tooltips in the All Thread Timeline for improved thread-level observability.

### 3. GPU Profiler

The GPU profiler provides performance statistics on GPU hardware components, kernels, and dispatches. On Windows systems, AMD uProf is one of the primary tools available to profile CPU and GPU codes targeting AMD "Zen"-based processors and AMD Instinct™ GPUs. For the Lead Performance Engineer role, this is essential when characterising workloads across the full CPU–GPU–NPU execution stack.

### 4. Power Profiling

With the AMD uProf power profiler you can monitor the frequency, thermal, and energy metrics of various components in the system, with a live timeline graph of various metrics available in the TIMECHART page. Power and thermal awareness is increasingly important for platform evaluation at AMD scale.

### 5. Remote Profiling

uProf supports remote profiling — connecting to remote Linux systems from a Windows host system, triggering collection and translation of data on the remote system, and reporting it in the local GUI. This is directly applicable to benchmarking infrastructure work on server, cloud, and HPC clusters.

### 6. System Analysis

uProf includes System Analysis capabilities to monitor system-level performance metrics and detect hotspots and micro-architectural issues in source code.

---

## Roofline Model and Arithmetic Intensity

uProf integrates a roofline modelling capability via the `AMDuProfPCM` companion tool. You can measure your code's arithmetic intensity (FLOPS per bytes transferred) and plot it on the roofline: if the workload is below the sloped line, it is memory bound; if below the flat ceiling, it is compute bound. Memory-bound workloads call for improved data locality, prefetching, or reduced data size; compute-bound workloads benefit from SIMD, reduced instruction count, or improved ILP.

This roofline workflow is a direct bridge to the job requirement for SIMD and vectorization analysis — uProf provides the evidence base for deciding whether hardware-aware SIMD optimisation will yield tangible throughput gains.

---

## Profile Configuration Types (CLI)

uProf is operated in two stages — collection and analysis — and offers multiple named configuration presets:

| Config flag | Purpose |
|---|---|
| `--config tbp` | Time-Based Profiling — identifies hotspot functions by wall-clock time |
| `--config assess` | Overall performance assessment — finds potential issues for deeper investigation |
| `--config ibs` | Instruction-Based Sampling — precise, skid-free event attribution |
| `--config callstack` | Full callstack capture — identifies where execution time is spent hierarchically |
| ITLB / I-cache configs | Detects poor instruction cache locality and TLB behaviour |

Results from `AMDuProfCLI collect` are saved to a timestamped output directory and can be analysed either via `AMDuProfCLI report` (producing a summary CSV) or loaded into the graphical GUI for interactive exploration.

---

## Toolchain Components

| Component | Role |
|---|---|
| `AMDuProf` (GUI) | Interactive visual analysis, timeline views, source annotation |
| `AMDuProfCLI` | Scriptable command-line collection and reporting, integration into CI/automation pipelines |
| `AMDuProfPCM` | Hardware performance counter monitoring; PCIe metrics; roofline data collection |
| `AMDuProfModelling.py` | Python plotting script for roofline visualisation output |
| `AMDSystemCheck` | Linux utility collecting OS, BIOS, and platform topology details for diagnostic context |

---

## HPC and Parallel Workload Support

uProf provides improved MPI and OpenMP reports. For large-scale parallel workloads — relevant to benchmarking infrastructure at AMD — uProf supports per-rank analysis and parallel profiling sessions where data volume itself becomes a second pipeline to manage.

---

## Backend and Scalability (uProf 5.3, May 2026)

uProf 5.3, released in May 2026, switched the default storage backend from SQLite to DuckDB (while retaining SQLite for compatibility), delivering significantly faster translation of CPU profiling data for modules with inline functions, lower Python profiling overhead during long runs, and shorter report generation times for large sessions. Visualization improvements include updated `AMDuProfPCM` HTML reports and a Linux timeline visualization for Function Tracing sessions in the GUI.

This matters operationally: when analysing many threads, long runtimes, parallel ranks, or mixed CPU/accelerator workloads, the database, translation, and reporting pipeline becomes a bottleneck in its own right.

---

## Virtualisation Support

uProf includes vIBS support for KVM and the `AMDSystemCheck` utility under Linux, which collects details about the operating system, BIOS, and platform topology. For cloud, lab, and server environments this is significant, because performance problems there often arise from a mix of hardware, firmware, hypervisor, and operating system interactions.

---

## Relationship to the Job Description

| JD requirement | uProf capability |
|---|---|
| System performance analysis and optimization | CPU/GPU/power profiling, hotspot detection, micro-architectural analysis |
| SIMD / vectorization concepts | Roofline modelling, arithmetic intensity measurement, compute vs. memory bound classification |
| CPU, GPU, NPU execution, memory hierarchy, I/O | Multi-layer profiling across all platform components |
| Concurrent programming, multi-threaded systems | Thread timeline analysis, OpenMP callstack stitching, unused thread metrics |
| Automation frameworks and benchmarking at scale | `AMDuProfCLI` for scripted, pipeline-integrated collection; DuckDB backend for large-session scalability |
| Cross-platform development (Windows, Linux) | Native support on both; remote profiling from Windows to Linux targets |
| Root-cause analysis | IBS precise attribution, PMC event correlation, TLB and cache miss breakdown |

---

## Positioning vs. Peer Tools

uProf fills the same conceptual niche as Intel VTune Profiler on the AMD platform. AMD uProf is free and works on all AMD Zen processors; both uProf and VTune require root or administrator access for hardware counter access. Where VTune is the natural choice on Intel silicon, uProf is the authoritative, hardware-aware tool for AMD Zen and Instinct platforms — making deep familiarity with it a core competency for this role.

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: In the context of the attached job description, thoroughly describe AMD uProf tool. Write the description in Markdown format.
