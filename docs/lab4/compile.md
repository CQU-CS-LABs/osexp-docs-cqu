### 编译方法

**Makefile修改**
在Makefile中取消LAB8	:= -DLAB8_EX1 -DLAB8_EX2(第13行)的注释
```
LAB1	:= -DLAB1_EX4 # -D_SHOW_100_TICKS -D_SHOW_SERIAL_INPUT
LAB2	:= -DLAB2_EX1 -DLAB2_EX2 -DLAB2_EX3
LAB3	:= -DLAB3_EX1 -DLAB3_EX2
LAB4	:= -DLAB4_EX1 -DLAB4_EX2
LAB5	:= -DLAB5_EX1 -DLAB5_EX2
LAB6	:= -DLAB6_EX2
LAB7	:= -DLAB7_EX1 # -D_SHOW_PHI
LAB8	:= -DLAB8_EX1 -DLAB8_EX2
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
  entry  0xA00000A0 (phys)
  etext 0xA0021000 (phys)
  edata 0xA0251470 (phys)
  end   0xA0254750 (phys)
Kernel executable memory footprint: 2254KB
memory management: default_pmm_manager
memory map:
    [A0000000, A2000000]

freemem start at: A0295000
free pages: 00001D6B
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
Initrd: 0xa005d3d0 - 0xa02513cf, size: 0x001f4000, magic: 0x2f8dbe2a
ramdisk_init(): initrd found, magic: 0x2f8dbe2a, 0x00000fa0 secs
sfs: mount: 'simple file system' (318/182/500)
vfs: mount disk0.
kernel_execve: pid = 2, name = "sh".
user sh is running!!!
$ 
```
