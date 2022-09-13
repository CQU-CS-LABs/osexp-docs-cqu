## lab1中对例外的处理实现（中断异常）

(1) 外设基本初始化设置

Lab1实现了中断初始化和对键盘、串口、时钟外设进行中断处理。串口的初始化函数serial_init（位于/kern/driver/console.c）中涉及中断初始化工作的很简单：

```
......

// 使能串口1接收字符后产生中断

outb(COM1 + COM_IER, COM_IER_RDI);

......

// 通过中断控制器使能串口1中断

pic_enable(IRQ_COM1);
```

时钟是一种有着特殊作用的外设，其作用并不仅仅是计时。在后续章节中将讲到，正是由于有了规律的时钟中断，才使得无论当前CPU运行在哪里，操作系统都可以在预先确定的时间点上获得CPU控制权。这样当一个应用程序运行了一定时间后，操作系统会通过时钟中断获得CPU控制权，并可把CPU资源让给更需要CPU的其他应用程序。时钟的初始化函数clock_init（位于kern/driver/clock.c中）完成了对时钟控制器的初始化：

```
    ......

unsigned long timer_config;

unsigned long period = 200000000;

period = period / HZ;

timer_config = period & LISA_CSR_TMCFG_TIMEVAL;

timer_config |= (LISA_CSR_TMCFG_PERIOD | LISA_CSR_TMCFG_EN);

__lcsr_csrwr(timer_config, LISA_CSR_TMCFG);

pic_enable(TIMER0_IRQ);
```

这里关于CSR中的TCFG寄存器具体含义需要参阅[龙芯架构32位精简版参考手册_v1.02.pdf](https://mirrors.tuna.tsinghua.edu.cn/loongson/docs/loongarch/%E9%BE%99%E8%8A%AF%E6%9E%B6%E6%9E%8432%E4%BD%8D%E7%B2%BE%E7%AE%80%E7%89%88%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C_v1.02.pdf)的P64（PDF中的76页）。

(2) 例外初始化设置

对于LoongArch32架构的计算机来说，操作系统初始化中断首先需要完成例外处理程序的基地址设置。这一部分程序写在了`kern/init/init.c`中的`setup_exception_vector`函数。

其中，该函数中使用的`__exception_vector`地址位于文件`kern/trap/vectors.S`。为了简化，它直接跳转到了`kern/trap/exception.S`中的`ramExcHandle_general`处。

(3) 例外的处理过程

trap函数（定义在trap.c中）是对例外进行处理的过程，所有的例外在经过中断入口函数`ramExcHandle_general`预处理后 (定义在 exception.S中) ，都会跳转到这里。在处理过程中，根据不同的例外类型，进行相应的处理。在相应的处理过程结束以后，trap将会返回，被中断的程序会继续运行。整个中断处理流程大致如下：

1)产生例外后， CPU硬件完成了如下操作：

- 将CSR.CRMD的PLV、IE分别存到CSR.PRMD的PPLV和IE中，然后将CSR.CRMD的PLV置为0，IE置为0。

- 将触发例外指令的PC值记录到CSR.ERA中

- 跳转到例外入口处取值。（如果是TLB相关例外跳转到CSR.RFBASE，否则为CSR.EBASE）

2)经过例外向量的跳转，进入`exception.S`中的`ramExcHandle_general`。

在这里会完成例外现场的保存操作，切换到内核栈，将当前处理器的所有通用寄存器压入内核栈中，并使用CSR中的KS0和KS1寄存器用来辅助保存数据（否则保存过程中必然导致一些特定寄存器的修改）。

保存的数据按照trapfame结构进行，位于`kern/trap/loongarch_trapframe.h`。

3) 然后跳转进入`kern/trap/trap.c`中的`loongarch_trap`函数，开始了C语言程序的内核的处理。

这个函数中，会根据例外的类型完成例外的分类并进行处理，具体见`kern/trap/trap.c`。

4) 当`loongarch_trap`这一函数处理完毕后（处理过程可能包含对trapframe的修改），会返回到之前的汇编程序（`exception.S`中），完成寄存器状态的还原，然后使用ertn指令结束例外处理，恢复程序的执行。

至此，对整个lab1中的主要部分的背景知识和实现进行了阐述。请大家能够据此完成实验任务。