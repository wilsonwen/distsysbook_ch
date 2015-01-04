# %chapter_number%. 宏观的分布式系统

>分布式程序设计是在多台计算机上解决问题的艺术。

任何计算机系统都需要完成两个基本的任务

- 存储
- 计算

分布式程序设计是在多台计算机上解决问题的艺术。通常来说，是因为这个问题单机无法解决。

没有人要求你必须使用分布式系统。如果有无限的金钱和无限的研发时间，我们就不需要分布式系统。所有的计算和存储都可以在一个神奇的盒子上完成，一个超级迅速和超级可靠的系统。

但是，很少人有无限的资源。所以，他们要找到现实世界中，找到开销与收益的平衡点。在小规模下，升级硬件是一个可行的策略。不过，随着问题体积变大，你会达到硬件更新无法在单节点下解决问题的临界点。在这种情况下，我热烈欢迎你来到分布式系统的世界。

现实是，最有价值的是中档的商用硬件设备，只要其维护费用可以通过容错软件来降低。

计算主要受益于可以以内存访问替代网络访问的高端硬件。而高端硬件的性能会受限于需要节点间大量通信的任务。

![cost-efficiency](images/barroso_holzle.png)

正如上图所示[Barroso, Clidaras & Holzle](http://www.morganclaypool.com/doi/abs/10.2200/S00516ED2V01Y201306CAC024),在统一内存访问的模型下，高端硬件与普通硬件的性能差别随着集群的规模降低。

理想情况下，添加一台机器可以线性的提升系统的性能和容量。不过这当然是不可能的，因为会有额外的开销。数据需要复制多次，计算任务需要协同工作等。这就是学习分布式算法的原因———分布式算法针对特定问题提供了有效的解决方法，同时明确了什么是可能发生的，正确的实现的最小开销是多少和什么是不可能发生的。

本书的重点在于传统的但商业相关的分布式程序设计和分布式系统：数据中心。例如，我不会讨论奇怪的网络设置或者共享内存设置里的问题。另外，我们的重点在与探讨系统的设计空间，而不是优化任何特定的设计，后者是一个更加专业的话题。

## 我们的目的：可扩展性和其他好处

我看待问题的方法，所有事情都需要考虑规模。

大多是事情在小规模时都是微不足道的。但是一旦你超过一定的规模、流量或其他物理限制时，问题就变得困难。拿起一块巧克力很简单，但是拿起一座山就很难。统计一间房间里的人数很简单，统计一个国家的人数就很难。

所以任何事情都伴随着规模 —— 可扩展性。非正式的说，在一个可扩展系统里，我们从小规模到大规模，事情不应该逐渐的恶化。下面是另一个定义：

<dl>
  <dt>[可扩展性](http://zh.wikipedia.org/wiki/可扩展性)</dt>
  <dd>是系统，网络或进程能够在处理能力范围内处理不断增长的负荷的能力，或者是能够扩展适应增长的能力。</dd>
</dl>

不断增长的是什么？你几乎可以从任何角度来衡量增长（人口数量，电力使用等等）。但是，有三个有趣的事情：

- 规模可扩展性：新增计算节点应该可以使系统线性增速；数据的增长不应该增加时延。
- 地理可扩展性：在合理范围内处理跨数据中心的时延，使用多个数据中心可以减短用户查询的响应时间。
- 管理可扩展性：新增计算节点不应该增加系统管理开销。

当然，在现实系统中，增长是同时发生在多个维度的；每一项度量只包含了增长的一些方面。

分布式系统能够持续满足用户的扩展需求。其中有两个特别相关的方面 - 性能与可用性 - 可以用多个方法来衡量。

### 性能 (和时延)

<dl>
  <dt>[性能](http://en.wikipedia.org/wiki/Computer_performance)</dt>
  <dd>由计算机系统使用特定时间与资源可完成的任务来量化</dd>
</dl>

基于此，我们希望达到以下一个或多个目标：

- 短暂的响应时间/低时延
- 高吞吐率（处理任务的速率）
- 资源占用率低

优化其中任一项都需要权衡相互的影响。例如，系统可以通过处理较大的批处理任务，从而减少额外操作，达到更高的吞吐率。折衷是，由于批处理单个任务的响应时间会更长。

我发现低时延 - 短响应时间 - 是性能中最有趣的方面，因为其与物理限制有很强的关联。使用经济资源解决时延问题，比解决其他性能问题更加困难。

时延的定义有很多，其中我最认同的是这个单词创造者的说法：

<dl>
  <dt>时延</dt>
  <dd>延迟的状态；事物启动到发生的间隔</dd>
</dl>

那么延迟意味着什么呢？

<dl>
  <dt>延迟</dt>
  <dd>存在但未其作用</dd>
</dl>

这个定义很cool，因为它强调了事情发生到其作用的间隔时间。

例如，想象一下你被僵尸病毒感染了。时延就是你感染病毒到变成僵尸的时间。这就是时延：事情发生却未可见的时间。

我们假设，分布式系统只处理一件高层次的任务：给定一个任务，系统用所有数据计算出一个结果。换句话说，将分布式系统想成是一个具备特定计算能力的数据存储，计算方法为：

`result = query(all data in the system)`

那么，影响时延的不是旧数据的数量，而是新数据起作用的速度。例如，时延由写操作对读者可见的时间来衡量。

这个定义的另一个关键点是，如何没有事情发生，就没有时延。数据不发生改变的系统，没有时延的问题。

在分布式系统中，有一个最低时延是无法克服的：光速限制了信息的传播速度；硬件操作有最低的时延开销。

最低时延的影响，由查询本身和信息传播的物理距离决定。

### 可用性 (和失效容忍性)

可扩展系统的第二个方面就是可用性。

<dl>
  <dt>[可用性](http://en.wikipedia.org/wiki/High_availability)</dt>
  <dd>系统可用的时间比例。如果一个用户不能访问系统，那么该系统即不可用。</dd>
</dl>

分布式系统允许我们实现单机系统很难实现的特性。例如，单机不能容忍任何的失效。

分布式系统由一群不可靠的部分组成，在其之上构建一个可靠的系统。

没有冗余的系统的可用性，与其下层部分一样。基于冗余构建的系统能够容忍分区部分失效，从而更加可用。请注意，冗余在不同的层面意义不一样 - 组件，服务器，数据中心等等。

Formulaically, availability is: `Availability = uptime / (uptime + downtime)`.

Availability from a technical perspective is mostly about being fault tolerant. Because the probability of a failure occurring increases with the number of components, the system should be able to compensate so as to not become less reliable as the number of components increases.

For example:

<table>
<tr>
  <td>Availability %</td>
  <td>How much downtime is allowed per year?</td>
</tr>
<tr>
  <td>90% ("one nine")</td>
  <td>More than a month</td>
</tr>
<tr>
  <td>99% ("two nines")</td>
  <td>Less than 4 days</td>
</tr>
<tr>
  <td>99.9% ("three nines")</td>
  <td>Less than 9 hours</td>
</tr>
<tr>
  <td>99.99% ("four nines")</td>
  <td>Less than an hour</td>
</tr>
<tr>
  <td>99.999% ("five nines")</td>
  <td>~ 5 minutes</td>
</tr>
<tr>
  <td>99.9999% ("six nines")</td>
  <td>~ 31 seconds</td>
</tr>
</table>


Availability is in some sense a much wider concept than uptime, since the availability of a service can also be affected by, say, a network outage or the company owning the service going out of business (which would be a factor which is not really relevant to fault tolerance but would still influence the availability of the system). But without knowing every single specific aspect of the system, the best we can do is design for fault tolerance.

What does it mean to be fault tolerant?

<dl>
  <dt>Fault tolerance</dt>
  <dd>ability of a system to behave in a well-defined manner once faults occur</dd>
</dl>

Fault tolerance boils down to this: define what faults you expect and then design a system or an algorithm that is tolerant of them. You can't tolerate faults you haven't considered.

## What prevents us from achieving good things?

Distributed systems are constrained by two physical factors:

- the number of nodes (which increases with the required storage and computation capacity)
- the distance between nodes (information travels, at best, at the speed of light)

Working within those constraints:

- an increase in the number of independent nodes increases the probability of failure in a system (reducing availability and increasing administrative costs)
- an increase in the number of independent nodes may increase the need for communication between nodes (reducing performance as scale increases)
- an increase in geographic distance increases the minimum latency for communication between distant nodes (reducing performance for certain operations)

Beyond these tendencies - which are a result of the physical constraints - is the world of system design options.

Both performance and availability are defined by the external guarantees the system makes. On a high level, you can think of the guarantees as the SLA (service level agreement) for the system: if I write data, how quickly can I access it elsewhere? After the data is written, what guarantees do I have of durability? If I ask the system to run a computation, how quickly will it return results? When components fail, or are taken out of operation, what impact will this have on the system?

There is another criterion, which is not explicitly mentioned but implied: intelligibility. How understandable are the guarantees that are made? Of course, there are no simple metrics for what is intelligible.

I was kind of tempted to put "intelligibility" under physical limitations. After all, it is a hardware limitation in people that we have a hard time understanding anything that involves [more moving things than we have fingers](http://en.wikipedia.org/wiki/Working_memory#Capacity). That's the difference between an error and an anomaly - an error is incorrect behavior, while an anomaly is unexpected behavior. If you were smarter, you'd expect the anomalies to occur.

## Abstractions and models

This is where abstractions and models come into play. Abstractions make things more manageable by removing real-world aspects that are not relevant to solving a problem. Models describe the key properties of a distributed system in a precise manner. I'll discuss many kinds of models in the next chapter, such as:

- System model (asynchronous / synchronous)
- Failure model (crash-fail, partitions, Byzantine)
- Consistency model (strong, eventual)

A good abstraction makes working with a system easier to understand, while capturing the factors that are relevant for a particular purpose.

There is a tension between the reality that there are many nodes and with our desire for systems that "work like a single system". Often, the most familiar model (for example, implementing a shared memory abstraction on a distributed system) is too expensive.

A system that makes weaker guarantees has more freedom of action, and hence potentially greater performance - but it is also potentially hard to reason about. People are better at reasoning about systems that work like a single system, rather than a collection of nodes.

One can often gain performance by exposing more details about the internals of the system. For example, in [columnar storage](http://en.wikipedia.org/wiki/Column-oriented_DBMS), the user can (to some extent) reason about the locality of the key-value pairs within the system and hence make decisions that influence the performance of typical queries. Systems which hide these kinds of details are easier to understand (since they act more like single unit, with fewer details to think about), while systems that expose more real-world details may be more performant (because they correspond more closely to reality).

Several types of failures make writing distributed systems that act like a single system difficult. Network latency and network partitions (e.g. total network failure between some nodes) mean that a system needs to sometimes make hard choices about whether it is better to stay available but lose some crucial guarantees that cannot be enforced, or to play it safe and refuse clients when these types of failures occur.

The CAP theorem - which I will discuss in the next chapter - captures some of these tensions. In the end, the ideal system meets both programmer needs (clean semantics) and business needs (availability/consistency/latency).

## Design techniques: partition and replicate

The manner in which a data set is distributed between multiple nodes is very important. In order for any computation to happen, we need to locate the data and then act on it.

There are two basic techniques that can be applied to a data set. It can be split over multiple nodes (partitioning) to allow for more parallel processing. It can also be copied or cached on different nodes to reduce the distance between the client and the server and for greater fault tolerance (replication).

> Divide and conquer - I mean, partition and replicate.

The picture below illustrates the difference between these two: partitioned data (A and B below) is divided into independent sets, while replicated data (C below) is copied to multiple locations.

![Partition and replicate](images/part-repl.png)

This is the one-two punch for solving any problem where distributed computing plays a role. Of course, the trick is in picking the right technique for your concrete implementation; there are many algorithms that implement replication and partitioning, each with different limitations and advantages which need to be assessed against your design objectives.

### Partitioning

Partitioning is dividing the dataset into smaller distinct independent sets; this is used to reduce the impact of dataset growth since each partition is a subset of the data.

- Partitioning improves performance by limiting the amount of data to be examined and by locating related data in the same partition
- Partitioning improves availability by allowing partitions to fail independently, increasing the number of nodes that need to fail before availability is sacrificed

Partitioning is also very much application-specific, so it is hard to say much about it without knowing the specifics. That's why the focus is on replication in most texts, including this one.

Partitioning is mostly about defining your partitions based on what you think the primary access pattern will be, and dealing with the limitations that come from having independent partitions (e.g. inefficient access across partitions, different rate of growth etc.).

### Replication

Replication is making copies of the same data on multiple machines; this allows more servers to take part in the computation.

Let me inaccurately quote [Homer J. Simpson](http://en.wikipedia.org/wiki/Homer_vs._the_Eighteenth_Amendment):

> To replication! The cause of, and solution to all of life's problems.

Replication - copying or reproducing something - is the primary way in which we can fight latency.

- Replication improves performance by making additional computing power and bandwidth applicable to a new copy of the data
- Replication improves availability by creating additional copies of the data, increasing the number of nodes that need to fail before availability is sacrificed

Replication is about providing extra bandwidth, and caching where it counts. It is also about maintaining consistency in some way according to some consistency model.

Replication allows us to achieve scalability, performance and fault tolerance. Afraid of loss of availability or reduced performance? Replicate the data to avoid a bottleneck or single point of failure. Slow computation? Replicate the computation on multiple systems. Slow I/O? Replicate the data to a local cache to reduce latency or onto multiple machines to increase throughput.

Replication is also the source of many of the problems, since there are now independent copies of the data that has to be kept in sync on multiple machines - this means ensuring that the replication follows a consistency model.

The choice of a consistency model is crucial: a good consistency model provides clean semantics for programmers (in other words, the properties it guarantees are easy to reason about) and meets business/design goals such as high availability or strong consistency.

Only one consistency model for replication - strong consistency - allows you to program as-if the underlying data was not replicated. Other consistency models expose some internals of the replication to the programmer. However, weaker consistency models can provide lower latency and higher availability - and are not necessarily harder to understand, just different.

---

## Further reading

- [The Datacenter as a Computer - An Introduction to the Design of Warehouse-Scale Machines](http://www.morganclaypool.com/doi/pdf/10.2200/s00193ed1v01y200905cac006) - Barroso &  Hölzle, 2008
- [Fallacies of Distributed Computing](http://en.wikipedia.org/wiki/Fallacies_of_Distributed_Computing)
- [Notes on Distributed Systems for Young Bloods](http://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/) - Hodges, 2013
