## IO映射

通常Linux对所有计算机平台上都实现了IO端口，包括其中单一地址空间的cpu在内。但是即使外设总线为IO端口保留了分离的地址空间，但也不是所有设备都会把寄存器映射到IO端口上，ISA设备普遍使用IO端口，但是大多数PCI还是会把寄存器映射到某个内存地址区域。

**说在前面**

对IO寄存器的读写和对内存的读写有一个问题，就是要避免访问IO寄存器时由于CPU或者编译不恰当的优化而改变预期IO。因为读写内存时系统对应代码的优化使得指令可能会被重新排序，这个对于IO读写来说会发生一些奇怪的错误。需要使用到下面的宏：

```
#include<linux/kernel.h>
void barrier(void)
#include<asm/system.h>
void rmb(void);
void read_barrier_depends(void);
void wmb(void);
void mb(void)
```

这些宏的作用是在编译的时候隔离内存

#### 使用IO端口

在独占某个端口的时候，驱动程序必须声明需要操作的端口，这个接口的核心函数是request_region

```
struct resource *request_region(unsigned long first, unsigned long n, const char *name)
```

这个函数表明了我们使用了从first开始的n个端口。name指的是设备名。如果不再需要某组端口的话，调用：

```
void release_region(unsigned long start, unsigned long n)
```

存在一个检查端口集可用与否的函数是：

```
int check_region(unsigned long first, unsigned long n)
```

但是需要注意的是检查一个端口是否有用这个函数并不是原子操作，所以不建议使用。

接着拿到了端口就可以开始读写了，可用使用inb，outb，inw，outw，inl，outl来实施。当然有的平台也支持使用串操作来传输数据序列，可以使用函数insb，outsb，insw，outsw，insl，outsl

#### 暂停式IO

如果处理器时钟比外设时钟块，设备特别慢，则需要让主机等待设备处理数据，如果不想设备丢失数据或者其他问题的出现可以使用暂停式IO，这种暂停式IO通常以_p结尾。写IO时记得不同的平台使用的函数可能略微不同。

### IO内存

除了IO端口的使用之外，还有一种和设备通信的方法就是通过使用映射到内存的设备或者设备的内存。

在使用之前首先分配IO内存的区域，分配内存区域的函数入下：

```
struct resource *request_mem_region(unsigned long start, unsigned long len, char *name)
```

这个函数的目的就是从start开始分配len长度的内存区域。那么释放它就需要

```
void release_mem_region(unsigned long start, unsigned long len)
```

但是在许多系统上并不能使用这种方法直接访问内存，需要首先建立映射，使用ioremap的方法可以为IO内存区域分配虚拟地址。那么由ioremap返回的地址不能直接引用，而是应该使用内核所提供的accessor函数。当我们完成了映射后，就可以开始读写了，比如使用ioread8，ioread16，ioread32，iowrite8……，如果还需要在某一块内存上操作，则可以使用memset_io,memcpy_toio等。

但是某些硬件十分有趣，他们有时使用IO端口，有时使用IO内存。那么为了方便编写驱动程序，linux引入函数ioport_map：

```
void *ioport_map(unsigned long port, unsigned int count)
```

然后可以使用

```
void ioport_unmap(void *addr)
```

来销毁。



