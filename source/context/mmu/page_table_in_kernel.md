# Kernel与MMU

主要是从操作系统的角度，如果实现一个完整的地址翻译，来看我需要什么的功能？

假设我们现在使用4K页，或者2M页面，访问一个虚拟地址为0xfffff8007898的地址。
下面我们按照Kernel的视角来看如何处理。

全部以代码说明，切勿全是文字性的表述！！！

以Linux为主，从头到尾梳理

## 地址翻译相关的初始化

下面我们详细的分析Linux内核中LoongArch的TLB初始化相关内容。

:::{tip}
我们在内核中一般使用固定的页大小和大页的配置，比如我们使用基础4KB页表的话，大页就是2MB和1GB的页，
如果使用基础页16KB的话，那么大页就是32MB和64GB的配置。

``不建议将基础页4KB的页大小和基础页16KB的页大小混合使用。``

:::

1. 设置基础页大小
```c
// 设置CSR.TLBIDX.PS域，保证后面的tlb指令正常使用。具体可查看CSR.TLBIDX寄存器
write_csr_pagesize(PS_DEFAULT_SIZE);

// 设置STLBPS的值
write_csr_stlbpgsize(PS_DEFAULT_SIZE);

// 设置TLBRefill时PS专用页大小值，具体可看CSR.TLBREHI寄存器。
write_csr_tlbrefill_pagesize(PS_DEFAULT_SIZE);
```

下面是上面的具体实现，我们设置页大小为4K页，也可以配置为16KB或者64KB页大小。

后续我们都是按照4KB页大小来处理初始化。


```c
// 设置页大小，方便TLB后续指令需要

#define  CSR_TLBIDX_PS_SHIFT     24
#define  CSR_TLBIDX_SIZE         CSR_TLBIDX_PS_SHIFT
#define  CSR_TLBIDX_SIZEM        0x3f000000

/* TLB related CSR registers */
#define LOONGARCH_CSR_TLBIDX     0x10  /* TLB Index, EHINV, PageSize, NP */
#define LOONGARCH_CSR_STLBPGSIZE 0x1e


static inline void write_csr_pagesize(unsigned int size)
{
   csr_xchg32(size << CSR_TLBIDX_SIZE, CSR_TLBIDX_SIZEM, LOONGARCH_CSR_TLBIDX);
}

#define csr_xchg32(val, mask, reg) __csrxchg_w(val, mask, reg)

// __csrxchg_w 是指令csrxchg的包装，如下示例所示：

// csrxchg   rd, rj, csr_num
// 新写入的值val, 存放在rd寄存器，rj存放需要些位的掩码的值（如果是位1就是对应的位可以写入，否则不变）

#define PS_DEFAULT_SIZE PS_4K
#define PS_4K     0x0000000c

#define write_csr_stlbpgsize(val)   csr_write32(val, LOONGARCH_CSR_STLBPGSIZE)

static inline void write_csr_tlbrefill_pagesize(unsigned int size)
{
   csr_xchg(size << CSR_TLBREHI_PS_SHIFT, CSR_TLBREHI_PS, LOONGARCH_CSR_TLBREHI);
}

```

2. 设置软件或者硬件PTW相关设置。

