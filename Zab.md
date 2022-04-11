# Abstrct

```markdown
Zab is a crash-recovery atomic broadcast algorithm we designed for the ZooKeeper coordination service. ZooKeeper implements a primary-backup scheme in which a primary process executes clients operations and uses Zab to propagate the corresponding incremental state changes to backup processes. Due the dependence of an incremental state change on the sequence of changes previously generated, Zab must guarantee that if it delivers a given state change, then all other changes it depends upon must be delivered first. Since primaries may crash, Zab must satisfy this requirement despite crashes of primaries.
Applications using ZooKeeper demand high-performance from the service, and consequently, one important goal is the ability of having multiple outstanding client operations at a time. Zab enables multiple outstanding state changes by guaranteeing that at most one primary is able to broadcast state changes and have them incorporated into the state, and by using a synchronization phase while establishing a new primary. Before this synchronization phase completes, a new primary does not broadcast new state changes. Finally, Zab uses an identification scheme for state changes that enables a process to easily identify missing changes. This feature is key for efficient recovery.
*Experiments and experience so far in production show that our design enables an implementation that meets the performance requirements of our applications. Our implementation of Zab can achieve tens of thousands of broadcasts per second, which is sufficient for demanding systems such as our Web-scale applications.*
```

Zab是一种被设计用于Zookeeper的故障恢复原子广播算法。Zookeeper实现了一种主备模式，主进程执行客户端指令，并通过Zab将相应的状态增量变化传递给备进程。由于状态变化增量依赖于之前状态变化的顺序，Zab必须保证：如果Zab提交了一个状态变化，该变化依赖的所有其他状态变化应当先于提交。即使在主节点Crash的情况下，Zab也必须满足这一条件。

使用Zookeeper的应用需要提供高性能服务，因此同时提交多个客户端指令（***having multiple outstanding client operations at a time***）的能力是一项重要目标。Zab实现**multiple outstanding state change**通过保证以下条件实现：

+ 至多有一个主可以广播状态变化，并且该主广播的状态变化都汇总到状态中；
+ 在选主期间引入**同步阶段**(在同步阶段完成前，新主不能广播)。

最终，Zab使用一种**identification scheme for state changes**，该方案使进程可以轻松的发现状态变化丢失。

Zab可以达到每秒万级的广播能力。

# Introduction

```markdown
### Zookeeper Introduction
Atomic broadcast is a commonly used primitive in distributed computing and ZooKeeper is yet another application to use atomic broadcast. ZooKeeper is a highly-available coordination service used in production Web systems such as the Yahoo! crawler for over three years. Such applications often comprise a large number of processes and rely upon ZooKeeper to perform important coordination tasks, such as storing configuration data reliably and keeping the status of running processes. Given the reliance of large applications on ZooKeeper, the service must be able to mask and recover from failures. [1]
```

```markdown
**ZooKeeper is a replicated service, and it requires that a majority (or more generally a quorum) of servers has not crashed for progress. Crashed servers are able to recover and rejoin the ensemble as with previous crash-recovery protocols [2], [3], [4]. ZooKeeper uses a primary-backup scheme [5], [6], [7] to maintain the state of replica processes consistent. With ZooKeeper, a primary process receives all incoming client requests, executes them, and propagates the resulting non-commutative, incremental state changes in the form of transactions to the backup replicas using Zab, the ZooKeeper atomic broadcast protocol. Upon primary crashes, processes execute a recovery protocol both to agree upon a common consistent state before resuming regular operation and to establish a new primary to broadcast state changes. To exercise the primary role, a process must have the support of a quorum of processes. As processes can crash and recover, there can be over time multiple primaries and in fact the same process may exercise the primary role multiple times. To distinguish the different primaries over time, we associate an instance value with each established primary. A given instance value maps to at most one process. Note that our notion of instance shares some of the properties of views of group communication [8], but it presents some key differences. With group communication, all processes in a given view are able to broadcast, and configuration changes happen when any process joins or leaves. With Zab, processes change to a new view (or primary instance) only when a primary crashes or loses support from a quorum.**
```

Zookeeper是一个冗余副本服务，它要求集群中的大多数server处于正常状态。宕机的server能够恢复并且能够重新加入集群。Zookeeper使用一种主备模式维持各备份进程中状态值的一致性。在Zookeeper中，primary接收、执行client的请求，并将结果单向传递，状态变化增量以transaction的形式通过Zab协议传给备份。当primary宕机时，集群中各server执行恢复协议，包括以下两部分工作：
+ 在恢复正常工作前，达成一个一致性共识状态；
+ 建立（选举）一个新的primary来广播状态值变化；

