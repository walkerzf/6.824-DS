# The Google File System

## Introduction / Background

**Shared goals  as the previous distributed file systems** : performance ,scalability,reliability, and availability.

Our design have been driven by key observations of application workloads and technological environment ,**both current and anticipated** , that reflects a marked departure from some earlier file system assumptions.

**traditional choices and radical different points**

* **Component failures are the norm** rather than the exception. inexpensive commodity computers . Problems : app bugs ,OS bugs,human errors ,and the failures of h/w and power supply. **So constant monitoring ,error detection,fault tolerance ,and automatic recovery must be integral to the system.**
* **Files are huge by traditional standards. Multi-GB files are common.** It is unwieldy t o manage billions  of approximately KB-sized file even when the system could support it. **So the IO operation , block-size  have to be revisited**.
* **Most files are mutated by appending new data rather than overwriting existing data. Random write within a file are practically non-existent.** Once written, the file are only-read,and often sequential-read. So the **appending** becomes the optimization focus and atomicity guarantees.
  * some constitute large repo that data analysis programs scan through
  * data streams continuously generate by running apps
  * intermediate data produced by one machine and processed by another machine (MapReduce)
* **co-designing the apps and the file system API benefits** the overall system by increasing flexibility. E.g. relaxed consistent model ;atomic append operation.



## Design Overview

### Assumptions

* The system is built from many **inexpensive commodity** components that **often fail.** The monitor and detect,tolerate ,and recover promptly from failure.
* The system stores a modest number of **large files**. Multi-GB files. Small files exists(script.) but not optimized.
* The **word loads**
  * Read:
    * large streaming reads: individual operations typically read hundreds of KBs, more commonly 1 MB or more. Successive operations from the same client often read through a contiguous region of a file.
    * small random reads: reads a few KBs at **some arbitrary offset.** Performance-conscious applications **often batch and sort their small reads** to advance steadily through the file rather than go back and forth.
  * Write:
    * large ,sequential writes that appending data. Typical operation size is similar to the read operation. Once written, seldom modified,
    * **Small write at arbitrary offset**: not optimized
* **concurrent append to the file(well-defined semantics )**: The file is used as producer-consumer queues or for many-ways merging.
* **High sustained bandwidth** is more important than low latency.  **High Throughput rate**

### Interface

* familiar interface   but not implement  a standard API such as POSIX
* hierarchically in directories and identified by pathname
* usual operation: create delete open close read write
* **snapshot and record append**

### Architecture

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210118165311556.png" alt="image-20210118165311556" style="zoom: 67%;" />

```
Procedure:
First, using the fixed chunk size, the client translates the file name and byte offset specified by the application into a chunk index within the file. Then, it sends the master a request containing the file name and chunk index. The master replies with the corresponding chunk handle and locations of the replicas. The client caches this information using the file name and chunk index as the key. 
The client then sends a request to one of the replicas, most likely the closest one. The request specifies the chunk handle and a byte range within that chunk. Further reads of the same chunk require no more client-master interaction
until the cached information expires or the file is reopened. In fact, the client typically asks for multiple chunks in the same request and the master can also include the information for chunks immediately following those requested. This extra information sidesteps several future client-master interactions at practically no extra cost.
```

A GFS cluster  consists of **a single master** and **multiple chunk servers**  ,and is accessed by multiple clients. Each of these is typically a Linux machine running a user-level server process.

**Files** are divided into fixed-size **chunks(64MB).**  

* an immutable and globally unique 64 bit chunk handle assigned by the master at the time of creation
* Chunk server stores chunks on local disk . When reading,and writing , we need the chunk handle and byte range.
* Replication .By default ,three replicas.

**Master** maintains all file system metadata.

* namespace,access control info,mapping from files to chunks,the current locations of chunks
* system-wide activities such as chunk lease management , gc of orphaned chunks and chunk migration between chunk servers 
* **Heart Beating** between master and each chunk server to give instructions and collect states

GFS client code linked into each application implements the file system API and communicates with master and chunk server to read or write on behalf of the application.

**Clients interact with the master for metadata operations, but all data-bearing communication goes directly to the chunk servers.**

