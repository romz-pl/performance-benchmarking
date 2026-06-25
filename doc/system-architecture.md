# Understanding of System Architecture

> [!NOTE]
> 
> "Understanding of system architecture, including CPU, GPU, NPU execution, memory hierarchy, and I/O behavior."


This preferred experience point asks for a **systems-level mental model** of how modern computing platforms execute workloads — not just surface familiarity, but the ability to reason about *where time and bandwidth are spent* and *why*. In the context of AMD's benchmarking and performance engineering work, this knowledge is the analytical foundation from which all performance optimization flows.

---

## CPU Execution

Understanding CPU execution means knowing how a modern out-of-order superscalar processor actually runs code — well beyond the instruction set level.

Key concepts include:

- **Pipeline stages** — fetch, decode, dispatch, execute, retire — and how stalls at each stage manifest as lost throughput.
- **Instruction-level parallelism (ILP)** — how the CPU's out-of-order engine reorders instructions to hide latency, and what limits this (data dependencies, resource conflicts, branch mispredictions).
- **Branch prediction and speculative execution** — how mispredictions cause pipeline flushes and what coding patterns exacerbate this.
- **Execution unit layout** — integer ALUs, floating-point units, SIMD/vector units (e.g., AMD's AVX2/AVX-512 lanes), load/store units — and how instruction mix determines throughput bottlenecks.
- **Front-end vs. back-end bottlenecks** — distinguishing instruction-supply problems (i-cache misses, decode bandwidth) from execution-unit or memory-bound problems, which is exactly the kind of diagnosis tools like AMD uProf or Intel VTune surface.
- **Core frequency, boost behavior, and thermal headroom** — how dynamic frequency scaling affects benchmark reproducibility and how to control for it.

In AMD's context, this applies directly to Ryzen/EPYC CPU analysis, where understanding Zen microarchitecture pipeline specifics (e.g., op cache behavior, µop dispatch width) is necessary to interpret profiling data correctly.

---

## GPU Execution

GPU execution follows a fundamentally different model: massively parallel, latency-tolerant, throughput-optimized.

Key concepts include:

- **SIMT execution model** — Single Instruction Multiple Threads, where thousands of shader/compute threads execute in lockstep within a wavefront (AMD's term; NVIDIA calls it a warp). Divergent branches within a wavefront cause serialization and efficiency loss.
- **Compute unit (CU) structure** — AMD's RDNA/CDNA architectures organize execution into Compute Units containing SIMD units, scalar units, vector/scalar register files, and shared memory (LDS). Understanding occupancy — how many wavefronts can reside on a CU simultaneously — is central to GPU optimization.
- **Memory access patterns** — coalesced vs. uncoalesced global memory access, the cost of random scatter/gather, and how to structure data for maximum bandwidth utilization.
- **The roofline model** — characterizing workloads by their arithmetic intensity (FLOPs per byte of memory traffic) to determine whether they are compute-bound or memory-bandwidth-bound, and what optimization strategy applies.
- **GPU dispatch and command queues** — how work is submitted via APIs (HIP, ROCm, DirectX, Vulkan) and how kernel launch overhead and synchronization points affect end-to-end throughput.
- **Tensor/matrix cores** — specialized execution units (AMD's Matrix Cores in CDNA) that accelerate mixed-precision matrix multiplication, critical for AI inference/training workloads AMD is increasingly prioritizing.

For AMD specifically, this means familiarity with the **ROCm software stack** and HIP programming model alongside the hardware architecture of RDNA (consumer) and CDNA (data center/AI) GPUs.

---

## NPU Execution

The NPU (Neural Processing Unit) is the newest major compute element, now integrated into AMD's Ryzen AI processors (Hawk Point, Strix Point) under the **XDNA/XDNA 2 architecture**.

Key concepts include:

- **Dataflow architecture** — unlike CPUs and GPUs, NPUs are typically implemented as spatial dataflow arrays (similar to systolic arrays), where data flows through a fixed network of processing elements rather than being fetched from shared memory by a scheduler.
- **MAC-centric execution** — NPUs are optimized for multiply-accumulate operations that dominate neural network inference (convolutions, matrix multiplications, attention).
- **On-chip memory and tiling** — because DRAM bandwidth is the bottleneck for large models, effective NPU use requires carefully tiling model layers to fit in on-chip SRAM, minimizing off-chip traffic.
- **Quantization and precision** — NPUs typically operate in INT8, INT4, or mixed precision; understanding how quantization affects both accuracy and throughput is necessary to benchmark them meaningfully.
- **Software stack** — AMD's NPU is exposed through the **Ryzen AI Software platform** (ONNX Runtime EP, IREE, MLIR-AIE compiler stack). Performance benchmarking requires understanding how the compiler tiles, schedules, and lowers model graphs onto the physical array.
- **Heterogeneous dispatch** — in practice, workloads are split across CPU, GPU, and NPU based on the nature of each layer; understanding the scheduling and data-transfer cost between these is essential to whole-pipeline benchmarking.

This is a rapidly evolving area and directly relevant to AMD's AI PC platform ambitions — making NPU benchmarking infrastructure a high-priority engineering challenge.

---

## Memory Hierarchy

Memory hierarchy understanding is arguably the single most important systems knowledge for performance engineers, because **most real-world bottlenecks are memory bottlenecks**.

The hierarchy from fastest/smallest to slowest/largest:

| Level | Typical Latency | Bandwidth | Managed by |
|---|---|---|---|
| Registers | ~1 cycle | Extremely high | Compiler |
| L1 cache | ~4–5 cycles | ~1–2 TB/s | Hardware |
| L2 cache | ~12–15 cycles | ~500 GB/s | Hardware |
| L3 cache (LLC) | ~30–50 cycles | ~200–400 GB/s | Hardware |
| DRAM | ~70–100 ns | ~50–200 GB/s | OS / programmer |
| NVMe SSD | ~20–100 µs | ~5–15 GB/s | OS |

Key concepts:

- **Cache line granularity (64 bytes)** — all cache operations occur in 64-byte chunks; misaligned access, false sharing in multithreaded code, and poor spatial locality all cause unnecessary cache traffic.
- **Cache associativity and eviction policies** — how conflict misses arise from unfortunate stride patterns and how to detect them with hardware performance counters.
- **NUMA topology** — on multi-socket systems (EPYC) and within AMD's chiplet architecture, memory access latency depends on *which* memory controller and *which* die the data resides on. NUMA-unaware code can suffer 2–3× latency penalties. `numactl` and AMD's uProf expose this.
- **Memory bandwidth saturation** — understanding the STREAM benchmark and similar tools to characterize peak sustainable bandwidth, and how to determine when a workload has hit this ceiling.
- **Unified memory and HBM** — AMD's MI-series GPUs use HBM (High Bandwidth Memory) with ~3 TB/s bandwidth; understanding the different bandwidth/capacity/latency tradeoff versus GDDR or DDR is necessary for AI workload benchmarking.
- **Prefetching** — hardware and software prefetch behavior, and how access patterns that defeat the prefetcher cause performance cliffs that are visible in profiler data but not obvious in code review.

---

## I/O Behavior

I/O behavior covers how data moves between the processor complex and external subsystems — storage, networking, and peripheral devices.

Key concepts:

- **PCIe topology and bandwidth** — PCIe 4.0 x16 provides ~32 GB/s; PCIe 5.0 doubles this. GPU, NVMe, and NIC all compete for this bandwidth. Understanding how lane allocation and device placement affect saturation is essential for benchmarking data-center workloads.
- **DMA and IOMMU** — how peripherals transfer data to/from system memory without CPU involvement, and how IOMMU configuration (used heavily in virtualization and security) can introduce overhead.
- **NVMe and storage latency** — understanding queue depth, command set efficiency, and how OS block-layer overhead affects observed storage throughput. For AI workloads with large dataset pipelines, storage I/O is often the true bottleneck.
- **GPU↔CPU data transfer** — `cudaMemcpy`/`hipMemcpy` equivalents and why minimizing host↔device transfers is a core GPU optimization strategy. Understanding pinned (page-locked) memory and its effect on DMA transfer rates.
- **Network I/O for distributed workloads** — in HPC/AI training contexts (directly relevant given AMD's CDNA/Instinct positioning), understanding RDMA over InfiniBand or RoCE, collective communication patterns (AllReduce, AllGather), and how bandwidth and latency interact with compute time determines scaling efficiency.
- **Interrupt coalescing and polling** — high-throughput I/O systems (DPDK, SPDK, RDMA) bypass kernel interrupt handling entirely; understanding why this matters for tail latency and throughput is important when benchmarking storage and networking.

---

## Why This Matters for the Role

In AMD's benchmarking and performance engineering context, this systems knowledge is not academic — it is the **interpretive framework** that transforms raw profiler numbers into actionable findings. A performance engineer who understands the full stack can:

- Look at a VTune/uProf flame graph and immediately hypothesize whether a hotspot is compute-bound, memory-latency-bound, or bandwidth-bound.
- Design benchmark workloads that stress a *specific* subsystem in isolation — and know when cross-subsystem interference is contaminating results.
- Translate architectural changes (new cache topology, wider SIMD, HBM addition) into *expected* benchmark impact, enabling pre-silicon performance projection.
- Identify when a performance regression is a software regression versus a hardware configuration difference.

This is precisely why AMD lists it as a preferred qualification: the benchmarking infrastructure being built here ultimately serves architects and product teams making silicon decisions, and that requires engineers who can reason across the entire hardware-software stack.

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: In the context of the attached job description, thoroughly explain the following preferred experience: "Understanding of system architecture, including CPU, GPU, NPU execution, memory hierarchy, and I/O behavior." Write the description in Markdown format.
