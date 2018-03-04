# xv6-learn

学习 xv6 代码

## Makefile

512=0x200
1k=1024=0x400
4k=4096=0x1000

硬盘的首个扇区叫做，主引导扇区（最后两个字节为 0x55 0xAA，用于验证确实是主引导扇区的代码）。BIOS 会把该扇区加载到内存 0x7C00 的地方。

**为什么硬盘的扇区是512字节？我感觉就是遵从一个原理，一开始设计成512个字节了，后来就遵循了512个字节。可能没有什么绝对的理由必须是512字节。**

Makefile 中的 bootblock 目标对象就是主引导扇区代码。
Makefile 中的 kernel 目标对象就是xv6操作系统内核的主体代码。（资源的管理）
Makefile 中的 fs.img 目标对象就是基于内核的用户态代码。（基于系统调用的用户程序）

涉及到的命令
- gcc
- gas
- ld
- objdump
- objcopy
- dd

### dd

dd 

选项
- if input file 指定输入流
- of ouput file 指定输出流
- ibs 默认块大小为512字节
- obs 默认块大小为512字节
- count 
- skip 读取跳过块
- seek 写入跳过块
- cbs 转换块大小

在传输过程中，数据可以用conv选项修改以适应介质。
- noerror选项意味着如果发生错误，程序也将继续运行。
- sync选项表示填充每个块到指定字节。
- 转换选项notrunc意味着不缩减输出文件，也就是说，如果输出文件已经存在，只改变指定的字节，然后退出，并保留输出文件的剩余部分。没有这个选项，dd将创建一个512字节长的文件。

（后缀w表示2倍，b表示512倍，k表示1024倍，M表示1024 × 1024倍，G表示1024 × 1024 × 1024倍）

### gcc

注意参考 `man gcc`

gcc 其实是一个挺复杂的命令。

file.c 
	c source code that must be preprocessed.
file.i
	c source code that should not be preproceesd.
file.ii
	c++ source code that should not be preproceesd.
file.h
	header file.

选项
-E Stop after the preprocessing stage; do not run the compile proper.
-s Stop after thr compilation proper; do not assemble.
-c compile or assemble the source files, but do bot link.
默认就是全部过程，预处理，编译，汇编，链接。

`CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -O2 -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer`

**需要仔细的研究一下。**

- `-fno-pic` 

> -fPIC 作用于编译阶段，告诉编译器产生与位置无关代码(Position-Independent Code)，
> 则产生的代码中，没有绝对地址，全部使用相对地址，故而代码可以被加载器加载到内存的任意
> 位置，都可以正确的执行。这正是共享库所要求的，共享库被加载时，在内存的位置不是固定的。

- `-static`
- `-fno-builtin`
- `-fno-strict-aliasing`

`-I` 搜索头文件的目录

### ld

`-m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)`
`$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o`

- `-e start` 程序执行的入口 符号。
- `-Ttext 0x7C00` same as `--section-start` 指定程序链接后的起始地址。

### objdump

`-S` 反汇编

此处用来反汇编。应该是为了调试或者教学。

### objcopy copy and translate object files.

此处用来提取目标文件中的代码段到 bootblock（主引导扇区二进制代码）

### `./sign.pl` 设置

判断 bootblock 的大小。设置 格式。（0x55 0xAA）

## 主引导扇区 bootblock

bootmain.c bootasm.S => bootblock

## kernel xv6 内核代码

## fs.img 用户态程序




