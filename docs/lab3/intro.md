## 实验执行流程概述

到现在为止，ucore还一直在核心态“打转”，没有到用户态执行。提供各种操作系统功能的内核线程只能在CPU核心态运行是操作系统自身的要求，操作系统就要呆在核心态，才能管理整个计算机系统。但应用程序员也需要编写各种应用软件，且要在计算机系统上运行。如果把这些应用软件都作为内核线程来执行，那系统的安全性就无法得到保证了。所以，ucore要提供用户态进程的创建和执行机制，给应用程序执行提供一个用户态运行环境。接下来我们就简要分析本实验的执行过程，以及分析用户进程的整个生命周期来阐述用户进程管理的设计与实现。

显然，由于进程的执行空间扩展到了用户态空间，且出现了创建子进程执行应用程序等与前面的实验有较大不同的地方，所以具体实现的不同主要集中在进程管理和内存管理部分。首先，我们从ucore的初始化部分来看，会发现初始化的总控函数kern_init没有任何变化。但这并不意味着二者差别不大。其实kern_init调用的物理内存初始化，进程管理初始化等都有一定的变化。

- **在内存管理部分，用户进程管理需要增加用户态虚拟内存的管理**。为了管理用户态的虚拟内存，需要对页表的内容进行扩展，能够把部分物理内存映射为用户态虚拟内存。如果某进程执行过程中，CPU在用户态下执行（**在CSR.CRMD中的PLV域，如果为0，表示CPU运行在特权态；如果为3，表示CPU运行在用户态**。），则可以访问本进程页表描述的用户态虚拟内存，但由于权限不够，不能访问内核态虚拟内存。另一方面，不同的进程有各自的页表，所以即使不同进程的用户态虚拟地址相同，但由于页表把虚拟页映射到了不同的物理页帧，所以不同进程的虚拟内存空间是被隔离开的，相互之间无法直接访问。在用户态内存空间和内核态内存空间之间需要拷贝数据，让CPU处在内核态才能完成对用户空间的读或写，为此需要设计专门的拷贝函数（copy_from_user和copy_to_user）完成。但反之则会导致违反CPU的权限管理，导致内存访问异常。

- **在进程管理方面，主要涉及到的是进程控制块中与内存管理相关的部分**，包括①建立进程的页表和维护进程可访问空间（可能还没有建立虚实映射关系）的信息；②加载一个ELF格式的程序到进程控制块管理的内存中的方法；③在进程复制（fork）过程中，把父进程的内存空间拷贝到子进程内存空间的技术。另外一部分与用户态进程生命周期管理相关，包括让进程放弃CPU而睡眠等待某事件；让父进程等待子进程结束；一个进程杀死另一个进程；给进程发消息；建立进程的血缘关系链表。

当实现了上述内存管理和进程管理的需求后，接下来ucore的用户进程管理工作就比较简单了。首先，“硬”构造出第一个进程，它是后续所有进程的祖先；然后，在proc_init函数中，通过alloc把当前ucore的执行环境转变成idle内核线程的执行现场；然后调用kernl_thread来创建第二个内核线程init_main，而init_main内核线程有创建了user_main内核线程。到此，内核线程创建完毕，应该开始用户进程的创建过程，这第一步实际上是通过user_main函数调用kernel_tread创建子进程，通过kernel_execve调用来把某一具体程序的执行内容放入内存。具体的放置方式是根据ld在此文件上的地址分配为基本原则，把程序的不同部分放到某进程的用户空间中，从而通过此进程来完成程序描述的任务。一旦执行了这一程序对应的进程，就会从内核态切换到用户态继续执行。以此类推，CPU在用户空间执行的用户进程，其地址空间不会被其他用户的进程影响，但由于系统调用（用户进程直接获得操作系统服务的唯一通道）、外设中断和异常中断的会随时产生，从而间接推动了用户进程实现用户态到到内核态的切换工作。ucore对CPU内核态与用户态的切换过程需要比较仔细地分析。当进程执行结束后，需回收进程占用和没消耗完毕的设备整个过程，且为新的创建进程请求提供服务。在本实验中，当系统中存在多个进程或内核线程时，ucore采用了一种FIFO的很简单的调度方法来管理每个进程占用CPU的时间和频度等。在ucore运行过程中，由于调度、时间中断、系统调用等原因，使得进程会进行切换、创建、睡眠、等待、发消息等各种不同的操作，周而复始，生生不息。

## 进程控制块
加载应用程序并执行实际上属于执行用户进程，操作系统在创建并执行用户进程之前，首先要创建内核进程。在了解这些之前，我们先介绍进程控制块的设计。进程管理信息用一个名为proc_struct的数据结构表示，在kern/process/proc.h中定义如下：
```c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
    int exit_code;                              // exit code (be sent to parent proc)
    uint32_t wait_state;                        // waiting state
    struct proc_struct *cptr, *yptr, *optr;     // relations between processes
    struct run_queue *rq;                       // running queue contains Process
    list_entry_t run_link;                      // the entry linked in run queue
    int time_slice;                             // time slice for occupying the CPU
    struct fs_struct *fs_struct;                // the file related info(pwd, files_count, files_array, fs_semaphore) of process
    skew_heap_entry_t lab6_run_pool;            // FOR LAB6 ONLY: the entry in the run pool
    uint32_t lab6_stride;                       // FOR LAB6 ONLY: the current stride of the process 
    uint32_t lab6_priority;                     // FOR LAB6 ONLY: the priority of process, set by lab6_set_priority(uint32_t)
};
```
下面重点解释一下几个比较重要的成员变量：

● **mm：内存管理的信息**，包括内存映射列表、页表指针等。mm里有个很重要的项pgdir，记录的是该进程使用的一级页表的物理地址。由于*mm=NULL，所以在proc_struct数据结构中需要有一个代替pgdir项来记录页表起始地址，这就是proc_struct数据结构中的cr3成员变量。

● state：进程所处的状态。

● parent：用户进程的父进程（创建它的进程）。在所有进程中，只有一个进程没有父进程，就是内核创建的第一个内核线程idleproc。内核根据这个父子关系建立一个树形结构，用于维护一些特殊的操作，例如确定某个进程是否可以对另外一个进程进行某种操作等等。

