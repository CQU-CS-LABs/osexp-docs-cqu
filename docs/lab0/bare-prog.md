## 初识裸机程序

我们以往写用户程序时，通常都只关注代码本身，而将运行时的环境交给了编译器等系统软件进行处理，但我们若要编写裸机程序，就需要进一步揭开运行时环境的神秘面纱。

强烈建议同学们阅读《程序员的自我修养：链接、装载与库》，会对这些问题有更加清晰的认识，在重庆大学A区与虎溪的图书馆均有馆藏。

下表揭示了裸机程序与用户程序的区别：

| 对比对象     | 裸机程序                                                     | 常规用户程序                                          |
| ------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| 内存地址空间 | 自行管理物理地址空间，可以自行对虚拟内存进行配置后使自己运行在虚拟地址空间 | 由操作系统管理的虚拟地址空间（不考虑Linux NOMMU模式） |
| 系统调用     | 调用自己                                                     | 调用更高特权级的操作系统/固件                         |
| 栈的初始化   | 自行完成                                                     | 操作系统载入用户进程时完成（毕竟还要通过栈传递参数）  |
| BSS段的清空  | 自行完成                                                     | 操作系统分配虚拟页面时完成清零                        |

许多同学可能对于上表已经看懵了，不明白这些名词的具体含义，没关系，我们接下来一一解释：

### 调用栈

我们知道编写的程序可以进行函数调用，也可以在调用后返回。那么我们可以思考，记录函数执行位置，包括局部变量的状态等可以采用一种先进后出的数据结构，也就是栈。这个调用栈的数据结构在不同的指令集架构的ABI（Application Binary Interface）中定义不同。且它的生长方式是向下生长。

但我们需要先分配出一个栈，才能运行这种能进行函数调用的C语言程序。因此对于裸机程序而言，我们需要在汇编程序里先初始化GPR（通用寄存器）的sp指针，才能进入C程序。

### 程序里的各个存储区

我们可以尝试运行以下实验，编写如下C语言程序：

```c title="hello.c" linenums="1"
#include <stdio.h>
int main() {
	printf("Hello World\n");
	return 0;
}

```

然后，执行以下命令编译为目标文件并查看目标文件的头：

```shell
➜  gcc hello.c -c -o hello.o
➜  objdump -h hello.o       

hello.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001a  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000005a  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000005a  2**0
                  ALLOC
  3 .rodata       0000000c  0000000000000000  0000000000000000  0000005a  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000001f  0000000000000000  0000000000000000  00000066  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000085  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000038  0000000000000000  0000000000000000  00000088  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```

这里可以看到，我们的程序分为了`.text`、`.data`、`.bss`、`.rodata`各存储段。

其中：

- `.text`是代码段，放置的是我们的程序编译后的代码。

- `.data`是数据段，这里放的是初始化过的静态变量（包含全局变量还有`static`修饰后的局部变量）。

- `.bss`是存放未初始化的静态变量的区域。而在ELF文件中，它并不实际存储数据，仅用于告知操作系统载入进程时该段合法地址的存在。

- `.rodata`存放的是只读数据，例如字符串常量与全局const变量就存放在这个位置。

这些段的元数据放在我们编译的产物（ELF文件）的头部分。

### 链接

我们在大一的“程序设计基础”课程应该已经学过多文件的C语言程序编写以及链接过程，同学们应该也在其中学习了Makefile的基本使用。

许多同学也许会好奇，链接器是如何将这些未定义的函数找到对应的位置并进行链接的呢？

这里我们就得涉及到一个符号表的概念，在我们编译产生的ELF文件中，还有一个存放符号表的区域，其中符号指的是函数、变量等的名字，并记录他们对应的地址。这样在链接时就知道能够知道对应的位置，从而进行链接，生成出一个更大的可执行程序。

而对于裸机程序，我们还有一个需求是设定程序各存储段的排布以及地址偏移量。地址偏移量可以告知编译器进行直接跳转时所需要的地址，这个时候我们就需要引入一个叫做ld脚本的东西，它可以帮助我们规划需要放入最终编译结果的各段的地址。

!!! note

    小思考：
    
    1. 操作系统给用户进程分配虚拟内存空间时，为什么需要将对应页面清零后再分配？如果不清零会导致什么问题？（可以从安全等角度考虑）
    
    2. 我们写C语言程序时开的局部变量不赋初始值为什么经常会得到一个随机数？是否与前面提到操作系统给用户进程分配虚拟页面时会进行清零的结论相悖？
    
    3. Linux NOMMU为什么只在部分指令集架构存在（如RISC-V）？这些指令集提供了什么机制确保了NOMMU模式的进程隔离问题？

    4. 为什么我们在裸机程序中可以暂时不考虑malloc（new）这种堆内存分配？

    5. 栈能不能改成向上生长？有什么利弊？

    6. 结合程序的各段所需的最小权限不同，处理器的MMU可以提供哪些权限控制位确保程序因为漏洞导致任意内存读写时，攻击者能破坏的数据范围最低？例如如何才能让.text段和.rodata段不被修改，其他段的数据不可被执行？如果攻击者有能力破坏页表，还有什么安全机制可以加入处理器中进行更好的保护？


## 编写一个裸机程序

至此，我们开始真正编写一个裸机程序，大家在搭建好的环境中新建文件夹，一步步开始吧！

