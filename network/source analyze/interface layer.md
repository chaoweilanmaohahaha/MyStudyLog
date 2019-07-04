这个文件介绍接口层，可以说是数据链路层在初始化以及传输过程中的分析。

#### 数据结构

##### ifnet

该结构是存储了所有接口的通用信息，系统初始化的时候会为每一个网络设备分配一个独立的ifnet。初始化时会调用if_attach函数。

> struct ifnet {
> /*实现信息*/
> char	*if_name;		/* name, e.g. ``en'' or ``lo'' *//* 字符串，标识接口的类型 */
> struct	ifnet *if_next;		/* all struct ifnets are chained *//* 把所有接口的ifnet结构/*链接成一个链表，if_attach在系统初始化期间构造这个链表 */
> Struct  ifaddr *if_addrlist;/* linked list of addresses per if *//* 指向这个接口的ifaddr结构列表 */
> int	if_pcount;		/* number of promiscuous listeners *//* 监听者的数目 */
> caddr_t	if_bpf;			/* packet filter structure */ /* 分组过滤器结构 */
> u_short	if_index;	/* numeric abbreviation for this if *//* 在内核中唯一地标识这个接口  */
> short	if_unit;		/* sub-unit for lower level driver */ /* 标识多个相同类型的实例 */
> short	if_timer;		/* time 'til if_watchdog called */ /* 以秒为单位记录时间，直到内核为此接口调用函数if_watchdog为止 */
> Short  if_flags;	/* up/down, broadcast, etc. */ /* 接口的操作状态和属性，譬如接口正在工作和用于广播 */
> struct	if_data {
> /*硬件信息*/
> u_char	ifi_type;	/* ethernet, tokenring, etc *//* 指明接口支持的硬件地址类型，如以太网，令牌环 */
> u_char	ifi_addrlen;	/* media address length *//* 数据链路地址的长度 */
> u_char	ifi_hdrlen;	/* media header length */ /* 由硬件附加给任何分组的首部的长度 */
> u_long	ifi_mtu;	/* maximum transmission unit *//* 接口在一次输出操作中能传输的最大数据单元的字节数，如以太网是1500 */
> u_long	ifi_metric;	/* routing metric (external only) *//* routing metric，通常为0 */
> u_long	ifi_baudrate;	/* linespeed *//* 指定接口的传输速率 */只有SLIP接口才设置它
> 对于ifi_addrlen和ifi_hdrlen，例如，以太网有一个长度为6字节的地址和一个长度为14字节的首部。
>  /* 接口统计 */
> u_long	ifi_ipackets;	/* packets received on interface */
> u_long	ifi_ierrors;	/* input errors on interface */
> u_long	ifi_opackets;	/* packets sent on interface */
> u_long	ifi_oerrors;	/* output errors on interface */
> u_long	ifi_collisions;	/* collisions on csma interfaces */
> u_long	ifi_ibytes;	/* total number of octets received */
> u_long	ifi_obytes;	/* total number of octets sent */
> u_long	ifi_imcasts;	/* packets received via multicast */
> u_long	ifi_omcasts;	/* packets sent via multicast */
> u_long	ifi_iqdrops;	/* dropped on input, this interface */仅被SLIP设备驱动访问
> u_long	ifi_noproto;	/* destined for unsupported protocol */
> struct	timeval ifi_lastchange;/* last updated */记录任何统计改变的最近时间
> }	if_data;
> /* 函数指针，在系统初始化时，每个设备驱动程序初始化它自己的ifnet结构 */
> int	(*if_init)		/* init routine */初始化接口
> __P((int));
> int	(*if_output)		/* output routine (enqueue) */对要传输的输出分组进行排队
> __P((struct ifnet *, struct mbuf *, struct sockaddr *,
> struct rtentry *));
> int	(*if_start)		/* initiate output routine */启动分组的传输
> __P((struct ifnet *));
> int	(*if_done)		/* output complete routine */传输完成后的清除（未用）
> __P((struct ifnet *));	/* (XXX not used; fake prototype) */
> int	(*if_ioctl)		/* ioctl routine */处理I/O控制命令
> __P((struct ifnet *, u_long, caddr_t));
> int	(*if_reset)  /*复位接口设备*/
> __P((int));		/* new autoconfig will permit removal */
> int	(*if_watchdog)		/* timer routine */周期性接口例程
> __P((int));
> /* 输出队列 */
> struct	ifqueue {
> 		struct	mbuf *ifq_head;/*指向队列的第一个分组（下一个要输出的分组）*/
> 		struct	mbuf *ifq_tail;/*指向队列最后一个分组*/
> 		int	ifq_len;/*当前队列中分组的数目*/
> 		int	ifq_maxlen;/*队列中允许的缓存最大个数*/
> 		int	ifq_drops;/*统计因为队列满而丢弃的分组数*/
> 	} if_snd;			/* output queue */
> };

