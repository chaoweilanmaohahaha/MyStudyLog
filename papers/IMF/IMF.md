# IMF : Inferred Model-based Fuzzer

**可推断的模型化模糊分析工具**

## 论文的出发点

也就是这篇论文的背景以及出发点到底在哪里，这篇论文是基于MACOS系统开发的首先还是先要摸清MACos系统。这篇论文主要抓了一个叫iokit的东西，这个东西是在mac上做驱动开发和管理的，他就是一个系统硬件层面的一个抽象，这个存在于macos的内核态；那么用户在写程序的时候需要调用库IOKitlib（在用户态），那么这篇文章就是基于这个库的库函数来去制定这样一个分析工具。

这个工具准确的说是一个基于建模的一个API fuzzer，论文的中心思想就是自动生成一个API模型，然后使用这个模型来fuzz内核。（问题1：为什么不直接在原程序中进行这个fuzz呢?  因为根据api调用来触发漏洞很难，除非我们调用了api函数是以某些顺序和某些特定的值。所以才需要所谓的顺序依赖和调用值依赖）



## 论文的思路

这篇文章的中心思想是提出了一种模型化的模糊分析工具。具体就是对于应用程序调用系统的api过程进行一个建模，具体由三个部分组成：logger--- 日志系统，inferrer --- 推断器， fuzzer --- 模糊测试工具。

日志系统的使用过程是将所有可能调用到的API和目标应用程序当成参数，然后多次运行目标程序，然后记录下使用到的API和参数。推断器负责从日志系统生成的日志中推断出一个代码运行模型，这里面涉及了两个论文中认为比较重要的问题——顺序依赖（api运行顺序）和数据依赖（参数调用顺序）。根据生成的模型来使用特定的模糊测试工具来进行测试，输出系统可能存在的一些bug。



## IMF工具的探究

从github上把工具整个荡了下来，分析一下其中的源码，然后看一下具体实现，这个工具是用python编写的。

使用方法在readme中给出一下步骤：

```
$ ./gen-hook [output(hooking code) path]

$ clang  -Wall -dynamiclib -framework IOKit -framework CoreFoundation -arch i386\

         -arch x86_64 hook.c -o hook
$ DYLD_INSERT_LIBRARIES=[hooking library path] [program path] [program args]
$ ./filter-log [log dir] [output dir] [# of output(filtered log)] [# of core]
$ ./gen-fuzz [filtered logs path] [output(fuzzer code) path] [# of core]
$ clang -framework IOKit -framework CoreFoundation -arch i386 fuzz.c -o fuzz
$ ./fuzz -f [log path] -s [seed] -b [bitlen] -r [rate] -l [# of max loops]
```

hook.py 生成一个代码.c的代码文件，然后用第二步进行编译，钩子函数通常的写法是这压根

```
type fake_func (args){
    /*the input args state*/
    func(args)
    /*the output of func*/
}
```

DYLD_INSERT_LIBRARIES的目的是通过向宏DYLD_INSERT_LIBRARIES里写入动态库完整路径。就可以在执行文件加载时将该动态库插入。

Mac offers a way to override functions in a shared library with **DYLD_INSERT_LIBRARIES** environment variable (which is similar to LD_PRELOAD on Linux). When you make a twin brother of a function that is defined in an existing shared library, put it in you a shared library, and **you register your shared library name in DYLD_INSERT_LIBRARIES, your function is used instead of the original one**. This is my simple test.

filter.py 通过最长前缀匹配的方式，将众多个log过滤成只剩两个log

apifuzz 讲道理就是生成fuzz.c文件的一个函数

然后执行这个fuzz.c文件来查看



造成kernel panic的几个原因：

1. Linux在中断处理程序中，它不处于任何一个进程上下文，如果使用可能睡眠的函数，则系统调度会被破坏
2. 内核堆栈溢出，或者指针异常访问时
3. 除0异常、内存访问越界、缓冲区溢出等错误时
4. 内核陷入死锁状态，自旋锁有嵌套使用的情况
5. 在内核线程中，存在死循环的操作
6. 一般保护错是在PC机用户程序企图访问不可访问地址时出现的错误
7. 

## 论文的几个优点吧

1. 在相关工作中，作者将之前提出的模糊测试的工具分了种类，将他们重点分为了4类：随机模糊测试工具、有类型定义的模糊测试工具、基于api钩子的模糊测试工具、基于回馈机制的模糊测试工具。