作为primary的server必须得到集群中大多数的支持。由于server可能宕机并恢复，在一段时间内多个server可能充当过primary，而且同一个server也可能充当过多次primary。为了分辨不同的primary，我们为每个primary指定一个instanceid（epochid），每个primary周期的instanceid保证不一样。在Group Communication[8]中，处于同一view的server可以进行广播，并且当任一server离开或者加入集群时会触发配置更新。在Zab协议下，只有当primary宕机或者失去大多数server支持时才会切换到新的view（新的epoch）。

```markdown
Critical to the design of Zab is the observation that each state change is incremental with respect to the previous state, so there is an implicit dependence on the order of the state changes. State changes consequently cannot be applied in any arbitrary order, and it is critical to guarantee that a prefix of the state changes generated by a given primary are delivered and applied to the service state. State changes are idempotent and applying the same state change multiple times does not lead to inconsistencies as long as the application order is consistent with the delivery order. Consequently, guaranteeing at-least once semantics is sufficient and simplifies the implementation.
```

Zab协议的关键是状态更新的单向性，底层是对于状态更新顺序的依赖。保证primary产生的先前的状态更新被传递并接受是关键。状态变化需要是幂等的。因此，保证至少一次的语法就足够了。

```markdown
As Zab is a critical component of the ZooKeeper core, it must perform well. Some applications of ZooKeeper en- compass a large number of processes and use ZooKeeper ex- tensively. Previous systems have been designed to coordinate long-lived and infrequent application state changes [9], [10], [11]. We designed ZooKeeper to have high throughput and low latency, so that applications could use it extensively on cluster environments: data centers with a large number of well-connected nodes.
```



```markdown
When designing ZooKeeper, however, we found it difficult to reason about atomic broadcast in isolation. There are requirements and goals of the application that must be satisfied, and reasoning about atomic broadcast alongside the applica- tion enables different protocol elements and even interesting optimizations.

**Multiple outstanding transactions**: It is important in our setting that we enable multiple outstanding ZooKeeper opera- tions and that a prefix of operations submitted concurrently by a ZooKeeper client are committed according to FIFO order. Traditional protocols to implement replicated state machines, like Paxos [2], do not enable such a feature directly, however. If primaries propose transactions individually, then the order of learned transactions might not satisfy the order dependencies and consequently the sequence of learned transactions cannot be used unmodified. One known solution to this problem is batching multiple transactions into a single Paxos proposal and having at most one outstanding proposal at a time. Such a design affects either throughput or latency adversely depending on the choice of the batch size.

Figure 1 illustrates a problem we found with Paxos under our requirements. It shows a run with three distinct proposers that violates our requirement for the order of generated state changes. Proposer P1 executes Phase 1 for sequence numbers 27 and 28. It proposes values A and B for sequence numbers 27 and 28, respectively, in Phase 2 with ballot number 1. Both proposals are accepted only by acceptor A1. Proposer P2 executes Phase 1 against acceptors A2 and A3, and end up proposing C in Phase 2 to sequence number 27 with ballot number 2. Finally, proposer P3, executes Phase 1 and 2, and is able to have a quorum of acceptors choosing C for sequence number 27, B for sequence number 28, and D for 29.

Such a run is not acceptable because the state change represented by B causally depends upon A, and not C. Consequently, B can only be chosen for sequence number i+1 if A has been chosen for sequence number i, and C cannot be chosen before B, since the state change that B represents cannot commute with C and can only be applied after A.

**Efficient recovery**: One important goal in our setting is to recover efficiently from primary crashes. For fast recovery, we use a transaction identification scheme that enables a new primary to determine in a simple manner which sequence of transactions to use to recover the application state. In our scheme, transaction identifiers are pairs of values: an instance value and the position of a given transaction in the sequence broadcast by the primary process for that instance. Under this scheme, only the process having accepted the transaction with the highest identifier may have to copy transactions to the new primary, and no other transaction requires recovery. This observation implies that a new primary is able to decide which transactions to recover and from which process simply by collecting the highest transaction identifier from each process in a quorum.
```



# System Model



# Problem Statement



# Zab In Detail



# Analysis



# EVALUATION



# Related Work



