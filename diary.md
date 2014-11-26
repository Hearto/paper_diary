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
	 <tr><tr/>
2. **简单化(simplicity)：**大道至简，拒绝复杂反而能带来高效。正如原文所述：*simplicity is as important as fault-tolerance when building a large-scale reliable, maintainable system.*
	-  不追求最优化和通用性 (*avoid unnecessary optimization, and unnecessary generality*)；
	-  配置简单化，如采用纯文本的配置文件，同时减少或者避免在没有产生审计跟踪情况下的配置变更；


 
## **Autopilot系统概述** ##

Autopilot分为几个不同的组件，如下图：

![](http://i.imgur.com/CBIueLA.jpg)

可以看出Device Manager是枢纽，各个组件的功能和职责在后面章节有具体介绍。



**Autopilot采用的一致性策略：**在弱一致性和强一致性之间做一个简单的折衷。

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

8. **scale unit：**Autopilot采用了scale unit（大约500台computers）这样一个概念用于rollout新的code version。Device Manager在执行roll out和roll back时每次操作一个scale unit，当一个scale-unit中有足够量的机器部署运行正确后，才会进行下一个scale unit的操作。小到一个配置文件参数的修改，大至系统镜像的部署，Autopilot都采用的这种方式。

## 自动修复服务（AUTOMATIC REPAIR SERVICES） ##
Autopilot提供了一种简单的故障检测和恢复的机制，它的failure unit是针对一台计算机或者一个网络交换机而言的，而不是一个process，提供的修复手段也只有Reboot、Reimage和Replace，这样保证了autopilot本身不需要关心应用程序级别的异常。
![](http://i.imgur.com/bMLhcdZ.jpg) 


如上图，集群中的每一个节点被归类为Healthy、Failure 和 Probation中的一个，记录在Device Manager中。

1. **Watchdogs：**用于检测Autopilot的健康状况，Watchdogs将结果反馈到Device Manager，结果为 OK、Warning和Error中的一种，Warning和Error会带有相关的文字说明。watchdog与Device Manager通信采用一个简单的watchdog协议（脚本语言编写）。**Watchdog Service**周期性地检查操作系统镜像和manifest，发现硬盘和内存错误情况。watchdogs的数量不被限制，根据特定的场景可以添加一定数量的watchdogs。Device Manager在收到所有的watchdogs的结果后才会判定一台computer处于error状态，一旦这台computer被判定为error，它可能会在相当的一段时间无法工作，因此watchdog一定要减少误报；这样，意味着在watchdog报错一段时间后，Device Manager才会发现它然后采取相应的操作。
2. **Failure/Recovery State Machine：**通常情况下，Device Manager标识一节点处于Healthy状态。当watchdog递交了error报告并且这个节点最终被认为是Failure状态时，Device Manager会采取一个动作（DoNothing, Reboot, ReImage, Replace中的一个），具体动作依赖于这个节点的故障历史。
	- 如果一个节点相当一段时间内并没有异常，则首先会Donothing以期望该Error是一个误报或者瞬间错误；
	- BIOS引起的致命硬盘错误则会导致这个节点立即被标记为replacement；
	- 一段时间内反复发生非致命性错误会导致Device Manager执行reboot或者reimage操作，最终将其标记为replacment；
	- 数据中心里面的dead computers会被定期地移除，替代；
	- Autopilot会启发式地决定维修操作的顺序和时间。

3. **Repair Service**周期性地向Device Mananger请求处于Failure状态的computer列表，以及一些修复操作，然后Device Manager将这些computer的状态修改为Probation。处于Probation状态的computer根据watchdog提供的错误报告执行修复操作。
	- 如果修复成功，这些错误会很快消失；
	- 如果处于Probation状态的computer足够长的时间内没有发生错误，它会被重置为Healthy状态；
	- 如果一个computer长时间停留在Probation状态，它会被重置为Failure状态，然后采用进一步的修复操作。
4. Device Manager执行一次新的部署时，在rollout之前会把相关的computer从Healthy状态变为Probation状态，采用上面的模型，当computer恢复Healthy状态时认为部署成功，否则部署失败。这样保证了修复操作与新部署状态的统一。
5. 所有的信息都由watchdogs发往Device Manager，由其决定需要执行的操作，这样Device Manager能获取全局信息，可以控制同时在repair的机器数量，避免某种异常时导致所有的都报Error，同时repair大量机器，导致线上服务不可用。

## MONITORING SERVICES ##

1. Autopilot通过使用Performance counters和logs完成对components和applications的MONITORING SERVICES。Performance counters记录components的实时状态，logs记录components的操作。
2. **Collection Service**提供一种分布式的服务，它收集集群中各个节点的Performance counters和logs信息并聚合，存储在一个大的分布式系统中用于日后分析；
3. **Cockpit**是Autopilot的一种可视化分析工具。实时的performance counter信息存储在SQL数据库中以便可视化分析。Cockpit根据performance counter数据库中的内容生成图标和报告，使得操作员可以monitor多个Autopilot集群。

<br></br>

## CASE STUDY: INDEX SERVING & LESSONS LEARNED FROM VERSION 1 ##
文中第八部分CASE STUDY: INDEX SERVING部分简单地介绍了Autopilot在web-index serving cluster环境中的一些操作措施；第九部分LESSONS LEARNED FROM VERSION 1介绍了第一个版本的Autopilot的使用情况和存在的问题，以及后续版本待改进的地方。

## 总结 ##

这篇论文发表于2007年，讲述了Microsoft Autopilot的设计思想、系统架构以及一些主要组件的功能和工作方式。Autopilot意在解放人工、降低成本，实现集群数据中心的自动化管理。其中，Device Manager是Autopilot的大脑、控制中心、决策中心。打个比方，如果Autopilot是一支军队，那么Device Manager就是这支军队的唯一指挥官，军队里面的士兵们（围绕Device Manager的一些服务，如Provisioning Service，Deployment Service，Watchdog Service，Repair Service等）对这个指挥官的命令必须绝对地服从；这支军队里面的士兵们自己不需要思考问题，他们发现问题直接报告给指挥官即可，指挥官作出决策后再把命令分发给士兵们；士兵们直接接触军队里面的资源，如枪支器械等，但这些资源的购买、维修、调度、分配都取决于指挥官；这支军队的最大优点就是简单，易管理，不容易发生混乱，但是由于所有的决策都要通过这个唯一的指挥官，会导致一些问题处理的不及时；比如当一个士兵发现一个坦克有故障的时候，他报告给指挥官，但指挥官不会听信单个士兵的报告，只有当报告这件事情的士兵数量达到足够多的时候，指挥官才会采取相应的处理措施；如何保持枪支器械的质量一直是一个难题，试想无用的枪支器械无外乎就是一堆破铜烂铁，没有了它们这支军队又何言战斗力啊！

微软很少在公开场合谈论Autopilot，而且迄今为止也只在两份官方文件中谈到过这款工具：一份发布于2007年、如今早已过时的文章，就是这篇《Autopilot: Automatic Data Center Management》；另一个则是2013年发布的网页，其中解释了Autopilot的开发团队如何凭借在这套系统研发工作中的不懈努力赢得“杰出技术成就”奖。Autopilot的强大实力为微软带来了显著收益，因为它有效提高了该公司在驾驭其价值数十亿美元的计算设备时所表现出的工作效率。随着微软公司在云计算大师纳德拉的带领下逐步向“设备与服务”企业转型，Autopilot的重要性只会随着时间的推移外加微软冲击广阔数字化世界而愈发得到凸显。*摘自微软秘密曝光！Autopilot是隐藏云武器：[http://cloud.51cto.com/art/201402/428756.htm]()*




## **文中的一些概念：** ##

1. **拜占庭将军问题（Byzantine failures）**又称两军问题，是由莱斯利·兰伯特提出的点对点通信中的基本问题。含义是**在存在消息丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的**。因此对一致性的研究一般假设信道是可靠的，或不存在本问题。
	
	<tr>拜占庭假设是对现实世界的模型化，由于硬件错误、网络拥塞或断开以及遭到恶意攻击，计算机和网络可能出现不可预料的行为。拜占庭容错协议必须处理这些失效，并且这些协议还要满足所要解决的问题要求的规范。</tr>
	> *摘自百度百科：*[http://baike.baidu.com/view/2815670.htm?fr=aladdin]()

2. **审计跟踪（audit trail）**，是系统活动的流水记录。**该记录按事件从始至终的途径，顺序检查审计跟踪记录、审查和检验每个事件的环境及活动。**审计跟踪通过书面方式提供应负责任人员的活动证据以支持职能的实现。审计跟踪记录系统活动和用户活动。系统活动包括操作系统和应用程序进程的活动；用户活动包括用户在操作系统中和应用程序中的活动。通过借助适当的工具和规程，审计跟踪可以发现违反安全策略的活动、影响运行效率的问题以及程序中的错误。
	> *摘自百度百科：*[http://baike.baidu.com/view/4323485.htm?fromtitle=audit+trail&type=syn]()

3. **一致性问题：**分布式系统的一致性问题总是伴随数据复制而生， 数据复制技术在提高分布式系统的可用性、可靠性和性能的同时，却带来了不一致问题。
	- **强一致性：**按照某一顺序串行执行存储对象读写操作， 更新存储对象之后， 后续访问总是读到最新值。 假如进程A先更新了存储对象，存储系统保证后续A,B,C的读取操作都将返回最新值。
	- **弱一致性：**更新存储对象之后，后续访问可能读不到最新值。假如进程A先更新了存储对象，存储系统不能保证后续A,B,C的读取操作能读取到最新值。从更新成功这一刻开始算起，到所有访问者都能读到修改后对象为止，这段时间称为”不一致性窗口”，窗口内访问存储时无法保证一致性。
	- **最终一致性：**最终一致性是弱一致性的特例，存储系统保证所有访问将最终读到对象最新值。譬如， 进程A写一个存储对象，如果对象上后续没有更新操作，那么最终A,B,C的读取操作都会读取到A写入的值。 “不一致性窗口”的大小依赖于交互延迟，系统的负载，以及副本个数等。
	> *摘自bitstech-存储、分布式技术：*[http://www.bitstech.net/2014/08/11/consistency/]()

4. **Paxos算法**解决的问题是**在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性**。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。
	> *具体参见维基百科：*[http://zh.wikipedia.org/zh-cn/Paxos%E7%AE%97%E6%B3%95]()


## 附 ##
这是我仔细阅读过的第一篇英文论文，笔记采用了边读边记的方式，后面两篇论文我会试着采取先整体阅读一遍再记笔记的方式。

> *笔记参考《Microsoft AutoPilot – 管理大规模集群的利器》：* [http://leoncom.org/?p=650735]()
