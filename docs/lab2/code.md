## 以页为单位管理物理内存

在获得可用物理内存范围后，系统需要建立相应的数据结构来管理以物理页（按4KB对齐，且大小为4KB的物理内存单元）为最小单位的整个物理内存，以配合后续涉及的分页管理机制。每个物理页可以用一个Page数据结构来表示。由于一个物理页需要占用一个Page结构的空间，Page结构在设计时须尽可能小，以减少对内存的占用。Page的定义在kern/mm/memlayout.h中。以页为单位的物理内存分配管理的实现在kern/default\_pmm.c中。

为了与以后的分页机制配合，我们首先需要建立对整个计算机的每一个物理页的属性用结构Page来表示，它包含了映射此物理页的虚拟页个数，描述物理页属性的flags和双向链接各个Page结构的page\_link双向链表。

```c
struct Page {

    int ref;        // page frame's reference counter

    uint32_t flags; // array of flags that describe the status of the page frame

    unsigned int property; // used in buddy system, stores the order (the X in 2^X) of the continuous memory block

    int zone_num;  // used in buddy system, the No. of zone which the page belongs to

    list_entry_t page_link;// free list link

    list_entry_t swap_link;  // swap hash link

};
```

这里看看Page数据结构的各个成员变量有何具体含义。ref表示这页被页表的引用记数（在“实现分页机制”一节会讲到）。如果这个页被页表引用了，即在某页表中有一个页表项设置了一个虚拟页到这个Page管理的物理页的映射关系，就会把Page的ref加一；反之，若页表项取消，即映射关系解除，就会把Page的ref减一。flags表示此物理页的状态标记，进一步查看kern/mm/memlayout.h中的定义，可以看到：

```c
/* Flags describing the status of a page frame */

#define PG_reserved                 0       // the page descriptor is reserved for kernel or unusable

#define PG_property                 1       // the member 'property' is valid

#define PG_slab                     2       // page frame is included in a slab

#define PG_dirty                    3       // the page has been modified

#define PG_swap                     4       // the page is in the active or inactive page list (and swap hash table)

#define PG_active                   5       // the page is in the active page list
```

这表示flags目前用到了六个bit表示页目前具有的六种属性，bit 0表示此页是否被保留（reserved），如果是被保留的页，则bit 0会设置为1，且不能放到空闲页链表中，即这样的页不是空闲页，不能动态分配与释放。比如目前内核代码占用的空间就属于这样“被保留”的页。在本实验中，bit 1表示此页是否是free的，如果设置为1，表示这页是free的，可以被分配；如果设置为0，表示这页已经被分配出去了，不能被再二次分配。

另外，本实验这里取的名字PG\_property比较不直观，主要是因为我们可以设计不同的页分配算法（best fit, buddy system等），那么这个PG\_property就有不同的含义了。

在本实验中，Page数据结构的成员变量property用来记录某连续内存空闲块的大小（即地址连续的空闲页的个数）。这里需要注意的是用到此成员变量的这个Page比较特殊，是这个连续内存空闲块地址最小的一页（即头一页，Head Page）。连续内存空闲块利用这个页的成员变量property来记录在此块内的空闲页的个数。这里取的名字property也不是很直观，原因与上面类似，在不同的页分配算法中，property有不同的含义。

Page数据结构的成员变量page\_link是便于把多个连续内存空闲块链接在一起的双向链表指针（可回顾在lab0实验指导书中有关双向链表数据结构的介绍）。这里需要注意的是用到此成员变量的这个Page比较特殊，是这个连续内存空闲块地址最小的一页（即头一页，Head Page）。连续内存空闲块利用这个页的成员变量page\_link来链接比它地址小和大的其他连续内存空闲块。

