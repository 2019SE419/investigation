# Reading -- Mesos

## Background

共享可提高群集利用率，并避免按帧复制数据。 Mesos以细粒度的方式共享资源，从而允许框架通过轮流读取存储在每台计算机上的数据来实现数据局部性。 为了解决框架的异构问题，Mesos 必须要给框架自己调度的权限，这就是双层调度的真正由来。 

## Architecture

![16dd411594eb4470](.\images\16dd411594eb4470.jpg)

- 在Mesos之上运行的框架由两个组件组成：一个**调度程序**（scheduler），用于注册提供资源的服务器到master，以及在从属节点上启动以执行框架任务的**执行程序**进程（executor）。
- master节点上运行scheduler程序
- 用来加载运作在slave nodes上的框架的任务。 每一个slave上可以运行多个executor，而每个executor与框架相关。
-  master进程决定给每个slave提供多少资源，集群框架的调度器选择使用哪一种被调度的资源。 

![16dd48a7c7a099f9](.\images\16dd48a7c7a099f9.jpg)

不同的 Executor 之间如何竞争资源呢？**关键在于 Mesos Slave 会先从 master 发送当前持有的资源，而资源信息本身是框架无关的**。 

- slave先向master索要资源
- master将请求发给某一个scheduler
- scheduler为每一个task分配资源
- master将资源和分配信息下发给executor

有两点值得提示：首先过滤器只是代表resource offer模型的性能优化，框架仍然拥有拒绝任何资源以及选择任务执行的节点的最终决定权。其次，当workload包括细粒度的任务（比如mapreduce的workload）时，resource offer模型在缺少过滤器的情况下，性能出奇的好。特别的，我们发现一种叫delay scheduling的简单机制，使用这种机制的框架等待一定的时间，可以获得存储他们所需数据的节点，基本在等待1-5S内可以达到最佳的数据局部性。 

## Revocation

Mesos将资源分配的决定权交给了插件化的分配模块，那么用户可以根据其需求定制分配策略，mesos实现了两个分配模块，一个是基于**max-min fairness**，对不同的资源执行fair sharing，另一个是实现了严格的优先级。 

在普通的操作中，mesos在大部分任务都是短时间，只有在任务结束后重新分配资源的场景下有优势。通常这种情况会频繁的发生，这样一个新框架才能快速地获取它所需要的资源。 举个列子：如果一个框架的共享资源是集群的10%，那么这个框架需要等待平均任务长度的10%的时间来接收它需要的资源。然而，假如一个集群中充满了长任务，比如一个古怪的作业和贪婪的框架，分配模块可以取消任务，但取消任务之前，mesos会给予框架一个宽限期去释放资源。 

首先对于大部分框架，取消一个任务没有多大影响，但对于那些任务相互依赖的框架，取消掉一个任务会带来很大的影响，我们允许这些框架通过分配模块来公开每一个框架的（guaranteed allocations）最低保障分配来避免取消任务，（guaranteed allocations）最低保障分配是只一个框架在不失去task的情况下所需要持有的资源量；框架通过一个api调用来读入其（guaranteed allocations）保障分配，分配模块负责满足框架的（guaranteed allocations）保障分配；当前，mesos是指简单地提供了（guaranteed allocations）保障分配的语义：加入一个框架拥有的资源低于其（guaranteed allocations）保障分配，那么任何一个task都不应该被取消，如果高于（guaranteed allocations）保障分配，那么任何的任务都可以被取消。 其次，决定何时触发取消操作，mesos必须知道在已经连接的框架中，那些框架使用的资源超出分配给它的资源限定。框架可以通过一个api调用表示它对资源的兴趣。 

## Isolation

Mesos通过利用现有的操作系统隔离机制来提供多个框架执行器(framework executors)在同一个slave上运行的性能隔离(performancd isolation)。既然这些机制是平台依赖的，那么mesos通过一个可插拔的隔离模块来提供做种隔离机制。 

## Making Resource Offers Scalable and Robust

Mesos 需要一些特殊的实现来补足 offer 机制的不足。

第一，如果每次框架都进行某些资源的拒绝，这样显然是非效率的，所以 Mesos 提供了叫做 filters 的短路机制；第二，框架需要时间响应 offer，所以 Mesos 提供的资源是按照框架自身所需资源的份额；第三，如果框架长期没有响应，就取消（rescind）这个 offer。

## Fault Tolerance

既然所有的框架都依赖mesos master，所有master的容错至关重要，为了达到这一点，我们通过将master设计成软状态，因此一个新的master可以通过slave和framework schedulers持有的信息完整地重建其内部状态。特别的，master的唯一状态就是active slave和active frame的列表和运行的tasks。这些信息足够计算每一个框架正在使用和运行多少资源。我们通过zookeeper的leader election来实现对master的热备。当active master失败时，slaves和schedulers会连接下一个被选举出了的master，重新填充其状态。

出了处理master的失败，mesos也会向框架的scheduler上报节点失败和executor crash。Framework可以根据他们选择的策略相应这些错误。

最后，处理scheduler的失败，mesos允许一个框架注册多个schedulers，当一个失败时，mesos master会提示另一个接管，但framework必须使用他们自己的机制在scheduler之间共享状态。

## Mesos Behavior

###  Definitions, Metrics and Assumptions

- framework ramp-up time: 一个新框架得到它所需要的资源所花费的时间（对于公平调度策略，一般来说就是等待上一个任务完成的时间）。
- job completion time: 作业完成时间，假设每个框架只有一个作业。
- system utilization：总的集群利用率

我们用两个维度来描述工作负载（workloads）：弹性和任务持续时间分布。

一个弹性框架（elastic framework）如hadoop和dryad能对集群资源进行伸缩，比如它一旦获取到集群节点就立即使用以及任务执行完后立刻释放节点。相反对于刚性框架（rigid framework），如MPI，只有在获取到一个固定数量的资源后，才能运行其作业，而且不能进行动态地伸缩（在有新资源时进行扩展提升性能，在性能受到很大影响时进行缩减）。对于任务的持续时间，考虑同构和异构这两种分布。

### Heterogeneous Tasks

当长短任务交错时，关键是如何防止长任务持久的占据资源。Mesos 的策略也很简单：每个节点上运行的 slot，其中长任务不能超过50%。论文还提到， Mesos 并不 care 你是长任务还是短任务，它只是设置一个超时时间然后严格执行（kill 掉 task）。但这种策略是否会导致任务的低效重复执行呢？

### Framework Incentives

这一部分是对 Mesos 的使用者最有用的，相当于使用指南。文章列了4点：

- 短任务：可分配的 slot 变多；最小化任务失败的损失
- 无最小分配：反例就是 MPI
- 弹性（scale down）：可以很容易减少需求，这样更容易得到资源
- 不接受未知资源：这个跟 DRG 算法有关

### Limitations of Distributed Scheduling

在 Mesos 列出的限制之中，碎片化是一直被人所诟病的。文章中指出，这种长短任务交替的场景是一个装箱（Bin-packing）问题，而这个问题很难被优化（NP-hard）。另外，还有一种情况是小作业阻塞了长任务，此时 Mesos 通过限制每个 slave 最小分配的资源来保证长任务不会总是无法请求到所需资源。

