
---
layout: post
Author: Levy5307
tags: [论文]
comments: true
---

Three chanllenges for replicating executing of any VM running any operating system and workload 

- correnctly capturing all the input and non-determinism necessary to ensure deterministic execution of a backup virtual machine 
- correctly applying the inputs and non-determinism to the backup virtual machine
- doing so in a manner that doesn`t degrade performance

## FT(Fault-Tolerance) Protocol 
-------------------
### Output Requirement: 
If the backup VM ever takes over after a failure of the primary, the backup VM will continue executing in a way that is entirely consistent with all outputs that the primary VM has sent to the external world.

The Output Requirement can be ensured by delaying any external output(typically a network packet) until the backup VM has received all information that will allow it to replay execution at least to the point of that output operation.

只有backup接收到该操作必须的所有数据时，primary才能向客户端发送该操作的output(将该操作的output给delay了)

Given the above constraints, the easiest way to enforce the Output Requirement is to create a special log entry at each output operation. Then the Output Requirement may be enforced by this specific rule:

### Output Rule: 
The primary VM may not send an output to the external world, until the backup VM has received and acknowledged the log entry associated with the operation producing the output.

注解：如果backup没有接收到所有关于该操作相关的所有日志entry，而发送了该操作的output给clients。那么如果primary挂掉了，backup则无法恢复到primary挂掉之前的样子。就产生了不一致性

## Detecting and Responding to Failure 
-------------------
### Respond to failure 

#### If the backup VM fails: 
the primary VM will go live -- that is , leave recording mode(and hence stop sending entries on the logging channel) and start executing normally.

#### If the primary VM fails: 
the backup VM should similarly go live, but the process is a bit more complex. Because of its lag in execution, the backup VM will likely have a number of log entries that it has received and acknowledged, but have not yet been consumed(because the backup VM hasn`t reached the appropriate point in its execution yet). The backup VM must continue replaying its execution from the log entries until it has consumed the last log entry. At that point , the backup VM will stop replaying mode and start executing as a normal VM(the backup VM has been promoted to the primary VM)

注解：primary挂掉的情况比较复杂，因为backup有一些日志，只是复制过来了，但是并没有去真正执行。所以需要backup执行完所有的log entries，然后再改变状态变成一个真正的primary

### Ways to detect failure 
- VMware FT uses UDP heartbeating betweent servers. In addition, 
- VMware FT monitors the logging traffic between primary and backup.

A failure is declared if heartbearting or logging traffic has stopped for longer than a specific timeout. 
注解：只要有heartbeat停止了，或者监控的数据传输停止了，就认为是发生了故障

### Split-brain problem 
When either a primary or backup VM wats to go live(as primary), it executes an atomic test-and-set operation on the shared storage. If the operation succeeds, the VM is allowed to go live; IF the operation fails, then the other VM must have already gone live, so the current vM actually halts itself(commits suicide).

If the VM cannot access the shared storage when trying to do the atomic operation, then it just waits until it can.

注解：使用共享存储的方式来解决脑裂导致的多主问题，获取到原子锁的则继续当做主。这样不会带来可用性的问题，因为只有共享存储无法访问了，primary和backup才会都无法获取锁，此时的VM ware其实也无论如何都不能对外提供服务了。所以对可用性没影响

### final aspect 
Once a failure has occurred and one of the VMs has gone live, VMware FT automatically restores redundancy by starting a new backup VM on another host.

## Practical implementation of FT 
-------------------
### Starting and Restarting FT VMs 
Using modified form of VMotion functionality of VMware vSphere 

### Managing the Logging Channel 
The hypervisors maintain a large buffer for logging entries for the primary and backup VMs.

primary产生一些log entries放入到log buffer中, 同样，backup从它的log buffer中获取log entries. primary的log buffer中的log entries将会尽快的放入到logging channel, 然后backup将该log entries读取出来放入到其自己的log buffer中。每当从logging channel读取出一些log entries，backup发送acknowledgement给primary（该acknowledgement告知primary可以将相应的output发送给client了）

