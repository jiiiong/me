# XRP: In-Kernel Storage Functions with eBPF

## 初读感受

这次初读篇论文的感受是非常的困难。

我带着李老师给的建议，带着系统的观点去看这篇论文，将里面涉及到的知识组织起来，形成一个索引表。
首先可以肯定的这是一个很好的方法，但是因为缺乏对索引表中知识最基本的、概念性的理解，所以整个索引表的组织逻辑非常混乱，很多时候知识把设计的知识点强塞进去。
不过我至少知道有哪些知识节点，并且由他们大概的关系，如果没有这个方法，我可能连看过什么都忘了。

**下一步**，我想结合幻灯片重新理一下文章的大结构，总结出一些问题。对于

对于李老师上一次提到的论文中的**方法论**，我不是很清楚是什么意思，是指整篇论文从背景开始一直到实验结论的研究方法论吗。

## 初步：总结

### 软件成为瓶颈

微秒级别的存储设备的出现使得kernel存储栈成为访问存储设备时的巨大开销。例如对optane ssd r5800x其软件开销占到了48%。

### kernel byass

一种消除这种软件开销的方法是kernel bypass，用户空间的软件直接访问存储设备。

但每个应用需要构建自己的FS，不乏共享文件系统，同时因为各自拥有完整的权限，存在隔离的问题，不安全

最后因为无法使用中断，其中能用cpu轮询的方式确定i/o是否完成。造成两个问题

* 当i/o不是瓶颈时，浪费cpu时钟

* 当存在过多线程时，因缺少sync，造成尾延迟严重变差，影响吞吐量

  <mark>可以通过sync操作解决？？？</mark>

### 延迟分析

对i/o读的总延迟进行构成分析，可以得知主要是通过FS和bio时的开销大，nvme的驱动层开销很小。所以作者考虑将**存储函数从用户空间下放到内核的nvme的驱动程序中，从而绕过大部分存储栈的想法。**

<mark>将存储函数offload至内核中可以利用内核天生的隔离和资源调度(共享？？能接受interrupt？？)能力</mark>

### 如何offload，可能的好处

**可能的收益**

考虑on-disk b-树索引结构，他在应用从发起最开始的查询请求后，之后会有许多依赖于上一次i/o的存储访问操作，称这一连串的i/o为i/o链，可以将i/o链offload到驱动上的存储函数实现。

**如何offlload**

具体的如何卸载存储函数到内核的驱动程序中，作者考虑了使用eBPF，作者设计了一类xrp类型的eBPF hook，设计了该类型的上下文，从而支持各种由eBPF prog描述的存储函数。

作者还考虑了将XRP和io_uring结合的情况，考虑到io_uring在单独使用时对**同步**的要求，他们是互补的。

### 实现中的挑战

在具体实现XRP时面临了几个挑战。

* 首先是offset地址所处上下文问题，在驱动层单独的offset是没有意义的，其需要和文件系统结合使用；

* 第二是允许文件系统并发的读和写。

  write只在page cache上有所反应，xrp在驱动层是无法感受到的（感受到了可以实现同步吗？？？）

  另外，改变数据结构的layout的操作也会使得同步出现问题。（即便文件系统的读写能实现同步，也解决不了这个问题啊，需要额外的sync逻辑，好像可以通过metadata digest的RCU实现）

  都可以通过在中断中使用锁解决，但开销很大，是因为获取fs block lock的开销大和阻塞的问题吗<mark>???</mark>

  考虑几个问题

  * 文件系统的读写需要哪些同步，他原来的同步是怎么实现的

    我猜是文件系统的读写sync通过FS block lock来实现

  * 在加入XRP之后，对原有的同步会造成什么影响

    我猜是**resubmission i/o无法被FS感知，需要XRP自己实现与FS发出的i/o和其他bpf存储函数的同步**。这要求XRP系统能够感知这些i/o操作，问题是page cache的存在，使得文件系统的i/o可能无法被XRP感知到，所以无法在XRP层建立sync逻辑。考虑在XRP中直接调用FS层lock实现sync逻辑，但是从interrupt层获取lock开销很大。

    这里又有一个问题，<mark>XRP自己怎么实现同步操作呢</mark>。

  * 如何解决上述影响

    考虑不使用page cache，而是user-defiend cache。会有什么效果呢。顶层的用户可以感知到任何i/o，当从user-defined cache读写时可以通知在驱动层的XRP从而实现同步。

**观察**

他们发现on-disk数据结构是相对稳定的，且index存储在几个大文件，且每个index不跨文件。

**设计原则**

基于这个观察，他们为了解决第一个问题提出几个思路，一次只对一个文件使用xrp，从而减少文件系统metadata的大小。采用相对稳定的on-disk数据结构，同时采用RCU实现metadata digest的同步（一致性），也解决了上述第二个挑战中的layout变化问题。

