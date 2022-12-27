---
title: 基于xv6 riscv实现学习os 其零：helloworld
typora-root-url: ../../source
date: 2022-11-13 21:32:18
category: os
tags: 
  - [xv6-riscv]
  - [mini-riscv-os]
  - [bootloader]
  - [riscv]
---

![30933181](/images/xv6-riscv-os-0/30933181.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">学习os的时间开始了! pixiv:30933181</center> 

# 前言

这个系列的目的还是以讲解xv6-riscv的代码以及记录我在做的事情为主，也会掺杂许多mini-riscv-os的代码介绍（关于xv6-riscv和mini-riscv-os的链接请看参考），并非教程倾向（但也会尽可能讲解一些基础知识），**很多细节不会讲到**。如果想要更详细的教程我建议你查看参考资料中引用的内容，在这一期我会列出一部参考的项目。

compiler的坑还没走多远，我又要开新的坑了，这是我很久之前想做但不敢做的事情。以前也做过一些尝试，比如说《30天自制os》以及6.828，前者讲的相对比较容易理解一些，但是当时的我缺少实践，后者难度较高，看不懂课后习题只能去查看别人的实现，东抄西抄总算抄完了前四章的内容，最后只留下了一些概念的印象。对我来说学会什么东西只有通过具体去实现，自己很难从什么概念去理解某个东西，也因此之前学的很多知识其实都是非常肤浅无用的。

实现os这件事情看起来是挺吓人的，本身复杂的概念和各种实现，同时需要许多前置知识。同时大多数os的开始都是离谱的x86bootloader，我想这个应该劝退了非常多的人。但最近发现做一个最简易的os或许并没有那么可怕，搜了一些项目，最简单的功能很少的系统只有一两千行代码，相对比较容易学习，同时riscv的bootloader部分没有乱七八糟的历史遗留，十分简洁，不会再因为这个劝退别人。也在这里感谢他们的付出。

# 注意事项

1. mini-riscv-os是针对riscv32，而xv6针对的是riscv64，导致一些汇编上、编译选项以及一些其他的内容会有所不同
2. 代码引用会直接使用项目名/路径的格式

此后不再赘述

# 环境配置

## 交叉编译工具链

参考链接

[https://pdos.csail.mit.edu/6.828/2019/tools.html](https://pdos.csail.mit.edu/6.828/2019/tools.html)

我是在mac（M1）下开发的，homebrew在安装riscv-tools的时候会提示需要安装一些依赖。在我配置的时候遇到了flock这个依赖搞不定的问题，发现直接brew install flock安装的flock是其他东西，因此需要卸载flock并且使用brew tap的命令，安装好依赖再去按riscv-tools

```cpp
brew uninstall flock
brew tap discoteq/discoteq
brew install flock
brew install riscv-tools
```

## qemu

这个没什么好说的，直接用包管理安装就是

# 启动所需代码

## bootloader

### 做了什么

1. 设置栈的起始地址
2. 跳转到c代码中

### 代码

mini-riscv-os/01-HelloOs/start.s

```assembly
.equ STACK_SIZE, 8192

.global _start

_start:
    # setup stacks per hart
    csrr t0, mhartid                # read current hart id
    slli t0, t0, 10                 # shift left the hart id by 1024
    la   sp, stacks + STACK_SIZE    # set the initial stack pointer 
                                    # to the end of the stack space
    add  sp, sp, t0                 # move the current hart stack pointer
                                    # to its place in the stack space

    # park harts with id != 0
    csrr a0, mhartid                # read current hart id
    bnez a0, park                   # if we're not on the hart 0
                                    # we park the hart

    j    os_main                    # hart 0 jump to c

park:
    wfi
    j park

stacks:
    .skip STACK_SIZE * 4            # allocate space for the harts stacks
```

csrr是从csr（Control and Status Register）寄存器中read值，而其中的csrr reg, mhartid则是将hart id读到对应的reg中。hart是riscv中硬件线程的最小单位，在riscv的spec中是这样描述的

> A RISC-V compatible core might support multiple RISC-V-compatible hardware threads, or harts, through
> multithreading.

这里的代码判断如果hart id不是0就跳到park这个循环中。实质上是只开启了一个hart

xv6-riscv/kernel/entry.S

```assembly
# qemu -kernel loads the kernel at 0x80000000
        # and causes each hart (i.e. CPU) to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
.global _entry
_entry:
        # set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        # jump to start() in start.c
        call start
spin:
        j spin
```

xv6的启动代码中考虑了多个hart启动的情况，给每一个hard都设置stack的起始地址。而stack的起始地址是写在其他的c代码中

```c
// entry.S needs one stack per CPU.
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];
```

## c代码

在c代码中打印出一个血统纯正的helloworld。这里其实隐含了很多的内容，但是暂且知道这样做就可以打印出helloworld即可。

对于xv6来说在进入os的main之前有许多设置状态的内容，这里暂且不讨论。

mini-riscv-os/01-HelloOs/os.c

```cpp
#include <stdint.h>

#define UART        0x10000000
#define UART_THR    (uint8_t*)(UART+0x00) // THR:transmitter holding register
#define UART_LSR    (uint8_t*)(UART+0x05) // LSR:line status register
#define UART_LSR_EMPTY_MASK 0x40          // LSR Bit 6: Transmitter empty; both the THR and LSR are empty

int lib_putc(char ch) {
	while ((*UART_LSR & UART_LSR_EMPTY_MASK) == 0);
	return *UART_THR = ch;
}

void lib_puts(char *s) {
	while (*s) lib_putc(*s++);
}

int os_main(void)
{
	lib_puts("hello, world\n");
	while (1) {}
	return 0;
}
```

## ldscript

这里主要是需要指定这么几项内容

1. 对于qemu来说，启动之后会读位于0x80000000这个地址的内容，因此我们需要将我们的内容放到这个地址开始。
2. 指定OUTPUT_ARCH( "riscv" )
3. 指定汇编入口地址，比如ENTRY( _entry )

xv6-riscv/kernel/entry.S

```
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;

  .text : {
    *(.text .text.*)
    . = ALIGN(0x1000);
    _trampoline = .;
    *(trampsec)
    . = ALIGN(0x1000);
    ASSERT(. - _trampoline == 0x1000, "error: trampoline larger than one page");
    PROVIDE(etext = .);
  }

  .rodata : {
    . = ALIGN(16);
    *(.srodata .srodata.*) /* do not need to distinguish this from .rodata */
    . = ALIGN(16);
    *(.rodata .rodata.*)
  }

  .data : {
    . = ALIGN(16);
    *(.sdata .sdata.*) /* do not need to distinguish this from .data */
    . = ALIGN(16);
    *(.data .data.*)
  }

  .bss : {
    . = ALIGN(16);
    *(.sbss .sbss.*) /* do not need to distinguish this from .bss */
    . = ALIGN(16);
    *(.bss .bss.*)
  }

  PROVIDE(end = .);
}
```

## makefile

mini-riscv-os/01-HelloOs/Makefile

```makefile
CC = riscv64-unknown-elf-gcc
CFLAGS = -nostdlib -fno-builtin -mcmodel=medany -march=rv32ima -mabi=ilp32

QEMU = qemu-system-riscv32
QFLAGS = -nographic -smp 4 -machine virt -bios none

OBJDUMP = riscv64-unknown-elf-objdump

all: os.elf

os.elf: start.s os.c
	$(CC) $(CFLAGS) -T os.ld -o os.elf $^

qemu: $(TARGET)
	@qemu-system-riscv32 -M ? | grep virt >/dev/null || exit
	@echo "Press Ctrl-A and then X to exit QEMU"
	$(QEMU) $(QFLAGS) -kernel os.elf

clean:
	rm -f *.elf
```

这里没什么好讲的，绝大多数选项都用不到，唯一要注意的是-march的值

riscv是一种模块化的指令集，不同的名字代表支持的扩展指令集不同，关于详情参考

[RISC-V#ISA_base_and_extensions](https://en.wikipedia.org/wiki/RISC-V#ISA_base_and_extensions)

之后直接通过make命令编译出elf之后通过qemu启动就好

# 参考

[https://github.com/cccriscv/mini-riscv-os](https://github.com/cccriscv/mini-riscv-os)

[https://github.com/mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv)

[Specifications - RISC-V International](https://riscv.org/technical/specifications/)