### 初始化的汇编代码

```asm title="start.S" linenums="1"
.extern main
.text
.globl _start
_start:
    # Config direct window and set PG
    li.w    $t0, 0xa0000011
    csrwr   $t0, 0x180
    /* CSR_DMWIN0(0x180): 0xa0000000-0xbfffffff->0x00000000-0x1fffffff Cached */
    li.w    $t0, 0x80000001
    /* CSR_DMWIN1(0x181): 0x80000000-0x9fffffff->0x00000000-0x1fffffff Uncached */
    # Enable PG
    li.w    $t0, 0xb0
    csrwr   $t0, 0x0
    /* CSR_CRMD(0x0): PLV=0, IE=0, PG */
    la  $sp, bootstacktop
    la  $t0, main
    jr  $t0
poweroff:
    b poweroff
_stack:
.section .data
    .global bootstack
bootstack:
    .space 1024
    .global bootstacktop
bootstacktop:
    .space 64
```

我们将这个文件保存为`start.S`。

这里程序主要完成了对CSR的DMWIN的设置，并修改CSR_CRMD开启虚拟地址翻译模式，然后从栈地址直接进入到main函数。而main函数来源于外部的extern，我们接着写main对应的代码。

### 编写简单串口输出C程序

串口是一种通信方式。在LoongArch32的QEMU中，有一个ns16550a规格的串口，位于物理地址0x1fe001e0。

该串口通过MMIO的方式访问，我们需要通过编写ns16550a对应的驱动代码完成串口的打印，这一部分感兴趣的同学可以自行上网搜索，助教已经给出一个只有输出功能的驱动范例，如下：

```c title="main.c" linenums="1"
#define UART_BASE 0x9fe001e0
#define UART_RX     0   /* In:  Receive buffer */
#define UART_TX     0   /* Out: Transmit buffer */
#define UART_LSR    5   /* In:  Line Status Register */
#define UART_LSR_TEMT       0x40 /* Transmitter empty */
#define UART_LSR_THRE       0x20 /* Transmit-hold-register empty */
#define UART_LSR_DR         0x01 /* Receiver data ready */

void uart_put_c(char c) {
    while (!(*((volatile char*)UART_BASE + UART_LSR) & (UART_LSR_THRE)));
    *((volatile char*)UART_BASE + UART_TX) = c;
}

void print_s(const char *c) {
    while (*c) {
        uart_put_c(*c);
        c ++;
    }
}

void main() {
    print_s("\nHere is my first bare-metal machine program on LoongArch32!\n\n");
}
```

我们将这个文件保存为`main.c`。

细心的同学可能会注意到，我们前文提到串口地址是在0x1fe001e0，为什么这里代码写成了0x9fe001e0呢？这是因为我们前面设置了DMW，完成了0x80000000-0x9fffffff的映射，并设置的Uncached属性。

!!! warning

    如果使用Cached地址访问串口，例如对应的DMWIN0下的0xbfe001e0，尽管在QEMU中不会有任何错误，但在实际有Cache的CPU硬件上会导致串口访问失去原子性，导致无法得到串口输出。

### 编写链接脚本

这里我们的链接脚本需要做的事情就是指定一个起始地址，并删去程序中不需要的存储区段，因此最终产生链接脚本如下：

```lds title="lab0.ld" linenums="1" hl_lines="3"
SECTIONS
{
    . = 0xa0000000;
    .text : { *(.text) }
    .rodata : { *(.rodata) }
    .bss : { *(.bss) }
}
```

然后保存为`lab0.ld`。

其中，我们把开始地址放在0xa0000000是基于我们配置的DMWIN0考虑的，对于代码和数据这类访问，我们当然希望放在带有Cache属性的地址段，这样运行起来速度快一些。

### 编写Makefile

```makefile title="Makefile" linenums="1"
TOOL	:=	loongarch32r-linux-gnusf-
CC		:=	$(TOOL)gcc
OBJCOPY	:=	$(TOOL)objcopy
OBJDUMP	:=	$(TOOL)objdump
QEMU	:=	qemu-system-loongarch32

.PHONY: clean qemu

start.elf: start.S main.c lab0.ld
	$(CC) -nostdlib -T lab0.ld start.S main.c -O3 -o $@

qemu: start.elf
	$(QEMU) -M ls3a5k32 -m 32M -kernel start.elf -nographic

clean:
	rm start.elf
```

然后保存为`Makefile`。

关于Makefile的内容大家可以上网寻找相关资料。这里编译时添加参数`-nostdlib`是为了防止stdlib编译到我们的程序中，毕竟这是一个裸机程序。

### 编译运行

在放置了`start.S`、`main.c`、`lab0.ld`、`Makefile`的文件夹下执行`make qemu`：

```shell hl_lines="6"
➜  make qemu 
qemu-system-loongarch32 -M ls3a5k32 -m 32M -kernel start.elf -nographic
loongson32_init: num_nodes 1
loongson32_init: node 0 mem 0x2000000

Here is my first bare-metal machine program on LoongArch32!


```

看到"Here is my first bare-metal machine program on LoongArch32!"表示我们的裸机程序运行成功！

### 退出QEMU

在nographic模式下，可以按 ++ctrl+a++ 一次，然后按 ++x++ 退出。