在初始情况下，也许这个物理内存的空闲物理页都是连续的，这样就形成了一个大的连续内存空闲块。但随着物理页的分配与释放，这个大的连续内存空闲块会分裂为一系列地址不连续的多个小连续内存空闲块，且每个连续内存空闲块内部的物理页是连续的。那么为了有效地管理这些小连续内存空闲块。所有的连续内存空闲块可用一个双向链表管理起来，便于分配和释放，为此定义了一个free\_area\_t数据结构，包含了一个list\_entry结构的双向链表指针和记录当前空闲页的个数的无符号整型变量nr\_free。其中的链表指针指向了空闲的物理页。

```c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */

typedef struct {

            list_entry_t free_list;                                // the list header

            unsigned int nr_free;                                 // # of free pages in this free list

} free_area_t;
```

有了这两个数据结构，ucore就可以管理起来整个以页为单位的物理内存空间。接下来需要解决两个问题：

- 管理页级物理内存空间所需的Page结构的内存空间从哪里开始，占多大空间？

- 空闲内存空间的起始地址在哪里？

对于这两个问题，我们首先根据memlayout.h给出的核心态内存基地址KERNBASE（0xa0000000）和KMEMSIZE（512M）计算出最大的物理内存地址KERNTOP（定义在memlayout.h中），最大物理内存地址maxpa等于KERNTOP（定义在page\_init函数中的局部变量），在该实验中Page size为4096 bytes即2^12 bytes，所以需要管理的物理页的个数npage的计算方式如下：

```c
#define KERNTOP             (KERNBASE + KMEMSIZE)

maxpa = KERNTOP

npage = KMEMSIZE >> PGSHIFT    # PGSHIFT = 12
```

这样，我们就可以预估出管理页级物理内存空间所需的Page结构的内存空间所需的内存大小为：

```c
sizeof(struct Page) * npage
```

由于加载内核的结束地址（用全局指针变量end记录）以上的空间没有被使用，所以我们可以把end按页大小为边界取整后，作为管理页级物理内存空间所需的Page结构的内存空间，记为：

```c
pages = (struct Page *)ROUNDUP_2N((void *)end, PGSHIFT);
```

为了简化起见，从地址0到地址pages+ sizeof(struct Page) \*npage)结束的物理内存空间设定为已占用物理内存空间（起始0\~640KB的空间是空闲的），地址pages+sizeof(struct Page) \*npage)以上的空间为空闲物理内存空间，这时的空闲空间起始地址为

```c
uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);
```

为此我们需要把这两部分空间给标识出来。首先，对于所有物理空间，通过如下语句即可实现占用标记：

```c
for (i = 0; i < npage; i ++) {

SetPageReserved(pages + i);

}
```

然后，根据探测到的空闲物理空间，通过如下语句即可实现空闲标记：

```c
//获得空闲空间的起始地址begin和结束地址end

...

init_memmap(pa2page(mbegin), (mend - mbegin) >> PGSHIFT );
```

其实SetPageReserved只需把物理地址对应的Page结构中的flags标志设置为PG\_reserved，表示这些页已经被使用了，将来不能被用于分配。而init\_memmap函数则是把空闲物理页对应的Page结构中的flags和引用计数ref清零，并加到free\_area.free\_list指向的双向列表中，为将来的空闲页管理做好初始化准备工作。

关于内存分配的操作系统原理方面的知识有很多，但在本实验中只实现了最简单的内存页分配算法。相应的实现在default\_pmm.c中的default\_alloc\_pages函数和default\_free\_pages函数，相关实现很简单，这里就不具体分析了，直接看源码，应该很好理解。

其实实验二在内存分配和释放方面最主要的作用是建立了一个物理内存页管理器框架，这实际上是一个函数指针列表，定义如下：

