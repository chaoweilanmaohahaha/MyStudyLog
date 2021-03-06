# NEUZZ: Efﬁcient Fuzzing with Neural Program Smoothing

## Background

这篇论文可以说是挺有意思的一篇，我觉得其实问题的本质还是解决产生的种子的有效性。因为比如传统的 AFL 工具通过随机修改生成的种子其实大部分都是无用的，那么这样反而让模糊测试工具陷入困境。

本文并没有从种子结构触发，而是希望采用机器学习的一些手法，采用基于梯度的方法来提升测试的性能。但是其实在之前提出的一些使用梯度来优化测试的方法中发现，如果将模糊测试作为目标函数该函数就是不连续的，这会让梯度算法反而陷入困境。

在讲解这篇论文真正的内容之前还是要接受这里需要用到的知识点。

我们先来理解一个经典的优化模型：

> 对于一个给出的目标函数 F，对于它的参数向量 *x*（目标函数不止一个参数），一系列对于参数向量 *x* 的限制 C(x)，在满足 C(x)  的情况下求得 F 的最大值后者最小值。

想一下之前的梯度下降算法的介绍，一般就是给定一个起点，也就是初始的参数向量 x，随后迭代搜索来得到最优解。这一类的算法的关键就在于**寻找一个向最优方向前进的方法**。一般采用的就是**使用梯度配合高阶导的方法**。

但是对于目标函数是不连续的那种，简单的使用梯度并不能解决问题，因为常常会出现类似局部最优等情况(**这个问题有待确认**)。在后面的研究中也会介绍有关**平滑函数**的概念，专门来解决这个问题。将上面的问题再详细一点，对于一个凸函数而言，它一定能够求得全局最优，但是对于一个非凸函数而言就可能陷入局部最优了。

而本文中，模糊测试是可以被看作一个未加限制的最优解问题，可以理想地认为在给定固定输入 x 下找到最大数量的漏洞。一个看法认为目标函数 f 就是当某个输入种子 *x* 触发了漏洞就等于 1，否则为 0。但是简单的说这么设计一点意义都没有，所以这里针对的目标函数就是最大化代码覆盖率。**也就是获得一个输入向量，使得目标程序能够获得最大的代码覆盖率**。

## Overview

这个工具在原本的 fuzz 中还需要添加几个模块，这些模块有办法帮助 fuzz 工具找到更好的输入：

* 基于神经网路的程序平滑：这个模块的目的是为了去构造一个平滑函数来模拟被测程序的行为，在这里选择了使用神经网络，因为有研究表明神经网络在这一方面具有重要的帮助。
* 基于梯度的优化：根据神经网络的性质，一旦一个神经网络模型被及案例之后，就可以使用一些计算梯度的方法来获取梯度的计算，这在后文中会提到，因为有了上面的程序平滑技术，解决了之前梯度可能会陷入困境的问题。
* 增量学习：一开始的训练肯定使用的是最初的数据集，那么当在测试过程中产生了新的输入之后，就需要使用新的种子去重新生成新的模型。

## Details

下面看一下核心的一些技术：

### 程序平滑技术

**这一项技术的本身就是为了让基于梯度的方法适用于模糊测试。**在上面提到，如果把 fuzz 看成是一个未加限制的目标函数，那么它是不连续的，因为多个分支的跳跃性非常大，是一个近似离散的一个函数。

> 文中举的例子是一个分支判断当 g = 3^(a+b)，当 g 在不同区间中的一个分支情况，可以看到这个分支对 a和 b 的不同取值的反映是跳跃式的，这就是它要解决的一个问题。

那么在数学上为了使得一个不连续的函数变得平滑，会用到**卷积的手法**，找到一个掩码函数 g 去计算 f‘ 来代替原来的目标函数 f。

![NEUZZ](..\img\NEUZZ.png)

这是最主要的方法，虽然文中并没有介绍如何去获得 g 这个函数，不过文中给出了目前提出的程序平滑的现成方法，一共分为两种(*我觉得有必要读一下文中给的论文*)：

* 黑盒平滑：根据挑选程序的输入，来计算这些样本的卷积从而进行程序平滑，也就是只使用输入。
* 白盒平滑：通过一些类似符号执行技术的方法，探测程序的执行指令，根据执行关系来做程序平滑。

### 基于神经网络的程序平滑