对于第二个问题，他们使用user-defined cache。因为XRP缺少对page cache的感知，如果块被page cache缓存，那么同步读写就会出错。所以将cache交予用户管理，在访问cache时让用户可以通知XRP，从而保证同步。同时会重新提交i/o rq如果原先的失败的话。

### 具体实现

解决了上面这两个问题，xrp系统的**重提较逻辑**是，在i/o请求（read_xrp）返回的中断处理函数上下文中处理完对应的irq，<mark>此时处理bpf prog的上下文，共享此时进程的上下文？？？</mark>，然后马上激活对应的bpf函数。其中的文件系统的metadata digest是共享的，由RCU保证并发访问。

**eBPF hook**

其中hook 上下文的各个域，data，done，next_key, next_addr，scratch。前四个域是在中断处理程序中使用的，最后的scratch是用户层用户和BPF函数私有的。

**verifier**

修改了verifier，使得XPR的is_valid_access可以传递buffer大小来执行越界检查

**metadata digest**

在file system上，由两个函数接口，与中断处理程序共享logical-to-physical mapping，inode address，update更新这个共享，lookup进行地址翻译，同时采用访问控制。具体的实现方式有多种，cache版本和直接调用文件系统的函数等。<mark>对文件系统的并行读写是怎么处理的，话说现在的存储器支持同时读写吗？有没有可能两个i/o操作的顺序直接变反</mark>

**resubmit NVMe Request**

最后的形成NVMe request重用request structure，这是安全的，因为无论用户还是BPF prog无法访问这个结构，重用导致fanout的个数有限制，可以在首次I/O申请中设置dummy irq。

#### **同步**

<mark>XRP的设计还考虑了同步的问题，这一部分不太明白，考虑一下几个问题</mark>

* 需要同步什么操作
* 为什么要同步，不同步会怎么样
* 同步的方法由什么

**interaction with kernel shceduler**

同时注意到XRP会和Linux的调度系统产生综合反应，其中包括CPU调度和I/O调度

### 案例研究

在设计完XRP系统之后，便是修改现有的存储系统进行案例研究。分别设计研究了BPF-KV和WiredTiger，考虑了BPF程序、用户cache兼容、接口修改

### 实验测试

<mark>对于测试，我认为需要明确有哪些影响因素，有哪些受影响的性能点，通过控制变量来研究</mark>

回答几个问题

* overhead在哪
* 扩展到多线程的效果
* XRP支持什么操作
* 能否加速现实世界的系统

首先对BPF-KV进行延迟，他停用了cache从而专注于on-disk结构lookup的效果。

* 单线程的延迟提升效果
* 多线程的延迟效果，分析了最差延迟，发现99.9百分位数的SPDK因为busy-polling导致效果很差
* 进一步研究了哪些高延迟操作的所占的百分比

进一步研究了吞吐量，单线程不同深度，固定深度不同线程数

还研究了可扩展性，看在wordload达到nvme设备极限时，不同线程数的表现，同时还研究了不同吞吐量下的延迟表现

最后研究了XRP在现实系统中对吞吐量和尾延迟的影响

对未来XRP的发展方法，指出了三个点：

* 支持更多类型的用户自定义函数
* 作为一个将任务卸载至设备附近的通用接口
* 与XDP结合，从而同时绕过内核的存储和网络栈

### 初步：下一步

总结要交流的内容，定位、明确阐述读论文遇见的主要问题（同步、数据分析）。

## 问题

* bypass中需要polling for completion结合cpu调度对系统的影响。

  ans：突然懂了，需要对进程调度系统有充分的理解

  多核cpu也需要了解，多核的并行到底是怎么样的

* 数据分析

* 同步

  ans：明白了page cache和数据结构layout的影响和解决办法。

  * 但是XRP具体如何使用这些东西达到与FS的i/o同步、bpf 程序间同步，on-disk数据结构同步还是不知道。

## 总结

关于sync问题，李老师认为，**page cache无法被XRP感知，所以只能通过锁来实现同步**。或者不使用page cache，而是使用**user-managed cache，如此用户就可以对其有所感知**，就能够实现同步。

怎么说呢，逻辑上或者说定性上是理解的。但是更加具体的理解还是存在几个问题：

* 究竟什么操作需要同步
* 在使用page cache的时候，怎么使用lock来进行同步

我尝试看具体的代码，但是发现**内核部分的代码**和**eBPF部分的代码**都看不太懂。

### 10/14

XRP（eXpress Resubmission Path）其实对整个访问NVMe设备的path进行了修改，并在NVMe interrupt handler中expose了一个hook允许eBPF prog的挂载。

## 下一步计划 10/13