```c
static void setup_ptwalker(void)
{
   unsigned long pwctl0, pwctl1;
   unsigned long pgd_i = 0, pgd_w = 0;
   unsigned long pud_i = 0, pud_w = 0;
   unsigned long pmd_i = 0, pmd_w = 0;
   unsigned long pte_i = 0, pte_w = 0;

   pgd_i = PGDIR_SHIFT;
   pgd_w = PAGE_SHIFT - 3;
#if CONFIG_PGTABLE_LEVELS > 3
   pud_i = PUD_SHIFT;
   pud_w = PAGE_SHIFT - 3;
#endif
#if CONFIG_PGTABLE_LEVELS > 2
   pmd_i = PMD_SHIFT;
   pmd_w = PAGE_SHIFT - 3;
#endif
   pte_i = PAGE_SHIFT;
   pte_w = PAGE_SHIFT - 3;

   pwctl0 = pte_i | pte_w << 5 | pmd_i << 10 | pmd_w << 15 | pud_i << 20 | pud_w << 25;
   pwctl1 = pgd_i | pgd_w << 6;


   // 如果是启用了硬件的PTW的话，还需要额外设置PWCH的HPTW_En
   if (cpu_has_ptw)
      pwctl1 |= CSR_PWCTL1_PTW;

   // 将配置写入CSR中
   csr_write(pwctl0, LOONGARCH_CSR_PWCTL0);
   csr_write(pwctl1, LOONGARCH_CSR_PWCTL1);
   
   // 设置内核的全局目录基址PGD
   csr_write(virt_to_pgdcsr(swapper_pg_dir), LOONGARCH_CSR_PGDH);
   
   // 先初始化用户PGDL为无效的页表基址
   csr_write(virt_to_pgdcsr(invalid_pg_dir), LOONGARCH_CSR_PGDL);
   csr_write((long)smp_processor_id(), LOONGARCH_CSR_TMID);
}
```

根据我们上个章节的描述，Linux初始化的时候，使用了LoongArch的PWCH中的``Dir3_base``和``Dir3_width``，

设置``Dir4_base = 0``和``Dir4_width = 0``。

也就是说Linux LoongArch是按照下面的设置

| 配置的页表级数 | 启用的PWCH相关配置 | 启动的PWCL相关配置 |
|-|-|-|
| 2级页表(16KB) |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 25, Dir3_width = 11 | Dir2_base = 0, Dir2_width = 0 <br> Dir1_base = 0, Dir1_width = 0 <br> PTbase = 14, PTwidth = 11|
| 2级页表(64KB) |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 29, Dir3_width = 13 | Dir2_base = 0, Dir2_width = 0 <br> Dir1_base = 0, Dir1_width = 0 <br> PTbase = 16, PTwidth = 13|
| 3级页表(4KB)  |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 30, Dir3_width = 9 | Dir2_base = 0, Dir2_width = 0 <br> Dir1_base = 21, Dir1_width = 9 <br> PTbase = 12, PTwidth = 9|
| 3级页表(16KB)  |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 36, Dir3_width = 11 | Dir2_base = 0, Dir2_width = 0 <br> Dir1_base = 25, Dir1_width = 11 <br> PTbase = 14, PTwidth = 11|
| 3级页表(64KB)  |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 42, Dir3_width = 13 | Dir2_base = 0, Dir2_width = 0 <br> Dir1_base = 29, Dir1_width = 13 <br> PTbase = 16, PTwidth = 13|
| 4级页表(4KB)  |   Dir4_base = 0, Dir4_width = 0 <br> Dir3_base = 39, Dir3_width = 9 | Dir2_base = 30, Dir2_width = 9 <br> Dir1_base = 21, Dir1_width = 9 <br> PTbase = 12, PTwidth = 9|


3. 设置TLB相关的例外配置

```c
// 将handle_tlb_refill的代码拷贝到tlbrentry中。
memcpy((void *)tlbrentry, handle_tlb_refill, 0x80);

//将上述的拷贝结果同步，执行idbr指令
local_flush_icache_range(tlbrentry, tlbrentry + 0x80);

//将和TLB相关的异常处理代码拷贝到各自的例外入口处
for (int i = EXCCODE_TLBL; i <= EXCCODE_TLBPE; i++)
   set_handler(i * VECSIZE, exception_table[i], VECSIZE);

// 将TLB Refill的入口写入CSR.TLBRENTRY寄存器中
csr_write64(tlbrentry, LOONGARCH_CSR_TLBRENTRY);

// 设置例外入口CSR.EENTRY
csr_write64(eentry, LOONGARCH_CSR_EENTRY);
```

