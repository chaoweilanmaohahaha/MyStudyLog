# EnFuzz: Ensemble Fuzzing with  Seed Synchronization among Diverse Fuzzers

###### background

这片文章的关注点不在如何使用某个策略来强化某个fuzz工具，它着眼于实际的应用场景。作者提出的观点认为，对于真正fuzz所有真实的应用程序，每个不同的fuzz工具都会凸显出它不同的特点，这样每个应用程序使用不同的fuzz工具得到的结果也是不一样的，必然有好有坏。但是这样带来的问题就是对于用户来说无法使用固定一款fuzz工具在fuzz任意一款应用程序的时候都可以有一样好的结果，那么这篇文章中作者提出的一个看法就是要想办法结合各种fuzz工具，找到一种整合起来的fuzz工具。

**在背景中提及了一款ClusterFuzz，这一款fuzz是一款分布式的fuzz工具。像AFL这样的现成的fuzz工具，它本身就有一种parallel模式，你可以把它看成一种使用同一种fuzz工具的整合式fuzzer，在这里面提出了一种策略，称为种子同步。**

作者所作的要使用不同的fuzz工具，实现在全局上是一种异步的fuzzer但是在局部你可以认为是同步的因为它们的种子是同步的。为什么要在不同的fuzzer工具之间实行种子同步，文章在动机中提到，如果一个fuzzer工具可以很容易进入T1，但是很难进入T2；而另一个很容易进入T2却很难进入T1。那么如果在第一个fuzzer进入T1后把当前的装填分享给T2，那么就可以有更多的路径。那么要实现这样的功能，必须要挑选合适的fuzzer基于多个层面并且要涉及一个具体的同步机制。

###### overview

因此要构造这一个fuzzer工具，那就需要构建一系列的基本fuzzer连接它们来测试同一个目标应用。这个工具的执行流程的第一步需要挑选一些现成的fuzzer工具作为基本fuzzer；在选择完这些fuzzer后，我们就混合这些fuzzer工具然后使用一种全局异步和局部同步的机制，最后收集崩溃的信息和覆盖信息然后将这些信息融入报告中。

###### detail design

**选择基本fuzzer的策略**

在选择基本fuzzer的过程中，并不需要考虑测试工具的整体的分类，在这里可以选择生成式的fuzzer，或者是修改驱使的fuzzer。选择fuzzer必须基于好的方面才能使得集成的fuzzer更出色。这就根据三个维度：种子修改和选择的策略（比如那种有的fuzzer会专门去选择那些没怎么被击中的路径，修改种子去尝试命中比较罕见的分支），代码覆盖信息粒度（这个的意思是考虑代码覆盖的种类，比如使用代码块覆盖，或者使用边覆盖），输入的形成策略（就是上面所说的一种生成式的和一种修改式的，那么像生成式的适合去fuzz有复杂输入形式的程序，而修改式的适合去修改有复杂逻辑的程序）。**作者在这里的观点就认为有两种特性的fuzzer一定会比只有一种特性的fuzzer强。**在这篇文章中作者是人为地去选择了各个fuzzer。

**enfuzz架构设计**

这里就是介绍了作者的**全局异步局部同步的机制**，主要的想法就是识别并获得感兴趣的种子，这个种子可以是不同的fuzzer异步生成的，但是各个fuzzer又同时分享了这些种子。

![enfuzz](..\img\enfuzz.png)

在这个架构中每个fuzzer依然保留自己的一个本地种子队列，那么同时监视器还维持一个全局种子池，这是所有fuzzer种子的一个集合。所有新的分支信息会被保存到全局的覆盖map中，这个map除了记录分支情况，也有着去重和分类的功能。最后全局的crash表就记录了所有崩溃的信息以及是什么种子使这个应用软件崩溃。

这里有几个问题，首先如何将每个异步生成的种子放入全局种子池。那么这里指出，某一个fuzzer所需要做的，就是普通的fuzzing，那么如果某一个fuzzer发现了崩溃的状况，或者说某一个fuzzer发现了一个新分支，那么就把当前的种子加入到全局的种子池中。那么当我们已经有了这些种子后，监视者是如何将这些种子去分派给各个fuzzer的呢？每次都会使用这个池子里的那些可以对当前的这个fuzzer产生感兴趣的种子的一个种子。

###### evaluation

这篇文章作者一共实现了五种形态，分别是使用了不同分组的base fuzzers。在这里要克服以下的困难：

1. 封装调用各种的fuzzer的接口。
2. 像libFuzzer这种fuzzer遇到崩溃就会停止，那么需要使他可以保持。
3. 如何去掉重复的bugs
4. 有效同步种子

这个评估方法我觉得暴露出了一点问题，不过还是中的看一下。

首先第一次评估使用了LAVA-M数据集，同时作者的评估方式是使用了fuzz-Q（包含了AFL，AFLFAST，FAIRFUZZ和QSYM）。因为LAVA-M中的程序都是较小的开源程序，这个fuzz-Q中加入了静态分析，因此比较成功

第二次使用的是Google的fuzzertestsuite，作者使用的是AFL，AFLFast，libFuzzer和Radamsa，明显比单独的好。

第三次作者使用了不同给的组合来fuzz，同时有一个fuzz-，它并没有在这个工具上使用所谓同步种子自己。一共有五种组合。

最后作者fuzz了实际的一些应用程序。

###### limitiation

1. 作者认为base fuzzer还没有遍及所有维度。
2. 扩展性方面可以考虑以下fuzz的那个的应用程序或者是不同的fuzz任务，强化框架的扩展性；
3. 看看能否动态分配资源，类似执行周期和核。
