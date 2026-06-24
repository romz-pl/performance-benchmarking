# Cross-Platform Development Experience

> [!NOTE]
> 
> "Exposure to cross-platform development in environments such as Windows, Linux, macOS, or Android."

## What the Requirement Means

In the context of this role, "cross-platform development" does not simply mean writing code that compiles on multiple operating systems. It means understanding how **the same workload behaves differently across hardware/OS combinations** — and why. AMD's product portfolio spans consumer PCs (Windows), developer/server workloads (Linux), creative professionals (macOS via Radeon and Ryzen mobile), and increasingly mobile/embedded targets (Android). A performance engineer in this role must be able to run, instrument, and compare benchmarks across these surfaces without being blind to OS-level artefacts that distort the signal.

---

## Why It Matters for Benchmarking and Automation

Benchmarking infrastructure that targets only one OS is brittle. Cross-platform exposure shapes the engineer's ability to:

- **Design portable benchmark harnesses** — automation frameworks must handle differences in process management, file I/O paths, environment variables, shell semantics (PowerShell vs. bash), and scheduler policies without the benchmarks themselves becoming the source of noise.
- **Attribute performance differences correctly** — a 15% throughput gap between Linux and Windows for the same binary on the same silicon may originate in the kernel scheduler, the memory allocator (`jemalloc` vs. the Windows heap), driver stack depth, or NUMA affinity policy. Misattributing OS overhead as hardware regression leads to wrong product decisions.
- **Validate driver and compiler output across targets** — AMD ships AMDGPU drivers, ROCm (Linux-first), and Windows-only WDDM/DirectX paths. A performance regression may appear on one OS and not another because the compiler backend or driver path diverges. Cross-platform experience allows the engineer to isolate the variable.

---

## OS-Specific Performance Considerations Relevant to AMD Silicon

### Windows
- Primary surface for consumer CPU/GPU benchmarks (gaming, AI PC, NPU workloads).
- Profiling via **WPA (Windows Performance Analyzer)** and **ETW traces** is the standard; understanding the Windows thread scheduler, DPC latency, and DirectX/WDDM submission paths is essential.
- Power plans, SMT scheduling, and Heterogeneous Core (Zen architecture with efficiency cores) behaviour differ significantly from Linux `cpufreq` governors.

### Linux
- Dominant environment for data centre, HPC, and ROCm/GPU compute benchmarks.
- Fine-grained control over `perf`, `eBPF`, `numactl`, `taskset`, and `cgroups` enables precise performance isolation that is harder to achieve on Windows.
- AMD's ROCm stack is Linux-native; GPU kernel performance characterisation for AI/ML workloads happens primarily here.

### macOS
- Relevant for Radeon Pro GPU validation on Apple platforms (prior to Apple Silicon displacement) and for cross-vendor compiler/toolchain comparison.
- Apple's Metal API and distinct memory model (unified memory) create a meaningfully different baseline that stress-tests the portability of benchmark assumptions.

### Android
- Growing target for AMD embedded and APU strategies; relevant as AI inference moves to edge devices.
- Introduces additional complexity: ARM architecture (vs. x86), highly constrained thermal envelopes, heterogeneous CPU topologies (big.LITTLE), and Android's power management aggressively throttling sustained workloads — all of which can invalidate naive benchmark-to-benchmark comparisons.

---

## Practical Skills This Experience Implies

| Skill | Why It Is Needed in This Role |
|---|---|
| CMake / cross-compilation toolchains | Building the same benchmark binary for multiple targets reproducibly |
| Containerisation (Docker, WSL2) | Isolating OS environment variables in automated pipelines |
| CI/CD across OS runners | Running nightly perf regression suites on heterogeneous agent pools |
| Platform-specific profiler literacy | Mapping `uProf` / `VTune` (Windows/Linux) findings to the same hardware counter semantics |
| Scripting in both PowerShell and bash/Python | Writing automation that does not break when the OS underneath changes |
| Understanding ABI and runtime differences | Catching cases where `double` alignment, SIMD alignment requirements, or calling conventions silently change benchmark results |

---

## How This Connects to the Broader Role

The job description explicitly lists "Lead the design and evolution of benchmarking infrastructure, automation frameworks, and workload pipelines." An infrastructure that is OS-agnostic from the outset scales across AMD's full product matrix — from a Ryzen AI PC running Windows 11 to a Radeon Instinct node running ROCm on Linux. Cross-platform experience is therefore not a secondary preference; it is what separates a benchmarking system that serves one product team from one that serves the entire organisation.

---

> [!NOTE]
> 
> Generated by Claude.ai
>
> Model: Sonet 4.6
>
> Prompt: In the context of the attached job description, thoroughly explain the following preferred experience: "Exposure to cross-platform development in environments such as Windows, Linux, macOS, or Android." Write the description in Markdown format.
