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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->458.471<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->12.0<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->456327.5<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13707.3<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4861<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2266.92 ms, 2 threads: 2835.983 ms, 4 threads: 2289.415 ms, 8 threads: 2398.374 ms. Fastest: 1 thread.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version is slower because it executes one dot product at a time and during each of these executions it uses the python interpreter. It needs 458 ms to perform approximately 131,000 iterations (2 × 256 × 256), and in each one, the interpreter overhead far outweighs the actual useful work—a dot product of just 64 numbers.

The vectorization is faster because the python interpreter is called only once. The entire multiplication between matrices is performed by a precompiled library: OpenBLAS. It requires 12 ms. OpenBLAS uses AVX2/FMA SIMD instructions, each of which processes multiple numbers simultaneously (4 doubles—multiplication plus addition in a single operation) and reuses data stored in the cache.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Parallelism stopped helping at one thread. This happens because the VM has 2 vCPU but only 1 physical core. 

lscpu: 1 core × 2 threads

Therefore having more than one thread implies synchronizing the threads on the same core and same cache without any advantage in terms of time required for computation.

The plot shows that the time required to compute one step never decreases: the minimum is with 1 thread. The difference between the times with 4 and 8 threads may be due to noise, since each measurement is computed as the median of three mesurements and the VM is shared.

```
1  thread   2267 ms  
2  threads  2836 ms
4  threads  2289 ms
8  threads  2398 ms
```
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
The GPU was faster because the computation required was a multiplication between matrices, which implies performing identical, indipendent operations. This is the work most suitable for a GPU: Single Instruction Multiple Thread. The increase in performance is 33x: 

```
CPU time    4.6 s per step
GPU time    137 ms per step
```

The workload which doesn't benefit from GPU architecture is a computation with sequential dependencies or many branches, or a task that is too small, where the cost of data transfer and kernel launching outweighs the cost of the computation itself. This kind of workload is more suited for a multicore CPU which uses the MIMD principle.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
- work that cannot be divided: I encountered this limit during the GPU experiment. The training steps can't be parallelized, since every step depends on the previous one. However, it's possible to parallelize inside each step: the attention can be computed in a vectorized way. The fact that parallelization is possible only inside a single step contributes to justify the relatively small increase in performance between GPU and CPU (gpu_ms = 13707.3 ms, cpu_ms = 456327.5 => x33).

- hardware I don't have: I encountered this problem during Multi-threading and Attention experiments. In the former, since I only have one physical core, increasing the number of threads doesn't add computing capacity. In the latter, the vectorized version achieved a 38× speedup (458 → 12 ms) by distributing the workload across the SIMD lanes of a single core; going any further would require additional physical cores, which the VM does not have.

- coordination that costs more than it saves: I encountered this problem during the Multi-threading experiment. Adding one thread worsens the performance because they share the same physical resources (core, cache), therefore they gain nothing in performance and they must synchronize at each operation.

```
1  thread 2267 ms  vs 2  threads 2836 ms (+25%)
```


- data has to move: At each step, `get_batch` creates the batch in the CPU's RAM, and `.to(device)` copies it to the GPU's VRAM;
`loss.item()` transfers the loss from the GPU back to the CPU, forcing the CPU to wait for the GPU. Since moved data is little, the cost is due to latency and waiting, not volume: part of each 137 ms/step.

<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the experiment CPU vs GPU with a larger batch. If the speedup increases it means we weren't exploiting the full potential of the GPU so the provided explanation was correct. If it remains close to 33x, there may be another explanation for the performance gap between CPU and GPU
<!--/answer-->