**Neither** the client nor the chunk server caches file data. Because the operation always be stream through the huge files which offer little benefit.

**Clients do cache metadata.**

### Single Master

Advantages:

* vastly simplifies our design
* enables the master to make sophisticated chunk replacement and replication decisions using global knowledge.

Disadvantages need to sidestep

* minimize the master’s involvement in reads and writes so that it does not become a bottleneck
* So in the architecture , the client never read data from master , only ask the master about the metadata.

### Chunk Size

**64MB**

Each chunk replica is stored as a plain Linux file on a Chunk server and is extended only as need. **Lazy space allocation**  avoids wasting space due to internal fragmentation.

Advantages:

* **reduces client’s need to interact with the master** because the read and write operation on the same chunk only need one initial request to the master to get the chunk location information. Even for small reads , the client can comfortably cache all chunk location information for a multi-TB working set.(**64MB chunk => 64 bits)**
* since on a large chunk, **a client is more likely to perform many operations on a given chunk**, it can **reduce network overhead** by keeping a persistent TCP connection to the chunk server over an extended period of time.
* **reduces the size of the metadata** stored on the master. This allows us to keep the metadata in memory.

Disadvantages:

* A small file consists of a small number of chunks, perhaps just one. The chunk servers 
  storing those chunks may become **hot spots** if many clients are accessing the same file.

In practice , **hot spots have not been a major issue** because our applications mostly read **large multi-chunk** files sequentially.

```
However, hot spots did develop when GFS was first used by a batch-queue system: an executable was written to GFS as a single-chunk file and then started on hundreds of machines at the same time. The few chunkservers storing this
executable were overloaded by hundreds of simultaneous requests. 
```

We fixed this problem by storing such executables with **a higher replication factor** and by making **the batch-queue system stagger application start times.** A potential long-term solution is to **allow clients to read data from other clients in such situations.**

### Master  Metadata

In memory for speed.

* the file and chunk namespaces(persistent)
* the mapping from files to chunks (persistent)
* the location of each chunk’s replicas (not-persistent,the master ask each chunk server when startup or other chunk server joins in the cluster )

The first two types metadata are kept persistent by logging mutations to **an operation log** stored on the master’s **local disk** and replicated on **remoted machines.**

#### In-memory Data structures

* The master periodically scan through its entire state in the background to implement **chunk garbage collection,re-replication,chunk migration **

One potential concern for in-memory data structures is that  the number of chunks and capacity of the system is limited by how much memory that the master have.(64MB chunk => less than 64 bytes metadata) . Similarly , the file name space data typically needs less than 64 bytes because it stores file names compactly using prefix compression

#### Chunk Locations

The master does not keep a persistent record of which  chunk server have a replica of a given chunk. It simply **polls** chunk server for the info  at startup. 

* How to keep it up-to-date?

The master controls all chunk placement and monitors chunk server status with regular **HeartBeat** messages

Why this design?

* This eliminate the problem of keeping the master and chunk server in sync as chunk server join and leave the master ,change name ,fail ,restart, and so on.
* The chunk server have the final word about the what chunk does it have  or does not have. The consistent view about chunk on the master can  not avoid the errors on the chunk server.

#### Operation Log(Critical)

What ? **Operation Log** contains a historical record of critical metadata changes.

* persistent record of metadata 
* logical time of the order of concurrent operation

What is the time of changes is visible to the clients.

* We replicate the log operations on multiple remote machines and respond to a client operation only after  flushing the corresponding operation log to the disk both locally and remotely. (The master batches several log records together before flushing  thereby reducing the impact of flushing and replication on overall system throughput.)

**When master crashed , how it recover?**

The master recovers its file system state by replaying the operation log. To minimize startup time , the master checkpoints its state whenever log grows beyond a certain size so that  it can recover by loading the latest checkpoint from local disk ,and replaying only the limited number of log records

```
The checkpoint is a compact B-tree like form that can be directly mapped into memory and used for namespace lookup without extra parsing. This speed up the recovery.
```

**How to build a checkpoint?**

