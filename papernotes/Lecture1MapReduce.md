# MapReduce :Simplified Data Processing on Large Clusters

## Abstract

**What is MapReduce?**

MapReduce is a programming model and an associated implementation for processing and generating large data sets.

 **Map:** process a key/value pair to generate a set of **intermediate** key/value pairs. 

**Reduce:** merge **all** intermediate  values associated with the same intermediate keys.

Advantage: high scalable , easy to use , fault tolerance 



## Introduction

**Computations  in the past for special purpose:** process large amount of raw data(crawled documents, web request logs , etc),to compute various derived data  (inverted indices, various representations of the graph structure of web documents, summaries of the number of pages crawled per host).

**Why we use need Map Reduce?**

When the input data is very large and the computations have to be distributed across hundreds or thousands of machines to finish in a reasonable amount time.

**A new abstraction**

We cam express simple computations while we hide the messy details of parallelization , fault-tolerance,data distribution and load balancing in library.

**Map and Reduce Primitives**

Map and Reduce help us parallelize large computations easily and to use **re-execution as the primary** **mechanism** for fault tolerance.



## Programming Model

The computation takes a set of input key/value pairs, and produces a set of  output key/value pairs.

* Map: written by user, take an input pair and produces a set of intermediate key/value pairs. The MapReduce library **groups together all intermediate values associated with the same intermediate key *I* ** and passes them to the **Reduce** function.
* Reduce: accepts an intermediate key *I* and a set of values for that key. It merges together these values to form a possibly smaller set of values.  The intermediate values are supplied to the user’s reduce function via an iterator . This allows us to handle lists of values that are too large to fit in memory.

Examples : count number of occurrence , distributed grep, count of URL access frequency, reverse web-link graph, term-vector per host, inverted index , distributed sort.



## Implementation

The implementation depends on the environment.

* large clusters of commodity PCs connected together with switched Ethernet
* machines are typically dual-processor x86 processors running linux 
* commodity networking hardware 
* consist of hundreds of machines
* storage is provided by inexpensive disks . A distributed file system developed in-house is used to manage the data stored on  these disks.
* user submit jobs to a scheduling system

### Execution Overview



<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210126202103661.png" alt="image-20210126202103661" style="zoom:50%;" />

The **Map invocations** are distributed across multiple machines by automatically partitioning the input data into a set of **M** splits. The input splits can be processed in parallel by different machines.

**Reduce invocations** are distributed by **partitioning the intermediate key space into R  piece**s using a partition function (Mod R?).

1.  The MapReduce library in the user program first splits the input files into **M** pieces (16-64MB). It then **starts up many copies of the program** on a cluster of machines.
2. **One of the copies of the program is special : master!**  The rest are **workers** are assigned work by the **master.** There are **M** map tasks and **R** reduce tasks to assign. The **master** picks an idle worker and assign a map task or a reduce task.
3. The worker that is running  **map** task reads the contents of the corresponding input split. **It parses key/value pairs out of the input data and passes each pair to the user-defined Map function**. The intermediate key/value pair  is buffered in memory.
4. Periodically , the buffered pairs are written to local disk , partitioned into **R** regions by the partition function .The location of these buffered pairs on the local disk are passed back to the **master** who is responsible for forwarding these locations to the reduce workers.
5. When a reduce worker is notified by the master about the locations, it uses **remote procedure calls(RPC)**   to read the buffered data from the local disks of the map workers. **When the reduce worker has read all intermediate data , it sorts it by the intermediate keys so that all occurrences of the same key are grouped together.** The sorting is needed because typically **many different keys map to the same reduce tasks.** If the amount of intermediate data is too large to fit in memory , an external sort is used.(DB )
6. The reduce worker iterates over the sorted intermediate data and for each unique intermediate key encountered, it passes  the key and the corresponding set of intermediate values to the user’s Reduce function . The output of **Reduce** function is **appended** to **a final output file for this reduce partition.**(**GFS**)
7. When **all** map and reduce tasks have been completed, the master wakes up the user program. At this point , the  **MapReduce** call in the user program back to the user code.