这里需要有以下需要注意：
   - LoongArch的TLB Refill例外由于使用频率高，因此单独设立了例外入口。
   - TLB的其他与TLB相关的例外，比如PIL，PIS，PIF等异常，使用的是和CSR.EENTRY相关。具体怎么配置我们在下一章详细介绍
     这里只需要知道，需要设置！

到此为止，基本的初始化设置已经完成，剩下的我们主要讨论如何内核使用！


## 如何从直接地址翻译模式到映射地址翻译模式


假设我们CPU上电后，操作权限交给了Kernel，此时我们还是运行在物理地址0x200000，此时虚拟地址等于物理地址。

具体涉及到的相关配置寄存器，请查看上个章节我们的描述！

下面我们会示例介绍几种kernel使用的场景和初始化的方式。供操作系统使用，既可以单独使用，也可以组合使用。

1. **步骤1**： 操作系统Kernel在CPU上电的时候，此时CPU处于**直接地址翻译模式**, 此时``DA=0 && PG=1``

   此时CPU的地址模式是： 虚拟地址等于物理地址，VA = PA。

   此时访问内存的类型，通过CSR.CRMD的DATF和DATM决定。

   可以设置 CSR.CRMD.DATF = 2'b01，这时**CPU取指的路径**通过Cache读取（如果Cache缺失，在访问内存），    
   如果不设置也就是默认 CSR.CRMD.DATF = 2'b00，指令从内存中读取，不经过Cache。

   通用设置CSR.CRMD.DATM = 2'b01，这是访存指令load/store也是通过Cache读取数据，    
   如果不设置也就是默认 ``CSR.CRMD.DATM = 2'b00``，数据直接从内存读取或者写回，不经过Cache。


2. **步骤2**： 如果Kernel(此时CPU还是处于直接地址翻译模式)想要使用**直接映射模式**，需要做一些初始化的工作
为使能直接映射模式（DMW方式）做准备。

   主要操作如下：
   - 初始化相关DMW[x]寄存器，比如下面所示：
      假设我们的Kernel运行在0x9000xxxxxxxxxxxxxxxx的地址上
      ```asm 
      li.d  $t0, 0x9000000000000011
      csrwr $t0, 0x180 // 设置DMW0， PLV0，Cache
      ```
   - 确认当前的PC是运行在0x9000xxxxxxxxxxxxxxxx上，如果没有需要跳转到此地址去。
     如下所示：
     ```asm
     // 首先将虚拟地址的基址加载到寄存器t0上。
     li.d   $t0, 0x9000000000000000
     
     // 将当前指令的地址加载到寄存器t1上
     pcaddi $t1, 0

     // 将上一条指令的地址t1，加上基址t0，再赋值到t0，此时
     // t0保存着上一条指令的地址（加上了虚拟地址偏移）
     or     $t0, $t0, $t1

     // 此时我们需要跳转到上面pcaddi地址加C的位置，也就是隔了三条指令，
     // 如果pcaddi指令的地址，加上三条指令，正好是下条指令的地址，也就真好跳转到新的地址上了。
     jirl   $zero, $t0, 0xc
     ```

   - 此时，我们已经准备好了虚拟地址，下一步就是打开CSR.CRMD寄存器的配置
     ```asm
     // PLV=0, IE=0, DA=0, PG=1
     li.w   $t0, 0xb0  
     csrwr  $t0, 0x0
     ```
     此时我们已经工作在**直接映射模式**，注意此时我们还是在使用DMW，并没有开启页表映射的相关机制。

     如果想要开启页表的机制**页表映射模式**，可以初始化相关的寄存器，初始化TLB寄存器后，
     就可以使用页表映射来进行虚实地址转换了。

