## 下载实验资料包

我们将实验资料包放置在Github上，请大家自己安装Git（我们提供的实验Docker中已经自带），然后使用以下命令下载：

```bash
git clone https://github.com/cyyself/ucore-loongarch32.git -b cqu2022_noanswer
```

建议大家不要使用Github网页提供的Download ZIP功能，因为这样会丢失Git元数据，无法做到切换分支等功能。

如果要下载有答案的版本，可以使用`git checkout cqu2022`切换到该分支。

我们并不对同学们的代码进行查重，鼓励同学们在看了答案后反推操作系统的运行过程从而了解原理。

初次接触git的同学可以[参考这里](https://colabdocs.cqu.ai/tips/git/)。

## 运行实验

我们可以在Makefile中切换要运行的实验，将该实验之后的实验注释掉即可。

例如，运行实验1，将Makefile改为以下状态：

```makefile
LAB1	:= -DLAB1_EX2 -DLAB1_EX3 -D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
#LAB2	:= -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
#LAB3	:= -DLAB3_EX1 -DLAB3_EX2
#LAB4	:= -DLAB4_EX1 -DLAB4_EX2
```

运行实验2，以此类推如下：

```makefile
LAB1	:= -DLAB1_EX2 -DLAB1_EX3 #-D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
LAB2	:= -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
#LAB3	:= -DLAB3_EX1 -DLAB3_EX2
#LAB4	:= -DLAB4_EX1 -DLAB4_EX2
```

## 编译与运行实验

如果我们修改了代码，希望重新编译OS内核可以执行以下代码。

```bash
make clean # 清空编译产物
make -j 16 # 编译
make qemu  # 运行qemu
```

注，如果没有make clean，Makefile不一定会检测到你的修改，可能导致编译错误或者编译了stale状态。

## 实验调试

实验的调试需要使用gdb与qemu。gdb的使用方法需要大家自行学习。

若要进入gdb调试，大家可以开启一个终端执行`make debug`，在另一个终端执行`make gdb`。

## 如何快速找到每个实验我需要补充的代码

使用你的编辑器，在UCORE代码中搜索"YOUR CODE"字符串。