The master switches to a new log file and creates the new checkpoint in a separate thread. The new checkpoint  includes all mutations before the switch. It can be done in a minute or so for a cluster with a few million files without delaying incoming mutations. When completed , it must be written to local disk and remote replicas.

### Consistency Model

**Relaxed Consistency Model**

* GFS’s Guarantees 
* Implications for applications

#### Guarantees by GFS

For master metadata , its mutation is atomic because it is handled by master exclusively : namespace lock ,operation log defines a global total order of  these operations.

The state of file region depend on the type of the operation and  whether concurrent operations.

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210121111337718.png" alt="image-20210121111337718" style="zoom:50%;" />

**What is the meaning of consistency of file region?**

All clients will see the same data regardless of which replicas they read from.

**What is defined for a region?**

A region is **defined** after a file data mutation if it is **consistent** and clients will see what mutation writes in its entirety. So if a file region is **defined** , it must be **consistent.**



* Serial Success (aka many mutations succeed without interference from concurrent writers) : the file region is **defined** (consistent) ,all clients will see which mutation written.
* Concurrent Success: (**consistent but not defined**): clients will see the same data but it may not reflect what any one mutation  has written. **Typically,it consists if mingled fragments from multiple mutations.**
* A failed mutation make the region **inconsistent (not defined)** , different clients may see different data at different times from different replica.

**Our application need to distinguish the defined regions from undefined regions. But they do not need to distinguish between different kinds of undefined regions**

Data mutations : **write(at arbitrary offset)** and **record append(at the tail of the file or chunk,and atomically at least once)**. 

After Data Mutations, t**he offset** is returned to client and **marks the beginning of a defined region** that contains the record.  In addition, GFS may insert padding or record duplicates in between.

**How the mutated file region is guaranteed to be defined and contain the data written by the Last mutation?**

* applying mutations to a chunk in the same order on all its replicas
* using chunk version number to detect any replica that has become stale because it missed mutation while its chunk server was down. (This is the one situation **where the chunk server have the stale replica**. Because the client cache chunk locations , they may read from a stale replica before the information is refreshed.  **The window(the client cache the chunk location but not refreshed)** is limited by the cache entry’s timeout and the next open of the file (**reopen purges from the cache all chunk information for that file**). Moreover , as most of our files are **append-only**, a stale replica usually return a **premature**(过早的) end of chunk rather wrong or outdated data.)  When a reader retries and contacts the master , it will immediately get the current chunk location.

(Stale Replica will never  involved in a mutation or given to clients asking the master for chunk locations , they are GC at the earliest opportunity.)

**How the master/chunk server know fail happens **

* The master identifies failed chunk server by regular **handshake**.
* Detect data corruption  by checksum.

When a problem surfaces , the data is restored from valid replicas as soon as possible. A chunk is lost  irreversibly only if all its replicas are lost before GFS react (minutes). In this case , the client will receive clear error not wrong data.

#### Implications for applications

**The techniques used for GFS applications** : append rather than overwrites,checkpointing ,and writing self-validating ,self-identifying records.

Two typical use for appending mutation:

* In the case **a writer generates a file from beginning to end(1 -1).** It It atomically renames the
  file to a permanent name after writing all the data, or periodically checkpoints how much has been successfully written. **Checkpoints may also include application-level checksums**. Readers verify and process only the file region up to the last checkpoint, which is known to be in the defined state. **Regardless of consistency and concurrency issues**, this approach has served us well. Appending is far more efficient and more resilient to application failures than random writes. Checkpointing allows writers to restart incrementally and keeps readers from processing successfully written file data that is still incomplete from the application’s perspective.
* In the other typical use, **many writers concurrently append to a file for merged results or as a producer-consumer queue（k-1）**. Record append ’ s **append-at-least-once semantics preserves each writer’s output**. Readers deal with the occasional padding and duplicates as follows. Each record prepared by the writer **contains extra information like checksums so that its validity can be verified(self -validating)**. A reader can identify and discard extra padding and record fragments using the checksums. If it cannot tolerate the occasional duplicates (e.g., if they would trigger non-idempotent operations), it can filter them out **using unique identifiers in the records(self-identifying)**, which are often needed anyway to name corresponding application entities such as web documents. These functionalities for record I/O (except duplicate removal) are **in library code shared** by our applications and applicable to other file interface implementations at Google. With that, the same sequence of records, plus rare duplicates, is always delivered to the record reader.

