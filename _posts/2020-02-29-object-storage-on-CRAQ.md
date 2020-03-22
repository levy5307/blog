## Strong Consistency
-------------------
in our system provides the guarantee that all read and write operations to an object are executed in some sequential order, and that a read to an object always sees the latest written value.

## Eventual Consistency
-------------------
in our system implies that writes to an object are still applied in a sequential order on all nodes, but eventually-consistent reads to different nodes can return stale data for some period of inconsistency (i.e., before writes are applied on all nodes). Once all replicas receive the write, however, read operations will never return an older version than this latest committed write. In fact, a client will also see monotonic read consistency if it maintains a session with a particular node (although not across sessions with different nodes)

## Chain Replication
-------------------
写请求都发送到head结点，然后该写请求逐渐从头结点向后迁移，直到tail结点提交了写入时，则代表此次写入提交了。读请求都从tail结点读取。所以在tail结点可以保持所有操作的全局有序，并且读操作会获取到最新的写入数据。因此是强一致的。但是这样的代价是，该链表的scale-out能力被限制了。
不能从中间结点读取，这样会导致读取不同的结点获取不同的值，并且有可能查到一些陈旧的数据，所以只能满足了最终一致性，无法满足强一致性。

### CR比传统的主从方式也有优点：
即读和写操作分配到不同的结点上，并且写请求的压力在各个结点均匀分布(primary-secondary的方式先写入primary，然后由primary扩散至所有sencondary，这样primary是一个中心，而且读也要发送至primary, 压力会比较大。)。所以会拥有更好的吞吐。

## CRAQ: Chain Replication with Apportioned Queries
-------------------
- A node in CRAQ can store multiple versions of an object, each including a monotonically-increasing version number and an additional attribute whether the version is clean or dirty. All versions are initially marked as clean.

- When a node receives a new version of an object (via a write being propagated down the chain), the node appends this latest version to its list for the object.

	- If the node isn`t the tail,it marks the version as dirty, and propagates the write to its successor.

	- Otherwise, if the node is the tail, it marks the version as clean, at which time we call the object version (write) as committed. The tail node can then notify all other nodes of the commit by sending an acknowledgement backwards through the chain.

- When anacknowledgment message for an object version arrives at a node, the node marks the object version as clean. The node can then delete all prior versions of the object.

- When a node receives a read request for an object:

	- If the latest known version of the requested object is clean, the node returns this value.

	- Otherwise, if the latest version number of the object requested is dirty, the node contacts the tail and asks for the tail’s last committed version number (a version query). The node then returns that version of the object; by construction, the node is guaranteed to be storing this version of the object. We note that although the tail could commit a new version between when it replied to the version request and when the intermediate node sends a reply to the client, this does not violate our definition of strong consistency, as read operations are serialized with respect to the tail.

Note that an object’s “dirty” or “clean” state at a node can also be determined implicitly, provided a node deletes old versions as soon as it receives a write commitment acknowledgment. Namely, if the node has exactly one version for an object, the object is implicitly in the clean state; otherwise, the object is dirty and the properly ordered version must be retrieved from the chain tail

## 备注
-------------------
这里backup request可以借鉴这种实现方式来实现强一致性。即：在所有结点上对key维护不同的version，当向secondary读时，如果当前key是clean的，则直接返回。否则向primary查询当前key的最新version，然后返回给客户端。 

### CRAQ’s throughput improvements over CR arise in two different scenarios:
- Read-Mostly Workloads have most of the read requests handled solely by the C−1 non-tail nodes (as clean reads), and thus throughput in these scenarios scales linearly with chain size C.
	
- Write-Heavy Workloads have most read requests to non-tail nodes as dirty, thus require version queries to the tail. We suggest, however, that these version queries are lighter-weight than full reads, allowing the tail to process them at a much higher rate before it becomes saturated. This leads to a total read throughput that is still higher than CR.

