### 1. 补全代码
注意代码中有“LAB3”的注释的地方，代码中所有需要完成的地方都有“LAB3”和“YOUR CODE”的注释。同学们可以根据代码中的注释的提示，思考练习中的问题来补全代码。

### 2. Makefile修改 

```makefile
LAB1	:= -DLAB1_EX2 -DLAB1_EX3 #-D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
LAB2	:= -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
LAB3	:= -DLAB3_EX1 -DLAB3_EX2
#LAB4	:= -DLAB4_EX1 -DLAB4_EX2
```

### 3. 编译运行

编译并运行代码的命令如下：
```
make clean
make
make qemu -j 16
```

补全代码后可以得到如下显示界面（仅供参考）
```
chenyu$ make qemu -j 16
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0xA0000120 (phys)
  etext 0xA0022000 (phys)
  edata 0xA017C630 (phys)
  end   0xA017F910 (phys)
Kernel executable memory footprint: 1399KB
memory management: default_pmm_manager
memory map:
    [A0000000, A2000000]

freemem start at: A01C0000
free pages: 00001E40
## 00000020
check_alloc_page() succeeded!
check_pgdir() succeeded!
check_boot_pgdir() succeeded!
check_slab() succeeded!
kmalloc_init() succeeded!
check_vma_struct() succeeded!
check_pgfault() succeeded!
check_vmm() succeeded.
sched class: stride_scheduler
proc_init succeeded
kernel_execve: pid = 2, name = "exit".
I am the parent. Forking the child...
I am parent, fork a child pid 3
I am the parent, waiting now..
I am the child.
waitpid 3 ok.
exit pass.
all user-mode processes have quit.
init check memory pass.
initproc exit.
Lab3 Check Pass!
```