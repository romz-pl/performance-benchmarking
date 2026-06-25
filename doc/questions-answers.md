# AMD Lead Software Performance Engineer — Interview Q&A


## 1. Behavioural & Leadership

---

**Q1. Tell me about yourself and why you are applying for this role.**

I am a senior C++ engineer and scientific computing specialist with over 25 years of experience spanning HPC, numerical methods, quantitative finance, and enterprise codebase modernisation. My academic background — a PhD in Theoretical Quantum Physics and an MSc in Computer Physics from Warsaw University of Technology — gave me a rigorous foundation in performance-critical numerical work, which I carried into industry. I have scaled MPI-based finite element solvers to 4,096 processors on IBM BlueGene/P, worked on low-latency trading infrastructure, and more recently focused on profiling-driven optimisation and CI/CD automation for complex C++ codebases. This role sits precisely at the intersection of systems architecture, performance analysis, and automation — which is exactly where my skills and curiosity converge. AMD's cross-stack visibility (CPU, GPU, NPU) and the opportunity to influence product direction from benchmarking data make this a compelling next step.

---

**Q2. Describe a time when you led a complex technical initiative. How did you align stakeholders and deliver results?**

At a previous engagement I led the modernisation of a legacy HPC codebase targeting a multi-architecture cluster. The codebase had no profiling baseline, no automated regression tests, and three separate teams with different priorities. I started by establishing a shared performance baseline using hardware performance counters and a lightweight benchmark harness I wrote in Python. I then ran a series of short discovery workshops with each team to map bottlenecks to owners. By framing every discussion around data rather than opinion, I aligned stakeholders around a prioritised roadmap within six weeks. Over the following quarter we reduced end-to-end wall time by approximately 35% on the primary workload. The key lesson: performance data is a universal language that dissolves organisational friction.

---

**Q3. How do you mentor junior engineers on performance topics without creating dependency?**

I try to teach mental models rather than answers. When a junior engineer asks why a loop is slow, I do not simply profile it for them — I sit with them at the profiler, narrate what I am looking at and why, and ask them questions as we go. I also set up regular "perf deep-dive" sessions where we pick a real bottleneck from the current sprint and work through it as a group. Over time I deliberately step back and let them lead the investigation, only intervening when they are stuck. The goal is that within three months a junior should be able to run a full profiling cycle independently; within six months they should be proposing hypotheses before they open the profiler.

---

**Q4. Give an example of when you had to influence a technical decision without direct authority.**

During a project at Techland I identified that a third-party library we depended on was causing significant L3 cache pressure due to its allocation pattern. The library owner was in a different team and resistant to changing the integration. I produced a reproducible micro-benchmark showing a 2× throughput difference between the current usage pattern and a proposed alternative, with hardware counter evidence (LLC misses, memory bandwidth saturation). I presented this data to the relevant engineering lead and offered to implement the change myself with a rollback plan. The proposal was accepted. The takeaway: you rarely need authority if you have a compelling, reproducible data story.

---

## 2. Performance Analysis & Profiling

---

**Q5. Walk me through your profiling workflow when you encounter an unfamiliar bottleneck.**

I follow a top-down approach. First, I establish a reproducible benchmark so results are comparable across runs — controlling for CPU frequency scaling, NUMA topology, process affinity, and background noise. Then I use a sampling profiler (AMD uProf or Intel VTune) to get a coarse hotspot map. Once I have the top functions, I drill into hardware performance counters: retired instructions, cycles, LLC misses, branch mispredictions, DRAM bandwidth. If the bottleneck looks like a memory issue I move to a memory access analyser to understand spatial and temporal locality. If it looks like instruction throughput, I inspect the generated assembly and check whether the compiler vectorised the hot loop. Only after I understand the root cause do I attempt an optimisation, and I re-run the benchmark to verify the improvement is real and not noise.

---

**Q6. What is your experience with AMD uProf specifically?**

I have used uProf primarily for CPU profiling on AMD Zen-family processors: time-based sampling, hardware event counting (from the Performance Monitoring Unit), and the Power Profiler for thermal and energy analysis. uProf's CPU Profiling mode gives call-graph attribution that I find cleaner than VTune for Zen targets because it uses AMD's own PMU events rather than approximating them. I have also used its Memory Access Analysis feature to surface false sharing in multi-threaded code — it instruments NUMA access latency and highlights hot cache lines. One practical note: I always pin the process to a specific CCX (Core Complex) when profiling to avoid inter-die latency noise on multi-CCD configurations.

---

**Q7. How do you distinguish a memory-bound bottleneck from a compute-bound one?**