```c
struct pmm_manager {

    const char *name;                                 // XXX_pmm_manager's name

    void (*init)(void);                               // initialize internal description&management data structure

                                                      // (free block list, number of free block) of XXX_pmm_manager

    void (*init_memmap)(struct Page *base, size_t n); // setup description&management data structcure according to

                                                      // the initial free physical memory space

    struct Page *(*alloc_pages)(size_t n);            // allocate >=n pages, depend on the allocation algorithm

    void (*free_pages)(struct Page *base, size_t n);  // free >=n pages with "base" addr of Page descriptor structures(memlayout.h)

    size_t (*nr_free_pages)(void);                    // return the number of free pages

    void (*check)(void);                              // check the correctness of XXX_pmm_manager

};
```

重点是实现init\_memmap/ alloc\_pages/free\_pages这三个函数。

### 物理内存页分配算法实现

如果要在ucore中实现连续物理内存分配算法，则需要考虑的事情比较多，相对课本上的物理内存分配算法描述要复杂不少。下面介绍一下如果要实现一个FirstFit内存分配算法的大致流程。

lab2的第一部分是完成first\_fit的分配算法。原理FirstFit内存分配算法上很简单，但要在ucore中实现，需要充分了解和利用ucore已有的数据结构和相关操作、关键的一些全局变量等。

**关键数据结构和变量**

first\_fit分配算法需要维护一个查找有序（地址按从小到大排列）空闲块（以页为最小单位的连续地址空间）的数据结构，而双向链表是一个很好的选择。

kernel/include/list.h定义了可挂接任意元素的通用双向链表结构和对应的操作，所以需要了解如何使用这个文件提供的各种函数，从而可以完成对双向链表的初始化/插入/删除等。

kern/mm/memlayout.h中定义了一个 free\_area\_t 数据结构，包含成员结构

```c
  list_entry_t free_list;         // the list header   空闲块双向链表的头

  unsigned int nr_free;           // # of free pages in this free list  空闲块的总数（以页为单位）
```

显然，我们可以通过此数据结构来完成对空闲块的管理。而buddy\_pmm.c中定义的free\_area变量就是干这个事情的。

kern/mm/pmm.h中定义了一个通用的分配算法的函数列表，用pmm\_manager表示。其中init函数就是用来初始化free\_area变量的,first\_fit分配算法可直接重用buddy\_init函数的实现。init\_memmap函数需要根据现有的内存情况构建空闲块列表的初始状态。何时应该执行这个函数呢？

通过分析代码，可以知道：

```
kern_init --> pmm_init-->page_init-->init_memmap--> pmm_manager->init_memmap
```

所以，default\_init\_memmap需要根据page\_init函数中传递过来的参数（某个连续地址的空闲块的起始页，页个数）来建立一个连续内存空闲块的双向链表。这里有一个假定page\_init函数是按地址从小到大的顺序传来的连续内存空闲块的。链表头是free\_area.free\_list，链表项是Page数据结构的base-\>page\_link。这样我们就依靠Page数据结构中的成员变量page\_link形成了连续内存空闲块列表。

**设计实现**

default\_init\_memmap函数将根据每个物理页帧的情况来建立空闲页链表，且空闲页块应该是根据地址高低形成一个有序链表。根据上述变量的定义，default\_init\_memmap可大致实现如下：

```c
default_init_memmap(struct Page *base, size_t n) {

    assert(n > 0);

    struct Page *p = base;

    for (; p != base + n; p ++) {

        assert(PageReserved(p));

        p->flags = p->property = 0;

        set_page_ref(p, 0);

    }

    base->property = n;

    SetPageProperty(base);

    nr_free += n;

    list_add_before(&free_list, &(base->page_link));

}
```

如果要分配一个页，那要考虑哪些呢？这里就需要考虑实现default\_alloc\_pages函数，注意参数n表示要分配n个页。另外，需要注意实现时尽量多考虑一些边界情况，这样确保软件的鲁棒性。比如

```c
if (n > nr_free) {

return NULL;

}
```

这样可以确保分配不会超出范围。也可加一些assert函数，在有错误出现时，能够迅速发现。比如 n应该大于0，我们就可以加上

```c
assert(n > 0);
```