● kstack: 每个线程都有一个内核栈，并且位于内核地址空间的不同位置。对于内核线程，该栈就是运行时的程序使用的栈；而对于普通进程，该栈是发生特权级改变的时候使保存被打断的硬件信息用的栈。uCore在创建进程时分配了 2 个连续的物理页（参见memlayout.h中KSTACKSIZE的定义）作为内核栈的空间。这个栈很小，所以内核中的代码应该尽可能的紧凑，并且避免在栈上分配大的数据结构，以免栈溢出，导致系统崩溃。**kstack记录了分配给该进程/线程的内核栈的位置**。主要作用有以下几点。首先，当内核准备从一个进程切换到另一个的时候，需要根据kstack 的值正确的设置好 tss，以便在进程切换以后再发生中断时能够使用正确的栈。其次，内核栈位于内核地址空间，并且是不共享的（每个线程都拥有自己的内核栈），因此不受到 mm 的管理，当进程退出的时候，内核能够根据 kstack 的值快速定位栈的位置并进行回收。

● **tf：中断帧的指针**，总是指向内核栈的某个位置：当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。**当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值**。除此之外，uCore内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的trapframe，uCore 在内核栈上维护了 tf 的链，可以参考trap.c::trap函数做进一步的了解。

● cr3: cr3 保存页表的物理地址，目的就是进程切换的时候方便直接使用 lcr3实现页表切换，避免每次都根据 mm 来计算 cr3。mm数据结构是用来实现用户空间的虚存管理的，当某个进程是一个普通用户态进程的时候，PCB 中的 cr3 就是 mm 中页表（pgdir）的物理地址；而当它是内核线程的时候，cr3 等于boot_cr3。而boot_cr3指向了uCore启动时建立好的饿内核虚拟空间的页目录表首地址。

为了管理系统中所有的进程控制块，uCore维护了如下全局变量（位于kern/process/proc.c）：
● static struct proc *current：当前占用CPU且处于“运行”状态进程控制块指针。通常这个变量是只读的，只有在进程切换的时候才进行修改。

● **static struct proc *initproc：本实验中，此指针将指向第一个用户态进程**。

● static list_entry_t hash_list[HASH_LIST_SIZE]：所有进程控制块的哈希表，proc_struct中的成员变量hash_link将基于pid链接入这个哈希表中。

● list_entry_t proc_list：所有进程控制块的双向线性列表，proc_struct中的成员变量list_link将链接入这个链表中。


##  创建用户进程

### 1. 应用程序的组成和编译
    
我们首先来看一个应用程序，这里我们假定是hello应用程序，在user/hello.c中实现，代码如下：
```c
#include <stdio.h>
#include <ulib.h>

int main(void) {
    cprintf("Hello world!!.\n");
    cprintf("I am process %d.\n", getpid());
    cprintf("hello pass.\n");
    return 0;
}
```
hello应用程序只是输出一些字符串，并通过系统调用sys_getpid（在getpid函数中调用）输出代表hello应用程序执行的用户进程的进程标识--pid。

首先，我们需要了解ucore操作系统如何能够找到hello应用程序。这需要分析ucore和hello是如何编译的。在本实验源码目录下执行make，可得到如下输出：
```
……

+ cc user/hello.c

loongarch32-linux-gnu-gcc -c  -Iuser/libs -Ikern/include -fno-builtin-fprintf -fno-builtin -nostdlib  -nostdinc -g -G0 -Wa,-O0 -fno-pic -mno-shared -msoft-float -ggdb -gstabs -mlcsr   user/hello.c  -o obj/user/hello.o

loongarch32-linux-gnu-ld -T user/libs/user.ld  obj/user/hello.o --whole-archive obj/user/libuser.a -o obj/user/hello

……

sed 's/$FILE/hello/g' tools/piggy.S.in > obj/user/hello.S

……

# 注意，可以观察obj/user/hello.S文件，这里有使用.incbin去引入先前编译好的obj/user/hello
loongarch32-linux-gnu-gcc -c obj/user/hello.S -o obj/user/hello.piggy.o

loongarch32-linux-gnu-ld -nostdlib -n -G 0 -static -T tools/kernel.ld  obj/init/init.o  …… obj/user/hello.piggy.o …… -o obj/ucore-kernel-piggy
```
从中可以看出，hello应用程序不仅仅是hello.c，还包含了支持hello应用程序的用户态库：

- user/libs/initcode.S：所有应用程序的起始用户态执行地址“_start”，调整了gp和sp后，调用umain函数。
- user/libs/umain.c：实现了umain函数，这是所有应用程序执行的第一个C函数，它将调用应用程序的main函数，并在main函数结束后调用exit函数，而exit函数最终将调用sys_exit系统调用，让操作系统回收进程资源。
- user/libs/ulib.[ch]：实现了最小的C函数库，除了一些与系统调用无关的函数，其他函数是对访问系统调用的包装。
- user/libs/syscall.[ch]：用户层发出系统调用的具体实现。
- user/libs/stdio.c：实现cprintf函数，通过系统调用sys_putc来完成字符输出。
- user/libs/panic.c：实现__panic/__warn函数，通过系统调用sys_exit完成用户进程退出。


除了这些用户态库函数实现外，还有一些libs/*.[ch]是操作系统内核和应用程序共用的函数实现。这些用户库函数其实在本质上与UNIX系统中的标准libc没有区别，只是实现得很简单，但hello应用程序的正确执行离不开这些库函数。

【**注意**】libs/*.[ch]、user/libs/*.[ch]、user/*.[ch]的源码中没有任何特权指令。

a00c3160 _binary_obj_user_hello_end a00125ac file_open a00018d0 strcpy a0000240 ide_device_valid a01455c0 _binary_obj_user_forktree_start a000cb30 wakeup_queue a00156f0 vfs_set_bootfs a000d12c cond_signal a0011850 sysfile_fsync a001a57c dev_init_stdout a00b4ac0 _binary_obj_user_hello_start

在make的最后一步执行了一个ld命令，把hello应用程序的执行码obj/user/hello.piggy.o连接在了ucore kernel的末尾。并观察tools/piggy.S.in文件可以发现，我们定义了.global _binary_obj_user_$FILE_start和.global _binary_obj_user_$FILE_end。这样这两个符号就会保留在最终ld生成的文件中，这样这个hello用户程序就能够和ucore内核一起被 bootloader 加载到内存里中，并且通过这两个全局变量定位hello用户程序执行码的起始位置和大小。而到了与文件系统相关的实验后，ucore会提供一个简单的文件系统，那时所有的用户程序就都不再用这种方法进行加载了，而可以用大家熟悉的文件方式进行加载了。

（注，后续的文件系统采用initrd的方式，并非位于磁盘。）  
<br>

### 2. 用户进程的虚拟地址空间

在user/libs/user.ld描述了用户程序的用户虚拟空间的执行入口虚拟地址：
```c
SECTIONS {
    /* Load programs at this address: "." means the current address */
    . = 0x10000000;