The clearest signals come from hardware counters. A compute-bound hotspot will show high "retired instructions per cycle" (close to the theoretical IPC ceiling for the microarchitecture) and low "memory load latency" or LLC miss rate. A memory-bound hotspot will show low IPC despite a high cycle count, elevated LLC misses, high DRAM bandwidth utilisation, and often long memory load latency (visible in tools like uProf's Memory Access view or VTune's Memory Access analysis). The roofline model is a useful conceptual frame: I compute the arithmetic intensity of the kernel (FLOPs per byte) and compare it to the machine's compute and bandwidth ceilings. If the kernel sits below the ridge point, more compute optimisation is wasteful — the bottleneck is bandwidth.

---

**Q8. Describe a concrete optimisation you made and the measurable outcome.**

In a scientific computing project involving a finite element assembly loop, sampling profiling revealed the loop was spending roughly 60% of its time on irregular memory gathers — indirect addressing into large element stiffness matrices. The loop was not vectorising because of the gather pattern. I restructured the data layout to group element data by cache-line-friendly blocks and rewrote the inner loop using explicit AVX2 gather intrinsics with software prefetching two iterations ahead. I also reordered the mesh element list using a Hilbert-curve space-filling order to improve locality across the element range. The combined change reduced wall time for assembly from ~18 s to ~7 s on a representative mesh, a ~2.5× speedup confirmed across five repeated runs with variance under 2%.

---

## 3. Benchmarking & Automation

---

**Q9. How do you design a benchmarking system that produces trustworthy, reproducible results?**

