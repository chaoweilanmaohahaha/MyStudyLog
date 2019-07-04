# What you corrupt is not what you crash- Challenges in fuzzing embedded device

嵌入式设备的问题：空间错误（越界）、时间错误（UAF），嵌入式设备没有防御机制比如MMU，出现错误并不会出现crush的事件，因此不能够直接观测崩溃的事件。

* type1：存在通用操作系统:MMU
* type2：专门的嵌入式操作系统:watchdog
* type3：将用户代码和操作系统编译在一起:no effect

行为：能观测得到的crash，重启，挂起，晚崩溃，错误输出，没有影响。

如何在没有可信的反馈的情况下进行fuzz

​	