3. **步骤3**： 如果不想使用步骤2的直接映射模式，想直接进入**页表映射模式**，也是可以的。    
   此时，我们需要做的事情如下所示： （CPU目前处于``DA=0 && PG=1``， 直接地址翻译模式）    
   假设我们的虚拟地址是0xffff_fxxx_xxxx_xxxx，到物理地址xxx_xxxx_xxxx的映射。      
   
   - 先可以做一些简单的其他相关初始化工作。

   - 此时我们需要准备初始化我们的TLB寄存器，
     比如设置PGDL和PGDH，以及PWCL和PWCH等，还要设置异常入口等。

   - 准备一个临时的页表关系，将当前物理地址映射到虚拟地址。比如
     VA[47:0] = PA[47:0]。将页表表项按照我们上个章节，在内存中建立映射的关系。

     注意此时一定要建立一个虚拟地址等于物理地址的映射关系！（可以参考下章节的代码例子）

   - 接着设置CSR.CRMD的DA和PG``DA=0 && PG=1``，即打开我们的MMU，使能映射地址翻译模式！

   - 建立对应目标虚拟地址和物理地址的映射关系：比如0xffff_fxxx_xxxx_xxxx，到物理地址xxx_xxxx_xxxx的映射关系。

     注意将页表同步到内存后，使用刷新DCache,ICache和TLB，确保没有旧的无效映射关系！

   - 这时候，再跳转到目标的虚拟地址
      ```asm
      // 注意此时加载的虚拟地址偏移可根据实际情况来设置！
      li.d   $t0, 0xfffff00000000000
      pcaddi $t1, 0
      or     $t0, $t0, $t1
      jirl   $zero, $t0, 0xc
      ```
   
   - 接着我们可以接着处理剩余的内核初始化工作流程！

   - 此时我们就执行在**页表映射模式**下，所有的虚拟地址的转换，都是通过页表来翻译。


:::{tip}
从上面的初始化步骤可以看出，步骤1是我们必须存在的过程。后面，我们可能只选择直接地址翻译模式，或者直接使用页表映射模式
或者两者都同时存在。

一般，在kernel刚获得执行权后，我们会初始化直接使用**直接地址翻译模式**，后续再如果有需要我们再设置页表映射模式。
这样kernel在后续内核态的时候直接使用DMW，而不用进行页表映射，极大的加速了访问（因为页表映射模式还要访问TLB，可能还要访问内存等），而且操作方便简单！

或者对于熟悉**页表映射模式**的开发者，可以不用DMW的方式，直接按照步骤三的方式，初始化页表TLB相关内容，直接使用页表翻译虚实地址。
都是可以的。

比如，Linux就是使用**直接地址翻译模式**和**页表映射模式**，而一些简单的嵌入式内核，只有一种特权模式，可以使用**直接地址翻译模式**来处理。
:::



## 如何建立虚拟地址到物理地址的映射关系

下面的例子我们假设采用三级页表，使用4KB页大小。


```c
void __create_2MB_mapping() {
   
   /// map 2MB

   long vir_base = 0x1800000000;
   long phy_base = 0x1C000000;

   long phy_pfn = (long) phy_base >> _PFN_SHIFT;

   // 首先建立PGD页表，PGD基址是pgd_table
   pgd_t* gptr = pgd_table + pgd_index((long)vir_base);

   // 存放下一级PMD的基址
   *(long*)gptr = (long)pmd_table;
   
   // 建立PMD对应的映射
   pmd_t* pmdptr = pmd_table + pmd_index((long)vir_base);
   
   // 存放下一级PTE的基址
   *((long*)pmdptr) = (long)pte_table;
   
   // 为每一个PTE建立一个映射物理页。
   for (int i = 0; i < PTRS_PER_PTE; ++i) {
      pte_t* pteptr = pte_table + i;
      // *(pteptr) = pfn_pte((phy_pfn+i), PAGE_KERNEL);
      
      // 在叶子节点页表，还需要增加相应的页表属性，比如PAGE_KERNEL，或者PAGE_USER
      *(pteptr) = pfn_pte((phy_pfn+i), PAGE_USER);
   }
}

```

