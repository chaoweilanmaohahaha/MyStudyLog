# objdump 的使用

这个工具是 Linux 自带的一个反汇编工具，它是用来对 Linux 上的可执行文件进行反汇编的。

我只好从网上截取一下关于 objdump 的使用，其实 objdump 的一个很大的作用是查看可执行文件每一个区块的虚拟地址的分布情况，从而知道加载到内存之后，每一个指令的虚拟地址究竟是多少。

```
--archive-headers 
-a 
显示档案库的成员信息,类似ls -l将lib*.a的信息列出。 
 
-b bfdname 
--target=bfdname 
指定目标码格式。这不是必须的，objdump能自动识别许多格式，比如： 
 
objdump -b oasys -m vax -h fu.o 
显示fu.o的头部摘要信息，明确指出该文件是Vax系统下用Oasys编译器生成的目标文件。objdump -i将给出这里可以指定的目标码格式列表。 
 
-C 
--demangle 
将底层的符号名解码成用户级名字，除了去掉所开头的下划线之外，还使得C++函数名以可理解的方式显示出来。 
 
--debugging 
-g 
显示调试信息。企图解析保存在文件中的调试信息并以C语言的语法显示出来。仅仅支持某些类型的调试信息。有些其他的格式被readelf -w支持。 
 
-e 
--debugging-tags 
类似-g选项，但是生成的信息是和ctags工具相兼容的格式。 
 
--disassemble 
-d 
从objfile中反汇编那些特定指令机器码的section。 
 
-D 
--disassemble-all 
与 -d 类似，但反汇编所有section. 
 
--prefix-addresses 
反汇编的时候，显示每一行的完整地址。这是一种比较老的反汇编格式。 
 
-EB 
-EL 
--endian={big|little} 
指定目标文件的小端。这个项将影响反汇编出来的指令。在反汇编的文件没描述小端信息的时候用。例如S-records. 
 
-f 
--file-headers 
显示objfile中每个文件的整体头部摘要信息。 
 
-h 
--section-headers 
--headers 
显示目标文件各个section的头部摘要信息。 
 
-H 
--help 
简短的帮助信息。 
 
-i 
--info 
显示对于 -b 或者 -m 选项可用的架构和目标格式列表。 
 
-j name
--section=name 
仅仅显示指定名称为name的section的信息 
 
-l
--line-numbers 
用文件名和行号标注相应的目标代码，仅仅和-d、-D或者-r一起使用使用-ld和使用-d的区别不是很大，在源码级调试的时候有用，要求编译时使用了-g之类的调试编译选项。 
 
-m machine 
--architecture=machine 
指定反汇编目标文件时使用的架构，当待反汇编文件本身没描述架构信息的时候(比如S-records)，这个选项很有用。可以用-i选项列出这里能够指定的架构. 
 
--reloc 
-r 
显示文件的重定位入口。如果和-d或者-D一起使用，重定位部分以反汇编后的格式显示出来。 
 
--dynamic-reloc 
-R 
显示文件的动态重定位入口，仅仅对于动态目标文件意义，比如某些共享库。 
 
-s 
--full-contents 
显示指定section的完整内容。默认所有的非空section都会被显示。 
 
-S 
--source 
尽可能反汇编出源代码，尤其当编译的时候指定了-g这种调试参数时，效果比较明显。隐含了-d参数。 
 
--show-raw-insn 
反汇编的时候，显示每条汇编指令对应的机器码，如不指定--prefix-addresses，这将是缺省选项。 
 
--no-show-raw-insn 
反汇编时，不显示汇编指令的机器码，如不指定--prefix-addresses，这将是缺省选项。 
 
--start-address=address 
从指定地址开始显示数据，该选项影响-d、-r和-s选项的输出。 
 
--stop-address=address 
显示数据直到指定地址为止，该项影响-d、-r和-s选项的输出。 
 
-t 
--syms 
显示文件的符号表入口。类似于nm -s提供的信息 
 
-T 
--dynamic-syms 
显示文件的动态符号表入口，仅仅对动态目标文件意义，比如某些共享库。它显示的信息类似于 nm -D|--dynamic 显示的信息。 
 
-V 
--version 
版本信息 
 
--all-headers 
-x 
显示所可用的头信息，包括符号表、重定位入口。-x 等价于-a -f -h -r -t 同时指定。 
 
-z 
--disassemble-zeroes 
一般反汇编输出将省略大块的零，该选项使得这些零块也被反汇编。 
 
@file 
可以将选项集中到一个文件中，然后使用这个@file选项载入。
```

在 Linux 中，可执行程序一般是 ELF 格式，可以类比在 Windows 中的 PE 格式，那么 objdump 在做逆向分析的时候有用的一点就是能够知道目标可执行程序的 ELF 区代码、数据与地址，对后续 patch 目标程序做准备。