## 实验步骤如下

### 异常中断

根据实验任务“异常中断”的问题（2）和（3）结合所学知识完成相关程序的撰写，而后按照如下步骤检验程序。

**Makefile修改**

在Makefile中取消LAB1 := -DLAB1_EX4 -D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT(第6行)的注释

```bash
LAB1    := -DLAB1_EX4 -D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
# LAB2    := -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
# LAB3    := -DLAB3_EX1 -DLAB3_EX2
# LAB4    := -DLAB4_EX1 -DLAB4_EX2
```

-D_SHOW_100_TICKS选项可在终端每是毫秒打印一行"100 ticks" -D_SHOW_SERIAL_INPUT选项会在终端打印键盘的输入 编译并运行代码的命令如下：

```shell
make

make qemu -j 16
```

注：如果make后报错，首先检查是否使用了Dockers，其次在切换实验的时候（即使用不同的makefile时）需要使用`make clean`进行切换）

补全代码后可以得到如下显示界面（仅供参考）

```bash
chenyu$ make qemu -j 16

(THU.CST) os is loading ...



Special kernel symbols:

  entry  0xA00000A0 (phys)

  etext 0xA001F000 (phys)

  edata 0xA0151820 (phys)

  end   0xA0154B00 (phys)

Kernel executable memory footprint: 1239KB

LAB1 Check - Please press your keyboard manually and see what happend.

100 ticks

100 ticks

100 ticks

100 ticks

100 ticks

100 ticks

100 ticks

got input

100 ticks

got input s

got input d

got input g

100 ticks

got input s

got input g

100 ticks
```