上面假设虚拟地址是0x1800000000，对应需要映射的物理地址是0x1C000000。

上述中第Level 1级页表，也就是PTE，我们建立了``PTRS_PER_PTE=512``个物理页大小``PAGE_SIZE=4KB``      
所以上述的映射的内存总大小为:

```
Total_MEM_Size = PAGE_SIZE * PTRS_PER_PTE = 4KB * 512 = 2MB
```


下面是页表表项相关的定义：
```c
/* Page table bits */
#define  _PAGE_VALID_SHIFT      0
#define  _PAGE_ACCESSED_SHIFT   0  /* Reuse Valid for Accessed */
#define  _PAGE_DIRTY_SHIFT      1
#define  _PAGE_PLV_SHIFT        2  /* 2~3, two bits */
#define  _CACHE_SHIFT           4  /* 4~5, two bits */
#define  _PAGE_GLOBAL_SHIFT     6
#define  _PAGE_HUGE_SHIFT       6  /* HUGE is a PMD bit */
#define  _PAGE_PRESENT_SHIFT    7
#define  _PAGE_WRITE_SHIFT      8
#define  _PAGE_MODIFIED_SHIFT   9
#define  _PAGE_PROTNONE_SHIFT   10
#define  _PAGE_SPECIAL_SHIFT    11
#define  _PAGE_HGLOBAL_SHIFT    12 /* HGlobal is a PMD bit */
#define  _PAGE_PFN_SHIFT        12
#define  _PAGE_PFN_END_SHIFT    48
#define  _PAGE_NO_READ_SHIFT    61
#define  _PAGE_NO_EXEC_SHIFT    62
#define  _PAGE_RPLV_SHIFT       63

#define _ULCAST_ (unsigned long)

/* Used only by software */
#define _PAGE_PRESENT   (_ULCAST_(1) << _PAGE_PRESENT_SHIFT)
#define _PAGE_WRITE     (_ULCAST_(1) << _PAGE_WRITE_SHIFT)
#define _PAGE_ACCESSED  (_ULCAST_(1) << _PAGE_ACCESSED_SHIFT)
#define _PAGE_MODIFIED  (_ULCAST_(1) << _PAGE_MODIFIED_SHIFT)
#define _PAGE_PROTNONE  (_ULCAST_(1) << _PAGE_PROTNONE_SHIFT)
#define _PAGE_SPECIAL   (_ULCAST_(1) << _PAGE_SPECIAL_SHIFT)

/* Used by TLB hardware (placed in EntryLo*) */
#define _PAGE_VALID     (_ULCAST_(1) << _PAGE_VALID_SHIFT)
#define _PAGE_DIRTY     (_ULCAST_(1) << _PAGE_DIRTY_SHIFT)
#define _PAGE_PLV       (_ULCAST_(3) << _PAGE_PLV_SHIFT)
#define _PAGE_GLOBAL    (_ULCAST_(1) << _PAGE_GLOBAL_SHIFT)
#define _PAGE_HUGE      (_ULCAST_(1) << _PAGE_HUGE_SHIFT)
#define _PAGE_HGLOBAL   (_ULCAST_(1) << _PAGE_HGLOBAL_SHIFT)
#define _PAGE_NO_READ   (_ULCAST_(1) << _PAGE_NO_READ_SHIFT)
#define _PAGE_NO_EXEC   (_ULCAST_(1) << _PAGE_NO_EXEC_SHIFT)
#define _PAGE_RPLV      (_ULCAST_(1) << _PAGE_RPLV_SHIFT)
#define _CACHE_MASK     (_ULCAST_(3) << _CACHE_SHIFT)
#define _PFN_SHIFT      (PAGE_SHIFT - 12 + _PAGE_PFN_SHIFT)

#define PLV_KERN        0
#define PLV_USER        3
#define PLV_MASK        0x3

#define _PAGE_USER     (PLV_USER << _PAGE_PLV_SHIFT)
#define _PAGE_KERN     (PLV_KERN << _PAGE_PLV_SHIFT)

#define _PFN_MASK      (~((_ULCAST_(1) << (_PFN_SHIFT)) - 1) & \
                       ((_ULCAST_(1) << (_PAGE_PFN_END_SHIFT)) - 1))

#define _CACHE_SUC     (0<<_CACHE_SHIFT) /* Strong-ordered UnCached */
#define _CACHE_CC      (1<<_CACHE_SHIFT) /* Coherent Cached */
#define _CACHE_WUC     (2<<_CACHE_SHIFT) /* Weak-ordered UnCached */


#define __READABLE     (_PAGE_VALID)
#define __WRITEABLE    (_PAGE_DIRTY | _PAGE_WRITE)

#define PAGE_KERNEL     __pgprot(_PAGE_PRESENT | __READABLE | __WRITEABLE | \
                                 _PAGE_GLOBAL | _PAGE_KERN | _CACHE_CC)
#define PAGE_KERNEL_SUC __pgprot(_PAGE_PRESENT | __READABLE | __WRITEABLE | \
                                 _PAGE_GLOBAL | _PAGE_KERN |  _CACHE_SUC)
#define PAGE_KERNEL_WUC __pgprot(_PAGE_PRESENT | __READABLE | __WRITEABLE | \
                                 _PAGE_GLOBAL | _PAGE_KERN |  _CACHE_WUC)

#define PAGE_USER       __pgprot(_PAGE_PRESENT | __READABLE | __WRITEABLE | \
                                 _PAGE_USER | _CACHE_CC)

```

