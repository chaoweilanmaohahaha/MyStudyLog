# Bash

如果从内而外去描述应用的执行，就是我们需要通过**应用程序来输入命令**，然后内核回芯片的**驱动程序**，随后调用相关的硬件。那么一般而言我们输入命令是要通过 shell，所以只要有操作系统那么就离不开 shell 这个东西。

一般谈及 shell 指的是命令行模式下的 shell，因为非命令行版本的 shell 界面各不相同，而命令行版的几乎每家都是一样的，并且在远程连接的时候，命令行模式的 shell 速度一定比较快，而且不容易掉线。

Linux 下会使用哪一个版本的 shell 呢？其实 shell 有很多种，包括 sh、C shell、K shell 等。那么 Linux 一般所使用的就是 Bash，也是 sh 的增强版本。如果你要知道你的 Linux 下有多少个 shell，只需要到 /etc/shells 这个文件下查看，这些都是合法的 shell，这个文件存在的意义是方便其他服务运行时查找合法的 shell。那么既然有这么多种 shell，怎么知道用户默认使用的是哪个 shell 呢？如果仔细观察 /etc/passwd 文件，会发现在每一行的最后包含了每个用户的默认 shell。

## 功能

bash shell 既然是 sh 的加强版，就说明其较强的功能。其中包括了：

* 历史命令：记录历史命令在 ~/.bash_history 中，这里记录的是前一次登陆时的命令，而这一次的命令都存在于缓存内，只有注销后才会记录到文件里。
* 命令补全：使用 Tab 能命令补全，这个功能只有 bash 里面有，命令补全包括了可以补全命令，还可以做到文件补全，更可以做到参数补全；
* 命令别名：可以给一些复杂的命令操作起一个别名；
* 任务管理、后台控制；
* 程序脚本化：相当于写一个 shell 脚本
* 通配符的使用

为了方便 shell 的操作，bash 已经内置了很多命令，包括 cd、umask，当执行这些命令的时候可以优先调用 shell 中的内置命令。但是有的时候你怎么判断一个命令是不是 shell 内置的呢？可以使用 type 命令。当我们对某个命令查询其 type 的时候，它可能会有如下的选项：

* file：外部命令
* alias：命令为命令别名设置的名称
* builtin：bash 的内置命令

由于 type 可以打印命令所在文件路径，所以它也可以做类似 which 命令的操作。

**在这里补充一个内容，当命令特别长时，可以使用\\加回车来换行，但是中间不能带有空格，否则\\作为转义。**

## 变量

变量的目的简单地说就是使用一个不变的内容代替一个一直会变的内容。为什么要使用变量，比如系统存在对于每个用户的一些偏好性设置，每一位用户对于同一个服务可能有不同的爱好，那么如果在代码中对每一个用户都硬编码变量内容则代码就显得冗长了。 

如何查看一个变量的内容呢？使用 echo 命令，但是前提是在变量前加上 $，或者使用 ${var} 的形式：

```
echo ${PATH} // 读取路径变量
```

如果需要对一个变量进行修改或者设置，最简单的就只要使用赋值：

```
a=1
echo ${a}
```

当一个变量名未设置内容时，默认内容是空。注意在设置变量的过程中，= 两边不能有空格，如果变量中有空格则可以使用单引号或者双引号结合起来，但是注意双引号会保留原本的特性，单引号则保留文本，例如：

```
b="${a}" // b = 1
b='${a}' // b = ${a}
```

如果你需要执行某个命令来赋值额外的信息，可以通过反引号或者 $() 来实现功能，例如：

```
b=`ls` or b=$(ls -al)
```

如果你是在以前变量的基础上扩展了变量，比如在 PATH 变量中添加了一些路径，使用 ${var} 直接累加内容。最后如果想要将变量转换为环境变量，使用 export 命令，而如果需要取消某个变量的设置，使用 unset 命令。

### 环境变量与自定义变量

环境变量在系统中好比是全局变量，它可以提供很多的功能。使用 env 命令可以查看环境变量的设置，这里不展开讲解每一个变量的作用了。那么除了环境变量，每一个用户也能定义自己的变量，要想查看这些自定义的变量需要使用 set 命令，这里面有几个变量是比较重要的，比如：

* PS1：命令提示字符，也就是可以设置执行某个命令后再次出现的提示字符，其中的转义字符其实代表了不同的意义；
* $：本 shell 的 PID
* ？：关于上个执行命令的返回值