If backup VM encounters an empty log buffer, it will stop execution. this pause will not affect any clients of the VM, because backup VM is not communicating externally.

If primary VM encounters a full log buffer, it will stop execution similary. However, this pause can affect clients of the VM.
Therefore, our implementation must be designed to minimize the possibility that the primary log buffer fills up

注解：backup的log buffer为空导致的stop不会影响客户端，但是如果primary的log buffer满了，导致其stop了，则会影响到了客户端，应该尽量避免。

如果backup重放一个execution的速度比primary记录一个execution慢太多的话：

1.会导致上述问题，即产生了full log buffer，然后stop而影响到了客户端
2.会导致bakcup与primary之间的lag太大，如果primary挂了，backup必须执行完这些lag的execution, 导致backup接替成为primary慢了很多

所以VM FT有一个额外的机制来降低primary VM的速度：

在primary和backup的sending和acknowledging之间，我们添加了一些额外的信息来表明backup和primary之间的lag。如果lag太大，VMware FT则会降低primary VM的速度(通过分配较少的cpu)。该过程是通过很多个ping-pong来实现的，即：逐渐调节的。如果lag变大，则降低primary VM的速度; 如果lag变小，则提高primary VM的速度。直到达到平衡。

### Implementation Issues for Disk IOs 
有许多关于磁盘IO的细微问题：

首先，由于磁盘操作是非阻塞的，因此可以并行执行，因此访问统一磁盘位置的同时磁盘操作可能会导致不确定性。同样，由于我们在磁盘操作中，使用DMA来完成内存与磁盘之间的数据传输，因此，访问同一块内存区域也会导致不确定性。VM FT的解决方案是检测此类的IO争用，使其串行化（在primary和backup上以同样的顺序执行）

其次，磁盘操作也会与内存操作产生竞争，因为磁盘通过DMA访问VM的内存。为了解决这个问题，可以使用为相应的内存区域添加page protection(类似于锁)，当磁盘操作时，则setup page protection, 如果有对改块内存的读操作，则需要等待该磁盘操作完成。但是其代价太高了。我们使用bounce buffers来解决。也就是磁盘读写均通过bounce buffer，并且IO操作完成时，将数据复制给目标内存。(这会减慢磁盘操作，但是我们实际上并没有见到太多的性能损失)

第三，当发生failover时，新选举的primary无法知道此前的primary执行的一些io操作是否开始执行、以及是否执行成功，我们的做法是认为执行失败，然后重新开始执行该io操作，因为io操作是幂等的，重复操作并不会产生影响。

### Implementation Issues for Network IOs 
1.reduce VM traps and interrupts
2.reduce the delay for transmitted packets(等待backup接收到请求相关的所有entries后，才能向客户端回应该请求)。Our primary optimizations in this area involve ensuring that sending and receiving log entries and acknowledgements can all be done without any thread context switch.(在接收和发送log entries的过程中，避免发生线程context切换)

## Design Alternatives 
-------------------
Shared vs. Non-shared Disk 

### Shared Disk 
如果主和从使用共享存储空间，只有primary才实际向磁盘中写入，并且写入磁盘必须遵循Output Rule来延迟写入。

### Non-shared Disk
当无法获取shared存储空间时，使用非共享存储空间是一个非常有用的办法。当不使用共享存储空间时，backup VM同样需要磁盘写入，他需要和primary VM的磁盘写入保持同步。因此向priary磁盘的写入无需延迟写入。

但是使用Non-shared disk有一个缺点，就是无法通过像shared disk那样通过原子操作来避免脑裂的问题。这是需要引入一些外部的组件来解决，比如通过大多数投票等方式来决定那个是primary。

### Executing Disk Reads on the Backup VM
在之前的设计中，不会从backup去读取。因为读操作被当成一个input，由primary执行，并且通过logging channel传递给backup

可以通过backup来执行读操作，以便于减少logging channel的压力。但是有几个问题：

