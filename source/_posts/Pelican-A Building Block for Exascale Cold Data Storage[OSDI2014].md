---
title: paper-reading[FAST2014-Pelican]
---
<h1>[OSDI2014]Pelican[COLON] A Building Block for Exascale Cold Data Storage</h1>

建议去USENIX看看这篇paper的presentation和slides

对"cold data"的描述是云存储中很少访问的数据，这类数据占了相当大的比例

- pelican是面向EB级冷数据存储、基于磁盘的机架级存储单元
- 将访问不频繁的数据存储在早期的基于磁盘的云存储中，是昂贵的，因为这些磁盘的配置基于热数据的工作负载，使用的配置提供低访问延迟以达到顶峰性能(peak performance，这会导致机架的上行链路带宽被完全充满，完全利用)，为了达到这一点，会有大量的供电、冷却和服务器支持，这带来了很大的成本开销，，如何为这类数据设计高性价比的存储成为一个挑战
- pelican的原型能够存储超过5PB的原始数据(raw data)
- pelican被设计为仅支持冷数据工作负载，每PB提供的峰值持续读取速度为1GB/PB/s
- 通过精心设计的软件栈使得pelican能够在仅有8%磁盘工作的时候仍能够满足冷数据工作负载的需求
- right-provisioning，一个刚好能够满足冷数据工作负载(如何分析负载得到的配置)的硬件配置
  - 根据冷数据的访问模式(写一次，很少读write once rarely read)，不需要所有的磁盘都同时工作来提供高性能，那么为了高性能而设置的大量供电、冷却、服务器设施就是不必要的了
  - 大量的电力供应、冷却机构、服务器被移除，使得这些硬件只占用机架极少一部分的空间，从而使得52U的机架能够放置1152个磁盘
  - 只有2个server进行管理
  - 上行带宽为4*10Gbps
  - 磁盘也是Archival级别的，不是commodity级别，这种磁盘对spin-up延迟和容量作了优化
  - right-provisioning削减了大量的硬件资源，这导致的结果是，并非所有的磁盘都能在任何时候启动
  - 实际上，冷却机构只能让8%磁盘同时旋转，如果超过，就会导致有的磁盘过热从而影响使用寿命
  - 存储系统的配置下，冷却机构只能让96个磁盘同时转动(一共1152个磁盘，机架大小为52U，~22disks/U)
  - right-provisioning虽然减少了成本，增加了存储密度，但是硬件资源的减少带来了明显的资源约束问题，作者将会采用软件的方法进行解决

- pelican将大量的磁盘分成磁盘组管理，对于1152个磁盘，分成48个磁盘组进行管理，组内的磁盘接受同样的磁盘物理约束(电力、带宽、冷却、故障域等)，在同一个组的磁盘会同时启动
- pelican通过分组进行数据布局和IO调度

第2节提供Pelican硬件的概述，并描述数据布局和I/O调度器，但在讨论硬件结构之后、数据布局之前，作者首先详细定义了资源约束，然后再将layout和I/O schedule是如何解决资源约束问题的

资源约束：

- 对活动磁盘组的约束
- 定义了资源域(resource domain)，有四种：power cooling vibration bandwidth
- paper里的图为了简单展示，没有呈现vibration domain和bandwidth domain
- power domain：16个中的2个disks可以活动
- cooling domain: 12个中的1个disk可以活动
- vibration domain：2个中的1个disk可以活动

在这样的资源约束下，数据布局，或者说数据放置的目标是最大化请求的并发度，也就是说，对于到达的请求，让尽可能多的请求能够被同时服务，冲突感知的数据放置

