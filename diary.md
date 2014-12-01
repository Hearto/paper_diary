# Autopilot: Automatic Data Center Management #
*by Microsoft Research Silicon Valley*
> *原文地址:*[http://research.microsoft.com/pubs/64604/osr2007.pdf]()


## 初识Autopilot ##
1. 文章摘要部分对Autopilot的说明为：
*Autopilot is responsible for automating software provisioning and deployment; system monitoring; and carrying out repair actions to deal with faulty software and hardware.*
可以理解为**Autopilot是这样一种东西，它负责着自动化软件的配置和部署；系统监测；软件和硬件的故障修复工作。**
	
	**Autopilot是一款帮助微软将数百万台服务器以及上万PB海量数据融合成一整套庞大强劲计算及存储资源池的工具，**<a>http://cloud.51cto.com/art/201402/428756.htm</a>一文中如是说。


## **关于设计原则(DESIGN PRINCIPLES)：** ##


1. **容错能力(fault tolerance)：**既然Autopilot想让员工享受8x5而不是24x7，所以它必须具备强悍的容错能力。现实中，无法避免组件不期地出现错误，系统的健壮性、可靠性必须得到保证。
	- 错误应被自动处理或抑制，阻止其扩张转变为故障，导致系统瘫痪； 
	- 没有任何implementation是万无一失的，但应该尽力把风险降到最低；
	- 当然，未雨绸缪，最好先想办法避免错误，比出现了再去处理强；
	- 硬件方便，单个计算机宕机不影响机群，单个机群宕掉不影响其他机群。
<br></br>
2. **简单化(simplicity)：**大道至简，拒绝复杂反而能带来高效。正如原文所述： *simplicity is as important as fault-tolerance when building a large-scale reliable, maintainable system.*
	- 不追求最优化和通用性 (*avoid unnecessary optimization, and unnecessary generality*)；
	- 配置简单化，如采用纯文本的配置文件，同时减少或者避免在没有产生审计跟踪情况下的配置变更；


 
## **Autopilot系统概述** ##

Autopilot分为几个不同的组件，如下图：

![](http://i.imgur.com/CBIueLA.jpg)

可以看出Device Manager是枢纽，各个组件的功能和职责在后面章节有具体介绍。



1. **Autopilot采用的一致性策略：**在弱一致性和强一致性之间做一个简单的折衷。
	- **Device Manager**（大约5-10计算机），存储着系统所有的“ground truth”状态信息，采用强一致性策略，使用轻量级库、Paxos算法。原文中对此的描述为：*All of the information about the ―ground truth‖ state that the system should be in, along with the logic to update this ground truth, is held in a single strongly-consistent state machine called the Device Manager that is typically distributed over 5–10 computers.*
	- Autopilot提供了一些叫作**satellite**的服务，原文对其描述为：*These satellites lazily perform actions to bring themselves or their clients up to date when they discover that the Device Manager state requires it. Similarly, if a satellite discovers a fault or inconsistency in the cluster, it will not attempt to rectify it but instead will report the problem to the Device Manager.***satellite服务执行一些“发现”性质的actions，这些satellite服务自己不处理fault，而是将fault报告给Device Mannager，Device Manager将处理结果再反馈给satellite。Autopilot通过这样一种方式区分satellite与Device Manager的系统状态：在不失正确性的情况下，satellite能保持他们的状态弱一致。**在这种模型中，satellite从Device Manager接收消息称之为pull，反之为push。系统允许satellite服务“慢悠悠”的工作着，但必须保证最终的正确性和一致性。


## **LOW-LEVEL服务** ##

1. **filesync service：**广泛应用在Autopilot和Application，用于文件同步，确保本地磁盘文件的正确性。filesync既可以作为client从远程服务器获取文件，也可以作为server向其他client提供这种服务。自己实现filesync的优势在于拥塞控制，避免电脑或者交互机网卡异常超载。
2. **application manager:**用于保证proccess的正常运行。每一个application process使用一个独立目录保存process所需的binaries、共享库、配置文件以及启动脚本。application manager根据这些配置信息确保process的正常运行。
3. **Provisioning Service**:周期性地扫描网络，发现有新的机器加入时，向Device Manager发送请求获取该机器运行所需的系统镜像，然后使用计算机的management接口完成系统的安装、启动和burn-in测试，这几个步骤完成后通知Device Manager。Provisioning Service使用几台计算机作冗余，通过相应的操作从这些机器中选出一个leader，Provisioning Service所需的信息在leader启动时从Device Manager获得。
4. **machine types**： Device Manager database中存储着集群中的每一个computer对应的machine type。computer与machine types的映射目前是手动静态配置的。Autopilot 本身不负责迁移或者负载均衡，而是applications根据machine types自动分发任务实现自身的负载均衡。从deployment角度来讲，machine types之间的差别主要体现在配置文件和application binaries，而通常不依赖硬件配置。
5. **configuration manifest**：存储在Device Manager database，用于描述一个文件集合。每个节点可以一次存储多个manifests，但是只能有一个处于active状态，application manager只运行处于acitve manifests里面的process。这样新版本的applications可以pre-load,而处于非active状态，同样也方便新版本的回滚。Device Manager中记录着不同type的computer可以运行的程序manifests。
6. **Deployment Service**：集群中的每台计算机周期性地查询Device Manager database以获得manifests信息。Deployment Service相当于一些弱一致副本的集合，里面存储着这些manifest对应程序、配置以及数据等。如果manifests缺失，则随机从Deployment Service副本中挑选一个。Device Manager对manifests有着最高的控制权，它始终保存着最新的manifests信息。
7. **deployment new version：**集群中的每一种machine type维系着一个active manifest，因此相同machine type的机群运行着相同的application binaries，具有相同的配置。在准备部署一个新版本时，
	- 最新版本的manifest首先存储到Deployment Servers，同时通知Device Manager，将对应的manifest存放到对应的machine type机器上；
	- 执行一次kick，或者等机器本身的scheduled synchronization task运行，使得所有待部署的computer获取manifest更新；
	- 当足够多的机器获取了最新的manifest时，Device Manager激活这个最新版本；
<br></br>
8. **scale unit：**Autopilot采用了scale unit（大约500台computers）这样一个概念用于rollout新的code version。Device Manager在执行roll out和roll back时每次操作一个scale unit，当一个scale-unit中有足够量的机器部署运行正确后，才会进行下一个scale unit的操作。小到一个配置文件参数的修改，大至系统镜像的部署，Autopilot都采用的这种方式。

## 自动修复服务（AUTOMATIC REPAIR SERVICES） ##
Autopilot提供了一种简单的故障检测和恢复的机制，它的failure unit是针对一台计算机或者一个网络交换机而言的，而不是一个process，提供的修复手段也只有Reboot、Reimage和Replace，这样保证了autopilot本身不需要关心应用程序级别的异常。
![](http://i.imgur.com/bMLhcdZ.jpg) 


如上图，集群中的每一个节点被归类为Healthy、Failure 和 Probation中的一个，记录在Device Manager中。

1. **Watchdogs：**用于检测Autopilot的健康状况，Watchdogs将结果反馈到Device Manager，结果为 OK、Warning和Error中的一种，Warning和Error会带有相关的文字说明。watchdog与Device Manager通信采用一个简单的watchdog协议（脚本语言编写）。**Watchdog Service**周期性地检查操作系统镜像和manifest，发现硬盘和内存错误情况。watchdogs的数量不被限制，根据特定的场景可以添加一定数量的watchdogs。Device Manager在收到所有的watchdogs的结果后才会判定一台computer处于error状态，一旦这台computer被判定为error，它可能会在相当的一段时间无法工作，因此watchdog一定要减少误报；这样，意味着在watchdog报错一段时间后，Device Manager才会发现它然后采取相应的操作。</br>
2. **Failure/Recovery State Machine：**通常情况下，Device Manager标识一节点处于Healthy状态。当watchdog递交了error报告并且这个节点最终被认为是Failure状态时，Device Manager会采取一个动作（DoNothing, Reboot, ReImage, Replace中的一个），具体动作依赖于这个节点的故障历史。
	- 如果一个节点相当一段时间内并没有异常，则首先会Donothing以期望该Error是一个误报或者瞬间错误；
	- BIOS引起的致命硬盘错误则会导致这个节点立即被标记为replacement；
	- 一段时间内反复发生非致命性错误会导致Device Manager执行reboot或者reimage操作，最终将其标记为replacment；
	- 数据中心里面的dead computers会被定期地移除，替代；
	- Autopilot会启发式地决定维修操作的顺序和时间。
<br></br>
3. **Repair Service**周期性地向Device Mananger请求处于Failure状态的computer列表，以及一些修复操作，然后Device Manager将这些computer的状态修改为Probation。处于Probation状态的computer根据watchdog提供的错误报告执行修复操作。
	- 如果修复成功，这些错误会很快消失；
	- 如果处于Probation状态的computer足够长的时间内没有发生错误，它会被重置为Healthy状态；
	- 如果一个computer长时间停留在Probation状态，它会被重置为Failure状态，然后采用进一步的修复操作。
<br></br>
4. Device Manager执行一次新的部署时，在rollout之前会把相关的computer从Healthy状态变为Probation状态，采用上面的模型，当computer恢复Healthy状态时认为部署成功，否则部署失败。这样保证了修复操作与新部署状态的统一。
5. 所有的信息都由watchdogs发往Device Manager，由其决定需要执行的操作，这样Device Manager能获取全局信息，可以控制同时在repair的机器数量，避免某种异常时导致所有的都报Error，同时repair大量机器，导致线上服务不可用。

## MONITORING SERVICES ##

1. Autopilot通过使用Performance counters和logs完成对components和applications的MONITORING SERVICES。Performance counters记录components的实时状态，logs记录components的操作。
2. **Collection Service**提供一种分布式的服务，它收集集群中各个节点的Performance counters和logs信息并聚合，存储在一个大的分布式系统中用于日后分析；
3. **Cockpit**是Autopilot的一种可视化分析工具。实时的performance counter信息存储在SQL数据库中以便可视化分析。Cockpit根据performance counter数据库中的内容生成图标和报告，使得操作员可以monitor多个Autopilot集群。

## CASE STUDY: INDEX SERVING & LESSONS LEARNED FROM VERSION 1 ##
文中第八部分CASE STUDY: INDEX SERVING部分简单地介绍了Autopilot在web-index serving cluster环境中的一些操作措施；第九部分LESSONS LEARNED FROM VERSION 1介绍了第一个版本的Autopilot的使用情况和存在的问题，以及后续版本待改进的地方。

## 总结 ##

这篇论文发表于2007年，讲述了Microsoft Autopilot的设计思想、系统架构以及一些主要组件的功能和工作方式。Autopilot意在解放人工、降低成本，实现集群数据中心的自动化管理。其中，Device Manager是Autopilot的大脑、控制中心、决策中心。打个比方，如果Autopilot是一支军队，那么Device Manager就是这支军队的唯一指挥官，军队里面的士兵们（围绕Device Manager的一些服务，如Provisioning Service，Deployment Service，Watchdog Service，Repair Service等）对这个指挥官的命令必须绝对地服从；这支军队里面的士兵们自己不需要思考问题，他们发现问题直接报告给指挥官即可，指挥官作出决策后再把命令分发给士兵们；士兵们直接接触军队里面的资源，如枪支器械等，但这些资源的购买、维修、调度、分配都取决于指挥官；这支军队的最大优点就是简单，易管理，不容易发生混乱，但是由于所有的决策都要通过这个唯一的指挥官，会导致一些问题处理的不及时；比如当一个士兵发现一个坦克有故障的时候，他报告给指挥官，但指挥官不会听信单个士兵的报告，只有当报告这件事情的士兵数量达到足够多的时候，指挥官才会采取相应的处理措施；如何保持枪支器械的质量一直是一个难题，试想无用的枪支器械无外乎就是一堆破铜烂铁，没有了它们这支军队又何言战斗力啊！

微软很少在公开场合谈论Autopilot，而且迄今为止也只在两份官方文件中谈到过这款工具：一份发布于2007年、如今早已过时的文章，就是这篇《Autopilot: Automatic Data Center Management》；另一个则是2013年发布的网页，其中解释了Autopilot的开发团队如何凭借在这套系统研发工作中的不懈努力赢得“杰出技术成就”奖。Autopilot的强大实力为微软带来了显著收益，因为它有效提高了该公司在驾驭其价值数十亿美元的计算设备时所表现出的工作效率。随着微软公司在云计算大师纳德拉的带领下逐步向“设备与服务”企业转型，Autopilot的重要性只会随着时间的推移外加微软冲击广阔数字化世界而愈发得到凸显。*摘自微软秘密曝光！Autopilot是隐藏云武器：[http://cloud.51cto.com/art/201402/428756.htm]()*




## **文中的一些概念：** ##

1. **拜占庭将军问题（Byzantine failures）**又称两军问题，是由莱斯利·兰伯特提出的点对点通信中的基本问题。含义是**在存在消息丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的**。因此对一致性的研究一般假设信道是可靠的，或不存在本问题。

	拜占庭假设是对现实世界的模型化，由于硬件错误、网络拥塞或断开以及遭到恶意攻击，计算机和网络可能出现不可预料的行为。拜占庭容错协议必须处理这些失效，并且这些协议还要满足所要解决的问题要求的规范。</tr>
	> *摘自百度百科：*[http://baike.baidu.com/view/2815670.htm?fr=aladdin]()
<br>
2. **审计跟踪（audit trail）**，是系统活动的流水记录。**该记录按事件从始至终的途径，顺序检查审计跟踪记录、审查和检验每个事件的环境及活动。**审计跟踪通过书面方式提供应负责任人员的活动证据以支持职能的实现。审计跟踪记录系统活动和用户活动。系统活动包括操作系统和应用程序进程的活动；用户活动包括用户在操作系统中和应用程序中的活动。通过借助适当的工具和规程，审计跟踪可以发现违反安全策略的活动、影响运行效率的问题以及程序中的错误。
	> *摘自百度百科：*[http://baike.baidu.com/view/4323485.htm?fromtitle=audit+trail&type=syn]()
3. **一致性问题：**分布式系统的一致性问题总是伴随数据复制而生， 数据复制技术在提高分布式系统的可用性、可靠性和性能的同时，却带来了不一致问题。
	- **强一致性：**按照某一顺序串行执行存储对象读写操作， 更新存储对象之后， 后续访问总是读到最新值。 假如进程A先更新了存储对象，存储系统保证后续A,B,C的读取操作都将返回最新值。
	- **弱一致性：**更新存储对象之后，后续访问可能读不到最新值。假如进程A先更新了存储对象，存储系统不能保证后续A,B,C的读取操作能读取到最新值。从更新成功这一刻开始算起，到所有访问者都能读到修改后对象为止，这段时间称为”不一致性窗口”，窗口内访问存储时无法保证一致性。
	- **最终一致性：**最终一致性是弱一致性的特例，存储系统保证所有访问将最终读到对象最新值。譬如， 进程A写一个存储对象，如果对象上后续没有更新操作，那么最终A,B,C的读取操作都会读取到A写入的值。 “不一致性窗口”的大小依赖于交互延迟，系统的负载，以及副本个数等。
	> *摘自bitstech-存储、分布式技术：*[http://www.bitstech.net/2014/08/11/consistency/]()
<br>
4. **Paxos算法**解决的问题是**在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性**。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。
	> *具体参见维基百科：*[http://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95]()

## 附 ##
这是我仔细阅读过的第一篇英文论文，笔记采用了边读边记的方式，后面两篇论文我会试着采取先整体阅读一遍再记笔记的方式。

> *参考资料《Microsoft AutoPilot – 管理大规模集群的利器》：* [http://leoncom.org/?p=650735]()

<br>
<br>
<br>
<br>

# Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center #
*University of California, Berkeley*

> *原文地址：*[http://mesos.berkeley.edu/mesos_tech_report.pdf]()

## 全文概览 ##

## Abstract & Introduction & Target Environment##
Mesos计算框架一个集群管理器，提供了有效的、跨分布式应用或框架的资源隔离和共享，可以运行Hadoop、MPI等framework。Mesos使用resource offer决定每一个framwork的资源分配，framework自身可以有选择地使用分配到的资源。

**背景：**新的framework不断涌现，一个集群中可能会同时运行多种framework，但没有一种framework会适合所有的application。

**目标：**针对不同的frameworks，提供一个通用的接口；resource offer & data locality。

文章第二章Target Environment部分，通过Facebook在部署hadoop集群时遇到的the cluster can only run Hadoop jobs为例，简述了Mesos适合的应用场景。

## Architecture ##

1.  **设计理念：**意在提供一种稳定、可扩展的调度管理机制使得多个frameworks共享同一个集群中的资源。*define a minimal interface that enables
efficient resource sharing, and otherwise push control
to the frameworks.*
2. **Overview:**Mesos也采用了master/slave模型，如下图：
![](http://i.imgur.com/jZXnaEg.jpg)
	+ **Mesos-master：**主要负责管理各个framework和slave，并将slave上的资源分配给各个framework；
	+ **Mesos-slave：**负责管理本节点上的各个mesos-task，比如：为各个executor分配资源；
	+ **Framework-scheduler：**注册到master，接收资源分配；
	+ **Framework-executor：**运行在slave上，执行framework的Task；
	+ **pluggable allocation module:**可插拔的分配模型——**master** 决定分配给每一个framework的资源量，**scheduler** 选择合适的资源分给它的Task；

![](http://i.imgur.com/uj5MNFF.jpg)<br>
	如上图,论文中举了一个简单的例子来说明Mesos的调度过程：<br>
	1）Slave1告诉master自己的资源情况（4 CPUs and 4 GB of memory free），Master唤醒资源分配模块；<br>
	2）Master将资源情况发送给Framework1；<br>
	3）Framework1的scheduler告知Master它的task信息（task1:2cpu,1gb ,task2:1cpu,2gb）；<br>
	4）Master把task发送到Slave1，Slave1上的Executor运行task；
<br>
3. **Resource Allocation：**Mesos的资源分配模块（allocation module）负责决定将资源分配给哪些framwork，和给每个framwork分配多少资源。

- **Dominant Resource Fairness (DRF):**是一种根据dominant resource（占有量最多的资源）的fairness分配策略。例如：集群有100cpu和100gb，framwork F1每个task需要4cpu和1gb，F2每个task需要1cpu和8gb，则DRF分配给F1 20个task（80cpu和10gb），分配给F2 10个task（20cpu和80gb），这样使得F1的dominant resource（cpu）和F2的dominant resource（ram）所占有的资源在数量上相等。
- **Supporting Long Tasks：**Mesos适合fine-grained（short Tasks）的环境，同样也有针对long tasks的处理措施。好一点的情况是：framwork本身能够标记task为short或者long，这样short task可以任意使用资源，而long task只能使用指定数量的资源；当framwork不标记task时，对于long task的**revocation**文中提到了两种机制：一种是要求资源分配模块提供guaranteed allocation（framework不会losing task）；另一种是资源分配模块根据framework的interest程度（通过API调用实现）决定对哪些task执行revocation操作。
- **Isolation：**Mesos的隔离模块（isolation modules）主要使用linux containers。文中提到，Mesos将来可能会采用virtual machines代替containers；
- **Resource Offers Scalable and Robust：**Mesos通过三种机制实现Resource Offers的高效和稳定：
	+ 使用过滤（Filter）机制：允许framework只接收“剩余资源量大于L的 slave”或者“只接收node列表中的slave”；
	+ 使用一种counts resources方法“刺激”frameworks快速响应；
	+ 取消并重新分配那些长时间没有响应的offer；
<br>
- **Fault Tolerance：**Mesos使用Zookeeper解决master的单点故障问题；
## Mesos Behavior ##
文中第四章讲述了Mesos针对一种和多种任务（Homogeneous and Heterogeneous Tasks）、Elastic和Rigid两种Frameworks的运行情况。Mesos将资源分为required和preferred：required是Framework运行所必须的资源；preferred是Framework可以使用但不是必须的资源。

1. **Elastic and Rigid Frameworks：elastic Framworks获得部分资源即可运行，而rigid Frameworks则需要获得它所需要的所有资源才能够运行。**文中4.2节计算了固定周期任务（constant durations task）和指数增长周期任务（exponential durations task）运行在elastic Framework和rigid Framework中的任务完成时间（job completion time）和系统利用率（system utilization）的理论结果。
2. **Placement Preferences:**Mesos采用了类似彩票调度（lottery scheduling）的资源分配策略，即根据frameworks的intended allocations的比例权重决定分配他们的preferred资源数量。
3. **Homogeneous and Heterogeneous Tasks ：** *毫无疑问，Mesos总是给short task更多的优待。* Mesos使用一种**Framework Incentives**机制提升jobs的响应时间：
	+ **short task优先：**short task所占资源少；任务异常时，short task更容易revocation；
	+  **资源及时分配（No minimum allocation）**而不是达到一个minimum allocation值才去分配，以便提升资源利用率；
	+  **frameworks不接收不明资源**，将资源留给需要的frameworks，提升系统利用率；
<br>
4. 文中第四章最后一节分析了Mesos调度策略存在的一些**局部性**：
	+ small task优先导致的large task饥饿问题；
	+ framework之间的相互依赖问题；
	+ 使用resources offers带来了framework调度的复杂性；

## Implementation & Evaluation ##

Mesos由大约一万行C++代码实现，使用了C++的libprocess库；此外，前面有介绍，Mesos的实现也借助于Zookeeper、hdfs、linux container等；这两章分别简单介绍了Hadoop、Torque and MPI、Spark framwork、ElasticWeb Server Farm在Mesos上的运行场景和运行情况；
 
## 总结 ##

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mesos诞生于UC Berkeley的一个研究项目，现已成为Apache Incubator中的项目，当前有一些公司使用Mesos管理集群资源，比如Twitter。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;总体上看，Mesos是一个master/slave结构，其中，master是非常轻量级的，仅保存了framework（各种计算框架称为framework）和mesos slave的一些状态，而这些状态很容易通过framework和slave重新注册而重构，因而很容易使用了zookeeper解决mesos master的单点故障问题。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mesos master实际上是一个全局资源调度器，采用某种策略将某个slave上的空闲资源分配给某一个framework，各种framework通过自己的调度器向Mesos master注册，以接入到Mesos中；而Mesos slave主要功能是汇报任务的状态和启动各个framework的executor（比如Hadoop的excutor就是TaskTracker）。
> *以上摘自：*[http://dongxicheng.org/mapreduce-nextgen/mesos_vs_yarn/]()

正如文中所述：Mesos实现的是一个*fine-grained sharing mode*（细粒度共享模型），Mesos针对short tasks环境运行效果会比较好，虽然文中也花了相当一部分篇幅讲述Mesos对long tasks的处理措施，但long tasks应该始终是Mesos架构中的一个瓶颈，其实这很容易理解，**就像一个平衡杆，一端提升了另一端没有不下降的道理，如果Mesos真能面面俱到，我想它早已飞上天了！！！**此外，data locality在分布式系统地位举足轻重，关于Mesos的data locality性能文中没有详细说明，有待进一步研究。


## 相关概念 ##

1. **Hive：**是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。 其优点是学习成本低，可以通过类SQL语句快速实现简单的MapReduce统计，不必开发专门的MapReduce应用，十分适合数据仓库的统计分析。
	> *摘自百度百科：* [http://baike.baidu.com/subview/699292/10164173.htm?fr=aladdin]()<br>
	> *官网：*[http://hive.apache.org/]()

2. **Zookeeper：**是Google的Chubby一个开源的实现，是高有效和可靠的协同工作系统，Zookeeper能够用来leader选举，配置信息维护等，在一个分布式的环境中，需要一个Master实例或存储一些配置信息，确保文件写入的一致性等。
	> *摘自百度百科：*[http://baike.baidu.com/view/3061646.htm?fr=aladdin]() 
	<br>
	> *官网：*[http://zookeeper.apache.org/]()

3. **lottery scheduling：彩票调度**，其基本思想是向进程提供各种系统资源（如CPU时间）的彩票。一旦需要做出一项调度决策时，就随机抽出一张彩票，拥有该彩票的进程获得该资源。在应用到CPU调度时，系统可以掌握每秒钟50次的一种彩票，作为奖励每个获奖者可以得到20ms的CPU时间。</br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 为了说明George Orwell关于“所有进程是平等的，但是某些进程更平等一些”的含义，可以给更重要的进程额外的彩票，以便增加它们获胜的机会。如果出售了100张彩票，而有一个进程持有其中的20张，那么在每一次抽奖中该进程就有20%的取胜机会。在较长的运行中，该进程会得到20%的CPU。相反，对于优先级调度程序，很难说明拥有优先级40究竟是什么意思，而这里的规则很清楚：拥有彩票f份额的进程大约得到系统资源的f 份额。
	> *摘自《读书笔记之进程调度（二）》*[http://www.cnblogs.com/wawlian/archive/2012/02/21/2361819.html]()
	<br>
	> *lottery scheduling维基百科：*[http://en.wikipedia.org/wiki/Lottery_scheduling]()

4. **SWIG：** 是一个非常优秀的开源工具，支持您将 C/C++ 代码与任何主流脚本语言相集成。此外，它向更广泛的受众公开了基本代码，改善了可测试性，让您的 Ruby 代码库某部分能快速写出高性能的 C/C++ 模块。
	> *摘自《开发人员 SWIG 快速入门》：*[http://www.ibm.com/developerworks/cn/aix/library/au-swig/]()
	<br>
	> *SWIG维基百科：*[http://en.wikipedia.org/wiki/SWIG]()

5. **LibProcess：**是一套基于Socket实现的通信协议库，它支持Protocal Buffer，通过两者结合，可实现一套很高效的基于消息传递的通信协议库，而Mesos底层通信协议正是采用了该库。
	> *摘自《Apache Mesos底层网络通信库libprocess介绍》（具体查看）：*[http://dongxicheng.org/apache-mesos/mesos-libprocess/]()

6. **Spark：**是UC Berkeley AMP lab所开源的类Hadoop MapReduce的通用的并行计算框架，Spark基于map reduce算法实现的分布式计算，拥有Hadoop MapReduce所具有的优点；但不同于MapReduce的是Job中间输出结果可以保存在内存中，从而不再需要读写HDFS，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代的map reduce的算法。
	> *摘自百度百科：*[http://baike.baidu.com/subview/123524/123524.htm?fr=aladdin]()
	<br>
	> *官网：*[http://spark.apache.org/]()

7. **Dryad和DryadLINQ：**是微软研究院的两个项目，用于辅助C#开发人员在在计算机集群或数据中心里处理大规模的数据。Dryad是一个在计算机集群或数据中心里并行地执行顺序程序的基础架构。DryadLINQ是“一个把LINQ程序转化成分布式计算指令，以便运行于PC集群的编译器”。这个转化过程可以分解为以下几步：
	- C#和LINQ数据对象转化为分布式的文件块。
	- LINQ查询转化为分布式Dryad任务。
	- C#方法转化为运行于Dryad任务节点上的代码。
	> *摘自百度百科：*[http://baike.baidu.com/view/5417085.htm?fr=aladdin]()
	<br>
	> *官网说明：*[http://research.microsoft.com/en-us/projects/dryadLINQ/]()

8. **HAProxy：**提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。HAProxy特别适用于那些负载特大的web站点，这些站点通常又需要会话保持或七层处理。HAProxy运行在当前的硬件上，完全可以支持数以万计的并发连接。并且它的运行模式使得它可以很简单安全的整合进您当前的架构中， 同时可以保护你的web服务器不被暴露到网络上。
	> *摘自百度百科：*[http://baike.baidu.com/view/2480120.htm?fr=aladdin]()
	<br>
	> *官网：*[http://haproxy.com/]()

## 附 ##

> *参考资料：*[http://dongxicheng.org/category/apache-mesos/]()