1.将会减缓backup的运行。因为要读取一些最新的内容时，backup必须等待所有的log entries实际的执行完（这些log entries或许已经在primary执行完了，但是在backup仅仅只是接收，并没有真正的执行）
2.必须有一些额外的工作来处理磁盘的失败读取。例如：如果backup读取失败、而primary读取成功，则backup需要重新读取、直到读取成功。因为backup读取的内容需要和primary读取的内容完全一致
3.在使用shared存储时，如果primary读之后紧跟着写，那么必须有一些额外的操作来延缓写操作，直到backup读取完之后再执行。

注：通过我们的性能测试，在backup执行读操作会减少1-4%的系统吞吐。但是也显著的减少了logging channel的带宽消耗

    所以在logging channel的带宽很有限的时候，使用bakcup读取时一个不错的选择。

## MIT课程笔记 
-------------------
在本文中讲述了，replication分为两个级别，即：application level和machine level

application level仅仅是保存应用看中的数据，例如GFS中的chunk，但是machine level要关注的数据就多的多，例如：内存，寄存器内容等等，

显而易见的，application level效率高的多，但是machine level可以互备的东西更完整。在本文中关注的是machine level

另外，本文只关注vm是单核的情况(非底层硬件单核, 底层硬件可以是multi-core，但是vm呈现给其guest os一定要是uni-core)，因为多核情况下，指令的执行顺序等变化太大(例如，对某个资源的锁获取具有一定随机性，如果primary和backup由不同的进程获取到，那执行结果将会大不相同)，目前vm团队还没有很好的适配。

```
FT's handling of timer interrupts
  Goal: primary and backup should see interrupt at 
        the same point in the instruction stream
  Primary:
    FT fields the timer interrupt
    FT reads instruction number from CPU
    FT sends "timer interrupt at instruction X" on logging channel
    FT delivers interrupt to primary, and resumes it
    (this relies on CPU support to interrupt after the X'th instruction)
  Backup:
    ignores its own timer hardware
    FT sees log entry *before* backup gets to instruction X
    FT tells CPU to interrupt (to FT) at instruction X
    FT mimics a timer interrupt to backup

FT's handling of network packet arrival (input) 
  Primary:
    FT tells NIC to copy packet data into FT's private "bounce buffer"
    At some point NIC does DMA, then interrupts
    FT gets the interrupt
    FT pauses the primary
    FT copies the bounce buffer into the primary's memory
    FT simulates a NIC interrupt in primary
    FT sends the packet data and the instruction # to the backup
  Backup:
    FT gets data and instruction # from log stream
    FT tells CPU to interrupt (to FT) at instruction X
    FT copies the data to backup memory, simulates NIC interrupt in backup
FT means the vm monitor(primary + backup)
```

```
+--------------+
|     app      |
|--------------|
|     os       |
| --------------
|  vm monitor  |
+--------------+
```
为什么要先copy到FT，而不是直接copy到primary呢？因为如果直接copy到primary，则无法知道copy的具体时间、以及是否copy了，所以会导致primary和backpu不一致

Output Rule会导致性能问题，因为必须要等待primary发送给backup，并且backup发送回复之后才能向客户端发送output，由于防止地震等灾害，primary和backup一般在不同的城市，所以通信的时间会比较长。这也是分布式系统的通病，所以很多实现都是对于high-level的操作才会遵守Output Rule，low-level的操作例如read-only操作，则直接由primary操作完成发送给客户端output，而不发给secondary

Q: 如果primary发送output之后挂了，此时相应的请求还在backup的buffer里，并没有执行。然后backup成为primary，又发送了一个多余的output，会不会带来问题？

A: 很幸运有TCP/IP协议，由于backup成为primary主之后，与原先的primary发送的output的TCP sequence相同，TCP协议会将其直接抛弃

   我们需要做的是，应该避免没有发送output，对于发送的多余的output，可以通过TCP或者应用层的查重来解决。
