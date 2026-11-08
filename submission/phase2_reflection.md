# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->145.05<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.963<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->182594.1<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->9890.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5003<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2082.949 ms, 2 threads: 1150.521 ms, 4 threads: 927.653 ms, 8 threads: 860.84 ms. Fastest: 8 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time in the Python interpreter: every multiply-add is a separate bytecode operation on one boxed number, with type checks and memory lookups in between, so the CPU’s arithmetic units sit mostly idle. The vectorized version hands whole matrices to NumPy’s BLAS, compiled C that streams contiguous memory through SIMD (AVX) instructions processing several floats per cycle, with good cache use. Same arithmetic, 150× faster (145.05 ms vs 0.963 ms).
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Going from 1 to 2 threads gave a 1.8× speed-up (2083 → 1151 ms), but 2 → 4 gave only 1.24× and 4 → 8 just 1.08× (928 → 861 ms). The curve flattens after 4 threads. I ran locally on a laptop with 28 logical CPUs, not Colab’s 2 vCPUs, so the limit wasn’t missing cores. The matrices in this small model (batch 16) are too small to split usefully: threads must synchronise after every operation and share memory bandwidth, and parts of each step (Python, the optimizer) stay serial.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is dominated by large, dense matrix multiplications (attention and feed-forward layers) where every output element is independent, which is exactly what a GPU’s thousands of cores do in parallel. Training was 18.5× faster on the GPU (182594 ms vs 9891 ms for 100 steps). A workload that would not benefit: generating text one token at a time with batch size 1, or branch-heavy sequential Python code, where each step depends on the previous one and the GPU cores sit idle.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stopped helping for a different reason in each experiment. Vectorization won 150× (145.05 → 0.963 ms) because the naive loop wasn’t hardware-limited at all; it simply wasn’t using the SIMD units the CPU already had. The thread sweep hit coordination cost and serial work: 8 threads gave only 2.4× over 1 thread (2083 → 861 ms), and 4 → 8 only 7%, even though my laptop has 28 logical CPUs, so the bottleneck was synchronisation, shared memory bandwidth and the parts of each step that cannot be divided (Amdahl’s law). The GPU brought hardware the CPU doesn’t have, giving 18.5× (182594 → 9891 ms), but it was capped by data movement and per-step overhead: a 4.9M-parameter model with small batches can’t fill the GPU, so kernel launches and host-to-GPU transfers take a large share of each step.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Repeat the CPU-vs-GPU run with batch sizes 16, 64 and 256, and extend the thread sweep to 16 and 28 threads. If the GPU speed-up grows well past 18.5× with bigger batches, the limit was per-step overhead and data movement, not compute. If 16+ threads get slower than 8, coordination cost dominates.
<!--/answer-->