```

在tools/kernel.ld描述了操作系统的内核虚拟空间的起始入口虚拟地址：
```c
SECTIONS {
    /* Load the kernel at this address: "." means the current address */
    . = 0xa0000000;
```
这样ucore把用户进程的虚拟地址空间分了两块，一块与内核线程一样，是所有用户进程都共享的内核虚拟地址空间，映射到同样的物理内存空间中，这样在物理内存中只需放置一份内核代码，使得用户进程从用户态进入核心态时，内核代码可以统一应对不同的内核程序；另外一块是用户虚拟地址空间，虽然虚拟地址范围一样，但映射到不同且没有交集的物理内存空间中。这样当ucore把用户进程的执行代码（即应用程序的执行代码）和数据（即应用程序的全局变量等）放到用户虚拟地址空间中时，确保了各个进程不会“非法”访问到其他进程的物理内存空间。

这样ucore给一个用户进程具体设定的虚拟内存空间（kern/mm/memlayout.h）如下所示：
![../img/lab3_image001.png](../img/lab3_image001.png) 

<br>

### 3. 创建并执行用户进程

在确定了用户进程的执行代码和数据，以及用户进程的虚拟空间布局后，我们可以来创建用户进程了。在本实验中第一个用户进程是由第二个内核线程initproc通过把hello应用程序执行码覆盖到initproc的用户虚拟内存空间来创建的，相关代码如下所示：
```c
// kernel_execve - do SYS_exec syscall to exec a user program called by user_main kernel_thread
static int kernel_execve(const char *name, unsigned char *binary, size_t size) {
    int ret, len = strlen(name);
    asm volatile(
    "addi.w   $a7, $zero,%1;\n" // syscall no.
    "move $a0, %2;\n"
    "move $a1, %3;\n"
    "move $a2, %4;\n"
    "move $a3, %5;\n"
    "syscall  0;\n"
    "move %0, $a7;\n"
    : "=r"(ret)
    : "i"(SYSCALL_BASE+SYS_exec), "r"(name), "r"(len), "r"(binary), "r"(size) 
    : "a0", "a1", "a2", "a3", "a7"
    );
    return ret;
}

#define __KERNEL_EXECVE(name, path, ...) ({                         \
const char *argv[] = {path, ##__VA_ARGS__, NULL};       \
                    kprintf("kernel_execve: pid = %d, name = \"%s\".\n",    \
                            current->pid, name);                            \
                    kernel_execve(name, argv);                              \
})

#define KERNEL_EXECVE(x, ...)                   __KERNEL_EXECVE(#x, #x, ##__VA_ARGS__)
……
// init_main - the second kernel thread used to create kswapd_main & user_main kernel threads
static int
init_main(void *arg) {
    #ifdef TEST
    KERNEL_EXECVE2(TEST, TESTSTART, TESTSIZE);
    #else
    KERNEL_EXECVE(hello);
    #endif
    panic("kernel_execve failed.\n");
    return 0;
}
```
对于上述代码，我们需要从后向前按照函数/宏的实现一个一个来分析。Initproc的执行主体是init_main函数，这个函数在缺省情况下是执行宏KERNEL_EXECVE(hello)，而这个宏最终是调用kernel_execve函数来调用SYS_exec系统调用，由于ld在链接hello应用程序执行码时定义了两全局变量：

- _binary_obj___user_hello_out_start：hello执行码的起始位置
- _binary_obj___user_hello_out_size中：hello执行码的大小
kernel_execve把这两个变量作为SYS_exec系统调用的参数，让ucore来创建此用户进程。当ucore收到此系统调用后，将依次调用如下函数:

```c
vector128(vectors.S)--\>
\_\_alltraps(trapentry.S)--\>trap(trap.c)--\>trap\_dispatch(trap.c)--
--\>syscall(syscall.c)--\>sys\_exec（syscall.c）--\>do\_execve(proc.c)
```
最终通过do_execve函数来完成用户进程的创建工作。此函数的主要工作流程如下：

- **首先为加载新的执行码做好用户态内存空间清空准备**。如果mm不为NULL，则设置页表为内核空间页表，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。由于此处的initproc是内核线程，所以mm为NULL，整个处理都不会做。

- 接下来的一步是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。这里涉及到读ELF格式的文件，申请内存空间，建立用户态虚存空间，加载应用程序执行码等。load_icode函数完成了整个复杂的工作。

load_icode函数的主要工作就是给用户进程建立一个能够让用户进程正常运行的用户环境。此函数有一百多行，完成了如下重要工作：

1. 调用mm_create函数来申请进程的内存管理数据结构mm所需内存空间，并对mm进行初始化；
mm是一个mm_struct结构的变量，这个数据结构表示了包含所有虚拟内存空间的共同属性(定义位于kern/mm/vmm.h)，具体定义如下：
```c
// the control struct for a set of vma using the same PDT
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    atomic_t mm_count;
    semaphore_t mm_sem;
    int locked_by;

};
```
    - mmap_list是双向链表头，链接了所有属于同一页目录表的虚拟内存空间。
    - mmap_cache是指向当前正在使用的虚拟内存空间，由于操作系统执行的“局部性”原理，当前正在用到的虚拟内存空间在接下来的操作中可能还会用到，这时就不需要查链表，而是直接使用此指针就可找到下一次要用到的虚拟内存空间。由于mmap_cache 的引入，可使得 mm_struct 数据结构的查询加速 30% 以上。
    - **pgdir 所指向的就是mm_struct数据结构所维护的页表。通过访问pgdir可以查找某虚拟地址对应的页表项是否存在以及页表项的属性等**。
    - map_count记录mmap_list 里面链接的 vma_struct的个数。

    涉及mm_struct的操作函数比较简单，只有mm_create和mm_destroy两个函数，从字面意思就可以看出是是完成mm_struct结构的变量创建和删除。

2. 调用setup_pgdir来申请一个页目录表所需的一个页大小的内存空间，并把描述ucore内核虚空间映射的内核页表（boot_pgdir所指）的内容拷贝到此新目录表中，最后让mm->pgdir指向此页目录表，这就是进程新的页目录表了，且能够正确映射内核虚空间；
setup_pgdir函数的定义如下(kern/process/proc.c)：
```c
// setup_pgdir - alloc one page as PDT
static int
setup_pgdir(struct mm_struct *mm) {
    struct Page *page;
    if ((page = alloc_page()) == NULL) { //申请一页的内存空间
        return -E_NO_MEM;
    }
    pde_t *pgdir = page2kva(page);
    memcpy(pgdir, boot_pgdir, PGSIZE);//将描述ucore内核虚空间映射的内核页表（boot_pgdir所指）的内容拷贝到此新目录表中
    //panic("unimpl");
    //pgdir[PDX(VPT)] = PADDR(pgdir) | PTE_P | PTE_W;
    mm->pgdir = pgdir; //将mm->pgdir指向此页目录表，使进程的页目录表能正确的映射内核虚空间
    return 0;
}
```

3. 根据应用程序执行码的起始位置来解析此ELF格式的执行程序，并调用mm_map函数根据ELF格式的执行程序说明的各个段（代码段、数据段、BSS段等）的起始位置和大小建立对应的vma结构，并把vma插入到mm结构中，从而表明了用户进程的合法用户态虚拟地址空间；
```c
mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)
```

4. 调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中，至此应用程序执行码和数据已经根据编译时设定地址放置到虚拟内存中了；

5. 需要给用户进程设置用户栈，为此调用mm_mmap函数建立用户栈的vma结构，明确用户栈的位置在用户虚空间的顶端，大小为256个页，即1MB，并分配一定数量的物理内存且建立好栈的虚地址<-->物理地址映射关系；

6. 至此,进程内的内存管理vma和mm数据结构已经建立完成，于是把mm->pgdir赋值到变量current_pgdir中，即更新了用户进程的虚拟内存空间，此时的initproc已经被hello的代码和数据覆盖，成为了第一个用户进程，但此时这个用户进程的执行现场还没建立好；

7. 先清空进程的中断帧，再重新设置进程的中断帧，使得在执行中断返回指令后，能够让CPU转到用户态特权级，并回到用户态内存空间，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；
这一步涉及练习1中需要补充的代码。如下所示：
```c
 #ifdef LAB3_EX1
