## 实验任务

### 异常中断

#### 练习1

结合LoongArch32的文档，列出LoongArch32有哪些例外？以及这些例外有哪两个例外向量入口？ 

#### 练习2

请编程完善`kern/driver/clock.c`中的时钟中断处理函数`clock_int_handler`，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用kprintf，向屏幕上打印一行文字”100 ticks”。

#### 练习3

请编程完善`kern/driver/console.c`中的串口中断处理函数`serial_int_handler`，在接收到一个字符后读取该字符，并调用kprintf输出该字符。

要求完成练习（2）、（3）提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程，并写出对练习（1）的回答。完成这练习（2）和（3）要求的部分代码后，运行整个系统，可以看到大约每1秒会输出一次”100 ticks”，而按下的键也会在屏幕上显示。

## 实验要求

1. 完成以上三个练习，给出编译运行后输出结果的截图。并在实验报告中对所写代码调用的函数的作用加以说明。

2. 需提交可以在我们给定的Docker环境中直接使用make qemu编译并运行的文件夹的压缩包。

## 组队分工建议

  建议三人都阅读LoongArch32文档与uCore代码后，分别完成上述3个练习。