那如果想要将一个自定义的变量改为环境变量，使得子进程也能使用，则可以使用 export 来共享变量。这么一讲其实变量本身一定有它的使用范围。你可以将自定义变量理解为局部变量，**而当一个 shell 启动时，操作系统会专门分配一片内存区域给这个 shell 存放环境变量，而使用 export 可以让自定义变量加入到这片区域内。当加载另一个 shell 时，子 shell 可以将父 shell 的环境变量所在的内存区域导入到自己的环境变量区块中。**

那么还有哪些和变量有关的操作呢？比如我们希望从键盘中读取变量内容可以使用 read 命令；如果想要声明一个变量可以使用 declare 或者 typeset，当然如果想要使用一系列变量可以使用 array 类型。

我们的 bash 还可以限制用户的某些系统资源的数量，这需要使用 ulimit 命令。而更特殊的对变量进行删除、取代、替换的操作需要用到 # 符号和 % 符号等规则，这里不详细展开，用到的时候再补充。

## 别名

假设我们有一个很常用，但是有很长的参数的一个命令，每次都敲有点麻烦，那么在 bash 中有一个**命令别名设置**功能可以对某些命令的使用进行别名设置和简化。别名设置的方法是使用 alias：alias 别名=‘命令’：

```
alias lm='ls -al | more'
```

当然这个命令还可以设置为替换已有的命令，如果想要知道一共设置了多少别名，则直接键入 alias，而要取消某个别名的话使用 unalias。

## 历史命令

history 命令可以输出以前使用过的命令，通过设置参数可以控制输出格式，将历史缓存存入哪个文件等信息。一般而言 history 命令的读取和记录方法是：

* 当使用 bash 登录 Linux 主机之后，系统会自动地由家目录的 ~/.bash_history 读取以前执行的命令，而其中存储的数据和 HISTFILESIZE 这个变量有关。
* 而假设在该次执行过程中又新输入了一些命令，则会在注销时将最新的 HISTFILESIZE 条命令更新到记录文件中。
* 那么假设需要在运行时直接将某些指令刷新到历史命令存储的文件中，则使用 history -w 命令。

但是如果说将 root 下执行的很多重要命令记录在 ~/.bash_history 中，如果这个文件被解析了就会有很严重的后果。并且假设出现多用户登录的场景，那么就会出现文件覆盖问题，所以尽可能使用单一 bash 登录，再用任务管理来切换不同任务。最后一个问题就是历史命令是无法记录执行时间的，这一点是要通过 ~/.bash_logout 来进行 history 的记录。

## 操作环境

再谈一下执行一个命令的运行顺序，现在可以总结为如下：

1. 以相对路径或者绝对路径执行命令；
2. 由 alias 找到该命令来执行；
3. 由 bash 内置命令执行
4. 通过 $PATH 这个变量的顺序查找到第一个命令来执行。

讲道理，当你使用命令行的终端来登录到系统时，应该有过一些欢迎用语，那么其实有很多的终端你会发现都有自定义的欢迎用语。这些信息都存放在 /etc/issue 中(其中也有一些转移字符作为特殊用法，这个具体看说明)。那么如果你是使用的远程登录进入的系统，则需要使用 /etc/issue.net 文件。还有就是如果你想让登录后的用户得到一些信息，应该怎么做呢？可以把信息放入 /etc/motd 文件中。

### 配置文件

讲道理，在你一进入 bash 时，你什么都没操作，就已经获得了一系列的变量或者配置了，这显然是系统帮我们做的，因此系统肯定去全局配置文件中读了相应的配置。那么如果每个人都想要使用不同配置的 bash 怎么办呢？可以使用个人偏好配置文件。在 linux 中对于 bash 的配置文件很多，其中不同的配置文件对 login shell 和 non-login shell 还不一样。login shell 指的是有完整登录流程的，而 non-login shell 指的是不需要登录取得 bash。

*  /etc/profile：login shell 会读。这个文件使用了 UID 来决定很多变量数据，这是每个用户取得 bash 后必须读的文件。这里面会设置包括：PATH、MAIL、umask、HOSTNAME 等。当然里面还有一个比较重要的操作就是调用 /etc/profile.d 下的脚本文件，它们都是用来改变环境变量的。
* ~/.bash_profile：这是 login shell 选读的第一个文件，在这个文件中你可以选择性地修改 PATH，但是假设你修改了文件想要立刻生效而不注销，则可以使用 source 命令。如果该文件不存在，则 login shell 还会依次去读 ~/.bah_login 和 ~/.profile。
* ~/.bashrc： non-login shell 会读。这里你可以设置个人的配置。并且还会主动调用 /etc/bashrc。