这样在n<=0的情况下，ucore会迅速报错。firstfit需要从空闲链表头开始查找最小的地址，通过list\_next找到下一个空闲块元素，通过le2page宏可以由链表元素获得对应的Page指针p。通过p-\>property可以了解此空闲块的大小。如果\>=n，这就找到了！如果<n，则list\_next，继续查找。直到list\_next==

&free\_list，这表示找完了一遍了。找到后，就要从新组织空闲块，然后把找到的page返回。所以default\_alloc\_pages可大致实现如下：

```c
default_alloc_pages(size_t n) {

    assert(n > 0);

    if (n > nr_free) {

        return NULL;

    }

    struct Page *page = NULL;

    list_entry_t *le = &free_list;

    // TODO: optimize (next-fit)

    while ((le = list_next(le)) != &free_list) {                                    

        struct Page *p = le2page(le, page_link);

        if (p->property >= n) {

            page = p;

            break;

        }

    }

    if (page != NULL) {

        if (page->property > n) {

            struct Page *p = page + n;

            p->property = page->property - n;

            SetPageProperty(p);

            list_add_after(&(page->page_link), &(p->page_link));

        }

        list_del(&(page->page_link));

        nr_free -= n;

        ClearPageProperty(page);

    }

    return page;

}
```

default\_free\_pages函数的实现其实是default\_alloc\_pages的逆过程，不过需要考虑空闲块的合并问题。这里就不再细讲了。注意，上诉代码只是参考设计，不是完整的正确设计。更详细的说明位于lab2/kernel/mm/default\_pmm.c的注释中。希望同学能够顺利完成本实验的第一部分。

### 系统执行中地址映射的过程

在lab1和lab2中都会涉及如何建立映射关系的操作，在龙芯架构中MMU支持两种地址翻译模式：直接地址翻译模式和映射地址翻译模式。直接地址翻译模式下物理地址默认直接等于虚拟地址的[PALEN-1:0]位（不足补0）。当处理器核的MMU处于映射地址翻译模式，具体又分为直接映射地址翻译模式和页表映射地址翻译模式。直接映射地址模式是通过直接映射配置窗口机制完成虚实地址的直接映射，映射地址翻译模式通过页表完成映射。

在lab1和lab2中我们在ucore的入口函数kernel_entry（entry.S文件中）中设置了CSR.CRMD的DA=0且PG=1，即处理器核的MMU处于映射地址翻译模式。同时，我们将CSR.DWM0寄存器的值设置为0xa0000001，表示 0xa0000000-0xbfffffff段的虚拟地址通过直接映射地址翻译模式映射到0x00000000-0x1fffffff的物理地址。将CSR.DWM1寄存器的值设置为0x80000011，表示0x80000000-0x9fffffff段的虚拟地址用过直接映射地址翻译模式映射到0x00000000-0x1fffffff的物理地址。除了这两段虚拟地址之外的虚拟地址都是通过页表映射地址翻译模式进行映射。

下面，我们来看看如何改变内核的起始地址。观察一下链接脚本，即tools/kernel.ld文件：

```
OUTPUT_ARCH(loongarch)

ENTRY(kernel_entry)

SECTIONS

{

    . = 0xa0000000;



  .text      :

  {

    . = ALIGN(4);

    wrs_kernel_text_start = .; _wrs_kernel_text_start = .;

    *(.startup)

    *(.text)

    *(.text.*)

    *(.gnu.linkonce.t*)

    *(.mips16.fn.*)

    *(.mips16.call.*) /* for MIPS */

    *(.rodata) *(.rodata.*) *(.gnu.linkonce.r*) *(.rodata1)

    . = ALIGN(4096);

    *(.ramexv)

  }
```

从上述代码可以看出ld工具形成的ucore的起始虚拟地址从0xa0000000开始，由于这个段的地址是直接映射地址段，所以其起始的物理地址为0x00000000，即

