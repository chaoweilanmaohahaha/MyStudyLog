# Digtool

本篇文章是对闭源操作系统windows系统进行漏洞挖掘，使用的技术是虚拟化监视。

关于漏洞检测的方法大致分为两类：路径探索和漏洞识别。像使用的AFL这种工具是典型的路径探索。漏洞识别就是记录探测到的路径上的异常情况。这篇文章主要针对的是漏洞识别技术。并且本文针对的是内核漏洞而不是应用程序的漏洞。目前与市场上的大多数用户漏洞识别的工具不同的是，市场上的工具针对的是开源的操作系统，依赖的是其中的具体实现。所以作者想到了使用虚拟技术，它就可以支持多种操作系统。但是市面上使用该技术的系统又只是针对单一的漏洞。对于闭源的操作系统的难点在于我们无法在编译时插入检测的代码，当然也不能直接改写内核代码。

本检测工具主要的对比对象是windows平台上的一个漏洞检测工具Driver Verifier，这个工具可以检测一些恶意函数被调用从而组织系统的崩溃，但是这个工具是平台上一个固有的工具，并且能够检测到的漏洞也很有限。

本文针对如下四种漏洞：

UNPROBE：对于一个输入缓冲区，不去校验用户指针，着就会导致类似非法地址指向，非法读写。

TOCTTOU：我认为就是所谓的double-fetch，从一片内存区域抓取两次数据，通常是检查一次用户指针，然后再用。

UAF：使用了系统中已经释放的区域

OOB：我认为就是缓冲区溢出，也就是进入了超出分配堆栈的边界。

## overview

hypervisor:被Hypervisor用来执行一个或多个虚拟机器的电脑称为主体机器（host machine），这些虚拟机器则称为客体机器（guest machine）。hypervisor提供虚拟的作业平台来执行客体操作系统（guest operating systems），负责管理其他客体操作系统的执行阶段；这些客体操作系统，共同分享虚拟化后的硬件资源

这里有个问题是在于内存的监控方面，对于闭源的操作系统我们没法在源代码中植入一些方法来监控内存的分配情况。并且我们要知道一点，对于每一个程序，进程他们都跑在自己的虚拟地址空间上，在windows平台就很难去监控物理地址空间的变化，因为在windows平台上他们的映射并不是线性的。这里用的是监控shadow page table(**shadow page table** mirrors what the guest is doing in terms of its own **page table**s and in terms of what the VMM translates the guest physical address to the host physical address)

整体的digtool涉及到三个部分：用户空间、内核空间和虚拟机

虚拟机就是将操作系统和硬件分离了，它也可以被称为VMM，虚拟监视器，可以访问磁盘内存等硬件设备，介于操作系统和硬件之间。虚拟机的目的是监视虚拟地址的入口，需要有一种机制从外部跟踪内存入口。使用SPT影子页表来进行监视。这个是他们自己开发的，一共有三个部分：VMM infrastructure、interface detection、memory detection。这个VMM infrastructure做一些初始化的工作，需要获取硬件环境和OS版本,并将原始的操作系统载入VM，interface detection 监视系统调用执行时从用户程序中输入的参数。这个监视可以量身定制。memory detection 监视内核内存使用情况。它可以设置监视区域以及设置监视目标。

内核空间主要做的事设置需要监视的区域，这需要和虚拟机进行交互，并且可以截获内核函数。实现是一个中间件，这个中间件在interface detection的作用是记录了所有的事件日志。这样可以用log analyzer分析日志。对于memory detection通过hook一些指定的与内存有关的函数来指定j监视内存区域。如果漏洞被找到，中间件记录并打断客户机OS通过单步调试或者软件中断，客户机操作系统会连接到一个debug工具。

用户层有一个loader，一个fuzzer和一个log analyzer，loader用来激活虚拟机，开启用户进程，然后触发fuzzer，然后将探测到的执行路径记录在log analyzer中。

### detail

VMM infrastructure：在完成初始化后将OS加载到一个虚拟机中，然后让虚拟机去监视这个。

* 虚拟页表监视：digtool使用SPT来监视，并且只监视对应的线程。使用BItMAP，每个bit代表某一个页表，如果某一位为1，则该页是要被监控的，则在SPT中将PTE中的P标志清除，触发一个缺页中断。只要有一个缺页中断产生，则检查该页在bitmap中的位置，如果是0则只要更新SPT；如果是1，则handle模块会处理这个中断例程，它会记录该中断信息，然后注入一个中断，这个中断会保存进入的地址信息和缺页中断的信息，然后连接到一个单步调试程序，并且设置MTF或者TF，通过单步调试程序可以不断监视页表的信息。
* 线程监视：因为只关注需要监视的线程， KPRCB 结构会包含某个处理器上的所有运行线程，可以这么得到FS-> KPCR-> KPRCB->CurrentThread.
* 内核和虚拟机的交互：一个是内核向虚拟机发出请求，虚拟机给与服务；二是虚拟机给内核消息，内核来处理。前者就是使用service interface，后者是shared memory。The service interfaces are implemented through a VMCALL instruction, which will trigger a VMEXIT to trap into the hypervisor. Thus, the service routines in the hypervisor can handle the requests. 后者需要虚拟机向共享的内存写入一些数据，然后内核从中拿到数据。

漏洞检测：

* 接口检测会监视运行情况，记录特征。他通过定义一些事件并且截获这些事件来执行，一共有十种事件：Syscall, Trap2b, Trap2e, RetUser, MemAccess, ProbeAccess, ProbeRead, ProbeWrite, GetPebTeb, and AllocVirtualMemory。前三个分别监视windows中的fast system call，0x2b和0x2e。当截获正在进入用户y页时，认为时RetUser事件。那么监视进入用户内存的事件记为MemAccess，截下来三个是检测用户地址是否被检查，ProbeAccess不能被hook得到，因为这在硬编码比较地址，则需要通过cpu仿真器去监视cmp指令中使用的寄存器。
  * UNPROBE的检测：探测在MemAccess之前是否有Probe***的事件，这个过程需要fuzzer和log analyzer的配合。
  * TOCTTOU的检测：如果两次MemAccess获取了同一个区域的数据，我们需要判断是否这事件在同一个调用中进行。
* 通过内存足迹来检测漏洞：就是检测内存的分配释放和进入。Illegal memory access will be captured by its page-fault handler in the hypervisor。为了跟踪分配的和释放的区域需要hook一些函数
  * UAF的检测:hook一些释放内存的函数,检测是否在这片区域还没分配之前已经进入.
  * OOB的检测:用平衡二叉树来保存分配的内存情况.