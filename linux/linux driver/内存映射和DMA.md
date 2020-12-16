## 内存映射和DMA

首先知道在linux处理内存的函数趋向使用指向page结构的指针，这个结构用来保存内核中需要知道的所有物理内存信息。

虚拟内存区VMA用于管理进程地址空间中不同该区域的内核数据结构，进程的内存映射包含程序的可执行代码区域，多个数据区和活动的内存映射。

### mmap设备操作

不是所有设备都可以使用mmap进行抽象，mmap的使用必须是以page_size为单位映射。mmap方法是file_operations结构的一部分，然后执行mmap系统调用时会调用该方法。

```
int (*mmap) (struct file *filp, struct vm_area_struct *vma);
```

vma就包含了访问设备的虚拟地址信息，要使用该函数，则需要为这个地址空间建立一个页表，使用如下函数：

```
int remap_pfn_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long pfn, unsigned long size, pgprot_t prot);
int io_remap_page_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long phys_addr, unsigned long size, pgprot_t prot)
```

如果对于内存的创建还需要更好的灵活性，则提倡使用VMA的nopage方法。

```
struct page *(*nopage) (struct vm_area_struct *vma, unsigned long address, int *type);
```

### 直接IO访问

有的时候直接对用户空间缓冲区执行IO操作也可以，如果需要传输的数据量非常大，直接进行数据传输，会大大提升速度。但是设置直接IO的开销非常大，比如必须和write系统调用同步执行。内核中实现直接IO的关键函数为get_user_pages函数,：

```
int get_user_pages(struct task_struct *tsk, struct mm_struct *mm, unsigned long start, int len, int write, int force, struct page **pages, struct vm_area_struct **vmas)
```

当直接IO操作完成后如果改变了某些页中的内容，就必须置位所有修改过的页

```
void SetPageDirty(struct page *page)
```

然后将页释放：

```
void page_cache_release(struct page *page)
```

#### 异步IO

有三个用于实现异步IO的file_operations方法：

```
ssize_t (*aio_read) (struct kiocb *iocb, char *buffer, size_t count, loff_t offset);
ssize_t (*aio_write) (struct kiocb *iocb, const char *buffer, size_t count, loff_t offset);
int (*aio_fsync) (struct kiocb *iocb, int datasync)
```

## 直接内存访问

DMA是一种硬件机制，允许外设和主内存之间传输它们的IO数据。

### 分配DMA缓冲区

这个缓冲区是为了暂存传输的数据，其中最大的一个问题是当传输的数据大于一页时，它们必须要占据连续的物理页。一些设备和一些系统中的高端内存不能用于DMA，这因为外设不能访问高端内存的地址。

使用DMA的设备驱动程序将与连接到总线接口上的硬件通信，硬件使用的是物理地址，程序是使用的是虚拟地址。而基于DMA的硬件使用的是总线地址。则在有的系统中要使用如下函数进行转换：

```
unsigned long virt_to_bus(volatile void *address);
void *bus_to_virt(unsigned long address);
```

DMA必须解决缓存一致性。根据DMA缓冲区期望保留时间长短，PCI代码区分为两种DMA映射：

* 一致性DMA映射：一致性DMA映射必须可以同时被CPU和外围设备访问。
* 流式DMA映射：通常单独的操作建立流式映射。

建立一致性的DMA可以调用pci_alloc_consistent，当不再需要缓冲区时，调用dma_free_coherent函数；而建立流式DMA映射时，必须告诉内核数据流动的方向，当只有一个缓冲区要被传输的时候，使用dma_map_single函数映射它。然后当完成传输后需要删除dma_unmap_single删除映射。流式映射有几个原则：缓冲区只能用作给定方向的传输；一旦缓冲区被映射，它将属于设备不是处理器的了；DMA处于活动期间不能撤销映射。

