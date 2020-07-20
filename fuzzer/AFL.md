# AFL

AFL：American Fuzzy Lop 是一款开源的fuzzing小工具，是一个使用暴力法的fuzzer。

###### INTRODUCTION

这里的操作分为对于有源码的程序操作和无源码的程序操作。

对于有源码的程序中，原本程序运行需要使用gcc或者g++先进行编译，然后在安装执行。那么一般大型的源程序软件为了方便都会编写makefile文件帮助更快捷的编译。而使用afl需要在源程序在编译过程中进行插桩，从而帮助它完成后续工作，因此需要修改其中使用的编译器为afl_gcc和afl_g++，这两个程序应该是对于gcc和g++的一次封装。

```
CC=/path/to/afl/afl-gcc ./configure
```

对于无源码的程序，则这个相当于是一个黑盒测试，这个需要通过qemu的用户级仿真实现，在afl中实现了这个功能。但是需要编译qemu。不过对于afl来说，测试无源码的程序的效率通常会比测试有源码的程序的效率低，因此使用这个软件最好测试有源码的软件比较合适。

然后有了目标程序，需要提供一个文件夹专门用作输入的文件（参数）来使用，还有一个文件夹用作输出文件来使用。然后我们就可以启动fuzz工具了。选取输入的时候有一些限制，**比如最好让输入的大小控制在1K以下，并且对于同一个功能的软件，只需要给一个测试用例就行了。**

对于使用stdin来作为参数输入的命令：

```
./afl-fuzz -i testcase_dir -o findings_dir /path/to/program […params…]
```

对于使用文件读取作为目标程序的命令：

```
./afl-fuzz -i testcase_dir -o findings_dir /path/to/program @@
```

可以指定 -f 参数来指定将修改过的输入输出到某个文件中。

在进行 fuzz 的过程中，会有输出信息输出在三个子文件夹中：

* queue：记录了所有生成的不同的输入；
* crashes：使得程序产生错误信号；
* hangs：使得程序运行超时的输入

这里再多说几句，为了提高 Fuzz 的性能，它支持 Parallel Fuzz，也就是同时可以多个核一起同时 Fuzz，下面在具体研究的时候我们再来看。

AFL 的修改策略让它很不适合去测试一些语法相关的程序，因此 AFL 提供了一种**字典**功能。它需要在某个文件夹下添加一个字典文件，然后在 fuzz 时使用 -x 参数。(也可以使用 libtokencap)

AFL -c 参数支持 crash exploration 模式。这种模式下 AFL 会从 crash 的种子中挑选一个运行，然后让程序保持在 crash 的状态。



###### ALGORITHM

afl这个工具使用的算法如下：

1. 将用户给的初始测试用例加载入队列中；
2. 取走队列中下一个需要使用的输入文件；
3. 尝试将测试用例缩减为最小，去除那些并不会改变程序执行的东西；
4. 使用一个传统fuzzing对策来改动文件；
5. 如果变动引起了一个新的状态变化，就将该分支记录下来，随后将这个这个变动重新插入队列中；
6. 回到第二步重复执行。

###### DETAIL

如果是开源的程序，那么需要对程序进行插装，主要在程序的分支处插入桩代码，这一步是在将源文件转化为汇编语言后完成的，使用了afl-as这个汇编编译工具。

那么在分支处插桩后，就可以捕获分支的覆盖率，也可以检测到粗略的分支命中率。代码覆盖率可以用来评估和改进测试过程，意思是说执行到的代码越多，那么找到bug的可能性也越大。那么对于代码覆盖率而言，一般会提供函数级别，代码基本块级别（代码基本块可以理解为编程的时候一个花括号内的内容，他要求只有一个入口，只有一个出口）和边界级别（控制流图中每个节点就是一个代码基本块，边就是跳转的连线）三种级别的覆盖率检测。那么afl中使用的就是边界级别的代码覆盖率。具体到afl中它使用了一个二元组来记录当前基本块+前一基本块的信息，来获取执行流程和代码覆盖情况。

