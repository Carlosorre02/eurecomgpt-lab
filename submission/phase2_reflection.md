# Phase 2 — Reflection: where does parallelism stop helping?

## 1. Your measurements

| Measurement | Value |
|---|---:|
| Naive attention, ms | <!--answer:attn_naive_ms-->449.328<!--/answer--> |
| Vectorized attention, ms | <!--answer:attn_vectorized_ms-->1.014<!--/answer--> |
| CPU training, ms for 100 steps | <!--answer:cpu_ms-->432132.8<!--/answer--> |
| GPU training, ms for 100 steps | <!--answer:gpu_ms-->14479.9<!--/answer--> |
| Final training loss | <!--answer:final_loss-->1.4135<!--/answer--> |

**Thread sweep:**

<!--answer:thread_sweep-->
2252.159, 2084.873, 2466.297, 2159.728

These are milliseconds per step for 1, 2, 4 and 8 threads, respectively.
The fastest measured configuration was 2 threads: 2084.873 ms/step.
<!--/answer-->

The completed notebook and threading plot are included in
[phase2_notebook.ipynb](phase2_notebook.ipynb).

## 2. Intra-node parallelism, three ways

**Why vectorizing wins**

<!--answer:vectorization_why-->
Naive attention took 449.328 ms, while vectorized attention took 1.014 ms.
The naive implementation repeatedly executes scalar operations through Python
loops, incurring interpreter overhead. Vectorization delegates matrix operations
to compiled numerical kernels. NumPy reported OpenBLAS and available SIMD
extensions, enabling optimized execution and processing multiple values per
instruction. Better memory locality can also contribute. The measured speedup
was 443.12 times; it cannot be attributed exclusively to SIMD.
<!--/answer-->

**Where the thread curve flattens, and why**

<!--answer:threading_plateau-->
The runtime reported 2 logical CPUs and an affinity covering 2 CPUs.
Moving from 1 to 2 threads reduced step time from 2252.159 to 2084.873 ms,
approximately a 1.08-times speedup. More threads did not improve that minimum:
4 took 2466.297 ms and 8 took 2159.728 ms. Extra threads share the same
available CPU resources and coordinate work and synchronization. Scheduling,
cache and memory contention are plausible contributors, but were not measured
separately. These non-monotonic results do not establish a universal optimum.
<!--/answer-->

**What made the GPU faster**

<!--answer:gpu_why-->
The CPU needed 432132.8 ms for 100 training steps, versus 14479.9 ms on
the T4, approximately a 29.8-times speedup. Transformer training contains
matrix multiplications in attention, projections and gradient computation,
providing substantial parallel work for GPU kernels. Here that benefit
outweighed launch and data-transfer overheads. A tiny, sequential,
branch-heavy task with little numerical work would not necessarily benefit:
GPU setup and transfers could cost more than the computation saved.
<!--/answer-->

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
The experiments expose different limits. Vectorization reduced attention
time from 449.328 to 1.014 ms by removing much Python-loop overhead and
using optimized kernels. It does not parallelize the entire program:
sequential setup, data handling and other unaccelerated work still limit
total speedup through Amdahl's law. We did not measure the remaining
serial fraction.

The thread sweep reached its best measured time at 2 threads, 2084.873 ms,
on a runtime exposing 2 vCPUs. Using 4 threads increased time to 2466.297 ms.
Additional software threads do not create additional CPU capacity, and
coordination can outweigh the benefit.

GPU training reduced 100-step time from 432132.8 to 14479.9 ms. This workload
provided enough parallel computation to justify GPU execution. Kernel
launches and host-device transfers remain costs; small workloads may fail
to amortize them. We did not measure that crossover or establish that
memory bandwidth or transfer overhead dominated these experiments.
<!--/answer-->

**What you would measure next**

<!--answer:next_experiment-->
I would repeat the CPU/GPU comparison across batch sizes, with warm-up runs,
repeated measurements and CUDA synchronization at timing boundaries.
I expect GPU speedup to shrink for tiny batches as fixed overheads dominate.
If speedup remained equally large at the smallest batches, that would weaken
this explanation for this model.
<!--/answer-->