After successful completion ,the output of the **MapReduce** is available in the **R** output files (one per reduce task,with file names as specified by the user).

**What is next?**

The **R** out put files do not need to merge , because they often will be passed  as the input files as another **MapReduce  or other distributed applications**.



### Master Data Structure

* **The state(idle ,in-progress or completed)** of each map and reduce task and the identity of the worker machine.
* **Conduit** through which the location of intermediate file regions is propagated from map tasks to reduce tasks. Therefore , for each completed map task, **the master stores the locations and sizes** of the **R** intermediate files regions **produced by the map task**. **Updates** to this location and size information are **received** as map tasks are completed. The information is **pushed** incrementally to workers that have *in-progress*  reduce tasks.



### Fault Tolerance

#### Worker Failure

**How to know the worker fails?**

The **master** pings every worker periodically . If no response is received from a worker in a certain amount of time , the **master** marks the worker failed.(heart beat message.)

**Different situations for worker failure.**

* any map tasks completed by the worker are reset back to their initial **idle** state and need to re-executed ,because their output is stored on the local disks that is inaccessible because the worker failure.
* any map or reduce task in progress on a failed worker is also **reset** to **idle** state and becomes eligible for rescheduling 
* Completed **reduce** task do not need to be re-executed since their output is stored in a global file system.

**What we need to do when reschedule.**

When map task A is executed first by worker A and then later executed by worker B (A failed), **all workers executing reduce tasks are notified of the re-execution,because they may read the data of the local disk in A **. Any reduce task  must read data from worker B.

   **MapReduce is resilient to large-scale worker failures.** 

#### Master Failure

It is easy to make the master write periodic checkpoints of the master data structures described above.

If there is only one master , the client can check for this condition and retry the **MapReduce** operation if they desire.

#### Semantics in the Presence of Failures

When the user-supplied map and reduce operators are **deterministic functions** of their input values, our distributed implementation **produces the same output as would have been produced by a non-faulting sequential execution of the entire program.**

We **rely on atomic commits** of map and reduce task outputs to achieve this property.

?

When the map and/or reduce operators are nondeterministic, we provide **weaker but still reasonable** semantics. In the presence of non-deterministic operators, **the output of a particular reduce task R1 is equivalent to the output for R1 produced by a sequential execution of the non-deterministic program.** However, t**he output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program.** 

Consider map task M and reduce tasks R1 and R2. Let e(Ri) be the execution of Ri that committed (there
is exactly one such execution). **The weaker semantics arise because e(R1) may have read the output produced** **by one execution of M and e(R2) may have read the output produced by a different execution of M.**
		?

### Locality

Read data from local disk  (which is the machine that has the replica)

**Network Bandwidth** is a relatively scarce resource in our computing environment. We converse network by taking advantage of the fact that the input data (**managed by GFS**) is **stored on the local disks of the machine** that make up out cluster.

**GFS** divides each file into 64MB block, and store several replicas on  different machines . The **MapReduce** master takes the location of the input files into account (which is managed by GFS master) and attempts to schedule a **map task** on the machine that has the **replica of the corresponding input data.**  If we can not do that , the master attempts to assign the **map** task to the **machine which is near the replica of the data**. (i.e. on a worker machine that is on the same network switch as the machine containing the data).

### Task Granularity

**Map: M pieces Reduce: R pieces**

Ideally,M and R should be much larger than the number of worker machines. 

Advantages: 

* each worker perform many different tasks improve dynamic load balancing 
* speed up recovery when worker failure, the task is small ;the many map tasks it has completed can be spread out across all the other worker machines



**Practical bounds for M and R**

Master must make **O(M+R) schedule** and keeps **O(M*R) state in memory** (the constant factors for memory usage are small : per map task  one byte.)

**R** is considered by user , are R separated files may be used by distributed applications or MapReduce.

**M**  ranges roughly from 16MB - 64 MB considering about locality ,because the chunk size is 64MB.

M = 200000 R =5000 using 2000worker

### Backup Tasks

**How to lengthen the total time taken for a MapReduce operation?**

