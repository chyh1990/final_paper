多核
=========

摘要
=========


Abstract
=========



引言
=========
当前基于AMD64/X86\_64架构的多核系统已经得到广泛应用。但随着40核甚至
80核的NUMA系统的出现，对已有操作系统的多核可扩展性能提出了苛刻的要求。
近年越来越多的研究发现【1,3】，由于现有操作系统在设计上的问题，OS在最新的多核系统中成为瓶颈的情况大大增多。

在造成Linux等现有OS在40到80核多核系统上可扩展性问题的各种因素中，一个核心原是CPU核间数据共享及其竞争。这种竞争体现在多个方面：
* 通过原子操作指令（x86中的Lock前缀）修改内核对象的引用计数
（reference count）Cache Miss/Snoop protocol
* 通过锁操作控制共享数据访问
* etc

受到编译器优化技术的启发，有研究者提出了一种在系统接口（如Posix规范）中发现可扩展潜能的方法[2]。他们称之为”可交换性法则“：只要两个操作的顺序可以交换，那么就可以用多核可扩展的方式实现这些操作。其中的原理是，如果两个操作可以交换（不影响操作结果，因此调用者无法分辨它们执行的顺序），那么他们就可以用不造成读-写冲突的方式实现。

基于这条规律，我们提出了基于模型验证（model
checking）和符号执行的方法，求解Posix标准中系统调用的可交换条件。对于操作系统中对堆（heap）变量和指针操作建模和推理的难点，提出了使用separation
logic 和类似KLEE[5]的解决方法。


操作系统系统调用和中断建模
===========
【TODO 进一步描述commutativty，cache，数据共享的关系】
【Posix interface model 和 实际系统的关系】
【例子：Filesystem， VM】

linux mm/mmap.c:1353

近年来，随着SMT求解器，如STP、MathSAT、Z3的进步，为符号执行和模型验证方法提供了越来越强大的后端支持，使对操作系统等一类高复杂度软件的形式化验证成为了可能。事实上，在OSDI近年的会议上，已有多项针对OS中文件系统子系统的建模工作[[5,6]，但他们的主要着眼点在使用模型验证和符号执行手段分析现有文件系统的Bug和错误上。在文件系统方面，MIT的Commuter[2]的一个创新点是把符号执行与可交换性法相结合，提出了在Posix文件系统API规范中发掘多核可扩展机会的方法。但他们的工作存在如下不足：

* TODO 
* TODO
* TODO

我们的主要工作是把Commuter的思路进一步扩展到OS的其他子系统，通过为操作系统虚拟内存管理、进程管理子系统进行建模，提供一种可以跨OS子系统分析操作系统接口多核可交换性/可扩展性的方法。

首先，我们认为对虚拟内存子系统系统调用建模发现潜在的可扩展性因素越来越有必要。正如文章[1][3]所说，在多核系统上OS的VM子系统对于VM密集型应用（如Java垃圾回收器）上已经成为了瓶颈，大大限制了多线程编程模型的多核可扩展性。通过建模发现Posix
VM API中可扩展性的潜能，以此为基础

另一方面，我们认为对OS的虚拟内存管理系统/管理系统建模是一项比对文件系统建模更有挑战性的工作，有如下两个原因。

第一，文件系统主要以数据结构的更新逻辑为主，对open、read等系统调用建模时可以把底层硬件（如磁盘）抽象为数组。因此已有工作对文件系统相关系统调用都没有考虑硬件的影响。而对虚拟内存子系统建模时，必须考虑硬件缺页中断等硬件相关的影响。

第二，实际操作系统中，对虚拟内存和进程的管理设计大量共享可变数据结构（shared
mutable)，如fork系统调用对物理页的写时复制（Copy-on-Write）策略，信号量的共享等。这种特性与Z3等SMT求解器以函数式语法描述程序约束条件的情况相对立，使得可交换性条件求解变得困难。


下面我们以操作系统中虚拟内存管理(Virtual
Memory)子系统为例，介绍简化的Posix系统调用API接口和中断处理例程的建模方法。
在OS的虚拟内存管理子系统中，有两个最重要的系统调用：mmap和munmap。与文件系统建模不同的是，为了相对完整地描述虚拟内存子系统的行为，模型中还必须包括VM子系统对缺页异常（pagefault
handler）的处理例程。[XXX 如何扩展论述？]

