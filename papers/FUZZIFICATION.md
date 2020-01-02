# FUZZIFICATION:  Anti-Fuzzing Techniques

###### Background

fuzzing这个技术可以不在知道程序内容的情况下找到程序中的bug。但是这也意味着，这项技术可以落入坏人的手中。所以这篇文章的出发点就是尽可能地隐匿fuzzing的结果，使得攻击者无法很快地发现漏洞。(有一个数据很有意思，实验表明模糊测试技术可以提升寻找漏洞地效率，提升了大约4.83倍)。

那么在这篇文章中作者提出的方法并不能阻止攻击者发现漏洞，而是尽可能让攻击者发现漏洞的代价变大，比如消耗更多的资源，消耗更多的时间。

因此作者认为衡量一个这个技术要从三个方面来看：

* 可以在给定的时间内发现更少的bug；
* 生成的这个受到保护的程序依旧可以很好地运行；
* 这些所谓受保护的代码不能很快地识别。

之前提出的代码混淆技术可以很好地混淆源程序，但是它降低了程序运行地性能；而且对于fuzzer地路径探索也没有做到什么保护作用。

软件多版本技术也是一个很好地技术，但是这个技术也不能很好隐藏源程序。

那么当下的研究的焦点主要集中在如何加快fuzzer的执行，这样可以发现更多的bug；同时现在的技术也着重于路径的覆盖，那就是说如果能够增加覆盖就能够更可能地挖掘出漏洞。

当然也有fuzzing的技术使用了混合的fuzzing技术，也就是说，使用了污点分析和符号分析可以加快fuzzer的性能。



###### Overview

这个技术的基本做法是需要使用源代码编译出两套代码，一个是使用了这项技术来生成了保护代码的，这个版本的程序是要被公布的，另一个版本的是给那些可信任的机构的。

衡量一个抗fuzzing技术的标准有四条：

* 可以有效减少bug数量的发现；
* 可以适应各个fuzzer；
* 加上了这个技术后再运行程序，只有轻微的开销；
* 可以组织攻击者去除保护代码

现如今的技术并不能满足这样的要求，他们包括：

* 代码打包和混淆技术：代码执行的开销很大。
* 漏洞注入技术：影响用户的使用；
* fuzzer地识别技术：没有通用性；
* 漏洞模拟技术：没有通用性。

针对以上提到的，作者就提出了三种技术：SpeedBump，BranchTrap，AntiHybrid。对于整个系统而言，它需要源程序的代码，一些输入的样例以及一个开销清单作为输入；1、首先通过源程序的编译，使用这些测试的数据获取代码块的依赖关系；2、使用上述的三种技术生成一个受保护的代码；3、评估当前的受保护代码运行的开销，如果这个开销够大，则重新生成。

###### Detailed Design

**SpeedBump：**首先说一下这个技术是为了有效地减慢fuzzer发现漏洞地速度。作者发现fuzzer更多地会陷入一些错误句柄，那么这个机制就在这些分支上注入延时。那么这个工具就会在第一次编译，通过使用给出的一些样例，来获取这些分支的位置。而在运行的过程中就可以生成一份清单，这份清单上就记录了所有很少运行到的分支情况。

延时的实现上不能使用普通的sleep函数，因为这样会让攻击者很快地发现，所以需要尽可能使用数学的运算并且和源代码有所关联（使用代码中内在的参数作为运算对象），使用了CSmith来生成。而且注意的是不是说主入的位置一定就是那些错误句柄的位置了，因为有些fuzzer就会识别这些句柄然后删除它们，这里的注入点是运行概率很小的地方。

**BranchTrap：**

***混淆输入***：如果我们可以插入大量的分支或者对这个收集的覆盖进行一个捣乱，让基于代码覆盖的fuzzer浪费大量的资源。

最快的想法就是在代码中插入很多条件跳转语句，而且这些条件跳转语句必须和输入有关。那么在这里，第一步是要生成足够数量的分支路径；第二这个主入的分支必须对于真正的执行没有太大影响；第三，这个分支一定是确定性的，一个输入对应一个分支（一些fuzzer会检测无确定的分支）。

作者的方法一共有三步：1、收集每个函数的尾部代码，其中所谓的尾部代码是每个函数的汇编表示；2、将有相同尾声的函数返回放到一个表中内；3、根据参数的异或计算获取表中的索引，然后调用这个函数尾声，随后返回源程序。

***污染状态***：这个的目的是防止fuzzer发现更多的感兴趣的输入。作者的基本思想是在源程序中插入大量的确定性的分支语句，这个语句的作用是在少数的执行中能够充满codecoverage，从而使接下来覆盖碰撞的可能性增大。那么针对那些已经尽可能避免覆盖碰撞的fuzzer呢？

**AntiHybrid**:
这个就是想办法屏蔽掉污点追踪和符号执行。首先需要知道的是这个方法的一个瓶颈就在于污点追踪和符号执行的性能上，这会花费很大的资源。符号执行还会碰到的问题就是路径爆炸问题，污点追踪对于隐含的路径分析效果不佳。那么一方面就是使用隐含数据流改写代码，使用路径爆炸来阻止符号执行，比如将直接的对比改为计算hash。

###### Evaluation

作者一共评估了四个内容，分别是fuzzer探测路径的情况（真实路径）、发现漏洞的情况、在实际软件上使用的情况与对抗黑客攻击的情况。

减少路径探索方面，比使用afl减少了70%,honggfuzz减少了67%，80%的qsym。

发现漏洞方面，在真实的程序上比使用afl减少了88%，honggfuzz减少了98%，94%的qsym。在LAVA-M测试集上使用Vuzzer降低了56%，qsym降低了78%，afl-qemu和hongfuzz效果就很不好了。

作者随后将这个技术运用到了实际的软件上，主要是介绍了使用在mupdf上，然后可以看到的是有很好的效果。

对抗现有技术中，目前方案中的speedbump中插入的一些代码因为使用了一些hash计算等代码，因此容易被攻击者匹配到模式。

###### Discussion

需要注意的一点是这一个框架只是提出了一种可以延缓攻击者进行攻击的方法，这样可以尽可能让安全的单位能更早的发现漏洞，但是只是提出了在另一个维度上的一个安全防护，真正的防护还是需要使用多种安全机制。

在运行效率和安全性的一个权衡。

在speedBump上插入一些延时代码是通过软件模拟的方式，这个方法在一些比较弱的机器上可能会有较大的性能问题，如果使用了硬件来帮助延时有可能暴露一些静态的代码特征。



---

### 污点追踪技术

DTA是一种信息流的分析方法，这种方法可以分析源数据和目的数据之间的依赖关系。使用这个技术可以跟踪数据在程序中的处理，并且记录下数据的传播，这样可以获得代码关键位置和数据之间的关系。这样有什么好处呢？就是说只要知道了在输入数据中有哪几位对最后的程序有用，就可以更有针对性的生成输入。

### ROP- 面向返回的编程

ROP通俗的说是将原来的代码块拼接起来，然后使用了一个预先准备好的一个返回栈，使用里面的拼接作为下一条指令。这有什么作用呢？其实这是一种缓冲区溢出攻击方法，它的优点在于它所使用的代码都是内存中合法的指令。而所用到的拼接的代码块可以这么理解：首先找到ret指令，然后在函数的开始到ret中间序列属于一个代码块，那么这些代码块就包含了内存中合法的各种操作，只需要将这些代码放入一个预先准备好的返回栈中，然后将代码重定向到这个代码栈中，就可以达到攻击的目的。这个攻击唯一的问题在于使用代码栈会占用一定的内存。