## System Interactions

**Goal:** minimize the master’s involvement in all operations. for avoid becoming bottleneck

A mutation is change the content or metadata of  a chunk as a write or record append.

We use **leases** to maintain a consistent mutation order across replicas. The master grants a chunk **lease** to one of the replicas ,**primary**.  The **primary** picks a serial order for all mutations to the chunk.

The **lease** mechanism is to minimize the management overhead on the master. A **lease** have a initial timeout (60s), As long as the chunk  is being mutated,  the primary can request and typically receive extensions from the master indefinitely. These extensions requests are piggybacked on the Heart Beat messages regularly exchanged between the master and all chunk servers.  

The master can revoke the lease or grant a new lease when the old lease expires.

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210123181149249.png" alt="image-20210123181149249" style="zoom: 33%;" />

1. The client ask the master which **chunk server holds the current lease for the chunk** and **the locations** of the other replicas. If no one has a lease , the master grants one  to be primary.
2. The master replies with **the identity of the primary** and **the locations** of the other replicas. The client **caches** this data for further mutations. It needs to contact the master **again only** when the primary becomes unreachable or it no longer holds a lease.
3. (Data)The client **pushes the data to all the replicas**. Each chunk server can store the data in the internal LRU buffer cache until it is used or aged out. By **decoupling the data flow from the control flow** ,  we can improve the performance by scheduling the expensive data flow based on the network topology.
4. Once all the replicas have acknowledged receiving the data, the client push a **write request** to the **primary** . The request identifies the data pushed earlier to all of the replicas . The **primary** assigns consecutive serial numbers to mutations it received . And applies the mutation to **primary’s local disk** in serial number.
5. The **primary** forwards the write request to all secondary replicas.  Each replica applies the mutation in the serial number assigned by the primary.
6. The secondaries all reply to the primary indicating that they have completed the mutations.
7. **The primary replies to the client**. Any errors encountered at any of the replicas are reported to the client. **In case of errors**, the write may have succeeded at the primary and an arbitrary subset of secondary replicas.  (If it failed in the primary ,it will not forward the write request ) After error, the write request is seen as failed , and the modified region is undefined , inconsistent. Retry mutation (3-7) , before retry the procedure.

If the write is large or straddles the chunk boundary, GFS client will break it into  multiple writes operations. There may be concurrent operation from other clients , because the GFS’s consistent model , the replica will not be bit identical but be identical containing fragments form different operations.

### Data Flow

We decouple the flow of data from the flow of control to use the network efficiently.

The control flows from the client to the primary and then to all secondaries , data is pushed linearly along a carefully picked chain of chunk server **in a pipelined fashion.** 

**Purpose:** fully utilize each machine’s network bandwidth ,avoid network bottleneck and high- latency links, and minimize the latency yo push through all the data.

**How to fully utilize the network bandwidth?**

The data is pushed **linearly** along a chain of chunk server rather than distributed in some other topology. Thus , each machine’s **full outbound bandwidth is used to transfer data** as fast as possible rather divided among multiple recipients.

**How to avoid the bottle neck and high latency links(inter-switch links are often both) as much as possible.**

Each machine forwards the data to the **closest** machine in the network topology that has not received it .

S1->S2->S3 or S4 The distance between the servers can be estimated from IP address.

**So we can minimize the latency by pipelined the data transfer over TCP**

Once the server receive **some** data (not all data), it starts forwarding **immediately.**  **Pipelining** is helpful to us because we use a switched network with full-duplex(全双工) links. Sending data immediately does not reduce the receive rate.

```
We do not think about congestion.
Transfer B bytes to R replicas.
Time cost = B/T +RL .
T is the network throughtput and L is Latency to transfer bytes between two machines.
```

### Atomic Record  Appends

