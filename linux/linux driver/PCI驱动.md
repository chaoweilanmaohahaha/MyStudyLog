

# PCI驱动程序

设计PCI架构一共有三个主要目标：获得在计算机和外设之间传输数据时更好的性能；尽可能的平台无关性；简化往系统中添加和删除外设的工作。

一个PCI外设由一个总线编号、一个设备编号和一个功能编号组成，允许单个系统拥有256个总线，每个总线上又能接上32个设备。

每个外设板会对下面三种信息进行应答：内存位置，IO端口，配置寄存器。**什么是配置空间，需要另外了解一下！**

### 引导

PCI设备上电时，硬件保持未激活状态，此时促会有内存和IO映射到计算机的地址空间，也不会有中断。但是每个PCI中都有处理PCI的固件，当系统引导时固件执行配置事务，这样方便为它提供的每个地址区域分配一个安全位置，这样驱动程序访问设备时，内存和IO已经映射到处理器地址空间了。

### 配置寄存器与初始化

PCI设备都有至少256个字节的地址空间，前64字节是标准化的，在PCI配置寄存器中，有一些是必需的，一些是可选的，可选寄存器依赖外设的实际功能。**虽然设计与体系结构无关，但是编写的时候还是要考虑到字节序的问题。**寄存器的说明要另外查找资料了。结构体struct pci_device_id结构体用来定义该驱动程序支持的不同类型的PCI设备列表。需要使用以下两个宏来初始化：

```
PCI_DEVICE(vendor, device)
PCI_DEVICE_CLASS(device_class, device_class_mask)
```

这个pci_device_id结构体需要导出到用户空间，这样系统要知道什么模块针对什么硬件设备：

```
MODULE_DEVICE_TABLE(pci, i810_ids)
```

这个语句会创建一个__mod_pci_device_table的局部变量，指向pci_device_id数组，在内核构建上depmod程序在所有模块中搜索该符号，然后将数据从模块中抽出，存放到modules.pcimap，这样当内核告知系统新的PCI设备已经被发现时，系统通过这个文件来搜寻要装载的合适的驱动程序。

### 注册PCI驱动程序

注册驱动程序主要是要创建struct pci_driver结构体，其中必须要定义的有四个成员，name，id_table，probe，remove。要让这个结构注册到PCI核心中，需要调用pci_register_driver函数，而卸载时则需要调用pci_unregister_driver函数。

### 激活PCI设备

在驱动程序访问PCI设备的设备资源之前，驱动程序必须调用pci_enable_device函数来唤醒设备：

```
int pci_enable_device(struct pci_dev *dev)
```

### 访问配置空间

在驱动程序检测到设备之后，通常需要读取或者写入三个内容：内存、端口和配置。配置空间又是至关重要的，因为这是找到设备映射到内存和IO空间的唯一途径。对于配置空间操作的函数都在头文件<linux/pci.h>中。

### 访问IO与内存空间

在内核中PCI设备的IO区域已经被集成到了通用资源管理中，因此我们并不需要访问配置变量来了解设备的映射，获取区域信息只要用一下函数就可以：

```
unsigned long pci_resource_start(struct pci_dev *dev, int bar);
unsigned long pci_resource_end(struct pci_dev *dev, int bar);
unsigned long pci_resource_flags(struct pci_dev *dev, int bar);
```

### PIC中断

计算机固件已经为设备分配了中断号，这个中断号保存在配置寄存器中，驱动程序只需要从PCI_INTERRUPT_LINE中获取值就行。

**如果需要了解一下外设接口和PC总线，可以查看MCA，EISA，VLB，SBus，NuBus等知识。**

