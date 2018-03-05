# bootblock

```
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock
```

仔细阅读 bootasm.S 和 bootmain.c 代码。

xv6 是支持多核CPU的，但是启动时是从单个CPU开始。（应该会切换到多CPU模式）
从16位实模式，经过短暂的16位保护模式，到32位保护模式。（保护模式区别于实模式，增加了内存保护，虚拟内存，多任务处理等）

## 代码阅读

学习汇编，汇编的格式分为`Intel 格式`和`AT&T 格式`
- https://www.ibm.com/developerworks/cn/linux/l-assembly/index.html

`.code16` 伪指令，指示下面代码按16位模式编译。 （需要在仔细看一下）
`.globl start` 指定程序的入口点。 （需要在仔细看一下，看看Intel格式有没有什么对应）
`start:` #（需要在仔细看一下）
`cli` 禁止 BIOS 中断。#（需要在仔细看一下）

Zero data segment registers DS, ES, and SS.

```
#include "asm.h" # (需要了解一下汇编语言中的 #include)
#include "memlayout.h"
#include "mmu.h"

# Start the first CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.code16                       # Assemble for 16-bit mode
.globl start
start:
  cli                         # BIOS enabled interrupts; disable

  # Zero data segment registers DS, ES, and SS. #（这里没有初始化CS，因为CS直接由CPU初始化。）
  xorw    %ax,%ax             # Set %ax to zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment

  # Physical address line A20 is tied to zero so that the first PCs #（早期的电脑，只有20根地址总线，最大支持1 MB物理内存寻址。）
  # with 2 MB would run software that assumed 1 MB.  Undo that. #（通过一致保持A20为0，来兼容早期的电脑。这里要开启A20地址线）
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
#（这里采用的方式和李忠的书上不一样的方式，应该是采用的键盘上的那种方式，兼容性更好。需要仔细看一下）
  # Switch from real to protected mode.  Use a bootstrap GDT that makes #（实模式到保护模式的过程）
  # virtual addresses map directly to physical addresses so that the #（GDT，cr0 控制寄存器？）
  # effective memory map doesn't change during the transition.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0

//PAGEBREAK!
  # Complete the transition to 32-bit protected mode by using a long jmp
  # to reload %cs and %eip.  The segment descriptors are set up with no
  # translation, so that the mapping is still the identity mapping.
  ljmp    $(SEG_KCODE<<3), $start32 （long jmp，为啥long jmp来，切换到保护模式下的CS段描述符和EIP）

.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain

  # If bootmain returns (it shouldn't), trigger a Bochs
  # breakpoint if running under Bochs, then loop.
  movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
  movw    %ax, %dx
  outw    %ax, %dx
  movw    $0x8ae0, %ax            # 0x8ae0 -> port 0x8a00
  outw    %ax, %dx
spin:
  jmp     spin

# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg

gdtdesc: #
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1 （这里为啥要减 1 ？记得说过，好像就是要求长度减 1）
  .long   gdt                             # address gdt

```