**GFS provides an atomic append operation called record append.**

**Traditional Write**: the client specifies the offset  at which data to be written.  Concurrent write to the same region is not serialization , so the region will end up containing data fragments from multiple writes from multiple clients. 

**Record append:** **the client specifies the data.** GFS appends the data to the file(the end of the file) at least once atomically ,and return the **offset** to the client.

If we do a traditional write , we could need additional complicated and expensive synchronization (e.g a distributed lock manager). But in our workload ,  we only append the data to the end of the file, serve as multiple-producer/single-consumer queues or contain merged results from many different clients.

**Record append is a mutation** which follows the control flow with a little extra logic at the primary (**In leases and mutation Order**) .

**The extra logic** ：The client pushes the data to all replicas of the  last chunk of the file Then, it sends its request to the primary. **The primary checks to see if appending the record to the current chunk would cause the chunk to exceed the maximum size (64 MB).** If so, it **pads the chunk to the maximum size**, tells secondaries to do the same, and replies to the client indicating that the operation should be **retried on the next chunk.** (Record append is restricted to be **at most one-fourth of the maximum chunk size** to keep worst case fragmentation at an acceptable level.) If the record fits within the maximum size, which is the common case, the primary appends the data to its replica, tells the secondaries to write the data at the exact offset where it has, and finally replies success to the client.

**When the fail happens on any replica**

The solution is same as the before. If the failure happens on the replica , the client will receive the error and retries the  operation. As a result , the replicas of the same chunk may contain different data possibly including duplicates of the same record in while or in part . From the consistent model in GFS , the GFS do not guarantee that all replicas are bytewise identical , and only guarantees the record is appended at lease once atomically.

**Why we can achieve this?**

This property follows readily from the simple observation  that for the operation to report success, that is (i.e.) the record must be written at the same offset on all replicas of the chunk . So the later data can be assigned a higher offset  or a different chunk even if we have a different primary.

**Consistent Defined**

The successful record append ‘s region will be defined hence consistent. The **intervening region** (failure happens) are inconsistent (hence undefined) .  Our client application can handle the inconsistent region using library code.

### Snapshot

**Snapshot** operation makes a copy of a file or a directory tree(source) almost instantaneously ,while minimizing any interruptions of ongoing mutations. 

**Usage:** **quickly create branch copies** of huge data sets (and often copies of those copies,recursively) ,or to **checkpoint the current state** before experimenting  with changes that can later be committed or rolled back easily.

**How we implement snapshot?**

We use **copy-on-write**. When the master receives a snapshot request, 

* it firsts **revokes any outstanding  leases** on the chunks **in the files it is about to snapshot.**  This ensures that any subsequent writes to these chunks will require an interaction with the master to find the lease holder. This will give the master an opportunity to **create a new copy of the chunk first.** ( Who creates the copy of the huge data set , Master.) 
* **After** the leases have been revoked or have expired, the  **master logs the operation to the disk.** (After log it to the disk!) It then **applies this log record to its in-memory state by duplicating the metadata for the source  file or directory tree**. The newly created snapshot files **point to the same chunks as the source files**
*  The first time when a client wants to write to a chunk C after the snapshot operation, it **sends a request to the master to find the current lease holder**. The master notices **the reference cnt for chunk C greater than one.** It **defers** replying to the client request and instead **picks a new chunk handle C’**. It then asks each chunk server that **has a replica of chunk C to create a new chunk called C’.** 
* By creating the new chunk on the same chunk server as the original , we ensure the data can be copied **locally** , not over the network(slower). From now on , the write request is no different from that for any chunk: the master grants one of the replicas a lease on the new chunk C’ and replies to the client , while can write the chunk not knowing it just been created from an existing chunk.



## Master Operation

The master executes all namespace operations. In addition,it manages chunk replicas throughout the system : it makes placement decisions , creates new chunks and hence replicas , and coordinates various system-wide activities to keep chunks fully replicated , to balance load across all the chunk servers, and to reclaim unused storage .

### Namespace Management and Locking

Many master operations can take long time such as **snapshot**. So we allow multiple operations  to be active and use locks over  regions of the namespace to ensure proper serialization.