/* LAB3:EXERCISE1 YOUR CODE
 * should set tf_era,tf_regs.reg_r[LOONGARCH_REG_SP],tf->tf_prmd
 * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel and enable interrupt. So
 *          tf->tf_prmd should be PLV_USER | CSR_CRMD_IE
 *          tf->tf_era should be the entry point of this binary program (elf->e_entry)
 *          tf->tf_regs.reg_r[LOONGARCH_REG_SP] should be the top addr of user stack (USTACKTOP)
 */
// 根据提示补充代码，完成练习。
#endif
```
我们需要补充的代码就是设置tf_era, tf_regs.regr[LOONGARCH_REG_SP],tf->tf_prmd这三个变量的内容。如果我们正确地设置了中断帧的内容，操作系统可以由内核态转为用户态运行，且重新开启中断响应。中断帧trapframe的结构定义如下(kern/trap/loongarch_trapframe.h)：
```c
struct trapframe {
    uint32_t tf_badv; // CSR[0x7]
    uint32_t tf_estat; // CSR[0x5]
    uint32_t tf_prmd; // CSR[0x1]
    uint32_t tf_ra;
    struct pushregs tf_regs;
    uint32_t tf_era; // CSR[0x6] same as epc on LOONGARCH
};
```
    - tf_regs: 当进程从内核态转为用户态时，要把堆栈寄存器的栈指针($sp)改为用户栈的地址，**即tf_regs[LOONGARCH_REG_SP]应设置为USTACKTOP**。
    - tf_era: 表示的是例外返回地址（触发例外指令的PC），为了使操作系统跳转到用户进程的第一条指令执行，练习1中的tf_era应设置为ELF执行文件设定的起始地址，**即tf_era应设置为elf->e_entry**。
    - tf_prmd: CSR[0x1]指的是控制状态寄存器例外前模式信息(PRMD)的内容(参考[龙芯架构32位精简版参考手册_v1.02.pdf](https://mirrors.tuna.tsinghua.edu.cn/loongson/docs/loongarch/%E9%BE%99%E8%8A%AF%E6%9E%B6%E6%9E%8432%E4%BD%8D%E7%B2%BE%E7%AE%80%E7%89%88%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C_v1.02.pdf)中p53)，若操作系统要从内核态切换至用户态，且开启中断响应，**tf_prmd应设置为PLV_USER | CSR_CRMD_IE**。


至此，用户进程的用户环境已经搭建完毕。此时initproc将按产生系统调用的函数调用路径原路返回，执行中断返回指令“ertn”（位于exception.S的最后一句）后，将切换到用户进程hello的第一条语句位置_start处（位于user/libs/initcode.S的第三句）开始执行。

## 进程退出和等待进程

当进程执行完它的工作后，就需要执行退出操作，释放进程占用的资源。ucore分了两步来完成这个工作，首先由进程本身完成大部分资源的占用内存回收工作，然后由此进程的父进程完成剩余资源占用内存的回收工作。

!!!note 
    为何不让进程本身完成所有的资源回收工作呢？
    这是因为进程要执行回收操作，就表明此进程还存在，还在执行指令，这就需要保证内核栈的空间和表示进程存在的进程控制块不被释放，所以需要父进程来帮忙释放子进程无法完成的这两个资源的回收工作。

为实现进程退出，用户态的函数库中提供了exit函数，此函数最终访问sys_exit系统调用接口让操作系统来帮助当前进程执行退出过程中的部分资源回收。我们来看看ucore是如何做进程退出工作的。

### ucore是如何做进程退出工作的

首先，exit函数会把一个退出码error_code传递给ucore，ucore通过执行内核函数do_exit来完成对当前进程的退出处理，主要工作就是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作。具体流程如下：

**1.** 如果current->mm != NULL，表示是用户进程，则开始回收此用户进程所占用的用户态虚拟内存空间；

    a) 首先执行“lcr3(boot_cr3)”，切换到内核态的页表上，这样当前用户进程目前只能在内核虚拟地址空间执行了，这是为了确保后续释放用户态内存和进程页表的工作能够正常执行；

    b) 如果当前进程控制块的成员变量mm的成员变量mm_count减1后为0（表明这个mm没有再被其他进程共享，可以彻底释放进程所占的用户虚拟空间了。），则开始回收用户进程所占的内存资源：

        i. 调用exit_mmap函数释放current->mm->vma链表中每个vma描述的进程合法空间中实际分配的内存，然后把对应的页表项内容清空，最后还把页表所占用的空间释放并把对应的页目录表项清空；

        ii. 调用put_pgdir函数释放当前进程的页目录所占的内存；

        iii. 调用mm_destroy函数释放mm中的vma所占内存，最后释放mm所占内存；

    c) 此时设置current->mm为NULL，表示与当前进程相关的用户虚拟内存空间和对应的内存管理成员变量所占的内核虚拟内存空间已经回收完毕；

**2.** 这时，设置当前进程的执行状态current->state=PROC_ZOMBIE，当前进程的退出码current->exit_code=error_code。此时当前进程已经不能被调度了，需要此进程的父进程来做最后的回收工作（即回收描述此进程的内核栈和进程控制块）；

**3.** 如果当前进程的父进程current->parent处于等待子进程状态：

current->parent->wait_state==WT_CHILD，

则唤醒父进程（即执行“wakup_proc(current->parent)”），让父进程帮助自己完成最后的资源回收；

**4.** 如果当前进程还有子进程，则需要把这些子进程的父进程指针设置为内核线程initproc，且各个子进程指针需要插入到initproc的子进程链表中。如果某个子进程的执行状态是PROC_ZOMBIE，则需要唤醒initproc来完成对此子进程的最后回收工作。

**5.** 执行schedule()函数，选择新的进程执行。

### 父进程如何完成对子进程的最后回收工作

这要求父进程要执行wait用户函数或wait_pid用户函数，这两个函数的区别是，wait函数等待任意子进程的结束通知，而wait_pid函数等待进程id号为pid的子进程结束通知。这两个函数最终访问sys_wait系统调用接口让ucore来完成对子进程的最后回收工作，即回收子进程的内核栈和进程控制块所占内存空间，具体流程如下：

**1.** 如果pid!=0，表示只找一个进程id号为pid的退出状态的子进程，否则找任意一个处于退出状态的子进程；

**2.** 如果此子进程的执行状态不为PROC_ZOMBIE，表明此子进程还没有退出，则当前进程（即子进程的父进程）只好设置自己的执行状态为PROC_SLEEPING，睡眠原因为WT_CHILD（即等待子进程退出），调用schedule()函数选择新的进程执行，自己睡眠等待，如果被唤醒，则重复跳回步骤1处执行；

**3.** 如果此子进程的执行状态为PROC_ZOMBIE，表明此子进程处于退出状态，需要当前进程完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列proc_list和hash_list中删除，并释放子进程的内核堆栈和进程控制块。自此，子进程才彻底地结束了它的执行过程，消除了它所占用的所有资源。

# 系统调用实现

系统调用的英文名字是System Call。操作系统为什么需要实现系统调用呢？其实这是实现了用户进程后，自然引申出来需要实现的操作系统功能。用户进程只能在操作系统给它圈定好的“用户环境”中执行，但“用户环境”限制了用户进程能够执行的指令，即用户进程只能执行一般的指令，无法执行特权指令。如果用户进程想执行一些需要特权指令的任务，比如通过网卡发网络包等，只能让操作系统来代劳了。于是就需要一种机制来确保用户进程不能执行特权指令，但能够请操作系统“帮忙”完成需要特权指令的任务，这种机制就是系统调用。

采用系统调用机制为用户进程提供一个获得操作系统服务的统一接口层，这样一来可简化用户进程的实现，把一些共性的、繁琐的、与硬件相关、与特权指令相关的任务放到操作系统层来实现，但提供一个简洁的接口给用户进程调用；二来这层接口事先可规定好，且严格检查用户进程传递进来的参数和操作系统要返回的数据，使得让操作系统给用户进程服务的同时，保护操作系统不会被用户进程破坏。

从硬件层面上看，需要硬件能够支持在用户态的用户进程通过某种机制切换到内核态。lab1讲述中断硬件支持和软件处理过程其实就可以用来完成系统调用所需的软硬件支持。下面我们来看看如何在ucore中实现系统调用。

## 1. 建立系统调用的用户库准备

在操作系统中初始化好系统调用相关的中断描述符、中断处理起始地址等后，还需在用户态的应用程序中初始化好相关工作，简化应用程序访问系统调用的复杂性。为此在用户态建立了一个中间层，即简化的libc实现，在user/libs/ulib.[ch]和user/libs/syscall.[ch]中完成了对访问系统调用的封装。用户态最终的访问系统调用函数是syscall，实现如下：

```
static inline int
syscall(int num, ...) {
    va_list ap;
    va_start(ap, num);
    uint32_t arg[MAX_ARGS];
    int i, ret;
    for (i = 0; i < MAX_ARGS; i ++) {
        arg[i] = va_arg(ap, uint32_t);
    }
    va_end(ap);

    num += SYSCALL_BASE;
    asm volatile(
      "move $a7, %1;\n" /* syscall no. */
      "move $a0, %2;\n"
      "move $a1, %3;\n"
      "move $a2, %4;\n"
      "move $a3, %5;\n"
      "syscall 0;\n"
      "move %0, $a7;\n"
      : "=r"(ret)
      : "r"(num), "r"(arg[0]), "r"(arg[1]), "r"(arg[2]), "r"(arg[3]) 
      : "a0", "a1", "a2", "a3", "a7"
    );
    return ret;
}
```

## 2. 与用户进程相关的系统调用

在本实验中，与进程相关的各个系统调用属性如下表所示，通过这些系统调用，可方便地完成从进程/线程创建到退出的整个运行过程。

| 系统调用名      | 含义                                        | 具体完成服务的函数                                                                |
| ---------- | ----------------------------------------- | ------------------------------------------------------------------------ |
| SYS_exit   | process exit                              | do_exit                                                                  |
| SYS_fork   | create child process, dup mm              | do_fork-->wakeup_proc                                                    |
| SYS_wait   | wait child process                        | do_wait                                                                  |
| SYS_exec   | after fork, process execute a new program | load a program and refresh the mm                                        |
| SYS_yield  | process flag itself need resecheduling    | proc->need_sched=1, then scheduler will rescheule this process           |
| SYS_kill   | kill process                              | do_kill-->proc->flags \|= PF_EXITING, -->wakeup_proc-->do_wait-->do_exit |
| SYS_getpid | get the process's pid                     |                                                                          |

## 3. 系统调用的执行过程

与用户态的函数库调用执行过程相比，系统调用执行过程的有四点主要的不同：

- 通过“syscall”指令发起调用；
- 通过“ertn”指令完成调用返回；
- 当到达内核态后，操作系统需要严格检查系统调用传递的参数，确保不破坏整个系统的安全性；
- 执行系统调用可导致进程等待某事件发生，从而可引起进程切换；

下面我们以getpid系统调用的执行过程大致看看操作系统是如何完成整个执行过程的。当用户进程调用getpid函数，最终执行到“syscall”指令后，CPU根据操作系统建立的系统调用中断描述符，转入内核态，并跳转到exception13处（kern/trap/exception.S），开始了操作系统的系统调用执行过程，函数调用和返回操作的关系如下所示：

> exception13(exception.S)--\>
> \_\loongarch_trap(trap.c)--\>trap\_dispatch(trap.c)--
> --\>syscall(syscall.c)--\>sys\_getpid(syscall.c)--\>……--\>\_\exception_return(exception.S)

在执行trap函数前，软件还需进一步保存执行系统调用前的执行现场，把与用户进程继续执行所需的相关寄存器等当前内容保存到当前进程的中断帧trapframe中（注意，在创建进程时，把进程的trapframe放在进程内核栈分配空间的顶部）。软件做的工作在exception.S的exception_handler函数部分：

```
exception_handler:
    // Save t0 and t1
    csrwr   t0, LISA_CSR_KS0
    csrwr   t1, LISA_CSR_KS1
    // Save previous stack pointer in t1
    move    t1, sp
    csrwr   t1, LISA_CSR_KS2
    //t1 saved the vaual of KS2,KS2 saved sp
    /*
        Warning: csrwr will bring the old csr register value into rd, 
        not only just write rd to csr register,
        so you may see the rd changed.
        It's documented in the manual from loongarch.
    */

    // check if user mode
    csrrd   t0, LISA_CSR_PRMD  
    andi    t0, t0, 3
    beq     t0, zero, 1f


    /* Coming from user mode - load kernel stack into sp */
    la      t0, current // current pointer
    ld.w    t0, t0, 0 // proc struct
    ld.w    t0, t0, 12 // kstack pointer
    addi.w  t1, zero, 1
    slli.w  t1, t1, 13 // KSTACKSIZE=8192=pow(2,13)
    add.w   sp, t0, t1
    csrrd   t1, LISA_CSR_KS2

