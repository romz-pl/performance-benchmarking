# AMD Benchmarking and Automation System


## 1. Overview and Strategic Context

AMD's benchmarking and automation infrastructure is not a single monolithic tool; it is a layered ecosystem spanning CPU, GPU, NPU, and memory subsystems, operating across Windows, Linux, and FreeBSD. The system serves two interlocking purposes:

- **Internal product validation** — continuous performance characterisation of pre-silicon and post-silicon platforms (Zen 5, RDNA 4, CDNA 4/5, NPU blocks in Ryzen AI APUs) before they reach market.
- **External competitive substantiation** — producing publishable data for industry-standard benchmarks (SPEC CPU, MLPerf, TPCx-AI) that support claims made in product launches and partner engagements.

The Warsaw team sits within AMD's broader global performance engineering organisation, meaning day-to-day work feeds directly into product roadmap decisions for platforms such as EPYC Turin (Zen 5), EPYC Venice (Zen 6, ~2026), Instinct MI350/MI400, and Radeon RX 9000-series GPUs.

---

## 2. The Profiling and Measurement Tool Stack

AMD's publicly documented profiling stack is organised into three tiers: a low-level SDK, a suite of Linux-native ROCm tools, and a cross-platform tool for Windows/Linux.

### 2.1 rocprofiler-sdk — The Foundation Layer

`rocprofiler-sdk` is the low-level library shipped with ROCm that underpins almost all GPU profiling on AMD hardware. It provides the API for device-activity tracing and hardware performance-counter collection, replacing the now-deprecated `rocprofiler` and `roctracer` libraries. All higher-level tools — `rocprofv3`, `rocprof-sys`, `rocprof-compute` — call into this SDK. The legacy tools `rocprof` and `rocprofv2` are scheduled to reach end-of-support by the end of Q2 2026.

### 2.2 rocprofv3 (ROCm GPU Trace Tool)

`rocprofv3` is the primary command-line GPU profiling tool, built directly on `rocprofiler-sdk`. Key capabilities:

- Collects HIP API traces, HSA API traces, offloaded kernel activity, memory copies, scratch-memory usage, and marker API annotations.
- Collects raw device hardware counters for per-kernel performance analysis.
- Since ROCm 7.0, writes profiling data to a **SQLite3-based database** by default (replacing CSV-centric legacy output); a separate `rocpd` tool then converts to CSV, OTF2, or Perfetto trace formats.
- Recommended for **focused GPU hotspot analysis** in multi-process MPI runs (the profiler is launched from within the MPI environment).

### 2.3 rocprof-sys (ROCm Systems Profiler, formerly Omnitrace)

`rocprof-sys` is designed for **system-wide, unified trace collection** — CPU, GPU, and MPI communication in a single pass. It:

- Instruments applications via binary rewriting or library interposition, without requiring source changes.
- Supports C, C++, Fortran, HIP, OpenCL, and Python.
- Collects call-stack samples (sampling-based profiling) alongside GPU kernel timelines.
- Outputs `.proto` Perfetto timeline traces for visual analysis in the Perfetto UI.
- Is the recommended first step when benchmarking an unfamiliar application: collect a full system trace, identify CPU/GPU hotspots, then drill down with `rocprofv3` or `rocprof-compute`.

### 2.4 rocprof-compute (ROCm Compute Profiler, formerly Omniperf)

`rocprof-compute` is used for **low-level roofline analysis and kernel micro-architecture profiling** on AMD Instinct (CDNA) GPUs. It:

- Runs multiple profiling passes to collect the complete set of hardware performance counters (single-pass collection is insufficient for full counter coverage on CDNA hardware).
- Generates roofline plots that distinguish compute-bound from memory-bandwidth-bound kernels.
- Integrates with a web-based or CLI dashboard for interactive inspection of per-kernel metrics (VALU utilisation, LDS bandwidth, L2 hit rate, HBM bandwidth, etc.).
- Is the tool of choice once `rocprofv3` has identified the kernel(s) of interest.

### 2.5 AMD uProf — Cross-Platform CPU and GPU Profiler

`uProf` (AMD MICRO-prof) is AMD's cross-platform profiling tool, covering **Windows, Linux, and FreeBSD**, and targeting both Zen-based CPUs and Instinct GPUs. It is the only AMD-native tool that works on Windows for Instinct GPU profiling. Current version as of mid-2026 is **5.3** (released 12 May 2026).

Key capabilities:

- **CPU Profiler**: statistical sampling via OS timer, core PMC (Performance Monitoring Counter) events, and IBS (Instruction-Based Sampling). Identifies hotspot functions, inclusive/exclusive time, and call-graph structure. Supports C, C++, Fortran, Java, Python, and .NET.
- **Threading Analysis**: visualises thread-state timelines, detects lock contention, and provides callstack stitching for OpenMP regions.
- **Power Profiling**: tracks CPU package power, per-core frequency, and thermal state alongside performance counters.
- **System Analysis** (`AMDuProfPCM`): platform-level metrics — memory bandwidth, PCIe utilisation, I/O throughput. PCIe metrics for Zen 3 server platforms were added in 5.3.
- **GPU Profiler**: GPU hardware component statistics, kernel dispatch information, and memory access patterns for Instinct (CDNA) GPUs.
- **Remote Profiling**: connect from a Windows host to a remote Linux system, trigger collection, and analyse locally in the GUI.
- **Roofline**: available for Zen-based CPUs (compute vs. memory-bandwidth characterisation).
- **Virtualisation support**: vIBS for KVM; `AMDSystemCheck` utility for BIOS/OS/topology inventory in cloud/lab environments.

**Version 5.3 highlights** (May 2026):
- Default backend switched from SQLite to **DuckDB** (SQLite retained for compatibility), substantially reducing report-generation time for large sessions.
- Faster inline-function resolution during CPU data translation.
- Lower Python-profiling overhead for long runs.
- Linux timeline visualisation for Function Tracing sessions in the GUI.
- Per-rank MPI analysis in HTML reports.
- New IBS metric `IBS_[LD,ST]_L1_DTLB_REFILL_LAT` for TLB-related load/store bottleneck analysis on Zen 4 and Zen 5.
- New `--session-name` CLI option; per-thread waiting-time collection on Linux.

### 2.6 Radeon GPU Profiler (RGP) and the Radeon Developer Tool Suite (RDTS)

For **RDNA graphics workloads** (gaming, graphics pipeline, shaders), the primary tool is the **Radeon GPU Profiler (RGP)**, part of the Radeon Developer Tool Suite. The RDTS was updated in 2026 to support AMD Radeon RX 9000-series (RDNA 4) GPUs and includes:

- **RGP**: frame- and dispatch-level GPU trace with pipeline-stall analysis.
- **RGA (Radeon GPU Analyzer) v2.12**: offline compiler and ISA disassembly tool for DirectX, Vulkan, SPIR-V, OpenGL, and OpenCL; enhanced ISA view for RDNA and CDNA in the 2026 update.
- **RRA (Radeon Raytracing Analyzer) v1.8**: ray-tracing BVH traversal performance analysis; RX 9000-series support added.
- **RGD (Radeon GPU Detective) v1.5**: post-mortem GPU crash analysis using DirectX debug information.

---

## 3. Industry-Standard Benchmark Suites Used at AMD

### 3.1 SPEC CPU 2017 and SPEC CPU 2026

AMD uses **SPEC CPU 2017** extensively for published SPECrate and SPECspeed results — the 2P EPYC 9965 achieved a `SPECrate2017_int_base` of 3230, compared to 2510 for the Intel Xeon 6980P. **SPEC CPU 2026**, released in 2026, extends coverage with explicit bare-metal cloud support and more multithreaded integer content, and AMD has stated public support for it as the authoritative CPU performance reference.

### 3.2 MLPerf Inference and Training

AMD participates in **MLCommons MLPerf** as a founding member. Key results:

- **MLPerf Inference v6.0** (April 2026): AMD Instinct MI355X (CDNA 4, 3nm, 288 GB HBM3E, FP4/FP6 support) submitted highly competitive results on Llama 2 70B and other data-centre inference tests.
- **MLPerf Training v6.0** (June 2026): AMD submitted results demonstrating training infrastructure advances on Instinct GPUs under ROCm.
- The benchmark automation pipeline for MLPerf submissions involves Python-based orchestration, multi-node HIP/ROCm workloads, and integration with the `rocprofiler-sdk` stack for performance validation.

### 3.3 TPCx-AI

AMD uses a multi-instance variant of the **TPCx-AI** benchmark (not directly comparable to published TPC results) for CPU AI-inference characterisation. The EPYC 9965 (192-core) in a 2P configuration achieves approximately 3.8× more AI test cases per minute than 2P Intel Xeon Platinum 8592+.

### 3.4 Workload-Specific Benchmarks

For competitive comparisons (particularly for EPYC Venice vs. NVIDIA Vera and Intel Xeon 6), AMD's performance labs run:

- **NGINX + WRK** — web-serving throughput under sustained concurrent load.
- **redis-benchmark** — high-speed in-memory key-value operations.
- **Memcached + memtier_benchmark** — in-memory caching and analytics.
- **SPECjbb2015-derived workloads** — server-side Java throughput and latency.
- **TPROC-C (TPC-C proxy on MySQL)** — OLTP transactional throughput.
- **HPC codes**: LAMMPS, HPCG, NAMD, OpenFOAM, GROMACS — validated under Ubuntu 24.04 with 6.8.0-40-generic kernel.

---

## 4. The Profiling Decision Framework

AMD published a structured three-question framework for selecting the right tool, which is the basis of the internal benchmarking workflow:

1. **Where should I focus?** → Use `rocprof-sys` (Linux) or `uProf` (Windows/Linux) to collect a system-wide trace and identify CPU/GPU hotspots.
2. **How well am I using the hardware?** → Roofline profiling: `rocprof-compute` for CDNA GPUs, `uProf` for Zen CPUs. Determine whether the bottleneck is compute-bound or memory-bandwidth-bound.
3. **Why am I seeing this performance?** → Collect hardware counters with `rocprofv3` (kernel-level), `rocprof-compute` (micro-architectural detail), or `uProf` (CPU PMC events and IBS).

---

## 5. Automation and Infrastructure Patterns

Although AMD's internal automation framework is not publicly documented in detail, the publicly available evidence — benchmark methodologies, ROCm CI, and job descriptions — reveals the following patterns:

### 5.1 Python-Centric Orchestration

All major ROCm tools expose Python APIs or are scriptable via CLI. The standard automation pattern is:

```python
# Pseudocode representative of AMD lab practice
subprocess.run(["rocprofv3", "--kernel-trace", "--output-dir", run_dir, app_binary])
results = parse_sqlite(run_dir / "results.db")
push_to_dashboard(results, build_id=os.environ["CI_PIPELINE_ID"])
```

Python is the primary glue language for workload orchestration, result parsing, regression detection, and report generation.

### 5.2 CI/CD Integration

Performance validation is integrated into CI pipelines (GitLab CI is used internally by AMD ROCm teams, as evidenced by the public ROCm GitHub repositories). A typical pipeline stage:

- **Trigger**: new compiler drop, driver build, or kernel patch.
- **Collection**: run a fixed workload matrix (e.g., GEMM sizes, GROMACS step count, MLPerf benchmark) under `rocprofv3` or `uProf` CLI.
- **Comparison**: diff results against a golden baseline stored in a database (SQLite or DuckDB, mirroring uProf's own backend choice).
- **Alerting**: flag regressions exceeding a threshold (e.g., >2% throughput drop or >5% latency increase).

### 5.3 Workload Matrix Management

For a product like EPYC 9005, the internal benchmark matrix covers: SPEC CPU, HPC codes (OpenFOAM, LAMMPS, NAMD), in-memory databases (Redis, Memcached), AI inference (LLaMA, Stable Diffusion), and power/thermal sweeps. Automation scripts parameterise across:

- Core counts and NUMA topologies (NPS1 vs. NPS2 vs. NPS4).
- SMT on/off.
- Power determinism modes.
- Kernel and BIOS firmware versions.

### 5.4 MPI and Multi-Node Scaling

For HPC workloads, the automation layer wraps MPI launchers (`mpirun`, `srun`) and feeds per-rank `rocprofv3` profiles into aggregated analysis pipelines. The uProf 5.3 per-rank MPI HTML report is the current standard output for MPI profiling summaries.

---

## 6. Platform Architecture Relevant to the Role

Understanding what is being benchmarked is as important as knowing the tools.

### 6.1 CPU — Zen 5 (EPYC 9005 "Turin") and Zen 6 (EPYC "Venice")

- **Zen 5**: full 512-bit AVX-512 datapath (or double-pumped 256-bit mode), ~17% IPC uplift over Zen 4, up to 192 cores (Zen 5c), DDR5-6000, SP5 socket.
- **Zen 6 (Venice)**: up to 256 cores, per-socket memory bandwidth increasing to 1.6 TB/s (from ~614 GB/s in Turin), N2-class process node; shipping in 2026.
- Memory hierarchy: L1/L2/L3 per-core, large L3 per CCD (chiplet), cross-CCD xGMI fabric; all relevant for cache-miss profiling and NUMA-aware benchmarking.

### 6.2 GPU — RDNA 4 (Radeon RX 9000) and CDNA 4/5 (Instinct MI350/MI400)

- **RDNA 4** (RX 9070 XT etc.): gaming/consumer GPU; profiled with RGP, RGA, DirectX/Vulkan tooling.
- **CDNA 4** (MI355X): 3nm, 288 GB HBM3E, FP4/FP6 support, 10 petaflops FP4; primary target for `rocprofv3` and `rocprof-compute` in data-centre AI inference.
- **CDNA 5** (MI400 series): planned 2026; AMD Helios rack-scale solution.

### 6.3 NPU — Ryzen AI

AMD's APUs (Ryzen AI 300 series and successors) include a dedicated NPU for Windows ML workloads. The benchmarking of NPU-accelerated inference uses the ONNX Runtime AMD NPU Execution Provider and is validated against DirectX Compute Graph Compiler workloads in the Windows ML ecosystem. NPU performance appears in AI PC benchmarks (image diffusion throughput, LLM prefill latency).

---

## 7. Key Metrics and What They Expose

| Metric | Tool | Subsystem |
|--------|------|-----------|
| IPC (Instructions per Clock) | uProf PMC | CPU core |
| Cache miss rate (L1/L2/L3) | uProf PMC / rocprofv3 | CPU/GPU memory hierarchy |
| DTLB miss latency (IBS_LD_L1_DTLB_REFILL_LAT) | uProf IBS (v5.3) | CPU TLB |
| Memory bandwidth utilisation | rocprof-compute roofline / uProf PCM | HBM / DDR5 |
| GPU VALU utilisation | rocprof-compute | CDNA shader engine |
| Kernel dispatch overhead | rocprofv3 HIP trace | GPU runtime |
| Thread waiting time | uProf threading / rocprof-sys | Concurrency |
| PCIe bandwidth | uProf PCM (Zen 3+) | I/O |
| Power / frequency telemetry | uProf power profiling / amd-smi | Platform |
| MPI per-rank load imbalance | uProf 5.3 HTML report | HPC scaling |

---

## 8. Relationship to the Role's Responsibilities

| JD Responsibility | Relevant System Component |
|---|---|
| Design and evolve benchmarking infrastructure | Python orchestration layer, CI/CD pipeline, SQLite/DuckDB result stores |
| Analyse CPU/GPU/memory/storage bottlenecks | uProf (CPU+power), rocprofv3 (GPU traces), rocprof-compute (roofline) |
| Partner with compiler and driver teams | rocprofiler-sdk API, RGA ISA analysis, uProf inline-function resolution |
| SIMD/vectorisation analysis | uProf PMC (AVX-512 utilisation), RGA ISA disassembly |
| Cross-platform development | uProf (Windows/Linux/FreeBSD), ROCm tools (Linux), RDTS (Windows) |
| Concurrent programming / threading | uProf threading analysis, rocprof-sys host+device trace |
| Automation frameworks at scale | Python CLI wrappers, CI pipeline integration, per-rank MPI automation |

---

## 9. Practical Interview Talking Points

Drawing on your own background, the most natural connections to make are:

- **MPI profiling at scale**: your 4,096-processor BlueGene/P FEM solver maps directly to AMD's per-rank MPI analysis in uProf 5.3 and `rocprofv3` multi-process tracing.
- **HPC benchmark suites**: your familiarity with LAMMPS, OpenFOAM, and similar codes means you understand what "representative workload" selection means and the difference between strong/weak scaling benchmarks.
- **SIMD/vectorisation**: Zen 5's full 512-bit AVX-512 datapath and double-pump mode create exactly the kind of ISA-level analysis opportunities you can discuss using RGA and uProf PMC counters.
- **Python automation**: your experience with Pandas, CPLEX/Gurobi pipelines, and Streamlit dashboards aligns with AMD's Python-centric benchmark orchestration and result visualisation.
- **Static analysis and CI hygiene**: your use of clang-tidy and sanitisers in CI contexts is directly transferable to maintaining a reliable, regression-free benchmarking pipeline.

---

*Document prepared June 2026. Sources: AMD Developer Portal, ROCm Blogs, AMD uProf 5.3 release notes, MLCommons MLPerf v6.0 results, AMD EPYC product pages, Phoronix EPYC Turin benchmarks.*

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: I am applying for the Lead Software Performance Engineer position, as described in the attached job description. I am interested in the currently available AMD benchmarking and automation system. In preparation for the interview, please provide a thorough description of the system. Write the description in Markdown format.
