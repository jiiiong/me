# [BPF & XDP Reference Guide](https://cilium.readthedocs.io/en/stable/bpf/)

## eBPF再理解

Linux的许多subsystems使用eBPF，需要修改这一部分的内核源码。

cBPF：classicBPF

Cilium在其<mark>data path</mark>中重度使用eBPF？？？

BPF architecture：不仅仅是虚拟机，一套指令集。在其上还有一堆infrastructures，如maps，helper functions，**tail calls**，**security hardening primitives**，**object pinning**，**infrastructure for allowing BPF to be offloaded to, e.g. NIC**。

<mark>比如XDP，为BPF在networking subsystem中进行的修改会不会影响原有的性能</mark>

## eBPF architecture

### 指令集：

* 将指令放在内核中执行的优势：
* event-driven
* 通用RISC指令集，用于编写C的一个子集
* 11个64b寄存器，1个pc，512B BPF stack<mark>BPF虚拟机？？？</mark>
* calling convention：r0，r1-r5，r6-r9，r10
* context，每个BPF prog type不一样，作用类似于函数的输入
* 操作通常为64位
* 最大指令数per prog
* instruction encoding

### helper functions

一组由core kernel定义的函数，可以用来在内核中存取数据

* 不同bpf prog type可用的helper funcs可能不一样
* call convention同上
* <mark>定义为宏</mark>，方便JIT编译和verifier检查，使helper function扩展更容易

### maps

* key-value store
* 用户空间用fd访问，ebpf程序也能访问
* 分为generic和non-generic
  * generic：可以用一套公用helper funcs来进行lookup之类的操作
  * no-generic：不适合于用唯一的helper func来实现，因为在执行期间有额外的（no-data）state<mark>？？？</mark>

### object pinning

* bpf prog和maps都作为内核资源，只能通过file descriptor访问。但fd会因为进程的结束而消失

* 所以实现minimal kernel space BPF file system来暂存fd

### tail calls

* 和函数调用不同，只使用jump，复用相同的栈
* 每个bpf prog使独立检查的，因此需要通过maps或者ctxt中的一些域进行链接
* 使用方法：
  * maps：BPF_MAP_TYPE_PROG_ARRAY
  * bpf_tail_call()：inline
  * 如果没找到对应的fd，会继续执行原来的bpf prog

* 用处：例如改变包过滤的表现

**BPF2BPF call：**

在BPF里面调用BPFsub prog

* 以前要用inline，现在能用函数的形式实现了。对指令cache有好处
* sub prog和tail call的混用需要内核5.9，并且存在限制

### JIT

* 不支持的架构需要用<mark>interpreter</mark>

### hardening

* read-only image：
* JIT constant blinding: trade off performance
* syscall

### offload

直接卸载到硬件。

## development environment

### LLVM

BPF back end compiler

## 下一步计划

* [BPF & XDP Reference Guide](https://cilium.readthedocs.io/en/stable/bpf/)的剩余部分需要继续看。
* 剩下的3篇论文