A -> B -> C -> D -> E (tuples: AB, BC, CD, DE)

AFL如果发现了一个新的路径会怎么做？它是用了一个全局的bitmap来保存了之前执行时所遇到的路径，

```
static const u8 count_class_lookup8[256] = {

  [0]           = 0, 
  [1]           = 1, 
  [2]           = 2, 
  [3]           = 4, 
  [4 ... 7]     = 8, 
  [8 ... 15]    = 16,
  [16 ... 31]   = 32,
  [32 ... 127]  = 64,
  [128 ... 255] = 128

};
```

afl将代码执行次数映射为一个8bit的数据，然后存入上述的桶中。如果某一次代码运行的过程中，出现了桶到桶之间的跳跃，则认为是一个感兴趣的数据。

接下来看输入文件是如何变异的，对于一个输入文件，则变异的过程是暴力的，其中包括对源文件的按位翻转，整数的加减，向原文件中添加入特殊内容，添加用户自定义或者系统自动生成的标记。

然后看一下多线程的操作。现在对于fuzz的流程有了一个大致的了解了，最后这个和AFL fuzz的过程的性能有关。对于afl来说，它本身有一个叫做fork server的。那么当启动fuzz的时候，这个fuzzer会和这个server进行通信，由这个server fork出多个进程，并且通过写时拷贝技术将已经停止的fuzz进程直接进行拷贝。

---

### 代码分析：

###### afl-gcc：

对于这个程序的main函数，一共做了几个操作：

首先是常规的判断，判断设备合法性，判断参数合法性，随后开始分别调用了find_as，edit_params，和execvp(cc_params[0], (char** cc_params))这三个操作。

find_as的目的顾名思义，就是从系统中查找汇编编译器，一共需要查看三个地方，分别是环境变量中，指定路径中，或者是指定的AFL_PATH变量中。

edit_params是编辑了参数之后喂给系统调用execvp来最终执行这个命令。其中有一步操作，gcc -B <目录>            将 <目录> 添加到编译器的搜索路径中。这样就可以指定汇编编译器，从而使用自己的编译器完成插桩。

###### afl-as

这个程序就是和编译汇编有点关系了，在afl-gcc中制定了编译程序的地址。这个程序比上面的gcc要庞大，先从main函数结构入手了。在经过了一些对于参数的判断之后，函数的一开始是通过当前的时间以及当前的进程号来设定了一个随机种子。然后同样需要一个edit_params的函数来编辑一下参数形式。接下去会进行一系列全局变量的设置，随后进入一个分支，如果在编辑参数时发现是调用了--version则进行一个分支操作，否则会调用一个叫做add_instrumentation的函数，然后就fork出一个新的进程来执行编译的操作了。这个add_instrumentation函数在做什么呢？

这一步就是在进行插桩，而插桩的内容就在afl-as.h中，它获取需要使用汇编编译的输入文件，然后将该文件读入一个临时文件中，而在这个文件中，**在开始阶段会插入一部分的桩汇编代码，同时在每个分支处也会插入桩代码，然后就使用as去编译该临时文件来生成.o文件**，从而完成了所谓的插桩。具体的插桩到底它做了什么这个需要具体再看一下代码。

具体以下面的例子来看：

```c
#include<stdio.h>
int main() {
    printf("hello world");
    return 0;
}
```

as 的目的是是要给程序中每一个可能的执行分支都要注入一个叫做 afl_maybe_log 的函数，另外就是在整个文件最后注入一个庞大的 afl 自己编写的 payload，里面有着很多可能触发到的函数。(实际目前来看，这个 afl_maybe_log 可能被注入在函数开头和所有有关跳转(jm开头的指令)后)，这样可以捕获控制流。

**AFL_MAYBE_LOG**