1:
    //saved EXST to t0 for save EXST to sp later(line 114) 
    csrrd   t0, LISA_CSR_EXST
    //return KS2
    csrrd   t1, LISA_CSR_KS2
    b common_exception


common_exception:
   /*
    * At this point:
    *      Interrupts are off. (The processor did this for us.)
    *      t0 contains the exception status(like exception cause on MIPS).
    *      t1 contains the old stack pointer.
    *      sp points into the kernel stack.
    *      All other registers are untouched.
    */

   /*
    * Allocate stack space for 35 words to hold the trap frame,
    * plus four more words for a minimal argument block.
    */
    addi.w  sp, sp, -156
    st.w    s8, sp, 148
    st.w    s7, sp, 144
    st.w    s6, sp, 140
    st.w    s5, sp, 136
    st.w    s4, sp, 132
    st.w    s3, sp, 128
    st.w    s2, sp, 124
    st.w    s1, sp, 120
    st.w    s0, sp, 116
    st.w    fp, sp, 112
    st.w    reserved_reg, sp, 108
    st.w    t8, sp, 104
    st.w    t7, sp, 100
    st.w    t6, sp, 96
    st.w    t5, sp, 92
    st.w    t4, sp, 88
    st.w    t3, sp, 84
    st.w    t2, sp, 80
    //st.w    t1, sp, 76
    //st.w    t0, sp, 72
    st.w    a7, sp, 68
    st.w    a6, sp, 64
    st.w    a5, sp, 60
    st.w    a4, sp, 56
    st.w    a3, sp, 52
    st.w    a2, sp, 48
    st.w    a1, sp, 44
    st.w    a0, sp, 40
    st.w    t1, sp, 36  // replace sp with real sp, now use t1 for free
    st.w    tp, sp, 32
    // save real t0 and t1 after real sp (stored in t1 previously) stored
    csrrd   t1, LISA_CSR_KS1
    st.w    t1, sp, 76
    csrrd   t1, LISA_CSR_KS0
    st.w    t1, sp, 72

    // replace with real value
    // save tf_era after t0 and t1 saved
    csrrd   t1, LISA_CSR_EPC
    st.w    t1, sp, 152

   /*
    * Save remaining exception context information.
    */

    // save ra (note: not in pushregs, it's tf_ra)
    st.w    ra, sp, 28
    // save prmd
    csrrd   t1, LISA_CSR_PRMD
    st.w    t1, sp, 24
    // save estat
    st.w    t0, sp, 20
    // now use t0 for free
    // store badv
    csrrd   t0, LISA_CSR_BADV
    st.w    t0, sp, 16
    st.w    zero, sp, 12
    // support nested interrupt

    // IE and PLV will automatically set to 0 when trap occur

    // set trapframe as function argument
    addi.w  a0, sp, 16
    li    t0, 0xb0    # PLV=0, IE=0, PG=1
    csrwr    t0, LISA_CSR_CRMD
    la.abs  t0, loongarch_trap
    jirl    ra, t0, 0
    //bl loongarch_trap
