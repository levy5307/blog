PacificA是微软实现的一款强一致性的分布式共识协议，具有简单易实现、可用性高的优点

## 理论基础
这一章的内容都是从微软发布的《[PacificA: Replicationi in Log-Based Distributed Storage System](https://www.microsoft.com/en-us/research/wp-content/uploads/2008/02/tr-2008-25.pdf)》总结而来，如有疑惑请移步。

### 前提条件
对于PacificA，需要系统满足下述条件：
1. server可能发生故障，但是只有fail-stop故障，不能是fail-slow
2. message可以被丢弃或者乱序，但是不能被修改
3. 网络分区可能发生
4. 不同服务器上的时钟不一定同步，甚至不一定是松散同步的，但是时间漂移有上限

### primary/backup结构
在PacificA中采取了primary-backup结构。同时我们将客户端请求分为两种：query和update。其中query不会更新数据、而update则会更新。
当一个replica group中的所有server都按照相同的顺序处理请求时，强一致性便可以得到保证。对于update请求，primary为其分配一个单调并连续增长的sn编号，并指示所有的secondary按照编号顺序执行update请求。我们为每个replica维护了一个prepare list和commit point，该prepare list是按照sn排序的。在prepare list中commit point之前的部分叫做committed list。committed list中的数据保证不会丢失（除非系统发生不可忍受的故障，即发生了所有replica的永久性故障）
在primary-backup模型中，所有的请求都会发送给primary。对于query请求，primary只需要本地处理就可以了，其获取当前最新的数据返回给客户端。但是对于update请求则需要所有的secondary都参与进来。其具体时序图如下：
![update时序图]()

由于：
- 只有当所有的secondary将该请求加入到prepare list中的之后，primary才会将其加入到committed list中，并且，
- 只有primary将该请求加入到committed list之后，secondary才会将其加入到committed list

所以，我们可以得到如下结论，我们称之为Commit Invariant。
***Commit Invariant***：Let p be the primary and q be any replica in the current configuration, committed(q) ⊆ committed(p) ⊆ prepared(q) holds.

当一个replica group的配置没有发生变化时，上述结论是成立的。在我们的replication模型里将数据管理和配置管理分割开来，接下来我们将看一下配置管理

### 配置管理
在我们的系统中，有一个叫做global configuration manager的组件存在，他用来管理所有replica group的配置，他保存当前的配置及其版本号

一个server可以根据failure detection探测到某个replica下线而删除某一个replica，相反也可以添加一个replica。这时该server需要将修改后的配置文件和当前配置的版本号发送给configuration manager。当版本号和configuration manager保存的版本号匹配时，新的配置将被采纳并将配置版本号+1

当发生网络分区时，配置冲突将会发生：primary尝试删除secondary；而secondary则尝试将自己提升为primary。此时configuration manager将如何处理这种情况呢？其实很简单，configuration manager只需要接收最早来的请求，而不管是primary发送来的、还是secondary发送来的。此时其接收到最早来的请求之后，会更新其本地保存的版本号，所以第二个之后来的请求都将会被拒绝。

任何遵从Primary Invariant的错误探测机制都可以用来删除一个replica: 
*** Primary Invariant***: At any time, a server p considers itself a primary only if the configuration manager has p as the primary in the current configuration it maintains. Hence, at any time, there exists at most one server that considers itself a primary for the replica group.

### 租期和错误探测
即使有configuration manager来维护当前配置，但是Primary Invariant也不一定能够保证。这是因为不同server上的配置文件的local view不一定相同，也即使说，有可能某些server上保存的是旧版本的配置文件、而没来得及更新。例如：old primary没有注意到重新配置导致产生了一个new primary，此时如果old primary还接收请求，有可能读取到一些旧的数据，这导致破坏了强一致性。

这个问题的解决方案就是采用租期。在一个租期内，primary定期的向secondary发送beacons，并等待其acknowledge。如果在一个固定的时间(lease period)内没有收到acknowledge，那么primary则认为其租期已经结束了，此时priimary将停止处理任何请求并且联系configuration manager去移除相应的secondary。同样对于secondary如果在一定时间(grace period)内没有收到来自primary的beacon，那么其同样会通知configuration manager，令其移除primary并令自己成为新的primary