这里作者提出的程序平滑方法可以说和上面的两种方法都不太一样，作者认为他使用的方法是一种灰盒平滑技术。

首先先解释为什么要使用神经网络，根据作者的观点，神经网络正好适用于这种由复杂行为的程序操作中：

* 它可以很好地为非线性的程序行为建立模型
* 神经网络还支持有效计算一些比如梯度，高阶导的操作，这在之后的 fuzz 中就要用到。
* 神经网络是可以从给出的数据集去预测一些没给出的输入产生的结果的。

为了训练这样一个模型来模拟程序的分支行为比如控制流，作者使用了**程序的输入文件**和**这些文件的代码覆盖**，将输入文件作为目标函数的 X 输入，代码覆盖情况作为 Y 标记。该训练的目的是将给出的**损失函数** L(y, f(x, $theta$)) 降到最低，这个损失函数是函数输出的代码覆盖和真实覆盖的差异。损失函数使用了二元交叉熵来计算了两者的距离。

在这篇文章中作者使用的是前馈神经网络，这里提一句在预处理输入数据集的时候因为会出现数据的偏向性，**因为大量数据可能获得相同的标签，因此在这里需要使用降维的方法**。

*这里总结一下，这个方法本质的目的应该就是使用了神经网络完全模拟了输入 -- 代码覆盖之间的关系，这样类似于模拟了程序的执行，起到了程序的平滑效果。*

### 基于梯度的优化

梯度主要是解决从原始的输入怎么改变生成新的输入的过程。在这篇论文中作者使用的是一个最简单的梯度搜索算法。

在这个项目中，计算梯度的为了找到一个输入，尽可能能让输出的代码覆盖能够更多的让 0 变成 1。

*作者在文中说梯度很好计算，暂时不知道为什么好计算。因为我目前觉得这个目标函数还是有点抽象，也要看看是不是和神经网络有关*

假设我们目前已经有这么个找最优改变的方案了，首先根据梯度等等信息，去识别梯度最高的一些输入位置，根据梯度计算下一步需要修改的位置，然后去迭代的尝试修改这些位置(根据给出的梯度信息)，使用的是位翻转的策略。

### 增量学习

神经网路模型的正确性很靠输入的正确性，所以如果想要获得更好的模型，最好的方法就是将后续 fuzz 工具生成的种子也加入到输入中继续训练，**但是有一个问题就是也要保证训练出来的模型不能忘记以前的记忆**。

通常解决上面的问题的方法有两种：

* 一种是将新的和旧的数据集都建立模型，然后使用集成学习的方法去构造模型
* 提炼旧数据生成一个浓缩版的旧数据集加入到新数据集中。

随着新的种子产生，输入的数据集会变得非常庞大，在这篇文章中作者是根据旧数据集中触发覆盖率的情况，剔除了一些覆盖情况重复的输入。

## Evaluation

作者的实验是在三个数据集上：10 个真实的程序、CGC、LAVA-M。然后会和 10 个现有的测试工具比较代码覆盖率和寻找漏洞的能力。

在寻找漏洞方面，NEUZZ 能够在有限的时间内从真实的程序中找到更多种类的漏洞。在 LAVA-M 的测试中，有一个非常有意思的地方，就是在解决 magic number 的问题上使用了 LLVM 来帮助，但是在这里作者还是使用了神经网络，*目前的理解就是先找到有关 magic number 的重要位置，然后局部进行暴力搜索来确定 magic number。**这个真的快了吗？***

但是反正作者的实验证实了 NEUZZ 找到了 LAVA-M 中所有的漏洞。

在 CGC 数据集上，NEUZZ 在随机挑选的 50 个目标程序上能够找到 31 个有 bug 的程序，属实最多。

如果分析代码覆盖率，NEUZZ 反正也是最好，性能也是最好。

相较于一些基于 RNN 技术实现的 fuzzer 工具，作者在四个目标程序上做了实验，不经查看了代码覆盖率，并且查看了模型的训练时间。可以发现因为使用的简单的神经网络，所以训练时间大大缩短，并且代码覆盖大大增加。**作者分析原因在于 NEUZZ 会识别重要修改位置，而其他的这些模型并没有这么做，只是简单的端到端训练。**

最后作者比较了使用神经网络和其他模型，最后还是发现使用神经网络确实必使用一些线性模型要好。