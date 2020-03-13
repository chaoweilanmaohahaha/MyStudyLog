# 调试 API

在之前的学习中，我们接触了调试器的强大功能。但是实际上在 Windows 本身就带了一些很强大的 API，简称调试 API。那么利用这些 API，我们可以加载或者捆绑一个正在运行的进程，从而获得被调试程序的底层信息，甚至修改。

当然如果可以，可以使用这些函数自己实现一个调试器。

## API

下面罗列一下可能使用的到的 API，如果要知道 API 的详细信息，就自行查找吧。

* ContinueDebugEvent：允许调试器恢复先前由于调试事件挂起的线程。
* DebugActiveProcess：此函数允许将调试器捆绑到一个正在运行的进程上。
* DebugActiveProcessStop：此函数允许将调试器从一个正在运行的进程上卸载。
* DebugBreak：给当前进程产生一个断点异常。
* DebugBreakProcess：在指定进程中产生一个断点异常。
* FatalExit：这个函数让调用进程强制退出，将控制权交给调试器。
* FlushInstructionCache：刷新指令告诉缓冲
* GetThreadContext：获取指定线程的执行环境
* GetThreadSelectorEntry：获取选择器和线程描述符表的入口地址
* IsDebuggerPresent：判断进程是否处于调试环境中
* OutputDebugString：将一个字符串传递给调试器
* ReadProcessMemory：读取指定进程的某区域的数据
* SetThreadContext：设置指定线程的执行环境
* WaitForDebugEvent：等待被调试进程发生调试事件
* WriteProcessMemory：在指定的进程的某个区域写入数据。

对于调试器而言，它要监视目标进程每一个调试事件，如果发生一个事件后，就要通知调试器处理这个事件。

上面提到的 WaitForDebugEvent 用来获取调试事件。返回的事件会放在 DEBUG_EVENT 结构中并且返回(详细有哪些事件这里不展开)，而这其中的 dwDebugEventCode 判断触发事件类型。而在 DEBUG_EVENT 中存放了一个联合体匹配了根据 dwDebugEventCode 的不同而匹配的具体信息。

## 具体使用

### 创建并跟踪进程

在通过 CreateProcess 创建进程时，需要在 dwCreateFlags 标志中设置可调试标志。创建后在 PROCESS_INFORMATION 结构获取创建进程和主进程的标识符。最后可以使用 IsDebuggerPresent 函数查看进程是否在调试器下运行。

那么在 API 中使用 DebugActiveProcess 函数将调试器固定到一个正在运行的进程上。

### 调试

很显然，调试过程中需要结合 WaitForDebugEvent 和 ContinueDebugEvent 来操作进程和接收调试事件。

### 处理事件

### 线程环境

进程拥有自己的私有空间和一个主线程，那么后续创建的其他线程和主线程在统一地址空间中。**进程并不执行代码，真正执行代码的线程。**那么在当前线程执行状态中保存了一个 CONTEXT 的结构，保存了线程所用的寄存器和栈。

对于线程环境，API 中也有相应的函数处理。那么注意的是，在使用 GetThreadContext 之前要调用 SuspendThread，否则线程可能会被调度，本质的目的是让线程在用户环境中稳定。然后可以的话再用 ResumeThread 恢复。

### 注入代码

如果说主入代码很短，大可以使用进程区块间的空隙，但是很长的话就只能先将目标进程的某个代码页保存，然后主入新的代码，最后执行写回。具体过程：

1. 创建一个进程；
2. 控制循环体和监听调试事件；
3. 挂起目标线程： SuspendThread
4. 修改目标页的读写权限：VirtualProtectEx
5. 读取目标页：ReadProcessMemory
6. 保存线程环境：GetThreadContext
7. 写入新代码页：WriteProcessMemory
8. 确认新指令中最后一个指令是 INT 3
9. 保存一份 CONTEXT 的拷贝
10. 设置新的 EIP
11. 恢复原线程执行。
12. 写入旧代码页
13. 恢复代码页的读写属性
14. 恢复原始线程的原始环境：SetThreadContext
15. 恢复原始线程的执行

**最后注意，一定要注意栈的平衡问题！**