```assembly
00000000004007f0 <__afl_maybe_log>:
  4007f0:	9f                   	lahf   
  4007f1:	0f 90 c0             	seto   %al
  4007f4:	48 8b 15 85 08 20 00 	mov    0x200885(%rip),%rdx        # 601080 <__afl_area_ptr>
  4007fb:	48 85 d2             	test   %rdx,%rdx
  4007fe:	74 20                	je     400820 <__afl_setup>
```

可以看到如果是刚进入这个程序，afl_area_ptr 这个指针肯定是空的，则进行 afl_setup 的操作。我们这里先不看初始化，如果已经有了指针的值，则直接触发 afl_store 函数：

**AFL_STORE**

```
0000000000400800 <__afl_store>:
  400800:	48 33 0d 81 08 20 00 	xor    0x200881(%rip),%rcx        # 601088 <__afl_prev_loc>  // 一开始 afl_prev_loc 为 0，但是其实代表以前的地址
  400807:	48 31 0d 7a 08 20 00 	xor    %rcx,0x20087a(%rip)        # 601088 <__afl_prev_loc>  // 此时的意思是 afl_prev_loc = rcx
  40080e:	48 d1 2d 73 08 20 00 	shrq   0x200873(%rip)        # 601088 <__afl_prev_loc> // 右移
  400815:	fe 04 0a             	incb   (%rdx,%rcx,1)  // share_men 加 1
```

这一段就是 AFL 的重要的一段计算 hash 的代码(从中看出，afl_area_ptr 的含义就是保存 Coverage 的 上 map)，为什么是 64K，**其实是在 AFL_fuzz 文件中开辟共享内存时规定了大小**。因为 rcx 初始值是 16 位的，在下面运算中可以发现没有办法改变 rcx 最大是 16 位的事实，所以，为什么会这样，因为在 add_instrument 中能够看到下面这段代码：

```c
cur_location = <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++; 
prev_location = cur_location >> 1;

// in add_instrument
fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32,
             R(MAP_SIZE));
```

在 AFL_STORE 后面就直接返回了。

**AFL_SETUP**

```
0000000000400820 <__afl_setup>:
  400820:	80 3d 71 08 20 00 00 	cmpb   $0x0,0x200871(%rip)        # 601098 <__afl_setup_failure> // 如果失败
  400827:	75 ef                	jne    400818 <__afl_return>
  400829:	48 8d 15 70 08 20 00 	lea    0x200870(%rip),%rdx        # 6010a0 <__afl_global_area_ptr> // 共享内存的地址
  400830:	48 8b 12             	mov    (%rdx),%rdx
  400833:	48 85 d2             	test   %rdx,%rdx
  400836:	74 09                	je     400841 <__afl_setup_first>
  400838:	48 89 15 41 08 20 00 	mov    %rdx,0x200841(%rip)        # 601080 <__afl_area_ptr> // 这边可以看到，当走完这个分支，afl_area_ptr 指针被赋值了
  40083f:	eb bf                	jmp    400800 <__afl_store> // 初始化后跳回 afl_store
```

接着看 afl_setup_first