* 先学习eBPF Subsystem：[BPF & XDP Reference Guide](https://cilium.readthedocs.io/en/stable/bpf/)，再接着看下一篇论文
* 对于这里的同步问题，可以以后再考虑
* 解决几个问题
  * RCU
  * 多核CPU，Linux的CPU调度系统以及其对中断的处理
  * tail latency和throughput的关系
  * 并发同步问题
* 我尝试看具体的代码，但是发现**内核部分的代码**和**eBPF部分的代码**都看不太懂。

---

# 会议

## 会前总结

这篇论文简单来说就是把用户空间的存储函数offload到内核中靠近NVMe设备的NVMe驱动层，从而绕过大部分内核存储栈。

### 论文整体逻辑

首先提出软件部分已经成为访问高速存储设备的瓶颈。

通过研究访问高速设备的延迟构成，说明kernel bypass的缺陷，进一步提出将用户存储函数offload到内核中的想法。

然后提出了offload可能的好处，比如可以在驱动层重提交有依赖的i/o链，并提出借助eBPF进行offload。

在具体实现时，提出了设计的两个挑战，分别是**地址翻译缺少文件系统的上下文**，和**带XRP文件系统的并发问题**。

对于第一个问题，作者提出了metadata digest。作者发现许多存储引擎的文件保持相对稳定，且每个index不会跨文件。因此提出metadata digest要遵循每次只缓存一个文件的metadata，且要面向稳定的数据结构。如此维护metadata digest的开销就不会太大。

对于第二个挑战，理解上还存在问题，下面再讨论。

在设计好XRP后，作者结合XRP进行了两个案例研究。然后展开实验测试研究了XRP的开销在哪，多线程可扩展性如何，XRP可以支持哪些存储函数，对现实产品效果如何。

### XRP并发问题

存储函数在XRP处会因为resubmitted I/O展开成一系列对NVMe设备的读写操作，相当于一系列文件系统发出的*read*和*write*。为保证存储函数的正确性，需要考虑存储函数和文件系统发出的*read*和*write*，存储函数和存储函数之间的并发存在同步问题。

---

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221011205507981.png" alt="image-20221011205507981"  />

P5

**个人理解**

带XRP的文件系统由*xrp_read*发出的*read*会因为resubmitted i/o被展开成一系列NVMe read request。在这个*read*期间，如果文件系统发出*write*请求在到达NVMe驱动层的时候，就需要XRP进行同步，例如保证*write*请求不会出现在一系列resubmitted i/o的中间。

但因为page cache的存在，文件系统的*write*操作可能根本无法被XRP所感知，最就会造成如下错误：

* 修改了NVMe request请求的块的内容
* 修改了NVMe request请求块的layout

可以使用锁解决这个问题，但从NVMe中断处理程序中访问锁开销很大。

<mark>**问题1：**</mark>这里的锁是指调用文件系统的锁吗？**如何理解为什么说开销很大？**为什么是从NVMe中断处理程序中调用锁，怎么调用才能实现并发。

<mark>**问题2：**</mark>考虑不存在page cache的情况，*write*直接到达driver，那么XRP怎么协调*write*和存储函数？

---

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221012121622736.png" alt="image-20221012121622736"  />

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221011205524687.png" alt="image-20221011205524687"  />

P6

<mark>**问题2：**</mark>怎么理解这句话。是指某些存储函数在遍历或者迭代的过程中可能会存在对device block的写操作，然后就需要通过lock来实现与其他存储函数，文件系统发出的*read*和*write*同步吗？

---

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221011205623714.png" alt="image-20221011205623714"  />

P6

<mark>**问题3：**</mark>作者提出使用user-managed cache，这时XRP能感受到对cache进行的*write*操作吗，操作会影响xrp_read吗？

观察BPF-KV和WiredTiger的cache策略和存储函数，我发现存储函数只对那些没有缓存的blocks进行操作

---

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221011214713016.png" alt="image-20221011214713016"  />

P8

<mark>**问题4：**</mark>这里指是要同步什么东西，是指**问题2**中的存储函数和直达NVMe驱动的*write*之间的同步吗？

### 实验设置与数据分析

这一方面的知识比较欠缺。

实验设置的一个朴素的想法是分别明确变量和因变量，通过控制变量观察其对因变量的影响。

该论文实验中的变量就有workload、线程数、index深度，因变量有延迟和吞吐量。

## 会中记录

系统和逻辑

page cache无法被XRP感知，可以借用锁实现同步

什么锁：eBPF的自旋锁，用户空间的锁

使用user-managed cache，用户可以自己控制

## 会后总结

李老师认为，**page cache无法被XRP感知，所以只能通过锁来实现同步**。或者不使用page cache，而是使用**user-managed cache，如此用户就可以对其有所感知**，就能够实现同步。

怎么说呢，逻辑上或者说定性上是理解的。但是更加具体的理解还是存在几个问题：

* 究竟什么操作需要同步
* 在使用page cache的时候，怎么使用lock来进行同步

我尝试看具体的代码，但是发现**内核部分的代码**和**eBPF部分的代码**都看不太懂。

### 下一步计划

先进一步了解eBPF的实现原理

