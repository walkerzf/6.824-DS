# The Design of a Practical System for Fault-Tolerant Virtual Machines

## Topic: Fault Tolerance

A commercial enterprise-grade system for providing fault-tolerant virtual machines is implemented , based on the approach of **replicating the execution of a primary virtual machine via a backup vm on another physical sever**.

Advantages:

* reduce performance for the apps runs on the  sever 10%; 

* the bandwidth keep the P and B lockstep is less  than 20Mbit/s,which is meaningful for longer distance FT

Is replication worth the Nx expense? => depends on the robust or availability you want to get

## Introduction

**Common approach** : Primary(P) / Backup(B). In the fail-stop failure case, B can always take over. The state of B must be keep identical to the P at all times, so that the backup can take over immediately when the failure happens.

**Two approaches:**

1. **State Transfer:** ship all changes to the B including CPU,memory,and I/o devices . which needs big width. Transfer is simple but slow on the Internet.
2. **Replicated State Machine**:  The idea is to model the severs as a deterministic state machines that are kept in sync by starting them from  **same initial state** and ensuring that they **receive the same input requests in the same order.** Since the sever can have non-deterministic operations , **extra coordination** must be used .But the amount of extra info is far less than the amount of all states about a sever.

 **Two Level Replicas** 

1. **Application state**: GFS works this way. Can be efficient; primary only sends high-level operations to backup . Application code (server) must understand fault tolerance, to e.g. forward op stream
2. **Machine level**: registers and RAM content.  might allow us to replicate any existing server w/o modification! requires forwarding of machine events (interrupts, DMA, &c) ,requires "machine" modifications to send/recv event stream...

**Prerequisites**

* A virtual machine running on top of a **hypervisor**(虚拟机器监视器) is an excellent platform for implementing the state-machine approach. A vm can be considered as  a **well-defined state-machine** which is controlled by hypervisor fully , **including delivery of all inputs ,all necessary info about non-deterministic operations on P.**

* Record the execution of a P and ensure that the B execute identically  which is known as **deterministic replay**. 
* **uni-processor**. for multi-processor vm , every access to shared memory can be seen as a non-deterministic opeartion.
* **fail-stop failures**: server failures  that can be detected before the failing server causes an incorrect externally visible action. h/w bugs or s/w bugs or human config errors are not fail-stop failures.

## Basic FT Design

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210116161327103.png" alt="image-20210116161327103" style="zoom:50%;" />

* hypervisor == monitor == VMM (virtual machine monitor)

* O/S+app is the "guest" running inside a virtual machine

* two machines, primary and backup,run on different physical server.**Virtual Lockstep**

All inputs(which are usually network packet==data + interrupt  and wired instructions) are sent to B via a network connection (aka TCP) known as the logging channel. Ordinarily, backup's output is suppressed by FT.

### Deterministic Replay Implementation

For a vm : 

inputs :incoming internet packet , disk read ,and input from the keyboard and mouse.

Non-deterministic events: virtual interrupt 

Non-deterministic operations : read the clock cycle of the day.

**Challenge  for replicating the execution of P and its workload ** 

1. correctly capturing all input  and non-determinism necessary  to ensure deterministic execution of B
2. correctly applying the capturing things to the B
3. doing so can not degrade performance