**The structure of namespace**

Unlike traditional fs,GFS does not have a per-directory data structure that lists all the files in that directory. Nor does it support aliases for the same file or directory( symbolic links or hard link). GFS represents its **namespace as a lookup table mapping full pathnames to metadata.**  With prefix compressing , the table can be efficiently represented in memory. Each node in the namespace tree (a file or a directory) **have a R/W Lock.**

As usual , each master operation acquire the lock before the operation.

`/d1/d2/d3../dn/leaf ` need acquire `/d1,/d1/d2,/d1/d2/d3` read-lock ,and depends the operation ,acquire the read or write lock on `/d1/d2/d3/../dn/leaf`.

```
Locking mechanism
create file /home/user/foo and snapshot /home/user to /save/user

In the creation, we need to acquire the read lock on /home and /home/user and write lock on /home/user/foo . 
In snapshot ,we need to acquire read lock on /home and /save , adn write lock ont /home/user and /save/user 
The two operations need to be serialized because they acquire conflicting lock on /home/user .
```

**File creation do not need acquire the write lock on the parent ,because they do not have the inode-like structure to be protected from modification. The read lock on the parent directory is sufficient to protect the parent directory from deletion or renamed or modification.**

**Advantage**

* The locking schema is that allows  concurrent mutations in the same directory. Because we only need the read(shared) lock on the parent directory and write lock on the file  .
* The read lock on the parent directory prevent the directory from being deleted , renamed,or snapshotted.
* The write lock serialize attempts to create a file with the same name twice.

Read/write lock are allocated lazily and deleted once they are not in use. 

Also the lock have **a total acquire order** to prevent deadlock : first by level in namespace and lexicographically with in the same level.

### Replica Placement

**Rack**

A GFS cluster is highly distributed at more levels than one. It typically has hundreds of chunk servers  spread across many **machine racks.** They can be accessed from hundreds of clients from the same or different  racks. So the communication between two machines **may  cross one or more network switches**.   Additionally , **bandwidth in or out of a rack may be less than the aggregate bandwidth of all the machines within the rack**. Multi-level distribution presents a unique challenge to distribute data for scalability ,reliability and availability.

**The purpose of the replica placement**

* maximize data reliability and availability 
* maximize the network bandwidth utilization 

**How to achieve the two purposes**

* spread chunk replicas across racks not only machines. 
  * Because when we only spread across machines , it only guards against **disk or machine failures** and fully **utilizes each machine’s network bandwidth** .  
  * we spread across machine racks .This ensures that some replicas of a chunk can **survive ** when  an entire rack is damaged or offline. It also means we can exploit the aggregate bandwidth of multiple racks. But the write traffic can walk through multiple racks  . This is a **trade-off.**

### Creation,Re-replication,Rebalancing

Chunk replicas are created for these three reasons.

1. Creation

**When the master creates a new chunk , it chooses where to place the initial empty replicas.**

* We want to place the  new replicas on chunk server with below-average disk space utilization. Over time this will equalize disk utilization across chunk server
* We want to limit the number of “recent” creations on each chunk server. Although  creation itself is cheap,it reliably **predicts imminent heavy write traffic** because chunks are created when demanded by writes ,a nd in out append-once-read-many workload they typically become practically read-only once they have been completely written.
* We want to spread replicas of a chunk across racks.

2. re-replication 

**The master re-replicate a chunk as soon as the number of available replicas falls below a user-specified goal.**

* A chunk server becomes unavailable
* one of replica is corrupted  because the disk is disabled  ,the replication goal increases.

**The priority of the chunk replication.**

* **how far it is from its replication goal** . The chunk have one replica have higher priority than a chunk have two replicas.
* The chunk have live files are preferred than chunk that belong to recently deleted files.
* The priority of chunk that blocks the client progress is boosted.

**How it generate a new replica**

The master picks the highest priority chunk and clones it by i 	nstructing some chunk server to copy the chunk data directly from an existing valid replica. The new replica placement is similar  in chunk creation.