##### ifaddr

可以看到在ifnet中有一个if_addr的结构列表，这个列表维护使用该接口通信协议的地址信息。

> struct ifaddr {
> 	struct	sockaddr *ifa_addr;	/* address of interface */指向接口的一个协议地址
> 	struct	sockaddr *ifa_dstaddr;	/* other end of p-to-p link */指向一个点对点链路上的另一端的接口协议地址或指向一个广播网中分配给接口的广播地址（如以太网）
> #define	ifa_broadaddr	ifa_dstaddr	/* broadcast address interface */
> 	struct	sockaddr *ifa_netmask;	/* used to determine subnet */指向一个位掩码，地址中表示网络部分的比特在掩码中被设置为1，地址中表示主机的部分被设置为0
> 	struct	ifnet *ifa_ifp;		/* back-pointer to interface */指回接口的ifnet结构的指针
> 	struct	ifaddr *ifa_next;	/* next address for interface */接口的下一个地址
> 	void	(*ifa_rtrequest)();	/* check or clean routes (+ or -)'d */支持接口的路由查找
> 	u_short	ifa_flags;		/* mostly rt_flags for cloning */支持接口的路由查找
> 	short	ifa_refcnt;		/* extra to malloc for link info */统计对结构ifaddr的引用，宏IFAFREE仅在引用计数将到0时才释放这个结构，结构ifaddr使用引用计数是因为接口和路由数据结构共享这个结构
> 	int	ifa_metric;		/* cost of going out this interface */支持接口的路由查找
> #ifdef notdef
> 	struct	rtentry *ifa_rt;	/* XXXX for ROUTETOIF ????? */
> #endif
> };

##### sockaddr

可以看到在ifaddr里又有是使用sockaddr结构的参数，这个结构维护了主机的地址，广播地址和网络掩码。

> struct sockaddr {
>     u_char    sa_len;            /* sockaddr的长度 */
>     u_char    sa_family;        /* 地址族，如AF_INET */
>     char    sa_data[14];        /* 具体地址数组 */
> };

虽然说ifnet和ifaddr已经指定了接口信息，但是部分设备会使用专用的ifnet结构，比如以太网和SLIP，像以太网使用arpcom和le_softc，SLIP使用sl_softc。

##### arpcom

> stuct arpcom {
>
> ​	struct ifnet ac_if;//代表接口的ifnet结构
>
> ​	u_char ac_enaddr[6];//以太网硬件地址
>
> ​	struct in_addr ac_ipaddr;//IP地址
>
> ​	struct ether_multi* ac_multiaddrs;//指向以太网的多播地址列表
>
> ​	int ac_multicnt;//多播地址列表的项数
>
> };

##### le_softc

> structl e_softc {
>
> ​	struct arpcom sc_ac;
>
> ​	#define sc_if sc_ac.ac_if//接口的ifnet结构
>
> ​	#define sc_addr sc_ac.ac_enaddr//以太网硬件地址
>
> ​	............................
>
> ​	.........硬件相关成员.........
>
> ​	............................
>
>  } le_softc[NLE];

对于以太网接口而言，一台电脑可能有多个以太网的接口，所以这个le_softc是个列表。在系统初始化的时候，系统内核除了分配了一个ifaddr结构之外，还会分配两个sockaddr_dl结构，分别表示链路层地址和地址掩码。

---

#### 操作方式

##### 初始化

在cpustart过程中，系统会扫描电脑中的设备，如果设备被识别，就会有一个初始化函数被调用，其中以太网使用leattach，SLIP使用slattach，环回采用loopattach。当调用该函数是会给每一个接口初始化一个ifnet结构，并加入到接口列表中。

##### 以太网的传输

一个以太网的传输要考虑接收报文和发送报文两个过程。

接收报文要从接口处的操作开始，当一个以太网接口收到一个帧时，接口会产生一个中断，然后调用处理中断的函数leintr，这个函数就是为了检测硬件中断，就会调用leread把这个帧转移到一个mbuf中。leread构造了帧头部ether_header然后判断是否是多播地址或者广播地址，然后会去使用m_devget构造mbuf生成以太网帧。然后调用ether_input函数，检查帧头部并且将帧内容放到指定的IP队列中。

发送报文时会调用ifnet结构中的if_output，比如以太网会连接到函数ether_output这个函数使用以太网格式封装数据内容，然后将帧放置到发送队列中。lestart函数会从接口输出队列中取出排队的帧，然后交给网卡发送。

###### 环回地址的传输

环回接口的if_output连接到函数loopoutput上，这个函数的作用就是将输出帧放入系统的输入队列。