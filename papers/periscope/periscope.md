## PeriScope: An Effective Probing and Fuzzing Framework for the Hardware-OS Boundary

大多数的内核攻击都在系统调用边界上，但是还有另外一种不需要借助系统调用的方法，就是通过攻破外部设备固件来进行攻击，因为许多驱动程序都是跑在内核空间上的。本篇文章着眼于攻击者从设备驱动入手，因此这是一个探测分析设备驱动的框架。

攻击的方法就是远程设置这个设备，然后使这个设备生成一些指定的输出，这些输出再去触发驱动程序的一个漏洞。作者设计了一个perifuzz的工具用来取得MMIO和DMA的读权限，然后修改读到的值。它不仅可以修改读取不同的地址的值，还能修改读取同一地址的值。

首先必须知道外部设备和操作系统怎么进行交互：

1. 中断：处理这个的方法是：外设会在某一跟中断线上发送一个中断请求，CPU会响应请求并且关闭这条中断线上的所有中断，此时CPU进入这个中断处理例程。注意中断处理历程的顶部和尾部之分。
2. MMIO：CPU会开辟一篇内核空间的虚拟地址来和外设进行映射，这样CPU可以使用平常一样的读写方式进行操作。
3. DMA：允许外设直接访问物理地址空间。

现在系统会使用一种叫做IOMMU的组件限制外设能够访问的物理地址，它是将外设所看到的虚拟地址转换为物理地址。



periscope的做法是在驱动程序这一端来关注数据流动的信息，因此它会在内核缺页例程机制上装上钩子函数，具体做法如下：

1. 当设备驱动程序建立了一个MMIO或者DMA映射，它会自动检测到并且进行注册；
2. 分析者需要指定某一个映射来进行监视。
3. 它将映射中的某一页标记为不在内核的页表中。
4. 当cpu访问这个被标记的页，则触发缺页。



perifuzz的目的是构造驱动程序的输入。

IOMMU/SMMU 保护：大部分的驱动程序信任外设使得它们能够直接连接物理地址，所有使用了所谓的IOMMU进行隔离。

perifuzz由三个模块组成：fuzzer，executor，injector。fuzzer运行在用户空间，它的目的是生成随机的输入以及处理执行的反馈结果，本文使用的是AFL作为fuzzer；executor的目的是生成一个输入文件给injector使用，通过的是地址映射的方式，它还负责记录内核崩溃前的数据；这个是periscope的一个接口，就是注册了一个pre-hook函数，监视着读取的数据，如果情况合适则复写目标寄存器中的值。需要注意的是injector从executor拿到的数据是一定是上一次每做过的数据，这样有助于发现overlapping fetches引起的double fetch bugs。

We insert these hooks into the dma_alloc_coherent and dma_free_coherent functions to track coherent
DMA mappings, into the dma_unmap_page function3 and dma_map_page to track streaming DMA mappings, and into ioremap and iounmap to track MMIO mappings.

用户空间使用debugfs和tracefs来监视数据。

发现的错误类型有：buffer overﬂows, address leaks, reachable assertions, and null-pointer dereferences，没有在流状DMA中检测double fetch，而在集成DMA中检测到了。

排除重复的问题后无法探测后面可能会发生的一些问题。系统的内部状态可能会影响到fuzz；	