……
```

自此，用于保存用户态的用户进程执行现场的trapframe的内容填写完毕，操作系统可开始完成具体的系统调用服务。

在sys_getpid函数中，简单地把当前进程的pid成员变量做为函数返回值就是一个具体的系统调用服务。完成服务后，操作系统按调用关系的路径原路返回到__exception.S中。然后操作系统开始根据当前进程的中断帧内容做恢复执行现场操作。其实就是把trapframe的一部分内容保存到寄存器内容。这时内核栈的结构如下：

```
exception_return:
    // restore prmd
    ld.w    t0, sp, 24
    li      t1, 7
    csrxchg t0, t1, LISA_CSR_PRMD
    // restore era no k0 and k1 for la32, so must do first
    ld.w    t0, sp, 152
    csrwr   t0, LISA_CSR_EPC
    // restore general registers
    ld.w    ra, sp, 28
    ld.w    tp, sp, 32
    //ld.w    sp, sp, 36 (do it finally)
    ld.w    a0, sp, 40
    ld.w    a1, sp, 44
    ld.w    a2, sp, 48
    ld.w    a3, sp, 52
    ld.w    a4, sp, 56
    ld.w    a5, sp, 60
    ld.w    a6, sp, 64
    ld.w    a7, sp, 68
    ld.w    t0, sp, 72
    ld.w    t1, sp, 76
    ld.w    t2, sp, 80
    ld.w    t3, sp, 84
    ld.w    t4, sp, 88
    ld.w    t5, sp, 92
    ld.w    t6, sp, 96
    ld.w    t7, sp, 100
    ld.w    t8, sp, 104
    ld.w    reserved_reg, sp, 108
    ld.w    fp, sp, 112
    ld.w    s0, sp, 116
    ld.w    s1, sp, 120
    ld.w    s2, sp, 124
    ld.w    s3, sp, 128
    ld.w    s4, sp, 132
    ld.w    s5, sp, 136
    ld.w    s6, sp, 140
    ld.w    s7, sp, 144
    ld.w    s8, sp, 148
    // restore sp
    ld.w    sp, sp, 36
    ertn

    .end exception_return
    .end common_exception
