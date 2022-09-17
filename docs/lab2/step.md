## 实验步骤如下

### 内存管理

**Makefile修改**

```makefile
LAB1  := -DLAB1_EX4 #-D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
LAB2  := -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
# LAB3  := -DLAB3_EX1 -DLAB3_EX2
# LAB4  := -DLAB4_EX1 -DLAB4_EX2
```

编译并运行代码的命令如下：

```bash
make
make qemu -j 16
```

补全代码后可以得到如下显示界面（仅供参考）

```bash
chenyu$ make qemu -j 16
(THU.CST) os is loading ...


Special kernel symbols:
  entry  0xA00000A0 (phys)
  etext 0xA0020000 (phys)
  edata 0xA0153370 (phys)
  end   0xA0156650 (phys)
Kernel executable memory footprint: 1242KB
memory management: default_pmm_manager
memory map:
    [A0000000, A2000000]


freemem start at: A0197000
free pages: 00001E69
## 00000020
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
check_slab() succeeded!
kmalloc_init() succeeded!
LAB2 Check Pass!
```

通过上图，我们可以看到ucore在显示其entry（入口地址）、etext（代码段截止处地址）、edata（数据段截止处地址）、和end（ucore截止处地址）的值后，探测出计算机系统中的物理内存的布局。然后会显示内存范围和空闲内存的起始地址，显示可以分为多少个page。接下来会执行各种我们设置的检查，最后响应时钟中断。