对于内存管理子系统，我们把操作系统对程序的接口分成两类：1)OS提供给用户态程序的系统调用接口，如mmap/mumap系列函数；和2)用户程序访问硬件页表发生缺页中断时，操作系统根据虚拟地址区域的元数据（如Linux中的VMA结构体）填写缺失的硬件页表项。对于由软件填充TLB的体系结构，如MIPS，也可以通过软件模拟页表的方法实现类似功能。

在编写操作系统的过程中，我们发现上述两类操作系统接口可以统一建模。由于在x86_64（或其他大部分体系结构中）缺页异常属于同步异常[10]，可以把读写缺页看做普通系统调用，通过添加如下两个虚拟系统调用对缺页进行建模：

void mem_write(pid, addr, value)
word mem_read(pid, addr)

系统调用
-----------

虚拟内存子系统中主要的用户态接口为mmap和munmap。以下以匿名页（Anonymous
Page）映射为例，介绍VM子系统的建模方法。

【此处用我写的vm.py，我认为对anon page的处理跟MIT他们是等价的，问题不大】

中断
-----------


基于模型的可交换性求解
===========

基本算法
-----------

在对关注的系统调用和中断建模之后，通过符号执行技术，可以求解N个系统调用之间的可交换性条件。对于N=2的情况具体算法如下：

设A(xi)和B(yi)为两个待求可交换性的模型函数，xi, yi为参数列表，返回值为a，b
系统初始状态为sigma0，写为Hoare逻辑的形式：

{sigma0} a = A(xi) {sigma1 a} b = B(yi) {sigma2 a b}

调换A和B的执行顺序

{sigma0} a' = A(xi) {sigma1 a'} b' = B(yi) {sigma2' a' b'}

通过符号执行技术和适当的判定过程，求解使得下面断言满足的条件：

sigma2 = sigma2' and a = a' and b = b'

及其对应的sigma0,a,b 即可得到系统调用A和B之间的可交换条件。'
这意味着在实现这对系统调用或中断的过程中，针对符合上式的特殊情况可以做优化，使得在这种情况下A、B的执行不会发生任何数据竞争现象，因而多核系统上可达到完美的可扩展性。

### 例子（reference counter？）

操作系统模型中求解的难点及解决思路
-----------
### 例子

[spec中fdtable共享例子]

### Separation logic

Separation Logic是对Hoare
Logic的一种扩展，理论上可以用于对使用共享可变数据结构（如Heap上分配的对象）的命令式程序做推导[7]。考虑如下程序：

XXX 多数sep logic论文使用reverse list。我们应该用OS中单向链表使用的例子

形式上Separation Logic定义了Separating conjuction运算符：P *
Q，表示程序的堆（Heap）可分割为两个独立的堆H1、H2，在H1上命题P成立，在H2上命题Q成立。直观上，通过Seperating
conjuction运算符和相关推理规则，简化了Heap上包含共享数据结构程序约束的描述。然而，文章[7]中并未给出Separation
logic的判定过程(decision
procedure)，Z3等SMT求解器未支持此类逻辑的可满足性判定。但是，对于separation
logic的一些特殊情况，学界已经提出了相关的判定过程。例如，对于操作系统中的使用最频繁的链表结构，文章[9]提出了求解下面形式约束的算法，并在Z3
SMT求解器中实现：

[公式 xxxx -> xxxxx]
其中xxx是xxx

### KLEE
KLEE是基于LLVM
bytecode的符号执行引擎。KLEE已经被广泛地应用在文件系统建模，大型开源软件（如GNU
coreutils）漏洞查找[5]等方面。KLEE中使用了一种简单的方法来避开指针和Heap对象约束求解的困难：记录指针可能引用的对象集合，在指针被dereference的时候，如果此指针可能指向N个对象，则fork出N个状态并进入调度队列，之后再继续进行符号执行。状态之间，KLEE使用写时复制（Copy-on-Write）的方式管理内存。有研究这对这种方法做扩展[11]，使




总结
============



Reference

1. RadixVM
2. Commuter
3. An Analysis of Linux Scalability to Many Cores
4. Non-scalable locks are dangerous
5. KLEE
6. Using Model Checking to Find Serious File System Errors
7. Separation Logic: A Logic for Shared Mutable Data Structures
8. Symbolic Execution with Separation Logic
9. Separation Logic Modulo Theories
10. Intel Manual
11. Parallel Symbolic Execution for Automated Real-World Software Testing
