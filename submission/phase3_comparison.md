# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->50.2<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.011<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.8<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking: local single-process word count 0.011 s < Spark TF-IDF 4.8 s < cloud MapReduce word count 50.2 s.

The like-for-like comparison is local vs cloud. Both run the same word count on the same 60 documents. They produce the same output.

Local single-process is the fastest. It runs in one Python process with all data in memory. Its time is almost pure computation.

Spark takes longer. Its job is also heavier, because TF-IDF needs two `reduceByKey` and a `join`. But most of its time is overhead:

- it starts the Python worker processes
- it serializes data between the JVM and Python
- it writes and reads shuffle files on local disk

Stage 0 alone took about 2.7 s, mostly to start the Python workers.

The cloud job takes the most time. Its computation is tiny. Almost all of its time is distributed-system overhead:

- Cloud Run cold starts
- one authenticated HTTP call per task, orchestrated by Workflows
- about 65 reads and writes to Cloud Storage
- the barrier, which waits for the slowest mapper
- retry backoff, because this was a `--chaos 0.3` run

Spark does more work than the cloud job and is still 10 times faster. This shows that the cloud job spends its time on overhead, not on computation. At this data size, overhead dominates.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job has 4 stages, so it has 3 shuffles. They come from:

- the `reduceByKey` in `term_freq`, which sums the counts of each term in each document
- the `reduceByKey` in `doc_freq`, which counts the documents that contain each term
- the `join` of tf and df in `build_tfidf_rdd`

The cloud job has 16 tasks. With 8 retries, they needed 24 attempts. So 8 out of 24 attempts failed, about 33%. This matches `--chaos 0.3`, which makes each attempt fail with probability 30%. The expected number of retries is about 16 × 0.3 / 0.7 ≈ 6.9, so 8 is a normal result.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations are lazy. They only extend the lineage. In this job they are:

- `flatMap`, `map` and `reduceByKey` in `term_freq`
- `map` and `reduceByKey` in `doc_freq`
- `map`, `join` and the final `map` with `to_row` in `build_tfidf_rdd`

`parallelize` only creates the source RDD from the list of documents.

The only action is `collect`.

What happened when I ran them:

- `build_tfidf_rdd` returned immediately and computed nothing. `toDebugString` is not an action. It just printed the recorded lineage.
- `collect` started job 0. The job ran 4 stages and 12 tasks. It returned 5975 rows to the driver in 4.8 s.

The Parquet file is written afterwards by pyarrow on the driver. That step is not Spark.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first `reduceByKey` is in `term_freq`, at line 31. It is shuffle boundary 1, between stage 0 and stage 1. In the lineage it is the step from `PairwiseRDD[2]` to `ShuffledRDD[3]`. Recovery starts at this boundary.

If one of its output partitions is lost:

- Spark recomputes only that partition. It reads the shuffle files written by stage 0 again.
- If the dead worker also held some stage 0 shuffle files, Spark re-runs only those stage 0 tasks. It starts from the parallelized input.
- All other partitions and all other shuffle files are reused.

I saw this reuse directly. I accidentally ran `collect` twice in the same session. In the second job only the final stage, stage 7, was executed. Its three earlier stages were SKIPPED, because their shuffle files were still there.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows. The `mr-wordcount` workflow in `workflow.yaml` fans out the tasks, provides the barrier and retries failed tasks.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function `mr-worker`, defined in `main.py`. Each map or reduce task is one invocation. It runs as the `mr-runner` service account.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->The Cloud Storage bucket `eurecomgpt-montini-eurecomgpt`. It holds the input splits, the shuffle files and the outputs of each job.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The `partition` function in `mapreduce.py`. It computes `zlib.crc32` of the word modulo the number of reducers. `run_map` in `main.py` calls it.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the `map_phase` step itself. A `parallel` step completes only when all its `for` branches have finished. `reduce_phase` is the next step. So it cannot start until the last map task has returned, retries included.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retrying is safe because each map task is idempotent:

- it writes to fixed names, `<job>/shuffle/split-i/part-r.json`
- `upload_from_string` overwrites a file that already exists
- the same input split always produces the same files

So running a task twice leaves the same state. This holds after a crash. It also holds if a timeout fires after the work was already done. Reduce tasks overwrite their outputs in the same way.

If `run_map` appended its pairs instead, a retried map would add them a second time. The reducers would count those words twice. The result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The run with `--chaos 0.3` still ended with `SUCCEEDED`. It took 50.2 s, against 44.4 s without chaos. It found 2221 distinct words, and the result was identical to the local reference.

In Cloud Logging I looked for `mr-worker` requests with HTTP status 503. I used `gcloud logging read` with the same filter as Logs Explorer. I found 8 failed requests:

- 7 within about 4 s at the start of the job, during the map phase
- 1 about 13 s later, probably a reduce task

The workflow retried each one with backoff. The extra time comes mostly from these backoff waits.
<!--/answer-->