Trustworthy benchmarking requires controlling or accounting for every source of variance. At the system level that means: fixing CPU frequency (disable Turbo Boost / CPB for steady-state measurements, or explicitly enable it if you are measuring peak burst performance), binding processes to specific cores via `taskset` or `numactl`, disabling NUMA balancing if it introduces noise, and flushing caches to a known state before each trial. At the software level I always run a configurable warmup phase to reach steady thermal state, then collect a statistically sufficient number of trials (I use the bootstrap confidence interval at 95% to determine sample size). Outlier detection (e.g., Grubbs' test) filters measurement artefacts. Results are stored with full metadata: hardware configuration, OS version, compiler flags, git commit hash. This makes the benchmark reproducible weeks later by anyone on the team.

---

**Q10. How have you built or improved automation frameworks for performance testing?**

On a recent project I built a Python-based benchmarking orchestration framework that ran nightly on a CI runner with dedicated bare-metal access. It used a declarative YAML configuration to specify workloads, compiler flag matrices, and target machines. Each run produced a structured JSON result ingested by a lightweight Streamlit dashboard that plotted performance trends, flagged regressions beyond a configurable threshold (expressed as percentage deviation from the rolling baseline), and filed a GitLab issue automatically when a regression was detected. The key design principles were: separation of workload definitions from execution logic, idempotent result storage (no overwriting historical data), and zero manual steps between commit and dashboard update.

---

**Q11. How do you prevent performance regressions from slipping into production?**

The most effective mechanism I have used is a performance gate in CI: the benchmark suite runs on every merge-request targeting the main branch. If any tracked metric regresses beyond a defined threshold (e.g., >3% wall-time increase or >5% throughput drop on a key workload), the pipeline fails and a report is attached to the MR. The thresholds are deliberately conservative at first and tightened as measurement noise decreases. I also maintain a "performance budget" concept per subsystem — each team knows their allocation of latency or memory, and any change that would exceed it requires a performance review before merging.

---

**Q12. What makes a good benchmark workload, and how do you select or construct one?**

A good benchmark must be representative of the production workload in terms of data sizes, access patterns, and instruction mix; reproducible across machines and time; sensitive enough to detect meaningful changes; and fast enough to run in CI. I start by profiling the production code to identify the dominant kernels by wall-time contribution (the 80/20 distribution is almost always present). I then extract or synthesise minimal kernels that reproduce those characteristics. Where possible I validate the synthetic benchmark against the full workload by checking that the same hardware counter ratios (IPC, LLC miss rate, memory bandwidth) match within ~10%. Micro-benchmarks that do not correlate with real workload behaviour are dangerous — they can show a 5× speedup on a loop that contributes 1% of total runtime.

---

## 4. SIMD & Vectorisation

---

**Q13. Explain how you approach vectorising a compute kernel. What are the key obstacles?**

My starting point is always the compiler's auto-vectorisation report (`-fopt-info-vec` in GCC, `/Qvec-report` in MSVC, `-Rpass=loop-vectorize` in Clang). If the compiler failed to vectorise, the report explains why: pointer aliasing, non-unit stride, data type mismatch, loop-carried dependencies. For aliasing I add `__restrict__` or `#pragma ivdep`; for non-unit stride I consider a data layout transformation (AoS → SoA). For reductions with loop-carried dependencies I restructure for partial sums. If the compiler still fails after these hints, I write explicit SIMD intrinsics (SSE4.2, AVX2, or AVX-512 depending on the target). The key discipline is: always measure the auto-vectorised version before reaching for intrinsics — modern compilers are often better than hand-written intrinsics for complex patterns because they can schedule around latency more freely.

---

**Q14. What is the difference between AVX2 and AVX-512, and when would you choose one over the other?**

AVX2 operates on 256-bit wide registers (eight 32-bit floats or four 64-bit doubles per cycle) and is available on all modern AMD Zen 2+ and Intel Haswell+ processors. AVX-512 doubles the register width to 512 bits and adds masked operations, scatter/gather, and richer integer operations. On paper AVX-512 offers 2× throughput for wide floating-point workloads, but the practical trade-offs are significant: on some microarchitectures (particularly older Intel cores) executing AVX-512 instructions causes a frequency downgrade that can reduce throughput for mixed workloads. AMD's Zen 4 introduced native 512-bit AVX-512 execution without frequency penalty. My decision rule: profile the target microarchitecture specifically; prefer AVX2 for broadly deployed code where the frequency penalty risk is real; use AVX-512 when the kernel is compute-bottlenecked and the target platform is confirmed Zen 4 or a similar native-512 design.

---

**Q15. What is false sharing and how do you detect and fix it?**

False sharing occurs when two threads write to different variables that happen to reside on the same cache line (typically 64 bytes). Each write invalidates the line in the other core's cache, causing coherency traffic and effectively serialising what should be independent operations. Detection: AMD uProf's Memory Access Analysis and Intel VTune's Threading analysis both surface false sharing by correlating high L1/L2 miss rates with specific cache lines accessed by multiple threads. Fix: pad shared data structures so each thread's working variables occupy distinct cache lines (`alignas(64)` in C++17, or explicit padding). For hot per-thread accumulators I use thread-local storage rather than a shared array indexed by thread ID.

---

## 5. Cross-Platform & Systems Architecture

---

**Q16. How do you approach a cross-platform performance project spanning Windows and Linux?**

I maintain a strict separation between platform-agnostic algorithmic code and platform-specific measurement or OS-interaction code. The benchmarking harness abstracts timer resolution (QueryPerformanceCounter on Windows vs. `clock_gettime(CLOCK_MONOTONIC)` on Linux), process affinity, and huge page allocation. For SIMD code I use a feature-detection layer (CPUID at runtime or compile-time CMake feature flags) so the same binary selects the optimal path on each platform. Performance characteristics can differ substantially between Windows and Linux on identical hardware due to scheduler behaviour, memory allocator differences (jemalloc vs. the default Windows heap), and security mitigations with different overhead profiles — so I always benchmark on both targets before drawing conclusions.

---

**Q17. How does the memory hierarchy influence your optimisation decisions?**

Modern CPUs have L1 (typically 32–64 KB, ~4 cycles), L2 (256 KB–1 MB, ~12 cycles), L3 (shared, 8–32+ MB, ~40–60 cycles), and DRAM (~100–200 ns). Most performance-critical code either fits comfortably in L2 and runs at compute speed, or it does not fit and becomes memory-bandwidth or latency bound. My optimisation decisions flow from this: for large datasets I focus on cache-friendly access patterns (sequential stride, spatial locality, tiling for blocked algorithms), software prefetching for predictable access patterns, and reducing the working set size (quantisation, structure packing). For multi-socket systems I additionally think about NUMA: allocating memory on the same node as the thread that accesses it, and using `numactl --localalloc` for benchmarking to avoid inter-socket latency masking algorithmic work.

---

**Q18. What is your understanding of GPU performance characteristics compared to CPU, and how does it affect benchmarking strategy?**

GPUs have very high memory bandwidth (e.g., 900 GB/s on HBM3-based accelerators vs. ~100 GB/s DDR5 on CPU) and massive parallelism (thousands of shader cores) but high latency for individual operations and limited branch divergence tolerance. Benchmarking a GPU workload requires measuring: kernel occupancy (are enough warps/waves in flight to hide memory latency?), memory bandwidth utilisation vs. the theoretical peak, and PCIe/NVLink transfer overhead (host-device copies often dominate for small payloads). For AMD GPUs I use ROCm's rocprof or AMD uProf's GPU profiling mode. A critical distinction: a workload that is CPU memory-bandwidth-bound does not automatically become compute-bound on a GPU — you must verify the arithmetic intensity exceeds the GPU's ridge point on its own roofline.

---

## 6. Concurrent Programming

---

**Q19. How do you test and validate the correctness of multi-threaded code?**

I use a layered approach. First, ThreadSanitizer (TSan) on every CI run to catch data races and lock-order violations — it has saved me from subtle races that are impossible to reproduce deterministically. Second, careful design review using established concurrency primitives rather than ad hoc synchronisation: I default to lock-based designs with well-understood semantics and move to lock-free only where profiling proves the mutex is the bottleneck. Third, stress testing with randomised thread scheduling using tools like `rr` (Mozilla's record-and-replay debugger) to explore more of the scheduling space. I also maintain an open-source lock-free SPSC queue (github.com/romz-pl/spsc-queue) where I applied all of these techniques rigorously, including formal reasoning about the memory ordering guarantees required at each atomic operation.

---

**Q20. Explain C++ memory ordering for atomics. When do you use `memory_order_acquire`/`release` vs. `memory_order_seq_cst`?**

`memory_order_seq_cst` (the default) provides the strongest guarantee: a single total order is observed by all threads for all seq_cst operations. It is the safest choice and the one I use when correctness is uncertain. The cost is that on architectures with relaxed memory models (e.g., ARM) it requires additional fence instructions. `acquire` and `release` form a synchronised-with relationship: a `release` store on thread A, when observed as a `load acquire` on thread B, guarantees that all writes by A before the store are visible to B after the load. This is sufficient for producer-consumer handoff (e.g., an SPSC queue) without the overhead of a full seq_cst fence. `memory_order_relaxed` is correct only for operations with no ordering requirements relative to other memory accesses — typically counters where you care about atomicity but not ordering. I always document the intended invariant in a comment adjacent to every atomic operation.

---

## 7. AMD-Specific & Strategic

---

**Q21. What do you know about AMD's Zen microarchitecture that is relevant to performance optimisation?**

Zen 4 (and Zen 5) are highly relevant: the chiplet design means that L3 cache is shared within a Core Complex Die (CCD) of up to eight cores, but inter-CCD and inter-die traffic (on multi-CCD CPUs like Threadripper or EPYC) incurs higher latency. For benchmarking this means NUMA-awareness is necessary even on a single-socket system when the CPU has multiple CCDs. Zen 4 introduced native 512-bit AVX-512 execution (two 256-bit FMA units fused), which changes the vectorisation trade-off compared to Intel's throttled AVX-512 implementation. Zen 5 further improved branch prediction and the out-of-order window size. For workloads with irregular control flow, the improved branch predictor in Zen 5 can materially change which optimisations are worth applying — worth re-profiling rather than assuming Zen 4 conclusions carry over.

---

**Q22. How would you approach establishing a benchmarking baseline for a new AMD platform within your first three months?**

In the first month I would audit the existing benchmark suite: catalogue what workloads are covered, what metrics are collected, how results are stored, and what the current noise floor is. I would identify gaps — workloads that are important to product decisions but not yet measured. In month two I would instrument the benchmark infrastructure to collect hardware performance counter data alongside wall time (not just throughput numbers in isolation), and establish a statistical baseline with confidence intervals for each metric. I would also run the suite on both the new platform and the previous generation to characterise the delta. By month three I would have a clear picture of where the new platform over- or under-performs relative to the roofline model, and I would present a written root-cause hypothesis for each significant deviation as input to the architecture and software teams.

---

**Q23. How do you handle a situation where benchmark results conflict with intuition or prior expectations?**

I treat unexpected results as the most valuable signal in the dataset. My first step is to rule out measurement error: verify the benchmark is correctly isolating the variable of interest, rerun on a different machine, check for thermal throttling or frequency variation in the results. If the result holds up, I dig into the microarchitecture-level explanation rather than dismissing it. Counterintuitive results often reveal something real — a hardware feature behaving differently than the specification implies, a compiler decision that was not obvious, or an interaction between subsystems. I document every such finding as a "performance anomaly" entry in the team's knowledge base, because it is invariably useful the next time someone hits the same surprise.

---

## 8. Process & Communication

---

**Q24. How do you communicate performance findings to non-specialists — product managers, architects, or executives?**

I follow a three-layer structure: (1) the headline — "Workload X is 40% slower than the target; here is the one-line root cause." (2) the evidence — one chart showing the bottleneck, labelled clearly, with a reference line for the target. (3) the recommendation and its cost — a concrete action, the expected gain, and the engineering effort. I deliberately avoid hardware counter acronyms in these presentations and use analogies: "the CPU is like a chef waiting for ingredients from a pantry in the next building — most of the time is delivery, not cooking." The goal is that the decision-maker can say yes or no to the recommendation with full information and no need to understand the microarchitecture details.

---

**Q25. What does "performance culture" mean to you, and how do you help build it?**

Performance culture means that the team treats performance as a first-class quality attribute alongside correctness, security, and maintainability — not something you bolt on before a release. Concretely it means: every significant code change has a measurable performance impact (positive, negative, or neutral) recorded in the review; benchmarks run in CI and regressions block merges; engineers know how to profile their own code before asking for help; and performance postmortems are blameless and shared broadly. Building this culture requires patience: I start with infrastructure that makes profiling easy (one command, immediate results), then recognition for engineers who find regressions early, and gradually the habits follow the incentives.

---

## 9. Rapid-Fire Technical

---

**Q26. What is the roofline model and how do you use it?**

The roofline model plots achievable performance (GFLOPs/s) as a function of arithmetic intensity (FLOPs per byte of DRAM traffic). The "roof" is either the peak compute throughput or the peak memory bandwidth multiplied by intensity — whichever is lower. Plotting a kernel on this chart immediately shows whether it is compute-bound or memory-bound and how far it is from the hardware ceiling. I use it to prioritise: if a kernel is far from the compute roof and above the ridge point, optimising its memory access is unlikely to help — I should focus on reducing instruction count. If it is far from the memory bandwidth roof and below the ridge point, data layout and prefetching are the levers.

---

**Q27. What is the difference between latency and throughput in the context of CPU execution?**

Latency is the number of cycles from when an instruction is issued until its result is available. Throughput (reciprocal throughput) is how many cycles must elapse before the same execution unit can accept a new instruction of the same type — often 1 cycle for pipelined units even if latency is 4–5 cycles. In a loop with independent iterations, the CPU can overlap multiple iterations and achieve throughput-limited execution. In a loop with a loop-carried dependency (each iteration depends on the result of the previous one), execution is latency-limited regardless of throughput. This distinction directly drives whether software pipelining or loop unrolling will help.

---

**Q28. What Python libraries do you use for performance data analysis and visualisation?**

Pandas for data ingestion and aggregation, NumPy for numerical transforms, SciPy for statistical analysis (confidence intervals, hypothesis testing, outlier detection), Matplotlib and Seaborn for static plots, and Streamlit for interactive dashboards shared with the team. For very large result sets (millions of benchmark rows) I use Polars instead of Pandas for its lazy evaluation and columnar storage model, which reduces memory pressure substantially.

---

**Q29. How do you validate that a performance optimisation has not introduced a correctness regression?**

The optimised output must be numerically equivalent to the reference implementation within an appropriate tolerance (for floating-point code, I use relative error with a configurable epsilon rather than exact bitwise equality, because reordering operations changes rounding). I maintain a correctness test suite that runs the optimised and reference implementations on the same input and compares outputs. I also run the full unit and integration test suite with AddressSanitizer and UndefinedBehaviorSanitizer enabled to catch memory errors or UB introduced by, for example, intrinsic misuse or unsafe pointer aliasing assumptions. Both the performance benchmark and the correctness suite must pass before I consider an optimisation complete.

---

**Q30. Where do you see AMD's opportunity in the AI accelerator and NPU space, and how does that relate to this role?**

AMD's CDNA architecture (Instinct series) has become a credible alternative to NVIDIA in HPC and AI training workloads, driven by the ROCm open-source software stack. The NPU on Ryzen AI (XDNA architecture) targets on-device inference for client platforms — a rapidly growing segment as AI-assisted workflows move to the edge. For this role, the relevance is that benchmarking infrastructure must evolve to cover heterogeneous execution: a workload may split across CPU, NPU, and GPU, and the performance bottleneck may be in data movement between them rather than in any single accelerator. Building benchmarking frameworks that instrument cross-device execution graphs, measure PCIe/Infinity Fabric transfer overhead, and attribute latency correctly to each stage is an area where I see both significant engineering challenge and strategic impact.

---

*End of document — 30 questions across 9 categories.*

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: I am applying for the Lead Software Performance Engineer position described in the attached job description. I am interested in a list of possible questions and answers that may be asked during the interview. Provide a comprehensive list of these questions and answers. Write the description in Markdown format.