**Deterministic Replay**: records the inputs of P and all possible non-determinism in a log entry(instruction #, type , data).

For Non-deterministic operations : sufficient info  allow the operation to reproduce the **same state change and output**

For Non-deterministic events: timer or IO completion interrupt. **The instruction number** will be recorded. 

The event will be delivered at the same point in the instruction stream .The VMware implements an efficient event recording and event recording mechanism that employs various techniques.

* When does the primary have to send information to the backup?
    Any time something happens that might cause their executions to diverge.
    Anything that's not a deterministic consequence of executing instructions.

### FT Protocol

**Output Requirement**: if the backup VM ever takes over after a failure of the primary, the backup VM will continue executing in a way that is **entirely consistent with all outputs that the primary VM has sent to the external world.**

If the P crash when it completes the operation , but not write the log entry ,so the value in B will remain old and takes over.

**Output Rule**: the primary VM **may not send an output to the external world, until the backup VM has received and acknowledged** the log entry associated with the operation producing the output.

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210116164154669.png" alt="image-20210116164154669" style="zoom:67%;" />

In this case ,we can not guarantee the all output occurs exact once.



### Detecting and Responding to Failure

#### **Failures in different cases:**

When **B fails** , the P go live and leaves the recording mode and runs normally.

When **P fails**,the B go live. Because the lag in its execution compared to P. B needs to continue **replaying its execution from the log entries until it has consumed the last log entry**. Then , the B will stop replaying mode and **become a new P.**(now missing a B ,which needs some device-specific operations). **For the purpose of Internet,** Vmware FT advertise the **MAC address** of the new P on the Internet. So the physical network switches will know on  what server the new P located. In addition, the new P need to reissue some disk I/O.

#### Detect failure

1. UDP heart beating between servers that running FT to detect when a server may have crashed
2. VMware FT monitors the logging traffic from P to B  and ack from B to P . Because a regular timer interrupt , the log traffic should never stop.

**Split - brain problem**

If the network crashed  but the P and B still runs, there will be corruption and problems for the clients communicating with  the VM. So we must ensure one of the P and B goes live.

When either  P or B wants to go live , it executes an **atomic test-and-set operation** on the shared storage. if the operation succeed , the VM goes live. If the operation failed  which means  the other VM goes live, the current VM will **halts itself (commits  suicide.)** If the VM cannot access the shared storage  when trying to do atomic operation ,it just waits . 

#### A new backup

If failure happens and only one VM goes live, the FT will start a new backup VM on another host.



## Practical Implementation of FT

### Starting and Restarting FT VMs

1. Starting (Restarting) a backup VM which is the same state of Primary VM which in  an arbitrary sate

   We hope that the mechanism **does not significantly disrupt the execution of P** . A modified form of **VMotion** **clones a VM to a remote host rather than migrating it,** which also sets up the log channel and cause the P enter the log mode as the primary , the source VM to enter replay mode as the new backup.  The FT VMotion typically interrupts the execution of P less than a second. Hence enabling FT on a running VM is an easy ,non-disruptive operation.

2. How to choose a server running the new backup VM

   Fault-tolerant VMs run in a cluster of servers that have access to the shared storage, so all VMs can typically run on any server in the cluster.

   ```
   This flexibility allows VMware vSphere to restore FT redundancy even when one or more servers have failed.
   VMware vSphere implements a clustering service that maintains management and resource information. When a failure happens and a primary VM now needs a new backup VM to re-establish redundancy, the primary VM informs the clustering service that it needs a new backup. The clustering service determines the best server on which to run the backup VM based on resource usage and other constraints and invokes an FT VMotion to create the new backup VM. The result is that VMware FT
   typically can re-establish VM redundancy within minutes of a server failure, all without any noticeable interruption in the execution of a fault-tolerant VM.
   ```

   

### Managing the Logging Channel

In this implementation , the hypervisors maintain a large buffer for logging entries for the P and B. 

<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210117093917911.png" alt="image-20210117093917911" style="zoom: 50%;" />

There will two cases we may meet.

The first case: **If the B encounters an empty log buffer** when it need to read the next log entry, it will stop execution until a new log entry available. This will not affect all client of VM.

The another case: **If the P encounter a full log buffer** which means it can not write a log entry into the buffer,it must stop execution until log entries can be flushed out through the logging channel. This stop in execution is a natural flow-control mechanism that slows P which will affect the client of VM , because in this case , the P will stop completely and unresponsive until it can log its entry and continue execution.



**Reason for P fills up the log buffer:**

* B VM is executing too slowly and therefore consuming log entries too slowly. In general , the overhead of recording and replaying in VMware deterministic replay is roughly the same.
* The server host of B is heavily loaded with other VMs

**Reason for avoiding the large lag between P and B**

* the unexpected stop in P
* If  P fails,the B must catch up by replaying all the log entries  and start run normally. The consuming time = failure detection time + current execution lag time.

**How to slow down the P to prevent  B from getting too far behind?**

* In our protocol for sending and acks log entries , we add additional info to determine the real-time execution lag between the P and B VMs. If we have a large lag time ,the VMware FT starts slowing down P y informing the scheduler to give a slightly smaller amount of the CPU.  We use a feedback loop.

### Operations on FT  VMs

For the special operations : power off or increase the shared CPU should be sent on the logging channel from the P to B.

VMotion is the operation can be done independently on B or P.

For a normal VMotion , we require that all outstanding disk IOs be quiesced just as the final switchover on VMotion occurs.

VMotion for P or B

**For P:** 

* this quiescing is easily handled by waiting until the physical IOs completed and delivering these completions to the VM.

For B :

*  there is not  an easy way to cause all IOs to be  completed at any required point, since **the backup VM must  replay the primary VM’s execution and complete IOs at the**
  **same execution point**.  The **primary VM may be running a workload in which there are always disk IOs in flight during  normal execution.** VMware FT has a unique method to solve  this problem. When a backup VM is at the final switchover  point for a VMotion, **it requests via the logging channel  that the primary VM temporarily quiesce all of its IOs.** The  backup VM’s IOs will then naturally be quiesced as well  at a single execution point as it replays the primary VM’s  execution of the quiescing operation.

### Implementation Issues for Disk IOs

There will be subtle implementations in there cases .

1. disk operation are non-blocking  , so they can execute in parallel. 

Simultaneous disk operations that access to the same disk location can lead to non-determinism. Our implementation of disk IO uses DMS directly to /from , which means the operations that access the same memory page can lead to non-determinism.

**Solution**: **Detect** this race condition , and **force** such operations to run sequentially. 

2. disk operation ( DMA) can race with a memory access by the app in OS 

I.E. There will be non-determinism when an app is reading a memory  block  at the same time the disk read is occurring to that block , This case is unlikely,but we need to detect it.

**Solution** :

* **Set up page protection temporarily** on the pages that are targets of disk operations ,which is an expensive operation.  So when the app want to access the same memory page which is the target of an outstanding disk operation will result in trap. The VM will be paused until the disk operation completes.
* **Bounce Buffer:** A bounce buffer is a temporary buffer that has **the same size** as the memory being accessed by a disk operation. A disk read operation is modified to read the specific data to the bounce buffer,**and the data is copied to guest memory only as the IO completion is delivered** . Similarly for disk write , write to the **bounce buffer** ,and read from this region

3. There will be issues when P fails with outstanding disk IOs ,and the B takes over.

The new P will never be sure the IOs completed successfully or not. In addition,because the disk IOs were not issued externally on  B , there will be no explicit IO completion for them as the newly-promoted P  continue to run, which would cause the VM to start an abort or reset procedure. 

**Solution**: Send an error completion that indicates that each IO failed. Re-issue the pending IOs during the go-live process of B .(They are idempotent.)

```
In computing, an idempotent operation is one that has no additional effect if it is called more than once with the same input parameters. For example, removing an item from a set can be considered an idempotent operation on the set.
```

### Implementation Issues for Network IO

FT disables the asynchronous network optimization which can bring non-determinism. Similarly ,code that pulls packet out of transmit queues asynchronously is disabled .These will result in traps now.

Two approaches to improve VM network performance.

1. Clustering optimizations to reduce VM traps and interrupts.

**Batch the packet transmit**. When the VM is streaming data at a sufficient bit rate, the hypervisor can do one transmit trap per group of packets, and in best case , zero traps,since it can transmit the packet as part of receiving packets. Similarly,can reduce interrupt for incoming packets by  only posting interrupt for a group of packets.

2. Reduce the delay for transmitted packets.

As noted earlier , the hypervisor must delay all transmitted packets until it get an ack from the B for the  appropriate log entries. **The key to reduce the delay is to reduce the time required to send a log message to the B and get an ack.** 

**Ensure that sending and receiving log entries and acks can be done without any thread context switch.** 

```
The VMware  hypervisor  allows functions to be registered with the TCP stack that will be called from a deferred-execution context (similar to a tasklet in Linux) whenever TCP data is received. This allows us to quickly handle any incoming log message on B and any acks reveivd by the B without any thread context switches.In addition,when P enqueues a packet to be transmitted , we force an immediate log flush of the associated output log entry by scheduling a deferred-execution context to do the flush.
```

## Design Alternatives

### Shared vs. Non-shared Disk

In the default design ,the P and B share the same virtual disk, which is **considered external** to the P and B ,so any write to the disk will be seen as a communication to the **external world** and any read will be seen as incoming packet. Only P can do actual write to the disk and write to the shared disk must be delayed according to the **output rule.**

Alternative Design: separate virtual disk (not shared).



<img src="C:\Users\15524\AppData\Roaming\Typora\typora-user-images\image-20210117163442618.png" alt="image-20210117163442618" style="zoom: 33%;" />(no disk read for backup VM)



In this design,the B does all disk writes to its virtual disks ,and in doing so, it naturally keeps the contents of its virtual  disks in sync with the contents of P’s virtual disks.

In this case , the virtual disk can be seen as the internal state of  each VM. So according to the Output Rule, the disk write of P do not need to be delayed.

```
The non-shared design is quite useful in cases where shared storage is not accessible to the primary and backup VMs. This may be the case because shared storage is unavailable ortoo expensive, or because the servers running
the primary and backup VMs are far apart (“long-distance FT”). 
```

**Disadvantage:** two copies of virtual disks muse be explicitly synced up in some manner when fault tolerance is first enabled. In addition, the disks can get out of sync after a failure,they must be resynced when B restart.(It is VMotion’s work ,clone the state and disk state).

When meeting a split-bran problem, we have the **atomic test-and-set operation**. In this case ,the system can **use some other external tiebreaker.**

### Executing Disk Reads on the B

In our two design , the B **never read** from its virtual disk.

If we have the B executes disk read operation and therefore eliminate the logging of disk read data. This approach can reduce the traffic on the logging channel for workloads that do a lot of disk reads.

Some problems:

* It may **slow down the backup VM’s execution**, since the backup VM must execute all disk reads and **wait** if they are not physically completed when it reaches the point in the VM execution where they completed on the primary.
* Also, some extra work must be done to deal with **failed disk read operations**. If a disk read by the primary succeeds but the corresponding disk read by the **backup fails,** then the disk read by the backup **must be retried** until it succeeds, since the backup must get the same data in memory that the primary has. **Conversely, if a disk read by the primary fails**, **then the contents of the target memory must be sent to the backup via the logging channel, since the contents of memory will be undetermined and not necessarily replicated by a successful disk read by the backup VM.**(slight different from the case when the B failed) 
* Finally, there is a subtlety if this disk-read alternative is used with the **shared disk** configuration. If the primary VM does a read to a particular disk location, followed fairly soon by a write to the same disk location, **then the disk write must be delayed until the backup VM has executed the first disk read.** This dependence can be detected and handled correctly, but adds extra complexity to the implementation.
  