**To keep cloning traffic from overwhelming client traffic** , the master limits the number of active clone operations  both for the cluster  and for each chunk server . Additionally , each chunk server limit the amount of bandwidth it spends on each clone operation by throttling its read requests to the source  chunk server.

3. Rebalance

The master runs it periodically : it examines the current replica distribution and moves replicas for better disk space and load balancing.

The fill up  process is gradual, the placement criteria .  remove criteria is to choose the chunk server that has the below  - average free space.

### Garbage Collection (GC)

When the file is deleted , GFS does not immediately reclaim the available physical storage. **It does so only lazily during regular garbage collection at both the file and chunk files.**

#### Mechanism

When a file is deleted by the application ,the master logs the operation  immediately just like other changes .

* The file just be renamed to a hidden name that includes the deletion timestamp.
  * During the regular scan of the master’s file system , it will be removed  after a configurable interval.
  * Before the time , it can be read under the new name ,and be undeleted by renaming it to the normal file .
* When the file is removed from the **name space** , **its in memory metadata is erased** 

In **chunk namespace** , the master identifies the **orphaned chunks** (which is not reachable from any file) and **erases the metadata for those chunks**.  In a **hear beat** message , each chunk server reports a subset of the chunk it has ,and the master replies the identity of **all chunks that are no longer in the master’s metadata**. The chunk server can remove its replicas of the chunk.

#### Discussion

**In our case , because the mapping from file-to-chunk mappings maintained exclusively by the master.** We can easily identify all reference to the chunks, they are Linux files under designated directories on each chunk server. Any such replica not known to master is **garbage**.

The GC approach to storage reclamation offers several advantages over eager deletion.

* It is simple and reliable in a large-scale distributed system where component failures are common. In the case of chunk creation failure , replica deletion message may be lost . **GC provides a uniform and dependable way to clean up any replicas not known to useful.**
* **It merges storage reclamation into the regular scans of namespaces and handshake with chunk servers. Thus it is done in batches and the cost is amortized.** Moreover , it is done when the master is relatively free . The master can respond more promptly  to client requests that demand timely attention
* The **delay** in reclaiming storage provides a safety net against **accidental ,irreversible deletion.**

Disadvantage

* **The delay sometimes hinders user effort to fine tune usage when storage tight.**  It means when applications that repeatedly create and delete temporary files may not be able to reuse the storage right away . We solve this issue by expediting storage reclamation. We can also allow users to **apply different replication and reclamation policies to different parts of the namespace.**

### Stale Replica Detection

**Chunk replicas may become stale if a chunk server fails and misses mutations to the chunk while it is down.** For each chunk , we maintain **chunk version number** to distinguish the new replica from the stale replica.

**Whenever the master grants a new lease on a chunk, it increases the chunk version number and informs the up-to-date replicas.** The master and these replicas all record the new version number in their **persistent** state. **This occurs before any client is notified and therefore before it can start writing to the chunk.** 

* If another replica is currently unavailable, its chunk version number will not be advanced. The master will detect that this chunkserver has a stale replica when the chunkserver **restarts and reports its set of chunks and their associated version numbers.** 
* If the master sees a version number **greater than **the one in its records, the master assumes that it failed when granting the lease and so takes the higher version to be up-to-date.

The master removes the stale replica when GC , and all these stale replicas not to exist at all when it replies to the client request for chunk location.

As another safeguard, **the master includes the chunk version number when it informs clients which**
**chunk server holds a lease on a chunk or when it instructs a chunk server to read the chunk from another chunk server in a cloning operation**. The client or the chunk server verifies the version number when it performs the operation so that it is always accessing up-to-date data.

## Fault Tolerance And Diagnosis

Big challenge : frequent component failure

### High Availability

**How to keep the overall system highly available ?**

* fast recovery
* replication

#### Fast recovery

Both the master and chunk server is designed to  restore their state and start in seconds not matter how they terminated.

#### Chunk replication

Each chunk is replicated on multiple chunk server on different racks.

#### Master Replication

The master state is replicated for reliability.  **Its operation log and checkpoint  are replicated on multiple machines. and non-violate**.

