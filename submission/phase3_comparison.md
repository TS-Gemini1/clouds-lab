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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->27.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.015<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->2.4<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->4<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The local run was the fastest (0.015 s), then Spark (2.4 s), and the cloud MapReduce was by far the slowest (27.7 s). Local and cloud are the fair comparison, because they did exactly the same word count. The data is so small that the cloud job barely spent any time counting. Most of the time went into calling the functions over HTTP, cold starts, saving and reading the shuffle files in Cloud Storage, and waiting for retries in the chaos run. Spark actually did more work (TF-IDF), but on one machine it was still much faster. So the cloud job was slow because of coordination, not because of the counting.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Spark had 4 stages, which means 3 shuffles. Each one comes from an operation that has to group data by key. The reduceByKey in term_freq, the reduceByKey in doc_freq, and the join. In the chaos run, 4 out of 16 tasks failed and got retried, so about 25%. That is a bit less than 30%, but with only 16 tasks some randomness is normal.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
In spark_tfidf.py, flatMap, map, reduceByKey and join are all transformations, and collect is the only action. When I ran build_tfidf_rdd, nothing was actually computed. Spark just built the plan, which is how toDebugString could print it straight away. Only when I called collect did Spark start a real job, with 4 stages and 12 tasks, and give me the 5975 rows.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If the worker holding one output partition of the first reduceByKey (the TF step) died, Spark would only need to recompute that one lost partition. It would start at the first shuffle boundary, between stage 0 and stage 1, and rebuild it by re-reading the shuffle files that stage 0 already wrote. The other partitions and the earlier work can be reused. Only if those shuffle files were lost too would Spark rerun the stage 0 tasks from the original documents.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->the mr-worker function<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->the Cloud Storage bucket<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the map_phase parallel step itself. A parallel step only finishes when all of its branches have finished, so reduce_phase, the next step, cannot start until the last map task is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retrying is safe because a map task always writes the same shuffle files and overwrites them. If it runs twice, the result is exactly the same as running it once. If run_map appended its pairs instead, a retried task would add its words a second time, so some counts would be doubled and the final word count would be wrong.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3, Logs Explorer showed 4 requests to mr-worker that returned 503 (the simulated worker crash), all at the start of the map phase. The workflow waited a moment and retried each one, and every retry worked. The job still finished with SUCCEEDED, but took 27.7 s instead of 12.1 s, and the result was still identical to the local reference (2221 words).
<!--/answer-->