下面是操作页表的一些函数：

```c
#define PAGE_SHIFT   12
#define PAGE_SIZE    (1UL << PAGE_SHIFT)

#define PTRS_PER_PGD ((PAGE_SIZE) >> 3)
#define PTRS_PER_PUD ((PAGE_SIZE) >> 3)
#define PTRS_PER_PMD ((PAGE_SIZE) >> 3)
#define PTRS_PER_PTE ((PAGE_SIZE) >> 3)

#define PMD_SHIFT (PAGE_SHIFT + (PAGE_SHIFT + PTE_ORDER - 3))
#define PMD_SIZE  (1UL << PMD_SHIFT)
#define PMD_MASK  (~(PMD_SIZE-1))

#define PUD_SHIFT (PMD_SHIFT + (PAGE_SHIFT + PMD_ORDER - 3))
#define PUD_SIZE  (1UL << PUD_SHIFT)
#define PUD_MASK  (~(PUD_SIZE-1))

#define PGD_SHIFT (PMD_SHIFT + (PAGE_SHIFT + PMD_ORDER - 3))
#define PGDIR_SHIFT  (PMD_SHIFT + (PAGE_SHIFT + PMD_ORDER - 3))

#define PGDIR_SIZE   (1UL << PGDIR_SHIFT)
#define PGDIR_MASK   (~(PGDIR_SIZE-1))

static inline unsigned long pte_index(unsigned long address)
{
   return (address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}
static inline unsigned long pmd_index(unsigned long address)
{
   return (address >> PMD_SHIFT) & (PTRS_PER_PMD - 1);
}
static inline unsigned long pud_index(unsigned long address)
{
   return (address >> PUD_SHIFT) & (PTRS_PER_PUD - 1);
}
#define pgd_index(a)  (((a) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))

static inline pgd_t *pgd_offset_pgd(pgd_t *pgd, unsigned long address)
{
   return (pgd + pgd_index(address));
};

```


## 页表的遍历页表


## 如果TLB中没有映射

## 如果页表项的V=0


## 如果写操作页表项D=0

## 如果权限不合法


## 虚拟地址到物理地址的映射
一个具体的示例代码