最后可能还有用的文件就是一个 ~/.bash_logout。这个文件可以帮助你在注销 bash 后，系统帮助用户完成一些操作后离开。

最后对于终端而言，他还有一些快捷键，如果输入 stty -a 你就能看到所有系统设置好的快捷键。比如你会发现在使用 vim 的时候不小心按下了 ctrl+s 则会卡住不动，因为这是停止输出的意思。如果你需要修改快捷键可以使用类似：

```
stty stop ^h
```

除此之外还可以使用 set 设置一些终端值。

最后在 bash 中有许多的通配符和特殊符号能够使用，不展开讲具体遇到的时候再补充，这些符号都不能在文件名中遇到。

### 重定向

一般而言正常的数据流向应该是，用户从键盘输入数据，然后执行命令后将结果返回到屏幕上。那么假设用户现在不想从键盘输入数据，想把执行结果存起来。那么就需要用到数据重定向。

在 linux 中一共有三个数据流：stdin，stdout，stderr。先看看输出：

标准输出(stdout)指的是将程序正常的输出结果，标准错误输出(stderr)指的是程序执行失败后的输出。正常情况下，这两个数据都会打印在屏幕上所以很难区分。假设现在想将 stdout 存起来，存放在一个文件中，则只需要 `(1)> file` 或者 `(1)>> file` 前者在文件存在的情况下会覆盖目标文件，后者代表追加。而 1 是 stdout 的代码，实际在系统中是它固定的文件描述符。同理对于 stderr 使用 `(2) > file` 和 `(2) >> file` 进行重定向。这两个可以组合到一个命令中，但是要注意不能一次性重定向到一个文件中，**这会造成次序错误**。正确的做法是使用 `&> file` 。

现在我不希望屏幕上出现任何信息，也不想将信息存起来，而是丢弃不用。在 linux 中有一个 /dev/null 的垃圾桶，它会吃掉任何导向这个设备的信息。

标准输入(stdin)同样也能重定向，使用 `<` 或者 `<<`。它的目的是让指令的数据不通过键盘输入，而是从一个文件中键入。这里的 `<<` 不再是追加的意思，后面会跟上**结束输入的字符**。

### 执行判断

现在我希望一次性执行两个或者更多的命令，在 shell 脚本中确实能完成这个功能。但是如果在命令行中呢？下面有几种情形用来处理连续的指令执行。

```
cmd1;cmd2;cmd3
```

上面是将每一个指令用分号隔开，这种属于不考虑任何命令的相关性，执行一系列的命令，也就是说无论前后的命令是否执行成功，都不会影响接下去的命令执行。

```
cmd1 && cmd2
```

上面这个符号在编程中也遇到，&& 说明后者的执行依赖于前者，假设 cmd1 失败了，cmd2 也不会执行。

```
cmd1 || cmd2
```

上面这个说明就算前者执行失败了，后者也会执行，但是一旦前者执行成功了，就不会执行后者。

在 linux 中最麻烦的是这些逻辑组合起来，因为 linux 的命令必然是从左到右执行的，而这些逻辑的判断实际是通过命令的返回值，也就是 $? 的值。所以当逻辑组合的情况要注意组合的顺序！

## 管道

现在有这样一个场景，当目标输出需要对输入进行一系列的变换，最后才能得到目标的格式，那需要怎么处理呢？在 bash 中存在一个叫做管道的东西可以帮助我们。

```
cmd1 | cmd2
```

管道不是连续执行，管道的能力是将前一个命令的正确输出信息作为后一个命令的标准输入。这个说明 cmd2 一定是能够接收标准输入的命令，比如 more、less、head 等。标准错误信息会在这里直接被忽略。

基于管道的作用，linux 中有一系列的操作：

* cut：可以将一段信息的某一段切出来，处理的单位是行；
* grep：分析一行中的信息，如果存在想要的就拿出来；
* sort：按照某个字段对数据进行排序；
* uniq：对排序后的数据，取非重复的数据；
* wc：统计数据中有多少字，多少行；
* tee：双向重定向，可以同时将标准输出和标准错误输出导出；
* tr：删除数据中的某些信息，或者替换
* col：tab 和空格之间的转换
* join：将有相同字段的两个文件中的两行加在一起
* paste：直接将两行加在一起
* expand：将 tab 自定义转成空格
* split：将一个大文件划分为几个小文件
* xargs：该命令的目的是产生某个命令的参数，比如说像 ls 是不支持标准输入的，但是可以使用 xargs 构造输入供 ls 使用；
* -：减号在这里的作用是代表 stdin 和 stdout。