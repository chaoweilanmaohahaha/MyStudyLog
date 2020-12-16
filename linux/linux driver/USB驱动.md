

# USB驱动

它的设计初衷是连接所有慢速设备到电脑上，但是现如今USB已经发展到可以连接几乎所有设备到电脑上了。USB主控制器负责询问每一个USB设备是否有数据发送，因此主机要轮询各种不同的外设。但是USB有一些很好地特性，比如可以要求一个固定的数据传输带宽，发送数据没有特殊的内容和格式要求。USB标准规范了一套任何特定类型的设备都可以遵守的标准，如果一个设备遵守了这个标准，就不需要再写一个特殊的驱动程序了。

## USB设备

USB设备由整个的配置，接口和端点组成，其中所谓端点，指的是一个个单向通道，分别有控制端点，中断端点，批量端点，等时端点，这个端点的定义使用struct usb_host_endpoint来描述，这个包含在了struct usb_endpoint_descriptor的结构体中，这个结构体还包含了所有USB指定的信息。许多的端口组合在一起构成接口，一个接口处理一种逻辑连接，struct usb_interface结构体描述了USB接口；那么所有接口又组合形成一个配置，一个设备可以有多个配置，使用了struct usb_host_config结构体描述，使用了usb_device结构体描述整个设备。**usb设备的sysfs设备命名为：根集线器-集线器端口号：配置.接口**

## USB urb

linux中的USB代码通过一个叫urb的东西和所有USB设备通信，这个请求块使用struct urb结构体来描述，urb使用异步方式往指定USB设备上的指定USB端点发送接收数据。一个urb典型的生命周期：

* usb设备驱动程序创建
* 分配给一个特定USB端点
* 由驱动递交给USB核心
* USB递交特定设备的特定USB主控制器驱动
* usb主控制器驱动处理，然后进行usb传送
* urb结束后，usb主控制器通知usb驱动

如果需要创建一个urb，则需要使用：

```
struct urb *usb_alloc_urb(int iso_packets, int mem_flags)
```

如果使用完了urb，则需要通知usb驱动程序已经使用完，靠函数：

```
void usb_free_urb(struct urb *urb)；
```

当然根据端点的不同还需要初始化不同的urb：（等时urb除外，这个需要手动初始化）

```
void usb_fill_int_urb //中断端点
void usb_fill_bulk_urb //批量端点
void usb_fill_control_urb //控制端点
```

当urb被正确创建和初始化后，则需要将器提交给usb核心：

```
int usb_submit_urb(struct urb *urb, int mem_flags)
```

如果需要终止某个urb，则调用如下函数：

```
int usb_kill_urb(struct urb *urb)
int usb_unlink_urb(struct urb *urb)
```

## 编写USB驱动程序

过程还是需要驱动程序吧驱动程序对象注册到USB子系统中，然后根据标识来判断是否安装了硬件。

struct usb_device_id提供了不同类型的该驱动程序所支持的USB设备，USB核心根据这个列表来判断设备使用的驱动程序。用来初始化这个结构体的宏包括：USB_DEVICE,USB_DEVICE_VER,USB_DEVICE_INFO,USB_INTERFACE_INFO。

### 注册

注册usb驱动程序时，不需要创建的主要的结构体是struct usb_driver，这个包含了许多变量和回调函数。当船舰了这个结构后，由函数usb_register_driver函数调用把它注册到usb核心，当驱动程序要被卸载时，使用usb_deregister_driver函数。

### 探测和断开

在探测时usb驱动程序会初始化任何可能用于usb设备的局部结构体，还应该把所有设备相关信息保存到局部的结构体中，这个获取可以通过配合使用usb_set_intfdata和usb_get_intfdata来实现，这个是将接口和相关数据结合。如果USB没有和任何设备相连接则需要使用usb_register_dev函数将USB设备注册到USB核心。在断开时需要使用usb_deregister_dev啊含糊来注销。

### 提交和控制

当驱动程序有数据发送给USB设备时，需要分配一个urb来传输数据，此时当数据被送到缓冲区时，urb要在可以被提交给核心之前初始化，当提交给核心后如果成功传输了urb，urb回调函数将会被调用。

## 不使用Urb

如果知识传输一些简单的数据，则没必要走一遍这个流程，可以使用usb_bulk_msg和usb_control_msg。