**A mutation to the state is considered committed only after its log record has been flushed to disk locally and on all master replicas.**

For simplicity , one master process remains in charge of all things, :mutations, GC .When it fails , it can restart almost instantly. **If its machine or disk fails , monitoring infrastructure outside the GFS starts a new master process elsewhere with the replicated operation log.**

Moreover, “shadow” masters provide read-only access to the file system even when the primary master is down. Because it is shadow , not the mirror,in that they may lag the primary slightly. (Similar in VM ware FT)

They enhance read availability for file that are not being actively mutated or applications that do not mind read stale info. In fact , the file content is read from the chunk server , the apps do not observe stale file content. So in short window , we can see stale file metadata .

**To keep itself informed, a shadow master reads a replica of  the growing operation log and applies the same sequence of changes to its data structures exactly as the primary does.** Like the primary, it **polls** chunk servers at startup (and infrequently thereafter) to locate chunk replicas and exchanges frequent handshake messages with them to monitor their status. **It depends on the primary master only for replica location updates resulting from the primary’s decisions to create and delete replicas.**

### Data Integrity

Each chunk server uses **check sum** to detect corruption of stored data. i**t would be impractical to detect corruption by comparing** **replicas across chunk servers.** Moreover, divergent replicas  may be legal: the semantics of GFS mutations, in particular  atomic record append as discussed earlier, **does not guarantee**
**identical replicas**. Therefore, each chunk server **must independently verify the integrity of its own copy by  maintaining checksums.**

**A chunk is broken up into 64KB blocks. Each has a corresponding  32 bit checksum. Like other metadata, checksums are kept in memory and stored persistently with logging, separate from user data.**

* For reads, the chunk server **verifies the checksum of data blocks that overlap the read range before returning any data to the requester, whether a client or another chunk server. Therefore chunk servers will not propagate corruptions to other machines.** If a block does not match the recorded checksum, the chunk server returns an error to the requestor and reports the mismatch to the master. In response, the requestor will read from other replicas, while the master **will clone the chunk from another replica.** After a valid new replica is in place, the master instructs the chunk server that reported the mismatch to delete its replica.

Checksum have little effect on performance  which needs little IO because they are all in memory.

* Checksum computation is heavily optimized for writes that append to the end of a chunk (as opposed to writes that overwrite existing data) because they are dominant in our workloads. **We just incrementally update the checksum for the last partial checksum block, and compute new checksums for any brand new checksum blocks filled by the append.** Even if the last partial checksum block is already corrupted and we fail to detect it now, the new checksum value will not match the stored data, and the **corruption will be detected as usual when the block is next read**
* In contrast, if a write overwrites an existing range of the chunk, **we must read and verify the first and last blocks of the range being overwritten, then perform the write, and finally compute and record the new checksums.** If we do not verify the first and last blocks before overwriting them partially, the new checksums may hide corruption that exists in the regions not being overwritten.

**During idle periods, chunk servers can scan and verify the contents of inactive chunks.** This allows us to detect corruption in chunks that are rarely read. Once the corruption is detected, the master can create a new  uncorrupted replica and delete the corrupted replica. **This prevents an inactive but corrupted chunk replica from fooling the master into thinking that it has enough valid replicas of a chunk.**

### Diagnostic Tools

Extensive and detailed diagnostic logging has helped immeasurably  in problem isolation, debugging, and performance analysis, while incurring only a minimal cost. Without logs, it is hard to understand transient, non-repeatable interactions between machines. GFS servers generate diagnostic logs that record many significant events (such as chunk servers going up and down) and all RPC requests and replies. These diagnostic logs can be freely deleted without affecting the correctness of the system. However, we try to keep these logs around as far as space permits.

The RPC logs include the exact requests and responses sent on the wire, except for the file data being read or written. By matching requests with replies and collating RPC records on different machines, we can reconstruct the entire interaction history to diagnose a problem. The logs also serve as traces for load testing and performance analysis.

The performance impact of logging is minimal (and far outweighed by the benefits) because these logs are written sequentially and asynchronously. The most recent events are also kept in memory and available for continuous online monitoring.