```
0000000000400841 <__afl_setup_first>:
  400841:	48 8d a4 24 a0 fe ff 	lea    -0x160(%rsp),%rsp
  400848:	ff 
  400849:	48 89 04 24          	mov    %rax,(%rsp)
  40084d:	48 89 4c 24 08       	mov    %rcx,0x8(%rsp)
  400852:	48 89 7c 24 10       	mov    %rdi,0x10(%rsp)
  400857:	48 89 74 24 20       	mov    %rsi,0x20(%rsp)
  40085c:	4c 89 44 24 28       	mov    %r8,0x28(%rsp)
  400861:	4c 89 4c 24 30       	mov    %r9,0x30(%rsp)
  400866:	4c 89 54 24 38       	mov    %r10,0x38(%rsp)
  40086b:	4c 89 5c 24 40       	mov    %r11,0x40(%rsp)
  400870:	66 0f d6 44 24 60    	movq   %xmm0,0x60(%rsp)
  400876:	66 0f d6 4c 24 70    	movq   %xmm1,0x70(%rsp)
  40087c:	66 0f d6 94 24 80 00 	movq   %xmm2,0x80(%rsp)
  400883:	00 00 
  400885:	66 0f d6 9c 24 90 00 	movq   %xmm3,0x90(%rsp)
  40088c:	00 00 
  40088e:	66 0f d6 a4 24 a0 00 	movq   %xmm4,0xa0(%rsp)
  400895:	00 00 
  400897:	66 0f d6 ac 24 b0 00 	movq   %xmm5,0xb0(%rsp)
  40089e:	00 00 
  4008a0:	66 0f d6 b4 24 c0 00 	movq   %xmm6,0xc0(%rsp)
  4008a7:	00 00 
  4008a9:	66 0f d6 bc 24 d0 00 	movq   %xmm7,0xd0(%rsp)
  4008b0:	00 00 
  4008b2:	66 44 0f d6 84 24 e0 	movq   %xmm8,0xe0(%rsp)
  4008b9:	00 00 00 
  4008bc:	66 44 0f d6 8c 24 f0 	movq   %xmm9,0xf0(%rsp)
  4008c3:	00 00 00 
  4008c6:	66 44 0f d6 94 24 00 	movq   %xmm10,0x100(%rsp)
  4008cd:	01 00 00 
  4008d0:	66 44 0f d6 9c 24 10 	movq   %xmm11,0x110(%rsp)
  4008d7:	01 00 00 
  4008da:	66 44 0f d6 a4 24 20 	movq   %xmm12,0x120(%rsp)
  4008e1:	01 00 00 
  4008e4:	66 44 0f d6 ac 24 30 	movq   %xmm13,0x130(%rsp)
  4008eb:	01 00 00 
  4008ee:	66 44 0f d6 b4 24 40 	movq   %xmm14,0x140(%rsp)
  4008f5:	01 00 00 
  4008f8:	66 44 0f d6 bc 24 50 	movq   %xmm15,0x150(%rsp)
  4008ff:	01 00 00                // 这一行朝上，可以看得出这里把所有的寄存器状态都压到栈中了
  400902:	41 54                	push   %r12 // 这个 r12 很像 rbp 指针
  400904:	49 89 e4             	mov    %rsp,%r12
  400907:	48 83 ec 10          	sub    $0x10,%rsp
  40090b:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
  40090f:	48 8d 3d c1 02 00 00 	lea    0x2c1(%rip),%rdi        # 400bd7  <.AFL_SHM_ENV>  // 这个在后面知道其实是保存了共享内存的标识符
  400916:	e8 d5 fc ff ff       	callq  4005f0 <getenv@plt>
  40091b:	48 85 c0             	test   %rax,%rax
  40091e:	0f 84 e2 01 00 00    	je     400b06 <__afl_setup_abort>
  400924:	48 89 c7             	mov    %rax,%rdi
  400927:	e8 54 fd ff ff       	callq  400680 <atoi@plt>
  40092c:	48 31 d2             	xor    %rdx,%rdx
  40092f:	48 31 f6             	xor    %rsi,%rsi
  400932:	48 89 c7             	mov    %rax,%rdi
  400935:	e8 36 fd ff ff       	callq  400670 <shmat@plt> // 调用 shmat，产生与共享内存的映射
  40093a:	48 83 f8 ff          	cmp    $0xffffffffffffffff,%rax
  40093e:	0f 84 c2 01 00 00    	je     400b06 <__afl_setup_abort>
  400944:	48 89 c2             	mov    %rax,%rdx  // 这里 rax 很像是 shmat 的返回值，返回的是附加好的共享内存地址
  400947:	48 89 05 32 07 20 00 	mov    %rax,0x200732(%rip)        # 601080 <__afl_area_ptr>
  40094e:	48 8d 15 4b 07 20 00 	lea    0x20074b(%rip),%rdx        # 6010a0 <__afl_global_area_ptr>  
  400955:	48 89 02             	mov    %rax,(%rdx) // 到这里意思是全局指针指向的是共享内存的地址
  400958:	48 89 c2             	mov    %rax,%rdx // 结合一下 AFL_STORE

```

