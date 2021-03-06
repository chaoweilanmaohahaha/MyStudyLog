# Send Hardest Problems My Way: Probabilistic Path Prioritization for Hybrid Fuzzing

## Background

这篇论文介绍的是如何优化 Hybrid Fuzzing 的一个想法，其实整体的想法不是特别复杂。

首先先说一下为什么 Hybrid。因为传统的 Fuzz 存在一个老毛病就是难以探索深层次的分支，而使用程序分析的方式能够帮助到模糊测试的，但是性能会大打折扣。在 AFL 的文档中本身页提到了，之所以不想添加一些分析技术的原因，就是看重效率。

Hybrid 所能做到的是取两者的长处融合，这样提升了模糊测试的能力(但是不一定提高了效率)。但是其实在配合使用 Fuzz 和 Concolic Execution 的方法上还是又很多地方值得思考(Concolic 是程序分析方式的一种，*暂时还没有搞清楚 Concolic Execution 和 Symbolic Execution 的区别，似乎前者是在后者的基础上加上了 模糊测试技术，具体我会看一下最初的几篇论文*)。 

如果要将两个技术混合的话，考虑后者性能问题，必然是尽可能使用模糊测试，只有万不得已的时候才会使用符号执行。那么在先前提出的方法中大多包含了两种：**按需执行**和**优化选择**。

* 按需执行：类似 Driller 的方法，先进行正常的模糊测试，只有当模糊测试陷入阻塞的时候，才将当前的还剩下的输入交给符号执行。但是这有一个问题，就是 fuzzer 的阻塞状态并不能真正代表已经很难进行下去了，**意思当 fuzzer 在阻塞的时候其实仍然可以发现新的分支。**另外这种策略也并不能识别阻塞 fuzz 的分支，特定情况下有可能导致符号执行作废。
* 优化选择：这个的想法是每一次分支选择是从 fuzz 和 符号执行中选择最优的方法来执行。但是这种想法在现实中很不现实，因为很难衡量所谓的最优，并且可能方法是非常费时的(搜集大量的路径限制)。并且有很多实现中会发现，这两种策略比较的指标都不一样，就很难进行比较了。

## Overview

先介绍一下 DigFuzz 的大致架构，其中他的 fuzzer 部分依旧基于 AFL，当 AFL 自身生成了新的种子之后，就会交给一个关键部分——路径概率模型来采样及构建执行树，这颗然后通过构造的执行树以及手头的 CFG，确认未命中的分支，随后将当前优先级最高的分支交给符号执行器，随后生成新的种子。

## Details

### Model

这个方法称为**区别分发策略**，它的核心是建立了一套概率模型。下面介绍一下这套概率模型。

这套概率模型基于**蒙特卡洛方法**：

> 蒙特卡洛方法是一种概率模型，求解的是某种随机事件的概率，或者是某个随机变量的期望值，通过某种“实验”的方法，以这种事件出现的频率估计这一随机事件的概率，或者得到这个随机变量的某些数字特征，并将其作为问题的解。

简单的说蒙特卡洛方法是把概率现象作为研究对象的数值模拟方法，但是这篇文章中也强调了，使用这种方法的前提必须保证：

* 样本的搜索空间是随机的；
* 是在大样本下搜索的；

所以在这篇论文中，为了估计一条路径执行的可能性，就是使用执行这条路径的统计概率来分析的；但是理论上执行路径是由每一个分支所构成的，计算分支的概率是可以做到的，那么想要计算路径的概率需要借助马尔科夫链模型，研究表明我们可以把执行路径看成是马尔可夫链：

> 马尔可夫链其实就是状态的转移，而且保证一点，就是某一个的状态发生的可能只与之前一个状态有关，与之前所有的状态都无关。

fuzz 的过程可以看成是一条马尔可夫链，所以当我们计算路径概率时，就是将路径上碰到的分支概率全部乘起来，而计算分支的概率，则是通过计算当前执行到该分支与兄弟分支的次数，计算出当前执行该分支的概率，那假设该分支并没有被执行到，根据概率学理论 rule of three，定义为 3/ 兄弟节点的执行次数。

根据计算出来的概率，我们形成了一个执行树，树节点就是代码块，边就是连接代码块的路径，每一个边上需要加上分支的概率。

### Implementation

实现其实就是围绕如何构造这棵执行树和对输入进行采样。

其实道理非常简单，就是获取了生成的输入后就查看分支的覆盖情况，然后求得该输入中的分支数据。

构造树的时候使用上面获得的分支情况，在借助了 CFG 的帮助下。首先通过当前的输入构造执行序列，随后通过分支击中的情况和 CFG 计算概率。

最后我们可以根据 CFG 的提示检测到许多我们还没有覆盖到的分支，然后就算这些路径的执行概率，按照从小到大排序，随后挑选概率最小的路径(这里认为这样的路径很难被处理到吧，所以优先对这样的分支进行处理)，交给符号执行器去发现新的输入。

## Evaluation

为了检测这个工具的性能，作者将这个工具和 Driller(按需执行)、MDPC(优化选择)、AFL 进行比较。其中还使用了 Driller 的 Random 配置，即不是从 Stuck 时才进行符号执行，而是从一开始就从输入中随机选择输入进行符号执行，这样同步来生成新输入。

实验在两个数据集上进行，CQE 和 LAVA-M。

在 CQE 数据集上可以看到 DigFuzz 优于其他的各个配置；在发现的漏洞方面也是 DigFuzz 更多，并且证明了它的性能是 Driller 论文中的两倍。最后从符号执行的贡献上来看，效率也是更加好一些。

在 LAVA-M 数据集上，漏洞检测方面其实大家表现的都差不多，并且其实覆盖率方面也是差不多，这是因为 LAVA-M 中的程序都是一些比较短小的程序，并且注入的漏洞相隔的非常近。

有意思的是在最后作者将实验部署到真实的程序上时发现，收到符号执行器的环境限制，基本上在真实的程序上跑不起来，所以就认为这是符号执行器的锅。

## Discussion

先谈一下作者对于该工具的评价，作者认为这个工具的基本问题是可能走到概率最低的分支也是最难解的，这样就会浪费时间，并且可能造成漏洞的分支也并不是一定是概率最低的路径。

以上来看，其实这样一个建模只不过是一种更合理的方法去处理符号执行和 fuzz 的关系。但是实际上它可能并没有使用符号执行直接朝向漏洞分支前进，所以这样的估计可能存在这偏差，不过这个方法确实在效率上提升了 hybrid fuzzing 继续探索路径的性能。可以说是通过建模的方法，对挖掘路径提供了一个导向的作用。

