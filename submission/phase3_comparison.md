# Phase 3 — Comparison Writeup

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds | <!--answer:cloud_elapsed_s-->57.7<!--/answer--> |
| Local single-process wall time, seconds | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for collect, seconds | <!--answer:spark_elapsed_s-->4.6<!--/answer--> |
| Number of stages in the Spark job | <!--answer:num_stages-->4<!--/answer--> |
| Task retries counted in the chaos run | <!--answer:chaos_retries-->6<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
The local word count was fastest (0.01 s), followed by Spark TF-IDF (4.6 s), then cloud MapReduce (57.7 s). Only local and cloud word count perform the same computation, although on different machines. Cloud execution adds workflow scheduling, function startup, Storage requests, polling and retries. Spark performs more work: term frequency, document frequency and a join. Its collect time includes task scheduling and shuffles, but excludes the earlier SparkSession startup. Even this heavier local job finished much sooner than cloud word count, showing how overhead dominates this small corpus.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
The four stages correspond to three shuffles: term_freq's reduceByKey, doc_freq's reduceByKey, and join. Six retries for 16 logical tasks represent 37.5% additional attempts. Across all 22 attempts, 6 returned HTTP 503: a 27.3% failure rate, reasonably close to chaos=0.3 for a small random sample. The logs do not establish how many distinct tasks needed retries.
<!--/answer-->

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
flatMap, map, both reduceByKey calls, and join are lazy transformations. parallelize creates the source RDD; toDebugString inspects its lineage without running the TF-IDF computation. build_tfidf_rdd constructs and returns that dependency graph. collect is the action: it triggers scheduling and computation, then returns the rows to the driver. save_parquet subsequently writes the collected Python rows with PyArrow; it is not a Spark RDD action.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
Recovery starts at the first shuffle, ShuffledRDD[3], between stages 0 and 1. Spark can rebuild the missing term-frequency partition by reading surviving stage-0 shuffle blocks and repeating its aggregation. Unaffected available shuffle outputs can be reused. If the failure also destroyed upstream shuffle blocks, Spark must rerun their producing map tasks from the source documents before recomputing dependent work.
<!--/answer-->

## 3. Your cloud MapReduce, mapped onto Hadoop

**JobTracker:**
<!--answer:role_jobtracker-->
Cloud Workflows: mr-wordcount, defined in workflow.yaml.
<!--/answer-->

**Workers:**
<!--answer:role_workers-->
The mr-worker Cloud Run function, with the mr_worker entry point in main.py.
<!--/answer-->

**HDFS:**
<!--answer:role_hdfs-->
Cloud Storage bucket eurecomgpt-512111-eurecomgpt, holding the job's input, shuffle and output objects.
<!--/answer-->

**Partitioner:**
<!--answer:role_partitioner-->
partition(word, num_reducers) in mapreduce.py: zlib.crc32(word.encode("utf-8")) % num_reducers.
<!--/answer-->

**The barrier.**

<!--answer:barrier_location-->
The map_phase parallel step is the barrier: it completes only when all its branches finish successfully. The following reduce_phase step therefore starts only after every mapper has finished, including any retries.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
Each task reads fixed inputs and overwrites deterministic output paths. Repeating a map task therefore replaces its shuffle objects with the same pairs. If a failed attempt had written some objects, a retry can safely replace them. Appending would instead duplicate pairs already written by an earlier attempt, so reducers would sum duplicates and overcount words.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
With chaos=0.3, the Workflows execution still reached SUCCEEDED. Cloud Logging request logs, inspected through gcloud, showed 6 HTTP 503 responses and 16 HTTP 200 responses: 22 attempts for 16 logical tasks. The client reported 2221 distinct words and identical=True; the report records matches_local=true. The retries therefore recovered the simulated failures without changing the final word counts.
<!--/answer-->
