# Roofline Model and Arithmetic Intensity

## Overview

Both concepts originate from high-performance computing and have become essential analytical tools for performance engineers working across CPU, GPU, and NPU architectures — precisely the domain of the AMD Lead Software Performance Engineer role.

---

## Arithmetic Intensity

**Arithmetic Intensity (AI)** is a measure of computational work performed per unit of data moved from memory. It is defined as:

```
Arithmetic Intensity = FLOPs / Bytes
```

where:

- **FLOPs** — the number of floating-point operations executed by a kernel or workload
- **Bytes** — the number of bytes transferred between the compute unit and the memory subsystem (DRAM or cache hierarchy, depending on the analysis level)

The unit is **FLOP/byte** (or equivalently, FLOP·B⁻¹).

### Intuition

Consider two contrasting examples:

| Workload | FLOPs | Bytes moved | AI |
|---|---|---|---|
| Dense matrix multiply (DGEMM) | O(N³) | O(N²) | O(N) — grows with N, memory-efficient |
| Vector addition (DAXPY) | O(N) | O(N) | ~0.25 — heavily memory-bound |
| Convolution (large filters) | High | Moderate | High — compute-bound |

A **low AI** workload moves a lot of data relative to the work it performs — it is *memory-bound*. A **high AI** workload performs significant computation per byte — it is *compute-bound*. The transition point between the two regimes is determined by the hardware.

### Practical considerations

- AI is not a single number for a given kernel — it depends on **which level of the memory hierarchy** is considered (L1, L2, LLC, DRAM). This gives rise to the concept of *roofline at different cache levels*.
- Data reuse (tiling, blocking, loop fusion) increases effective AI by serving more FLOPs from faster, closer cache rather than DRAM.
- On AMD hardware, the memory hierarchy spans L1/L2 per compute unit, the Infinity Cache (on RDNA/CDNA GPUs), HBM or GDDR, and system DRAM — each level has a different bandwidth and therefore a different ridge point (see below).

---

## The Roofline Model

The **Roofline Model** (Williams, Waterman, Patterson, 2009) is a visual and analytical performance model that places a workload's measured or theoretical performance against two hardware-imposed upper bounds, revealing whether the workload is limited by **compute throughput** or **memory bandwidth**.

### The two ceilings

```
Attainable Performance = min( Peak FLOP/s,  Peak Bandwidth × Arithmetic Intensity )
```

Plotted on a log-log graph with **Arithmetic Intensity** on the x-axis and **Performance (FLOP/s)** on the y-axis, this produces the characteristic *roofline* shape:

```
FLOP/s
  │                          ┌──────────────── Peak Compute Roof
  │                     ┌────┘
  │                ┌────┘   compute-bound
  │           ┌────┘
  │      ┌────┘
  │ ┌────┘
  │─┘  memory-bound
  └────────────────────────────────────────  FLOP/byte (AI)
            ▲
            Ridge Point
```

### Key features

**The memory bandwidth roof** (diagonal slope) — represents the maximum performance achievable given the hardware's memory bandwidth: `Performance = Bandwidth × AI`. This is the binding constraint for low-AI workloads.

**The compute roof** (horizontal ceiling) — represents the hardware's peak FLOP/s (e.g., peak FMA throughput on a CPU core or a GPU CU). This is the binding constraint for high-AI workloads.

**The ridge point** — the AI value at which the diagonal and horizontal roofs intersect:

```
Ridge Point AI = Peak FLOP/s / Peak Bandwidth
```

A workload whose AI exceeds the ridge point is potentially compute-bound; below it, memory-bound. On a modern GPU (e.g., AMD RDNA3/CDNA3), peak FP32 throughput may be in the tens of TFLOP/s while HBM bandwidth is in the TB/s range — yielding a ridge point of perhaps 10–30 FLOP/byte. On a CPU the ridge point is typically lower (often 2–10 FLOP/byte) due to relatively narrower FLOP/s peaks per memory bandwidth unit.

### The model's diagnostic value

The gap between a workload's **measured** performance and the nearest roofline ceiling is wasted potential. The Roofline Model immediately clarifies *which* optimisation strategy is relevant:

| Situation | Diagnosis | Action |
|---|---|---|
| Below memory roof, far from compute roof | Memory-bound | Improve data locality, increase tiling, prefetch, reduce bandwidth via compression |
| Below compute roof, above ridge point | Compute-bound | Increase ILP, exploit SIMD/vectorization, reduce branch divergence, fuse operations |
| Near a roof but below it | Instruction or pipeline bottleneck | Analyse latency vs. throughput, check occupancy (GPU), examine IPC (CPU) |
| Far below both roofs | Likely latency-bound or synchronisation overhead | Profile with uProf/VTune for stall cycles, memory latency, thread contention |

---

## Relevance to the AMD Role

The job description explicitly lists responsibilities that map directly onto these concepts:

- **"Analyze system behavior across CPU, GPU, memory, storage, and software layers to identify bottlenecks"** — the Roofline Model is the canonical framework for this analysis; computing AI and plotting against hardware rooflines immediately reveals the binding resource.

- **"SIMD or vectorization concepts and hardware-aware optimization techniques"** — vectorization raises peak compute throughput (widens the horizontal roof) and increases effective AI by performing more FLOPs per loaded cache line.

- **"Tools such as uProf, VTune, or WPA"** — both AMD uProf and Intel VTune provide Roofline view panels that overlay measured kernel performance against hardware rooflines automatically, enabling root-cause analysis without manual instrumentation.

- **"Benchmarking infrastructure"** — a robust benchmarking pipeline should record both FLOP counts and memory traffic (via hardware performance counters: `PERF_COUNT_HW_CACHE_*` on Linux, or AMD's Compute Performance Monitoring) so that AI and roofline position can be computed systematically across workloads and platform generations.

- **"CPU, GPU, NPU execution, memory hierarchy, and I/O behavior"** — each processing unit has its own roofline (with different peak FLOP/s and different memory bandwidth tiers), and a cross-platform performance engineer must reason about all of them simultaneously.

In summary, Arithmetic Intensity quantifies how efficiently a workload uses memory bandwidth relative to compute, and the Roofline Model provides the architectural context that transforms that number into a clear optimisation verdict. Together they are among the most productive diagnostic tools available to a performance engineer working on heterogeneous AMD platforms.

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: In the context of the attached job description, explain in detail the terms Roofline Model and Arithmetic Intensity. Write the description in Markdown format.