下面的代码是 AFL 的 forkserver 技术(下面暂时有些信息还不得而知)：

**AFL_FORK**

```
000000000040095b <__afl_forkserver>:
  40095b:	52                   	push   %rdx
  40095c:	52                   	push   %rdx
  40095d:	48 c7 c2 04 00 00 00 	mov    $0x4,%rdx
  400964:	48 8d 35 29 07 20 00 	lea    0x200729(%rip),%rsi        # 601094 <__afl_temp>
  40096b:	48 c7 c7 c7 00 00 00 	mov    $0xc7,%rdi
  400972:	e8 99 fc ff ff       	callq  400610 <write@plt>
  400977:	48 83 f8 04          	cmp    $0x4,%rax
  40097b:	0f 85 98 00 00 00    	jne    400a19 <__afl_fork_resume>
```

**AFL_WAIT_LOOP**

```
000000000400981 <__afl_fork_wait_loop>:
  400981:	48 c7 c2 04 00 00 00 	mov    $0x4,%rdx
  400988:	48 8d 35 05 07 20 00 	lea    0x200705(%rip),%rsi        # 601094 <__afl_temp>
  40098f:	48 c7 c7 c6 00 00 00 	mov    $0xc6,%rdi
  400996:	e8 a5 fc ff ff       	callq  400640 <read@plt>
  40099b:	48 83 f8 04          	cmp    $0x4,%rax
  40099f:	0f 85 59 01 00 00    	jne    400afe <__afl_die>
  4009a5:	e8 e6 fc ff ff       	callq  400690 <fork@plt>
  4009aa:	48 83 f8 00          	cmp    $0x0,%rax
  4009ae:	0f 8c 4a 01 00 00    	jl     400afe <__afl_die> // fork 失败
  4009b4:	74 63                	je     400a19 <__afl_fork_resume>
  4009b6:	89 05 d4 06 20 00    	mov    %eax,0x2006d4(%rip)        # 601090 <__afl_fork_pid>
  4009bc:	48 c7 c2 04 00 00 00 	mov    $0x4,%rdx
  4009c3:	48 8d 35 c6 06 20 00 	lea    0x2006c6(%rip),%rsi        # 601090 <__afl_fork_pid>
  4009ca:	48 c7 c7 c7 00 00 00 	mov    $0xc7,%rdi
  4009d1:	e8 3a fc ff ff       	callq  400610 <write@plt>
  4009d6:	48 c7 c2 00 00 00 00 	mov    $0x0,%rdx
  4009dd:	48 8d 35 b0 06 20 00 	lea    0x2006b0(%rip),%rsi        # 601094 <__afl_temp>
  4009e4:	48 8b 3d a5 06 20 00 	mov    0x2006a5(%rip),%rdi        # 601090 <__afl_fork_pid>
  4009eb:	e8 70 fc ff ff       	callq  400660 <waitpid@plt>
  4009f0:	48 83 f8 00          	cmp    $0x0,%rax
  4009f4:	0f 8e 04 01 00 00    	jle    400afe <__afl_die>
  4009fa:	48 c7 c2 04 00 00 00 	mov    $0x4,%rdx
  400a01:	48 8d 35 8c 06 20 00 	lea    0x20068c(%rip),%rsi        # 601094 <__afl_temp>
  400a08:	48 c7 c7 c7 00 00 00 	mov    $0xc7,%rdi
  400a0f:	e8 fc fb ff ff       	callq  400610 <write@plt>
  400a14:	e9 68 ff ff ff       	jmpq   400981 <__afl_fork_wait_loop>
```

afl_fork_resume 没什么说的，就是处理堆栈，把现场还原

---

###### afl-fuzz

