## 随心记录

这篇论文主要涉及的是调度系统。论文的整体逻辑和内容还是理解的。但是对**Linux 线程调度的基本了解** 还是不够。对整个**os社区对调度这一块的近期研究**不太了解，比如为为特定工作流定制特定调度策略。各种硬件的发展，比如NUMA、accelerator、什么socket之类的东西都不了解。

现在读论文的一个感受就是，碰见作者提到的一个概念或者一种情况，我只能大概地想，哦，可能是这样子的。他具体的对应着什么例子，细节如何我可能完全不了解。

我觉得我现在对操作系统其实是有基本的了解的，但是在各种时候还是要加深对其的了解。

# ghOSt

## background&design goal

背景：硬件变复杂，安全问题。--》优化系统处理关键负载的性能

简介liux的调度，现有的调度器都是很通用的，不适用于针对特定硬件和负载进行优化

实现调度器很困难：低级语言、debugging、同步问题

部署更加困难

user-level 线程：不确定性，不知道native线程什么时候被允许以及在哪个cpu上运行。可以分配cpu但在负载低时浪费cpu

shinjuku、和shenango虽然针对他们的目标有很好的性能，但缺陷多、代码量大，需要专用设备、无法与其他应用共存等。

BPF也不太行，表达力有限、能使用的内核数据结构也有限。是同步运行的。在ghOSt中，BPF prog被挂载在*pick_next_task*上来利用ghOSt调度过长引起的空闲cpu时间

### 设计目标

1. 表达能力强
2. 易实现和测试（代码层面上吗
3. 支持per-cpu model之外的模型
4. concurrent policy <--要求-- 机器性能的横向扩展
5. 非破坏升级、错误隔离

## 设计

### basic

* 调度Native线程

* 提出enclave概念，分割机器：4，5

* 用户空间agent，由pthread实现、每个cpu都有一个。

  实现调度策略逻辑、错误冗余（fallback to CFS etc.）：1，2，5

### kernel2agent

**其他实现方式**

* memory-map *task_structure*
* via sysfy files

**ker-user API**

* message通知线程状态变化和timer tick

* message  queue：在enclave初始化之后，有一个默认的queue。
* 对于per-cpu，thread归属于给将要运行cpu对应的queue
* 线程在加入ghOSt时，隐式分配给默认queue
* queue可以唤醒agent
* 可以改变message 的路由
* agent和kernel对线程状态的同步问题：sequence numbers
* auxiliary information：status words

### agent2kernel

必须能支持us级，和百个核心的可扩展性

* transaction：拥有语义完整性的快速，分布式提交，多个提交可同时指定同一个远程node
* 组提交：减少syscall，batch interrupt
* per-cpu，在transaction提交到内核中时会通过check ASN来保证上面提到的agent-kernel同步问题
* 通过bpf加速调度：消息传到用户空间，然后把调度决策传回内核很耗费时间。在这段时间内可以通过BPF利用空闲时间。

### centralized scheduler

为保证global agent持续运行

* 给与agent高优先级，可以减少延迟
* 但为了系统稳定性，inactive马上yield cpu，active cpu 将工作转移
* 用ThreadSN保证同步

### falut isolation和dynamic upgrade

* ghOSt kernel shceduler class 比默认调度器优先级低；agent优先级更高，ghOSt线程被占用后，由agent处理
* 不摧毁enclave，原地升级agent。失败的话，自动吧线程交给CFS
* 如果发现agent没有在规定时间内给可以运行的线程调度，则自动摧毁enclave

### 问题：从整个运行逻辑出发

* 怎么创建enclave
* cpu-queu-agent之间的关系，如果一个cpu和agent之间没有queue会怎么办
* thread怎么加入enclave
* message是如何发出的，这一部分是修改了什么的代码，实在调度类里实现的还是整个调度机制有了变化
* 消息路由变化

## 测试

### ghOSt开销

* 消息传递
* 本地调度
* 远程调度：组提交增加了调度吞吐量
* global agent 可扩展性

### 与自定义中心调度器的比较

对比了shinjuku的三种实现方式

shinjuku无法调度其他native进程。

shenango监测网络应用的负载，从而实现cpu共享

他们使用goOSt实现了结合两者的策略

### Google Snap

* 原理：动态唤醒线程，为了保证延迟，尝试real-time，但destabilize系统，尝试soft-real-time（分一点时间出去）
* 负载设计
* 调度策略设计
* 比较及原因分析

### Google Search

* 工作负载设计
* 实验设计：CCX
* 策略：计算硬件的拓扑逻辑，cpumask & idel cpus
* 针对NUMA和CCX的快速优化，体现了ghOst的优越性

### 保护VM不遭受LTF/MDS攻击

