# Lecture 1 Introduction

## Why do people build distributed systems?

*  to increase capacity via parallelism
*  to tolerate faults via replication
*  to place computing physically close to external entities
*  to achieve security via isolation

## But:

* many concurrent parts, complex interactions
* must cope with partial failure
* tricky to realize performance potential

## Four Topics

This is a course about infrastructure for applications.
  * **Storage.**
  * Communication.
  * **Computation**.

**The big goal: abstractions that hide the complexity of distribution.**

### Implementation

RPC, threads, concurrency control.

### Performance

The goal: scalable throughput

### Fault Tolerance

We'd like to hide these failures from the application.

* Availability -- app can make progress despite failures
* Recoverability -- app will come back to life when failures are repaired

Big idea: replicated servers.   （not use the non-volilate disk!）

### Consistency

*  Consistency and performance are enemies.

     Strong consistency requires communication,
        e.g. Get() must check for a recent Put().
     Many designs provide only weak consistency, to gain speed.
        e.g. Get() does *not* yield the latest Put()!

## Case : MapReduce

Purpose : multi-hour computations on multi-terabyte data-sets ,eg. build index or sort or analyze of the web

overall goal: easy for non-specialist programmers ,because programmer just defines Map and Reduce function s , which are  often fairly simple sequential code
  	MR takes care of, and hides, all aspects of distribution!

* Scales well.
  + Easy to program -- failures and data movement are hidden.

### Abstract view of a MapReduce job

* input is (already) split into M files

* **Input and output are stored on the GFS cluster file system**

  ```
    Input1 -> Map -> a,1 b,1
    Input2 -> Map ->     b,1
    Input3 -> Map -> a,1     c,1
                      |   |   |
                      |   |   -> Reduce -> c,1
                      |   -----> Reduce -> b,2
                      ---------> Reduce -> a,2
  ```

*  MR calls Map() for each input file, produces set of k2,v2
        "intermediate" data
        each Map() call is a "task"
    
*   MR gathers all intermediate v2's for a given k2, and passes each key + values to a Reduce call
     final output is set of <k2,v3> pairs from Reduce()s

```pesudo
Example: word count
  input is thousands of text files
  Map(k, v)
    split v into words
    for each word w
      emit(w, "1")
  Reduce(k, v)
    emit(len(v))
```

### MapReduce hides many details:

*   sending app code to servers
*  tracking which tasks are done
*  moving data from Maps to Reduces
*   balancing load over servers
*   recovering from failures

### Input and output are stored on the GFS cluster file system

* MR needs huge parallel input and output throughput.
    GFS splits files over many servers, in 64 MB chunks
      Maps read in parallel
      Reduces write in parallel
* GFS also replicates each file on 2 or 3 servers
    Having GFS is a big win for MapReduce

### Details for the procedure of map reduce 

  one master, that hands out tasks to workers and remembers progress.

1. master gives Map tasks to workers **until all Maps complete**
   Maps write output (intermediate data) to local disk
   Maps split output, by hash, into one file per Reduce task
2. **after all Maps have finished, master hands out Reduce tasks**
   each Reduce fetches its intermediate output from (all) Map workers
   each Reduce task writes a separate output file on GFS

![image-20201223001156163](C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20201223001156163.png)

What will likely limit the performance? => network  capacity

### How does MR get good load balance?

  Solution: many more tasks than workers.

### What about fault tolerance?

* What if a worker crash during a  mapreduce.
**MR re-runs just the failed Map()s and Reduce()s.**

* Suppose MR runs a Map twice, one Reduce sees first run's output,another Reduce sees the second run's output?

Correctness requires re-execution to yield exactly the same output. So Map and Reduce must be pure **deterministic functions**: they are only allowed to look at their arguments. no state, no file I/O, no interaction, no external communication.

* What if you wanted to allow non-functional Map or Reduce?

  Worker failure would require whole job to be re-executed,or you'd need to create synchronized global checkpoints.

### Details of worker crash recovery:

  * Map worker crashes:

    master notices worker no longer responds to pings.
    master knows which Map tasks it ran on that worker.

    * Those tasks' intermediate output is now lost, must be re-created ,master tells other workers to run those tasks
    * can omit re-running if Reduces already fetched the intermediate data
  * Reduce worker crashes.
    finished tasks are OK -- stored in GFS, with replicas.
    master re-starts worker's unfinished tasks on other workers.