**Straggler.** a machine that takes an **unusually long time** to complete **one of the last few map or reduce tasks in the computation.**

**Why straggler?**

* a machine with a bad disk may experience frequent correctable errors that slow its read performance from 30 MB/s to 1 MB/s. 
* The cluster scheduling system may have scheduled other tasks on the machine, causing it to execute the MapReduce code more slowly due to competition for CPU, memory, local disk, or network bandwidth.
* a **bug** in machine initialization code that caused processor caches to be disabled: computations on affected machines slowed down by over a factor of one hundred.

**How to alleviate the problem?**

* When a MapReduce operation is close to completion, the master schedules **backup executions of the remaining in-progress tasks. **The task is marked as completed whenever either the primary or the backup execution completes.
* We have tuned this mechanism so that it typically increases the computational resources used by the operation by no more than a few percent. We have found that this significantly reduces the time to complete large MapReduce operations. 

## Refinements



### Partitioning Function

The usually partition function is **Mod** depends on the number of output files that user want.

If we deal with the URL and we want  all entries for a single host to end up in the same output file , we may need **Hash( Host Name ( Url Key))**.

### Ordering Guarantees

We **guarantee**  that within a given partition, **the intermediate key/value pairs are processed in increasing key order.** This ordering guarantee **makes it easy to generate a sorted output file format** needs to support efficient random access lookups by key , or user of the output find it convenient to have the data sorted.

### Combiner Function

The combiner function is executed on each machine that performs a map task. Typically the same code is used on the reduce function.

**The different**

How the **MapReduce** Library handles the output of the function.

* The output of the reduce function is written to the final output file
* The output of a combiner function is written to an intermediate file that will be sent to a reduce task.

In the case of the word count , many words in each map task will produce the record “word,1” , these records will be merge in the reduce task finally , cost much. We need to combine it after the finish of the map and before sending it to the reduce.

### Input and Output Types

Input types:

* **Text:** treat each line as a key/value pair ; 
  * the key is the offset in the file 
  * the value is the contents of the line.
* Another format : a sequence of key/value pairs sorted by key
* add supported format through implementing the interface **reader**

It is similar in output types. 



### Side - effects

In some cases , users of MapReduce have found it **convenient to produce auxiliary files** as **additional outputs** from their map and/or reduce operators.  We rely on the application writer to **make such side-effects atomic and idempotent.**   

Typically the application writes to a temporary file and **atomically renames** this file once it has been fully generated .

We do not provide support for atomic two-phase commits of multiple output files produced by a single task. Therefore , tasks that produce multiple output files with cross-file consistency requirements should be  **deterministic** . (每次输出都需要相同)This restriction has never been an issue in practice.



### Skipping Bad Records

Sometimes there are bugs in user code that cause the **Map or Reduce** to crash deterministically on certain record.

* fix the bug , sometimes it not feasible ,perhaps the bug is in the third-party library 
* ignore a few records that cause the bug

Each worker process **installs a signal handler that catches  segmentation violations and bus errors**. Before invoking a user Map or Reduce operation, the MapReduce library **stores the sequence number of the argument in a global variable** . If the user code generates a signal, **the signal handler sends a “last gasp” UDP packet that contains the sequence number to the MapReduce master**. When the master has seen more than one failure on a particular record, it indicates that **the record should be skipped when it issues the next re-execution of the corresponding Map or Reduce task**.



### Local Execution

help debugging profiling , small-scale test 

The implementation of MapReduce support that we can  sequentially  execute all of the work for a MapReduce operation on the local machine.

### Status Information

The master runs an internal HTTP server and exports a set of status pages for human consumption.

* the progress of computation such as how many tasks have been completed , how many are in progress , bytes of input , bytes of  intermediate data , bytes of output
* links to standard error and standard output files generated by each task
* top-level status page shows which worker have failed , and which map and reduce tasks they were processing when they failed

The user can use these data to predict how long the computation will cost and whether or not more resources should be added to the computation. These pages can also be used to figure out when the computation is much slower than expected

### Counters

The MapReduce library provides a counter facility to count the occurrences of various events.