* 调度physical core，per-cpu model很难实现
* 安全VM  core调度策略：partitioned EDF scheme；软NUMA偏好
* 测试

## 未来的工作

* 将agent的部分职责，交给同步执行的BPF程序来加速ghOSt
* tick-less scheduling。在VM环境下，tick会导致VM退出到主机内核环境。

## ---

## 需要学习的内容:heavy_exclamation_mark:

* Linux的cpu调度机制（mechanism）
  * mechanism和policy有什么区别
  * round-robin policy
  * priority inversion
  * pthread
  * 线程和进程
  * 多核cpu、超线程cpu、NUMA、Socket、CCX、timer_tick
  * 如何支持多核
* 同步问题
  * 通信中的同步：同步（一问一答），异步（自己问自己的）
  * 操作系统中的同步：原语，RCU，任务抢占，中断
  * RCU
  * 读者与写者问题
  * ring buffer
* 其他
  * 什么叫us-scale workload
  * 什么叫amortization
  * cross-hyperthread speculative execution attacks
  * transaction memory
  * microbenmark
  * cpu saturation throughput
  * tial workload
  * partitioned EDF scheme

* 

# 会议

## 读完两篇论文后对大方向的思考:exclamation:

我又**重温了一下以前的笔记**。回答了几个问题。

首先，到底研究BPF的哪些方面。**BPF只是个工具**。

我读论文下来感觉更重要的两点分别

* 对操作系统本身有充分的了解。我现在只是很**笼统的理解**，具体的机制，例如cpu调度机制是如何进行的完全不够了解。还有例如操作系统中涉及的各种同步问题也是一概不知。这会造成我在读论文的时候，某些部分可能无法理解。例如XRP中的同步问题，还有ghOSt中kernel中ghOSt调度类怎么原有调度机制相兼容，怎么向agent发出message等等，这些都需要代码上的了解。

  所以我觉得，加强操作系统**具体的机制**和**代码层面上的认知**十分重要。

* 也**别局限于操作系统，视野要广**

* 另一个是对**领域现有研究**的了解。例如XRP中提到的存储设备中的技术，kernel bypass的技术；还有ghOSt中提到的针对特定工作负载提出调度策略从而提高性能；需要**多读论文，略读**，把整个**领域的研究情况**做出一个知识结构，从而有**整体的把握**。

差点忘了，我要学习的是

* **科研思维**，那种发现有价值的问题，提出解决方案，设计实验验证方案这样的研究思维

最近突然好像在**思考自己的思维模式**

比如说科研思维模式？比如我在读论文的时候我看不懂，但我甚至不知道自己的问题出在哪里，这时候我应该怎么思考，怎么去一步一步地解决他。

## 交流

李老师，我昨天刚重新梳理完ghOSt这篇论文。整体的逻辑和大概的内容还是能把握的。

不过，我的一个感受是对os具体的机制或者说代码层面上的理解还是不够，导致我对论文的细节把握不够；另一个感受就是对整个社区近期的研究的全面了解比较少，比如定制调度策略优化特定负载，NUMA，kernel bypass这些，需要拓宽文献的阅读范围，阅读速度，学习怎么快速抓住论文重点。

我看了一下剩下的两篇论文，分别是cpu调度和存储系统有关的。

所以下一步我想先加紧把剩下两篇看完，然后通过复现加强代码层面上的理

## 会中记录

**李：**

其实像这种就是你需要**再花时间去快速熟悉和了解**，找一些比较好的博客，基本一两天就能快速理解，像这种必须补的系统知识还是需要去补充了，至少你**看了论文知道自己还有哪些缺失的知识点和面**，之前去看也没这么**有针对性**。

代码这块可以先不用急，因为你还没怎么看代码。

等你看代码之后再针对代码这块的问题进行梳理和理解

**我：**

是的是的，在有大体知识框架的情况下，读论文后可以针对性地了解知识上点和面的缺失。如果直接看操作系统书之类的可能浪费时间还记不住。

我有记下一些不懂的概念，比如ring buffer，多核cpu，同步之类的问题，正打算google一下，先学一下基本的概念。

在后面再梳理具体代码

**李：**

可以的，而且这个磨刀不误砍柴工，去学习了这些内容再回去看论文又会有很深的新的理解



## 会后总结：下一步:exclamation::exclamation:

* 先看剩余的两篇论文
* **针对性的对缺失的知识进行了解**：学习操作系统的进程和调度部分
* 到时候可以分别复现一篇文件系统的和一篇调度系统的论文，学习如何学习architecture和code
* **别局限于os，开阔知识面**：数据库，网络，硬件等等
* 逐渐**思考自己的思考模式**：比如我怎么会想到问这个问题的
* 看一下**github**有什么东西