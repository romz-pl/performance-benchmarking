# Partnering Across Teams to Influence Technical Direction and Support Product Readiness

> [!NOTE]
> 
> "Partner with architecture, product, driver, compiler, and application teams to influence technical direction and support product readiness."

This responsibility sits at the heart of what distinguishes a **Lead** Software Performance Engineer from a purely individual contributor role. It is fundamentally about **translating performance data into decisions** — and doing so across organizational boundaries where each team speaks a different technical dialect and operates under different pressures.

---

## What It Actually Means

AMD's computing platforms are not built by a single team. A CPU or GPU product emerges from the coordinated work of architecture designers, hardware-facing driver engineers, compiler teams optimizing code generation, and application engineers tuning real-world software. Performance problems and opportunities rarely live cleanly within one team's scope — they emerge from the **interactions** between layers. This role requires someone who can navigate all of them.

"Influencing technical direction" means your performance data and analysis must be persuasive enough to change how other teams prioritize their work. "Supporting product readiness" means ensuring that, when a product ships, its performance story is validated, understood, and defensible.

---

## The Five Stakeholder Axes

### 1. Architecture Teams
These engineers make long-horizon decisions about microarchitecture — cache sizes, execution unit counts, memory subsystem design, interconnect topology. Your role here is to feed **empirical benchmark data** back into the architectural feedback loop. If your benchmarking infrastructure reveals that a particular workload class (e.g., sparse matrix operations relevant to AI inference, or memory-bound CFD kernels) is hitting a fundamental bottleneck in the current pipeline, that finding needs to reach architects early enough to influence the *next* product generation — not the one already in silicon. This requires translating low-level profiling results (e.g., from AMD's own **uProf**, or Intel VTune where cross-platform comparisons are needed) into architectural hypotheses that resonate with people designing at the ISA and microarchitecture level.

### 2. Product Teams
Product managers and product engineers are making **market-facing decisions**: which benchmark results to publish, which workloads to certify, how to position a platform against competitors. Your partnership here means providing them with **accurate, reproducible, and contextualized** performance numbers — while also flagging risks. If a key benchmark is performing below target three months before launch, the product team needs to know, understand why, and decide whether to accelerate a fix or adjust expectations. You are the technical authority bridging raw performance data and product narrative.

### 3. Driver Teams
GPU and CPU driver engineers operate close to the hardware, managing scheduling, memory management, power states, and hardware abstraction. Performance regressions frequently originate here — a driver update changes a heuristic, and a previously well-tuned workload degrades by 15%. Your benchmarking infrastructure must be capable of **detecting these regressions rapidly** and your relationship with driver teams must be close enough to drive fast root-cause analysis. Conversely, driver teams may be unaware that a particular optimization they could make (e.g., changing a scheduling policy for a specific workload pattern) would yield significant gains — your profiling data surfaces these opportunities.

### 4. Compiler Teams
Modern compilers (LLVM-based toolchains, OpenCL/HIP compilers for GPU) make thousands of optimization decisions that directly affect performance. **SIMD vectorization**, loop unrolling, inlining, and register allocation all depend on compiler heuristics. When a benchmark underperforms, one common culprit is a missed auto-vectorization opportunity or a suboptimal code generation path. Partnering with compiler teams means being able to read assembly output, identify vectorization failures (e.g., loops that should use AVX-512 or RDNA-native SIMD but don't), and file actionable bug reports or feature requests that are specific enough for compiler engineers to act on. This requires exactly the kind of **hardware-aware optimization** and SIMD familiarity listed in the job's preferred experience.

### 5. Application Teams
These are the engineers closest to end-user software — game engines, AI frameworks (PyTorch, TensorFlow, ONNX Runtime), HPC applications, creative tools. They have deep knowledge of their own workloads but may not fully understand the underlying hardware. Your partnership here often involves **co-optimization**: you bring hardware profiling expertise and platform knowledge; they bring workload semantics. Together you identify whether a performance gap is caused by suboptimal API usage, poor data layout (cache unfriendly access patterns), insufficient parallelism, or a missing hardware feature exposure. Application teams also provide the **most realistic benchmarks** — replacing synthetic microbenchmarks with real workloads that customers actually run.

---

## The Mechanics of Influence Without Authority

In a matrixed organization like AMD, this role does not carry direct authority over any of these teams. Influence is earned through:

- **Data credibility** — rigorous, reproducible benchmark results that others trust and cannot easily dismiss
- **Clear communication** — translating profiler output and assembly analysis into language each audience understands; architects think in cycles and pipeline stages, product managers think in percentages and competitive rankings
- **Timeliness** — findings delivered early enough in the product cycle to be actionable; a performance insight that arrives after tape-out is a post-mortem, not a contribution
- **Relationship capital** — built through consistent, honest, technically sound engagement over time

---

## Supporting Product Readiness

As a product approaches launch, this responsibility becomes explicitly **validation-oriented**. It encompasses:

- Certifying that target workloads meet performance commitments across the supported platform matrix (Windows, Linux, varying hardware SKUs)
- Identifying and triaging late-breaking regressions with enough specificity to unblock rapid fixes
- Producing the performance characterization data that feeds launch materials, press briefings, and competitive positioning
- Ensuring the automation infrastructure can reproduce results reliably — so that published numbers are not one-off measurements but **statistically stable outcomes** from a well-controlled benchmarking pipeline

---

## Why This Matters for the Role

This responsibility is listed as one of the core ongoing duties — not a milestone. It signals that AMD is looking for someone with the **technical depth to be credible** across all five team types and the **interpersonal and communication skills** to translate that credibility into organizational influence. For a performance engineer, being right about a bottleneck is necessary but not sufficient — the finding must propagate to the people who can fix it, at the right time, framed in the right way.

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: In the context of the attached job description, thoroughly explain the following responsibility: "Partner with architecture, product, driver, compiler, and application teams to influence technical direction and support product readiness." Write the description in Markdown format.