```
 phy addr  = CSR.DMW0[31:29]  : virtual addr[28:0]
```

**第一个阶段**从kernel_entry函数开始到pmm_init函数执行之前，ucore采用直接映射地址方式进行地址翻译，翻译方式如上。注意，由于0x80000000-0x9fffffff和0xa0000000-0xbfffffff才能进行直接映射，所以内核的大小不能超过512M。

**第二个阶段**（创建初始页目录表，开启分页模式）从pmm_init函数被调用开始，在pmm_init函数中创建了boot_pgdir，初始化了页目录表，正式开始了页表映射地址翻译模式。

### 建立虚拟页和物理页帧的地址映射关系

**建立二级页表**

在LoongArch32采用了与MIPS相同的软件定义页表的方式，因此内核可以自由采取页表的定义方式。由于该版本uCore自x86架构移植而来，因此沿用了x86的类似方式采用二级页表来建立逻辑地址与物理地址之间的映射关系。由于我们已经具有了一个物理内存页管理器default\_pmm\_manager，支持动态分配和释放内存页的功能，我们就可以用它来获得所需的空闲物理页。在二级页表结构中，页目录表占4KB空间，可通过alloc\_page函数获得一个空闲物理页作为页目录表（Page Directory Table，PDT）。同理，ucore也通过这种类似方式获得一个页表（Page Table，PT）所需的4KB空间。

整个页目录表和页表所占空间大小取决与二级页表要管理和映射的物理页数。假定当前物理内存0~16MB，每物理页（也称Page Frame）大小为4KB，则有4096个物理页，也就意味这有4个页目录项和4096个页表项需要设置。一个页目录项（Page Directory Entry，PDE）和一个页表项（Page Table Entry，PTE）占4B。即使是4个页目录项也需要一个完整的页目录表（占4KB）。而4096个页表项需要16KB（即4096*4B）的空间，也就是4个物理页，16KB的空间。所以对16MB物理页建立一一映射的16MB虚拟页，需要5个物理页，即20KB的空间来形成二级页表。

完成前一节所述的前两个阶段的地址映射变化后，为把0\~KERNSIZE（明确ucore设定实际物理内存不能超过KERNSIZE值，即512MB，131072个物理页）的物理地址一一映射到页目录项和页表项的内容，其大致流程如下：

1. 指向页目录表的指针已存储在boot_pgdir变量中。

2. 调用boot\_map\_segment函数进一步建立一一映射关系，具体处理过程以页为单位进行设置，即

```
linear addr = phy addr + 0xa0000000
```

设一个32bit线性地址la有一个对应的32bit物理地址pa，如果在以la的高10位为索引值的页目录项中的存在位（PTE\_P）为0，表示缺少对应的页表空间，则可通过alloc\_page获得一个空闲物理页给页表，页表起始物理地址是按4096字节对齐的，这样填写页目录项的内容为

```
  页目录项内容 = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
```

进一步对于页表中以线性地址la的中10位为索引值对应页表项的内容为

```
  页表项内容 = (pa & ~0x0FFF) | PTE_P | PTE_W
```

其中：

* PTE\_U：位3，表示用户态的软件可以读取对应地址的物理内存页内容

* PTE\_W：位2，表示物理内存页内容可写

* PTE\_P：位1，表示物理内存页存在

ucore的内存管理经常需要查找页表：给定一个虚拟地址，找出这个虚拟地址在二级页表中对应的项。通过更改此项的值可以方便地将虚拟地址映射到另外的页上。可完成此功能的这个函数是get\_pte函数。它的原型为

```c
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create)
```

下面的调用关系图可以比较好地看出get\_pte在实现上述流程中的位置：

![](G:\py\sy\ucore_la32_docs-master\lab2_figs\image007.png)

图6 get\_pte调用关系图

这里涉及到三个类型pte\_t、pde\_t和uintptr\_t。通过参见mm/mmlayout.h和libs/types.h，可知它们其实都是unsigned int类型。在此做区分，是为了分清概念。