这应该是这个程序的主入口了，首先有一个大的while循环用来从命令行获取参数，使用的是unistd.h的getopt函数。下面来看一下各个选项在这个它的大main函数里是怎么操作的：

* i选项：全局变量中in_dir表示了输入数据存放的文件夹，在这个case中进行赋值。
* o选项：全局变量中out_dir表示了输出数据存放的文件夹，在这个case中进行赋值。
* M和S选项： 设置masterid，也就是全局变量sync_id，这个选项和分布式模式有关，可能用来进行分布式的fuzz，具体需要查看文件parallel fuzzing.txt这个文件。
* f选项：设置一个全局变量out_file，功能我们后面来看
* x选项：设置读入的字典的位置extras_dir，这个是afl特有的一个功能，它可以由使用者指定输入的种子通过用户指定的字典来修改。
* t选项：设置延时，用户在这个后面添加对应的msec时间。
* m选项：设置每个子进程的内存限制，最大50M。
* d选项：暂时不知道是什么作用。
* B选项：这个选项在说明书中并没有展示，这个可以加载在fuzz过程中之前遇到的某个bitmap，然后使用它来进行修改再进行fuzz。
* C选项：crash模式。
* n选项：dumb模式。
* T选项：设置text banner语句在屏幕上，具体有什么用还不得而知。
* Q选项：使用qemu模式，也就是进行闭源二进制测试时使用。

这个函数真能正要做的第一步就是设置信号（setup_signal_handlers）,这个函数设置不同信号触发不同的信号句柄，使用的是signation。然后第二步是使用check_asan_opts()函数，似乎在查看是否在环境变量中设置了asan和msan的这两个东西。如果使用了-S选项，即使用了分布式模式，则需要对该模式的输出文件夹路径另外进行设置。

接下去程序设置了一堆有关环境变量的东西，随后保存了命令行输入命令的一个副本。然后设置一个叫banner的东西，并且检测程序是否跑在了终端上。然后获取cpu的核数，等等都是一系列在运行之前的必备检查。其中read_testcases就把所有输入队列中的文件添加到了一个队列当中，随后就是一个大的while循环。初步的猜测，在cull_queue中会消耗输入队列中的一个用例，然后使用这个用例去跑相应的程序，如果在输入队列中还有东西，就通过show_stats去刷新并且显示一些数据，如果发现已经到头了，那么就要改变策略了。（这个函数可能要通过实例才能具体得知里面的细节。）

###### afl-analyze

这个工具是用来分析要使用的用例的，这个很有意思，使用的方法依旧是afl-analyze [选项] programme。

* i选项：选择输入文件的文件名，也就是某个测试文件。
* f选项：选择该programme需要使用的文件。
* e选项：只关注边的覆盖率，但是不关注边击中的次数；
* m选项：限制内存；
* t选项：限制时间；
* Q选项：qemu模式；

这个函数的总体的做法是使用了read_initial_file函数来读取相应的文件内容，随后使用了run_target函数来运行相应的程序，随后调用analyze函数对输入的文件分析。

###### afl-tmin

这个工具的作用是对输入进去的用例进行简化，在涉及的参数上，和下面的showmap和analyze类似，其中最主要的一个函数是minimize函数，这个函数就是简化了使用的实例。

###### afl-showmap

这个函数的功能是显示被afl捕获的原始数据，总体的使用是afl-showmap [选项] programme。在这个函数中依旧有一个解析命令行的函数，稍微来看一下功能。

* o选项：选择输出的文件名，将元祖信息输出到这个文件中。
* m选项：限制内存
* t选项：限制时间
* q选项：不显示信息
* e选项：值显示边，不现实对应的击中次数。
* Q选项：qemu模式
* b选项（隐藏项）：用二进制格式写入文件
* Z和A选项（隐藏项）：在cmin中会用到

这个main函数在前面依旧是设置许多的参数并且对一些功能进行检查，随后的主要的函数是run_target函数，然后把结果通过write_results函数写入文件，因此这两个函数是这个里面的核心。