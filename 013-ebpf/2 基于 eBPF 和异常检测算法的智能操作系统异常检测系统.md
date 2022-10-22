# 2 基于 eBPF 和异常检测算法的智能操作系统异常检测系统

https://gitlab.eduxiji.net/CH3CHOHCH3/project788067-124730/-/tree/master

使用**BCC**

### 整体结构

**数据收集模**和**异常检测算法模块**

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221003164408219.png" alt="image-20221003164408219" style="zoom:50%;" />

### 数据采集模块

基于eBPF，使用BCC实现。

BCC使用Python和Lua作为前端，LLVM用作后端，C eBPF prog以字符串形式嵌入到程序中

#### 功能

* 系统资源监控
* 进程执行追踪
* 自定义模块加载

#### 总体设计

<img src="C:\Users\z9911\AppData\Roaming\Typora\typora-user-images\image-20221003164337783.png" alt="image-20221003164337783" style="zoom:50%;" />

#### 运行模块 

* 输入处理：命令行  --》 配置文件

* 装载功能模块
* 启动主循环

#### 插件模块

设计了统一前端的接口，和各自的eBPF后端

* xxx_generate_prog：动态采集eBPF prog
* xxx_attach_probe：挂载，不同的挂载点type的挂载方式不同
* xxx_print_header
* xxx_record

#### 测试

结合静态分析工具，可以分析程序异常的原因

### 异常检测算法模块

暂时没看

## <mark>问题</mark>

* 对hook自身的原理完全不了解： 好像和event有关
* 对bcc如何实现对不同类型hook point的挂载不了解

## 思考

eBPF的运用**需要充分理解Kernel、其中的data structure、kfuncs**，例如测试进程占用CPU时间的eBPF prog挂载点的合理选择、进程时刻的获取等。

## 下一步计划 10/3

学习eBPF Subsystem：[BPF & XDP Reference Guide](https://cilium.readthedocs.io/en/stable/bpf/)
学习BCC：[Learn eBPF Tracing: Tutorial and Examples](https://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html)
10/4-10-12 四篇论文

# 会议

### 会前总结

#### 进度

目前处于四步走的第一步当中，也就是eBPF技术的学习和经典论文的阅读上

我首先花了大概两天时间看了ebpf.io上的what-is-eBPF和一个演讲，eBPF：The next power tool of SRE’s。
总结下来，我个人对eBPF Subsystem的理解是：他是一个可以hook到内核或用户层应用各处以（受限）特权模式运行的沙盒虚拟机。

对于使用这个eBPF子系统的来说，eBPF子系统主要分为两部分。第一部分是内核空间部分，主要包括eBPF程序和eBPF maps。第二部分是用户空间部分，主要通过syscalls实现与内核中eBPF程序的交互。

然后就是关于上层的软件，bcc、bpftrace和cilium我认为就是将上述两个部分结合，并向上提供抽象接口的工具。

当时我遇到的问题是，我在思考我到底要要研究eBPF的哪一方面，

* 直接基于eBPF Subsystem开发应用
* 还是基于bcc，bpftrace，cililum开发高性能的软件？
* 研发类似于bcc，bpftrace之类的框架？

基于这个问题，我选择看一下操作系统比赛的作品和四篇论文，看他们是如何利用这个技术的。

因为操作系统比赛作品的文档和代码比较详细，我花了一天时间大概看了这一部分，然后发现他们主要基于BCC实现其中的数据收集模块。

另外我还了看一下OSDI 2022年的Best paper的abstract

#### 思考

结合这两部分我有一个思考就是，对eBPF的运用对内核代码和具体领域有充分的了解。

比如操作系统比赛的那个系统，他在测试进程占用CPU时间时hook点的选择上就需要充分理解内核中进程切换这一部分的代码。还有OSDI 2022年的best paper指出对于NVMe 设备，内核中通过存储栈对其访问已经成为了巨大的开销，所以其利用eBPF绕开存储栈。

所以我认为，前期单纯地学习eBPF技术是简单的，后期对领域的选择、了解和问题的发现才是难点所在。

#### 下一步计划

下一步我想先花8天时间看一下四篇论文，看一下其是如何运用eBPF技术的。

另外因为我现在虽然对eBPF有了概念性的理解，但对实际的接口和代码还是完全不熟悉的，所以我打算看一下what-is-ebpf中further reading给出的一篇bcc教程和有更多技术细节的eBPF的参考指南。

#### 问题

* 研究eBPF的哪一方面:heavy_check_mark:
  * eBPF技术在内核代码中的实现原理？
  * 直接基于eBPF Subsystem开发应用
  * 还是基于bcc，bpftrace，cililum开发高性能的软件？
  * 研发类似于bcc，bpftrace之类的框架？
* 对eBPF技术的学习有没有什么建议，从bcc入手，或者说学习具体的eBPF子系统中各种syscalls
* 对我的下一步计划有没有什么建议:heavy_check_mark:
* 如何了解各个领域有无什么指导性的意见:heavy_check_mark:

### 会中记录

系统的理解，新的路。

* :heavy_check_mark:第一步：基于subsystem的优化

  第二步：bcc

  总体来看

  

* :heavy_check_mark:索引表，工具书

* :heavy_check_mark:怎么利用一个新的方法论，在传统的系统结构上，构造一个相对新的系统。

* :heavy_check_mark:系统的思维

### 会后总结

* 对于研究eBPF哪一方面的问题，不要割裂开来看。我们是为了优化系统而使用eBPF技术，各种层次的技术都会用到。例如

  * 第一步：直接基于subsystem对系统进行优化
  * 第二步：将功能集成为上层框架，如此我们可以以一个新系统的方式看到使用了eBPF技术的旧系统

  <mark>还不是很理解，先看一下论文看他具体是怎么运用eBPF的，是怎么样一个**方法论**，其如何在传统的系统结构上，构造一个相对新的系统：bypass???</mark>

* **系统的思维**：即要总体来看，首先关注系统间各模块的功能、与其他模块的关系，模块的具体内容在需要时再去深入。

* 以系统思维对**知识体系**构建一个**索引表**。这要求我们**视野要广，知识体系要宽阔**，简单了解各种知识模块的内容，了解知识模块间的关系，而绝不是不是局限于某一具体知识的研究。

总结两点：**系统思维**和找出eBPF论文中运用的**方法论**





