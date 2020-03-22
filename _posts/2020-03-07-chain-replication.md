# Chain Replication Protocol

## Reply Generation

The reply for every request is generated and sent by the tail.

## Query Processing 

Each query request is directed to the tail of the chain and processed there atomically using the replica of objID stored at the tail.

## Update Processing 

Each update request is directed to the head of the chain. The request is processed there atomically using replica of objID at the head, then state changes are forwarded along a reliable FIFO link to the next element of the chain (where it is handled and forwarded), and so on until the request is handled by the tail.

## Coping with Server Failures

For this purpose, we employ a service, called the master, that:

- detects failures of servers,

- informs each server in the chain of its new predecessor or new successor in the new chain obtained by deleting the failed server,

- informs clients which server is the head and which is the tail of the chain.

Using Paxos to coordinate those master instances, so they behave in aggregate like a single process that does not fail.

## Three cases

(i) failure of the head, (ii) failure of the tail, and (iii) failure of some other server in the chain.

### Failure of the Head 

This case is handled by the master removing H from the chain and making the successor to H the new head of the chain.

### Failure of the Tail

This case is handled by removing tail T from the chain and making predecessor T− of T the new tail of the chain.

### Failure of Other Servers

Failure of a server S internal to the chain is handled by deleting S from the chain. 

This, however, could cause the Up- date Propagation Invariant to be invalidated unless some means is employed to ensure update requests that S received before failing will still be forwarded along the chain (since those update requests already do appear in HistiobjID for any predecessor i of S) The Update Propagation Invariant in this case is preserved by:

The master first informs S’s successor S+ of the new chain configuration and then informs S’s predecessor S−.The Update Propagation Invariant is preserved by requiring that the first thing a replica S− connecting to a new successor S+ does is: send to S+ (using the FIFO link that connects them) those requests in HistS− that might not have reached S+; 

Note: 也就是说，如果不用采用上述补救措施的话，而是S挂掉后不采取任何措施，由S+继任，并且S-继续向S+传递，这样会有一个问题：由于S挂掉了，那么S+中将会缺少一部分update操作日志(即中间会有一个空洞),  这样一致性都无法保证了

# Extending a Chain

A new server could, in theory, be added anywhere in a chain. In practice, adding a server T+ to the very end of a chain seems simplist. For a tail T+, the value of SentT+ is always the empty list, so initializing SentT+ is trivial. All that remains is to initialize local object replica HistT+ in a way that objID satisfies the Update Propagation Invariant.

# Strong Consistency

## What is Strong Consistency

- operations to query and update individual objects are executed in some sequential order 

- the effects of update operations are necessarily reflected in results returned by subsequent query operations.

## Why CR is Strong Consistency

Strong consistency thus follows because query requests and update requests are all processed serially at a single server (the tail).

# 链式复制的优点

- 链式复制的每个节点（除了尾结点）都会产生写复制操作，而主从复制的写复制操作集中在主节点，这样就增加了主节点的负担；

- 主从复制要想提供强一致性（consistency），一般都会用上分布式一致性（consensus）算法；而链式复制由于写复制的顺序性，更容易实现强一致性

- 链式复制可用性强，例如要保证N个节点挂掉，集群仍然可用，链式存储只要有N+1个节点就可以了，但是主从方式需要2N+1个节点
