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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->246.847<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->0.983<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->435613.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->14798.7<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4803<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2665.196 ms, 2 threads: 2039.544 ms, 4 threads: 2204.024 ms, 8 threads: 2238.965 ms. Fastest: 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version runs T x T = 65,536 dot products through Python loops: every multiply goes through the interpreter, with type checks and boxed floats, one scalar at a time, so the CPU spends its cycles on overhead and cache misses instead of arithmetic. The vectorized version hands whole matrices to OpenBLAS, which runs compiled, cache-blocked matrix multiplies with AVX2/AVX-512 SIMD instructions (X86_V3/V4 found) that process 8-16 floats per instruction: 251x faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Time per step falls from 2665.196 ms at 1 thread to 2039.544 ms at 2 threads, then rises to 2204.024 ms at 4 and 2238.965 ms at 8. The Colab runtime has only 2 vCPUs (one physical core with hyper-threading; PyTorch's default was 1 thread), so beyond 2 threads there is no extra hardware: extra threads time-share the same core. Each matrix multiply must still be split, synchronised at barriers and merged, and all threads compete for the same caches and memory bandwidth, so coordination costs more than it saves. Even 1 to 2 threads gives only 1.31x, because two hyper-threads share one core's execution units.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is dominated by dense matrix multiplies (QKV projections, attention scores, MLP) over a batch of 32 x 256 tokens: millions of independent multiply-adds with regular memory access, exactly what the T4's 2,560 CUDA cores run in parallel (29.4x faster than the CPU). A workload that would not benefit: sequential, branchy code with little arithmetic per byte, such as tokenizing text or generating one token at a time with batch size 1, where kernel-launch and CPU-GPU transfer overhead dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different limit. Vectorization (251x) removed interpreter overhead; after that the attention is bound by SIMD width and memory bandwidth, not by the code. The thread sweep hit hardware I did not have: with 2 vCPUs the best was 2 threads (2039.544 ms vs 2665.196 ms at 1 thread, only 1.31x because hyper-threads share one core). Past that it hit coordination that costs more than it saves: 8 threads took 2238.965 ms. The GPU gave 29.4x, far below the T4's peak: with only 4,935,680 parameters each kernel is short, so launch overhead, Python and moving each batch from CPU to GPU memory (data that has to move) take a large share. Work that cannot be divided remains in all three: layers run one after another and the optimizer waits for the full backward pass (Amdahl's law).
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Sweep the GPU batch size (16, 32, 64, 128) and record tokens per second. If throughput keeps rising almost linearly, the GPU was underused and launch/transfer overhead bound my speed-up; if it plateaus at once, compute was already saturated and my overhead explanation is wrong.
<!--/answer-->