```

这时执行“ertn”指令后，CPU根据内核栈的情况恢复到用户态，即“syscall”后的那条指令。这样整个系统调用就执行完毕了。

至此，实验三中的主要工作描述完毕。


# 附录 A：用户进程的特征

## 从内核线程到用户进程

对于内核线程来说，它们不需要在用户态执行用户进程的管理机制，既无法体现用户进程的地址空间，以及用户进程间地址空间隔离的保护机制，不支持进程执行过程的用户态和核心态之间的切换，且没有用户进程的完整状态变化的生命周期。

**内核线程的管理** 实现相对是简单的，其特点是直接使用操作系统（比如ucore）在初始化中建立的内核虚拟内存地址空间，不同的内核线程之间可以通过调度器实现线程间的切换，达到分时使用CPU的目的。由于内核虚拟内存空间是一一映射计算机系统的物理空间的，这使得可用空间的大小不会超过物理空间大小，所以操作系统程序员编写内核线程时，需要考虑到有限的地址空间，需要保证各个内核线程在执行过程中不会破坏操作系统的正常运行。这样在实现内核线程管理时，不必考虑涉及与进程相关的虚拟内存管理中的缺页处理、按需分页、写时复制、页换入换出等功能。如果在内核线程执行过程中出现了访存错误异常或内存不够的情况，就认为操作系统出现错误了，操作系统将直接宕机。在ucore中，就是调用panic函数，进入内核调试监控器kernel_debug_monitor。

内核线程管理思想相对简单，但 **编写内核线程对程序员的要求很高** 。从理论上讲（理想情况），如果程序员都是能够编写操作系统级别的“高手”，能够勤俭和高效地使用计算机系统中的资源，且这些“高手”都为他人着想，具有奉献精神，在别的应用需要计算机资源的时候，能够从大局出发，从整个系统的执行效率出发，让出自己占用的资源，那这些“高手”编写出来的程序直接作为内核线程运行即可，也就没有用户进程存在的必要了。

但现实与理论的差距是巨大的，能编写操作系统的程序员是极少数的，与当前的应用程序员相比，估计大约差了3~4个数量级。如果还要求编写操作系统的程序员考虑其他未知程序员的未知需求，那这样的程序员估计可以成为是编程界的“上帝”了。

从应用程序编写和运行的角度看，既然程序员都不是“上帝”，操作系统程序员就需要给应用程序员编写的程序提供一个既“宽松”又“严格”的执行环境，让对内存大小和CPU使用时间等资源的限制没有仔细考虑的应用程序都能在操作系统中正常运行，且即使程序太可靠，也只能破坏自己，而不能破坏其他运行程序和整个系统。

**严格** 就是安全性保证，即应用程序执行不会破坏在内存中存在的其他应用程序和操作系统的内存空间等独占的资源；

**宽松** 就算是方便性支持，即提供给应用程序尽量丰富的服务功能和一个远大于物理内存空间的虚拟地址空间，使得应用程序在执行过程中不必考虑很多繁琐的细节（比如如何初始化PCI总线和外设等，如何管理物理内存等）。

## 让用户进程正常运行的用户环境

在操作系统原理的介绍中，一般提到进程的概念其实主要是指用户进程。从操作系统的设计和实现的角度看，其实用户进程是指一个应用程序在操作系统提供的一个用户环境中的一次执行过程。这里的重点是用户环境。

### 用户环境的功能

从功能上看，操作系统提供的这个用户环境有两方面的特点。

一方面与 **存储空间** 相关，即限制用户进程可以访问的物理地址空间，且让各个用户进程之间的物理内存空间访问不重叠，这样可以保证不同用户进程之间不能相互破坏各自的内存空间，利用虚拟内存的功能（页换入换出），给用户进程提供了远大于实际物理内存空间的虚拟内存空间。

另一方面与 **执行指令** 相关，即限制用户进程可执行的指令，不能让用户进程执行特权指令（比如修改页表起始地址），从而保证用户进程无法破坏系统。但如果不能执行特权指令，则很多功能（比如访问磁盘等）无法实现，所以需要提供某种机制，让操作系统完成需要特权指令才能做的各种服务功能，给用户进程一个“服务窗口”,用户进程可以通过这个“窗口”向操作系统提出服务请求，由操作系统来帮助用户进程完成需要特权指令才能做的各种服务。另外，还要有一个“中断窗口”，让用户进程不主动放弃使用CPU时，操作系统能够通过这个“中断窗口”强制让用户进程放弃使用CPU，从而让其他用户进程有机会执行。

### 用户环境的组成部分

基于功能分析，我们可以把用户环境定义为如下组成部分：

- **建立用户虚拟空间的页表和支持页换入换出机制的用户内存访存错误异常服务例程** ：提供地址隔离和超过物理空间大小的虚存空间。
- **应用程序执行的用户态CPU特权级** ：在用户态CPU特权级，应用程序只能执行一般指令，对于特权指令，结果不是无效就是产生“执行非法指令”异常；
- **系统调用机制** ：给用户进程提供“服务窗口”；
- **中断响应机制** ：给用户进程设置“中断窗口”，这样产生中断后，当前执行的用户进程将被强制打断，CPU控制权将被操作系统的中断服务例程使用。

## 用户态进程的执行过程分析

在用户环境下运行的进程就是用户进程。那如果用户进程由于某种原因进入内核态，那在内核态执行的是什么呢？还是用户进程吗？
首先分析一下用户进程这样会进入内核态呢？回顾一下lab1，就可以知道当产生外设中断、CPU执行异常（比如访存错误）、陷入（系统调用），用户进程就会切换到内核中的操作系统中来。表面上看，到内核态后，操作系统取得了CPU控制权，所以现在执行的应该是操作系统代码，由于此时CPU处于核心态特权级，所以操作系统的执行过程就就应该是内核进程了。但是这样理解忽略了操作系统的具体实现。如果考虑操作系统的具体实现，应该如果来理解进程呢？

从 **进程控制块** 的角度看，如果执行了进程执行现场（上下文）的切换，就认为到另外一个进程执行了，及进程的分界点设定在执行进程切换的前后。到底切换了什么呢？其实只是切换了进程的页表和相关硬件寄存器，这些信息都保存在进程控制块中的相关域中。所以，我们可以把执行应用程序的代码一直到执行操作系统中的进程切换处为止都认为是一个应用程序的执行过程（其中有操作系统的部分代码执行过过程），即进程。因为在这个过程中，没有更换到另外一个进程控制块的进程的页表和相关硬件寄存器。

从 **指令执行** 的角度看，操作系统的主要功能是给上层应用提供服务，管理整个计算机系统中的资源。所以操作系统虽然是一个软件，但其实是一个基于事件的软件，这里操作系统需要响应的事件包括三类：外设中断、CPU执行异常（比如访存错误）、陷入（系统调用）。如果用户进程通过系统调用要求操作系统提供服务，那么从用户进程的角度看，操作系统就是一个特殊的软件库（比如相对于用户态的libc库，操作系统可看作是内核态的libc库），完成用户进程的需求。从执行逻辑上看，是用户进程“主观”执行的一部分，即用户进程“知道”操作系统要做的事情。那么在这种情况下，进程的代码空间包括用户态的执行程序和内核态响应用户进程通过系统调用而在核心特权态执行服务请求的操作系统代码。

为此这种情况下的进程的内存虚拟空间也包括两部分：用户态的虚地址空间和核心态的虚地址空间。但如果此时发生的事件是外设中断和CPU执行异常，虽然CPU控制权也转入到操作系统中的中断服务例程，但这些内核执行代码执行过程是用户进程“不知道”的，是另外一段执行逻辑。那么在这种情况下， **实际上是执行了两段目标不同的执行程序，一个是代表应用程序的用户进程，一个是代表中断服务例程处理外设中断和CPU执行异常的内核线程** 。这个用户进程和内核线程在产生中断或异常的时候，CPU硬件就完成了它们之间的指令流切换。

## 用户进程的运行状态分析

用户进程在其执行过程中会存在很多种不同的执行状态，根据操作系统原理，一个用户进程一般的运行状态有五种：创建（new）态、就绪（ready）态、运行（running）态、等待（blocked）态、退出（exit）态。各个状态之间会由于发生了某事件而进行状态转换。

### 进程运行状态如何转变

首先，在 **创建（new）态** ，操作系统完成进程的创建工作，而体现进程存在的就是进程控制块，所以一旦操作系统创建了进程控制块，则可以认为此时进程就已经存在了，但由于进程能够运行的各种资源还没准备好，所以此时的进程处于创建（new）态。

创建了进程控制块后，进程并不能直接执行，还需准备好各种资源，如果把进程执行所需要的虚拟内存空间，执行代码，要处理的数据等都准备好了，则此时进程已经可以执行了，但还没有被操作系统调度，需要等待操作系统选择这个进程执行，于是把这个做好“执行准备”的进程放入到一个队列中，并可以认为此时进程处于 **就绪（ready）态** 。

当操作系统的调度器从就绪进程队列中选择了一个就绪进程后，通过执行进程切换，就让这个被选上的就绪进程执行了，此时进程就处于 **运行（running）态** 了。

到了运行态后，会出现三种事件：

- 如果进程需要等待某个事件（比如主动睡眠10秒钟，或进程访问某个内存空间，但此内存空间被换出到硬盘swap分区中了，进程不得不等待操作系统把缓慢的硬盘上的数据重新读回到内存中），那么操作系统会把CPU给其他进程执行，并把进程状态从运行（running）态转换为 **等待（blocked）态** 。
- 如果用户进程的应用程序逻辑流程执行结束了，那么操作系统会把CPU给其他进程执行，并把进程状态从运行（running）态转换为 **退出（exit）态** ，并准备回收用户进程占用的各种资源，当把表示整个进程存在的进程控制块也回收了，这进程就不存在了。在这整个回收过程中，进程都处于退出（exit）态。
!!!info
    考虑到在内存中存在多个处于就绪(ready)态的用户进程，但只有一个CPU，所以为了公平起见，**每个就绪态进程都只有有限的时间片段** ，当一个运行态的进程用完了它的时间片段后，操作系统会剥夺此进程的CPU使用权，并把此进程状态从运行（running）态转换为就绪（ready）态，最后把CPU给其他进程执行。
- 如果某个处于等待（blocked）态的进程所等待的事件产生了（比如睡眠时间到，或需要访问的数据已经从硬盘换入到内存中），则操作系统会通过把等待此事件的进程状态从等待（blocked）态转到 **就绪（ready）态** 。这样进程的整个状态转换形成了一个有限状态自动机。