pde\_t全称为 page directory entry，也就是一级页表的表项（注意：pgdir实际不是表项，而是一级页表本身。实际上应该新定义一个类型pgd\_t来表示一级页表本身）。pte\_t全称为 page table entry，表示二级页表的表项。uintptr\_t表示为线性地址，由于段式管理只做直接映射，所以它也是逻辑地址。

pgdir给出页表起始地址。通过查找这个页表，我们需要给出二级页表中对应项的地址。虽然目前我们只有boot\_pgdir一个页表，但是引入进程的概念之后每个进程都会有自己的页表。

有可能根本就没有对应的二级页表的情况，所以二级页表不必要一开始就分配，而是等到需要的时候再添加对应的二级页表。如果在查找二级页表项时，发现对应的二级页表不存在，则需要根据create参数的值来处理是否创建新的二级页表。如果create参数为0，则get\_pte返回NULL；如果create参数不为0，则get\_pte需要申请一个新的物理页（通过alloc\_page来实现，可在mm/pmm.h中找到它的定义），再在一级页表中添加页目录项指向表示二级页表的新物理页。注意，新申请的页必须全部设定为零，因为这个页所代表的虚拟地址都没有被映射。

当建立从一级页表到二级页表的映射时，需要注意设置控制位。这里应该设置同时设置上PTE\_U、PTE\_W和PTE\_P（定义在mm/mmu.h）。如果原来就有二级页表，或者新建立了页表，则只需返回对应项的地址即可。

虚拟地址只有映射上了物理页才可以正常的读写。在完成映射物理页的过程中，除了要像上面那样在页表的对应表项上填上相应的物理地址外，还要设置正确的控制位。

只有当一级二级页表的项都设置了用户写权限后，用户才能对对应的物理地址进行读写。所以我们可以在一级页表先给用户写权限，再在二级页表上面根据需要限制用户的权限，对物理页进行保护。由于一个物理页可能被映射到不同的虚拟地址上去（譬如一块内存在不同进程间共享），当这个页需要在一个地址上解除映射时，操作系统不能直接把这个页回收，而是要先看看它还有没有映射到别的虚拟地址上。这是通过查找管理该物理页的Page数据结构的成员变量ref（用来表示虚拟页到物理页的映射关系的个数）来实现的，如果ref为0了，表示没有虚拟页到物理页的映射关系了，就可以把这个物理页给回收了，从而这个物理页是free的了，可以再被分配。page\_insert函数将物理页映射在了页表上。可参看page\_insert函数的实现来了解ucore内核是如何维护这个变量的。当不需要再访问这块虚拟地址时，可以把这块物理页回收并在将来用在其他地方。取消映射由page\_remove来做，这其实是page\_insert的逆操作。

建立好一一映射的二级页表结构后，由于分页机制在前一节所述的前两个阶段已经开启，分页机制到此初始化完毕。当执行完毕gdt\_init函数后，新的段页式映射已经建立好了。

在pmm\_init函数建立完实现物理内存一一映射和页目录表自映射的页目录表和页表后，ucore看到的内核虚拟地址空间如下图所示：

```
   Virtual memory map:                                          Permissions
                                                                              kernel/user

       4G ------------------> +---------------------------------+
                              |                                 |
                              |         Mapped(2G)              |
                              |                                 |
       KERNTOP  ------------> +---------------------------------+ 0xBFFF_FFFF
                              |                                 | 
                              |  Unmapped cached(DMW0 512M)     |
                              |                                 | 
       KERNBASE-------------> +---------------------------------+ 0xa0000000
                              |                                 |
                              |    Unmapped uncached(DMW1 512M) | RW/-- KMEMSIZE
                              |                                 |
                              +---------------------------------+ 0x80000000
                              |                                 |
                              |    User Mapped                  |
                              |                                 |
                              ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 0x00000000
```