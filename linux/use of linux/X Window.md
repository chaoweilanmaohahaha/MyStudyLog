# X Window

现在的 UNIX-like 操作系统不是只有服务器了，它也在桌面应用比如美化、制图等方面有了需求，这些都是需要**图形用户接口**的，这就是目前使用的 X Window System 做的事，它是一个很强大的窗口接口架构，同时还支持**网络以及跨平台**。

X Window 最早是由 MIT 开发的，当初的希望是 X Window 不要和硬件有强烈的相关性，否则这就和操作系统类似了。后来随着一版版的改进，目前最稳定进步最明显的一个版本就是 X11，后来的版本都是在该版本基础上改进的。那么所以首先  X11 它只是一个软件，而不是一个操作系统；其次它是通过网络架构来进行图形用户接口的执行与绘制。

## 组件

基本上 X Window 就是两个组件：X Server 和 X Client。

### X Server

X Server 的重点在于管理客户端的硬件，他需要管理比如键盘、鼠标、手写板、显示器、显卡等等。 X Server 它是可以有自己独特的设置的，因此可以和 Linux 的设置不同。因此每台客户端主机都需要安装 X Server，而服务器则是提供 X Client 软件。当它感知到了用户使用硬件的一些输入操作时，就会将信息通知 X Client。

### X Client

X Client 的最大工作是根据 X Server 给的数据，将它们转换成绘图数据，随后将这个绘图数据再反馈给 X Server，让 X Server 在显示器上绘制出图像。**X Client 并不需要知道 X Server 的硬件设备和操作系统**，只需要单纯处理数据，然后将绘图数据返回给 X Server。但是也有一个问题就是每一个 X Client 并不知道其他的 X Client 的存在，所以很有可能会出现绘图重叠问题。

### X Window Manager

为了防止出现 X Client 彼此不知道的问题，需要一个叫做 Window Manager 的组件来管理，它称为窗口管理器，负责全部的 X Client 的管理，同时还会提供一些控制元素、管理虚拟桌面、提供窗口的参数等等。我们经常听到的那些 KDE、GNOME 就是窗口管理器。

### Display Manager

Display Manager 的最大功能是提供登录环境，比如说像 GNOME 会使用 Display Manager 来提供 tty1 的图形用户界面模式登录。

## 启动

现在可以知道，要想启动 X Window System，就需要先启动管理硬件与绘图的 X Server，随后加载 X Client。那么整个具体启动过程：

* 使用 startx 命令，这个命令会去寻找用户或是默认的 X Server 和 X Client 配置文件，具体的找寻的目录顺序这里不赘述
* 执行 initx 命令，这个命令就是使用了刚才找到的配置文件参数对 X Server 和 X Client 进行加载那么和它们有关的配置文件有：
  * xserverc：xserver 的配置文件，这个配置文件最后会去调用一个很重要的 xorg.conf。
  * xinitrc：这里面都是启动 X Client 的默认脚本，说实话其实就是选择启动那个窗口管理器了。

最后提到一点，你其实会看到 X Server 的配置文件中会出现有关端口的参数，其实端口对应的是将 X Client 加载到这个接口上，而在 CentOS 中端口已经改为了一个 Socket 文件，默认启动的是 6000 端口，其他接口以此类推。

那么你到底需不需要启动 X Window 呢？如果你的系统用作了服务器，那么无需开启；那么如果是桌面系统，那肯定需要借助 X WIndow 来操作。

## 配置

X Server 的启动其实很重要，它的一系列配置文件是默认存放在 /etc/X11 目录下的，而其他相关的可能存放在：

* 总管模块：/usr/lib64/xorg/modules
* 显示字体：/usr/share/X11/fonts
* 显卡驱动：/usr/lib64/xorg/modules/drivers/

那么在 CentOS 中这些都通过统一的配置文件来规范，它就是 xorg.conf。

在 CentOS 6 以后其实系统默认已经没有了这个配置文件，如果你想添加依旧可以在 /etc/X11/xorg.conf.d/ 目录下建立 .conf 文件。

这个配置文件的内容是分区的，具体每个功能占一个区。如果你想要建立一个 conf 文件，可以借助 Xorg-configure 生成一个模板。比如说你想更新一个字体，就可以在其中字体存放的指定目录存放一个新的字体，随后使用 fc-cache 来更新字体缓存；当然也可以想办法调整显示屏分辨率，可以使用 xrandr 命令，当然如果需要强制刷新分辨率和频率，则需要直接修改 xorg.conf 文件。

最后提一点，显示还是离不开硬件的支持，那么如果需要类似 3D 加速这种功能，免不了要好一点的显卡，那就要更新显卡的驱动。目前流行的是 NVIDIA、AMD 和 Intel 三家，你可以直接下载对应硬件的驱动程序然后加载模块，随后将驱动放到指定的文件夹下修改 xorg.conf 就可以啦。