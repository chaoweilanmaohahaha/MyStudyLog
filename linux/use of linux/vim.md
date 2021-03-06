# vim 编辑器

如果常用 Linux 的命令行模式，则少不了使用一些文本编辑器，你可以使用文本编辑器对 Linux 的配置文件进行编辑和修改，所以如果作为系统管理员的话至少应该熟悉一种文本处理工具。

为什么都需要学习一下 vim，Linux 上其实有那么多的文本编辑器，但是所有的 UNIX-like 系统都会内置 vi 文本编辑器，并且最重要的其实是很多软件的编辑接口都会调用 vi。vim 有许多人性化的设计，包括它会根据文件的扩展名和文件内的开头信息判断该文件的内容并自动调用该程序的语法判断样式。

众所周知 vim 包含三种模式，**一般命令模式**、**编辑模式**和**命令行模式**。下面对于三种不同的模式记录一下各个模式下比较常用的命令：

一般命令模式下：

* 0：移动到该行最前面的字符处；
* $：移动到改行最后面的字符处；
* G：移动到该文件最后一行；
* gg：移动到该文件的第一行；
* n：光标向下移动 n 行；
* /word：向光标下面寻找名称为 word 的字符串；
* :n1,n2s/word1/word2/g：在 n1 和 n2 行之间寻找 word1 这个单词，并把该字符串替换为 word2；
* x：向后删除一个字符；
* dd：删除(剪切)该行；
* yy：赋值光标在的那一行；
* p：在光标下一行黏贴；
* u：恢复前一个操作；
* ctrl+r：重做上一个操作；

编辑模式：

* r：只替换光标所在的那一个字符；

## 缓存和恢复

一般主要的文本编辑软件都会有恢复的功能，而使用 vim 的时候，其实在被编辑的文件的目录下都会存在一个 .filename.swp 的文件。这个文件就是你当前编辑文件的一个缓存。当你在使用 vim 时出现意外而退出时，此时会导致缓存无法由于正常的流程结束，这时缓存不会消失。那么假设该文件名对应的缓存存在的话，则 vim 就会主动判断这个文件可能存在问题。一般而言，可能会有以下的问题：

* 可能有其他人或程序同时在编辑这个文件。
* 由于不知名的原因导致 vim 中断。

因此对于这个缓存文件，你可以选择只读、直接删除该文件、重新加载缓存内容(加载完后需要手动删除)、无论如何都修改该文件。

## 额外功能

大多数的发行版上已经将 vi 替换成了 vim。vim 除了带有 vi 的功能外，还有类似颜色显示和程序语法检测的功能。除此之外，它还有许多额外的功能。

### 可视区块

其实前面的几乎所有操作都是基于行的，那么假设需要对一列进行操作，或者对一个区块区域整体进行操作，这就可以用到可视区块。当处于一般命令模式下，按下 v 可以进行区块的字符选择，此时理论上被选中的部分会出现明显的反白。随后就可以对反白部分进行赋值或者黏贴的操作等。

### 多文件编辑

vim 是可以同时打开多个文件的，vim ./a ./b 就能同时打开 a 和 b 两个文件。那么通过 :files 可以列出 vim 打开的文件；在这个情况按下 :n 或者 :N 可以选择当前编辑上一个文件还是下一个文件。

### 多窗口功能

大部分的编辑软件存在划分窗口或者冻结窗口的功能，这可以将一个文件划分为多个窗口。这是为了在修改一个很大的文件时，如果查看到后面的数据，但是需要修改很前面的数据，多窗口功能就有用了。

在命令行模式中输入 :sp filename,这时就可以在新窗口中启动另一个文件，如果仅输入 :sp 则出现的是同一个文件在两个窗口间。使用 ctrl+w+ 上下箭头就可以切换需要修改的窗口，而在某个窗口中输入 :q 可以退出该子窗口。

### vim 补全

bash 中是存在命令或者参数文件名等的补全的，那么在程序编辑器中也有部分的编辑器会根据扩展名进行关键词补全，那么在 vim 中有如下三个和关键词补全相关的快捷键：

* ctrl+x 和 ctrl+n
* ctrl+x 和 ctrl+f
* ctrl+x 和 ctrl+o

这样就可以在弹出的框中使用上下键对关键字进行选择了。

### vim 环境设置与记录

一般在家目录底下，如果你是用过了 vim，就会多出一个 ~/.viminfo 文件，这个文件会记录所有曾经做过的操作，好让你下一次轻松的开始作业。每一个 Linux 发行版其实对 vim 的环境配置都不太一样，但是这些环境都是可以自己配置的，使用 :set *** 可以对 vim 的某个配置进行设置。

当然如果你希望将你个人使用 vim 的习惯永久记录下来，就可以在家目录下建一个 .vimrc 文件，在里面将所有的配置都设置一下。