- 数据被按blob组织，一个blob被存入pelican时，会使用纠删码进行编码，并且blob会被均分成多个片段(fragment)存到一组磁盘中(stripe)
- 这些片段所在的一组磁盘被要求能够同时活动，以最大化IO性能
- 让任意两组磁盘组同时活动，在硬件资源丰富的系统，是没有问题的
- 但是在pelican的right-provisioning的系统中，由于硬件资源紧缺，两组磁盘组可能会发生资源上的竞争(供电不够用、冷却不够用等)，比如这两组磁盘中的某两个disk的资源域重叠(在同一列的disks，发生冷却资源的竞争)
- 因此需要最小化资源冲突的概率
- 考虑一个blob经过编码产生n段放置到n个磁盘，如果随机放置，那么对于下一个要放置到n个磁盘的blob，发生冲突的概率符合O(n^2^)
- 让冲突在组与组之间发生
- 组与组之间要么完全冲突，要么完全不冲突
- 完全冲突的组组成的集合，称为class
- 静态地将磁盘分成组，通过按对角线分(paper有图)，这样能保证组内的每个disk都在单独的power domain和cooling domain，这样组内的disk不会发生资源冲突，只有可能组与组之间发生冲突
- 这使得冲突的概率变成符合O(n)
- pelican设置1个组有24个disks，对于1152个磁盘，有48个组，形成4个class，每个class有12个组，在不同class的组不会发生资源冲突，所以系统的并发度是4，也就是说，如果有4个对不同class上的请求，它们能够被同时服务
- 需要注意，paper声称pelican只是EB级冷数据存储的building block，这意味着实际部署的时候，必定是部署大量的pelican机架，而不是只有一台。实际上，作为microsoft的一个计划，project pelican后续的工作是把这种机架集成到microsoft的Azure云存储的冷数据存储层中
- 组内的18个磁盘用来存一个blob，而一个组有24个磁盘，这样，即使组内有磁盘损坏，仍然有可能在组内找到18个磁盘来存blob
- 通过组封装了约束，也就是说，当我想知道哪些blob请求可以并发时，只需要看这些blob是否在同一个class，如果在，则不能并发，否则能并发	

对于IO调度器来说，问题在于怎么去减少spin up latency的影响

- IO调度器采用均摊spin up latency的思路
- 均摊的方式是把操作组织成一个组来把spin up latency均分到每个操作上

评估使用的指标：

- completion time：指的是从客户端发出请求到最后一个数据字节被发送到客户端之间的时间。这个指标包括排队延迟、磁盘启动延迟、读取和传输数据的时间
- time to first byte：指的是从客户端发出请求到第一个数据字节被发送到客户端之间的时间。该指标包括排队延迟和任何磁盘启动与卷挂载的延迟
- service time：从调度器将请求出队列到与该请求相关的最后一个字节被传输的时间。该指标包括磁盘启动延迟和数据传输时间，但不包括排队延迟
- average reject time：这些实验使用开放回路工作负载；当提供的负载超过 Pelican 或 FP 系统的服务能力时，一旦调度器的队列满了，请求将被拒绝。此指标衡量被拒绝的请求比例。网卡（NIC）在 40 Gbps 的情况下以每秒 5 个请求成为瓶颈。因此，在所有实验中，除非另有说明，我们将实验速率设定为每秒 8 个请求。
- throughput：即机架网络的平均吞吐量，计算方法为实验期间通过网络链接传输的总字节数除以实验持续时间


![time drawio](/images/time.drawio.svg)


作者在USENIX做presentation的时候被MAID作者的学生问到了有关MAID的问题：

为什么你做的东西和我导师02年做的MAID很相似？

作者没有在presentation上直接回答，只是说pelican独特之处在于根据cold storage workload的要求做了right provisioning保证性价比并通过软件栈(mainly,data layout+IO scheduler)解决了由此引发的问题

第二个问题，来自其他人：

据他所知，磁盘工作到一定次数的周期就坏了，如果每天spin up and down一百多次，大概1年半就磁盘就坏了，这会导致在读取之前磁盘已经不可用

作者给出的回答是这些磁盘为archival storage进行专门的硬件硬件优化，对spin-up和spin-down的周期次数做了优化，是manageable的，至于有关工作次数/寿命相关参数的spec，it's confidential

关于pelican使用的archival disk，详见

[*8th USENIX Workshop on Hot Topics in Storage and File Systems (HotStorage 16)*]

Feeding the Pelican: Using Archival Hard Drives for Cold Storage Racks

<a href=https://www.computerweekly.com/blog/StorageBuzz/Microsoft-revives-MAID-with-Pelican-but-tape-can-still-sleep-easy>[Pelican is being developed and rolled out by Microsoft for its Azure cloud datacentres, and is explicitly only for those that are “not big enough” for tape infrastructure (Azure currently uses IBM TS3500 libraries), according to Russinovich.](https://www.computerweekly.com/blog/StorageBuzz/Microsoft-revives-MAID-with-Pelican-but-tape-can-still-sleep-easy)</a>