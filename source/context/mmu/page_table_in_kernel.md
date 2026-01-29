# Kernel与MMU

主要是从操作系统的角度，如果实现一个完整的地址翻译，来看我需要什么的功能？

假设我们现在使用4K页，或者2M页面，访问一个虚拟地址为0xfffff8007898的地址。
下面我们按照Kernel的视角来看如何处理。

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

异常号如下所示，具体的使用说明我们在下一章详细说明！
```c
/* ExStatus.ExcCode */
#define EXCCODE_RSV     0  /* Reserved */
#define EXCCODE_TLBL    1  /* TLB miss on a load */
#define EXCCODE_TLBS    2  /* TLB miss on a store */
#define EXCCODE_TLBI    3  /* TLB miss on a ifetch */
#define EXCCODE_TLBM    4  /* TLB modified fault */
#define EXCCODE_TLBNR      5  /* TLB Read-Inhibit exception */
#define EXCCODE_TLBNX      6  /* TLB Execution-Inhibit exception */
#define EXCCODE_TLBPE      7  /* TLB Privilege Error */
#define EXCCODE_ADE     8  /* Address Error */
   #define EXSUBCODE_ADEF     0  /* Fetch Instruction */
   #define EXSUBCODE_ADEM     1  /* Access Memory*/
#define EXCCODE_ALE     9  /* Unalign Access */
#define EXCCODE_BCE     10 /* Bounds Check Error */
#define EXCCODE_SYS     11 /* System call */
#define EXCCODE_BP      12 /* Breakpoint */
#define EXCCODE_INE     13 /* Inst. Not Exist */
#define EXCCODE_IPE     14 /* Inst. Privileged Error */
#define EXCCODE_FPDIS      15 /* FPU Disabled */
#define EXCCODE_LSXDIS     16 /* LSX Disabled */
#define EXCCODE_LASXDIS    17 /* LASX Disabled */
#define EXCCODE_FPE     18 /* Floating Point Exception */
   #define EXCSUBCODE_FPE     0  /* Floating Point Exception */
   #define EXCSUBCODE_VFPE    1  /* Vector Exception */
#define EXCCODE_WATCH      19 /* WatchPoint Exception */
   #define EXCSUBCODE_WPEF    0  /* ... on Instruction Fetch */
   #define EXCSUBCODE_WPEM    1  /* ... on Memory Accesses */
#define EXCCODE_BTDIS      20 /* Binary Trans. Disabled */
#define EXCCODE_BTE     21 /* Binary Trans. Exception */
#define EXCCODE_GSPR    22 /* Guest Privileged Error */
#define EXCCODE_HVC     23 /* Hypercall */
#define EXCCODE_GCM     24 /* Guest CSR modified */
   #define EXCSUBCODE_GCSC    0  /* Software caused */
   #define EXCSUBCODE_GCHC    1  /* Hardware caused */
#define EXCCODE_SE      25 /* Security */
```


(exception_table_example)=
```c
// 所有异常表
void *exception_table[EXCCODE_INT_START] = {
   [0 ... EXCCODE_INT_START - 1] = handle_reserved,

   [EXCCODE_TLBI]    = handle_tlb_load,
   [EXCCODE_TLBL]    = handle_tlb_load,
   [EXCCODE_TLBS]    = handle_tlb_store,
   [EXCCODE_TLBM]    = handle_tlb_modify,
   [EXCCODE_TLBNR]      = handle_tlb_protect,
   [EXCCODE_TLBNX]      = handle_tlb_protect,
   [EXCCODE_TLBPE]      = handle_tlb_protect,
   [EXCCODE_ADE]     = handle_ade,
   [EXCCODE_ALE]     = handle_ale,
   [EXCCODE_BCE]     = handle_bce,
   [EXCCODE_SYS]     = handle_sys,
   [EXCCODE_BP]      = handle_bp,
   [EXCCODE_INE]     = handle_ri,
   [EXCCODE_IPE]     = handle_ri,
   [EXCCODE_FPDIS]      = handle_fpu,
   [EXCCODE_LSXDIS]  = handle_lsx,
   [EXCCODE_LASXDIS] = handle_lasx,
   [EXCCODE_FPE]     = handle_fpe,
   [EXCCODE_WATCH]      = handle_watch,
   [EXCCODE_BTDIS]      = handle_lbt,
};


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

:::{tip}
复位将重新处理器核中的所有逻辑，将电路置于确定的状态。这里将给出复位后处理器的状态的定义。

复位后第一条指令的 PC 是 0x1C000000。由于复位撤销后 MMU 一定处于直接地址翻译模式，所以复

位后所取的第一条指令的物理地址也是 0x1C000000。

复位撤销后，处于确定状态的寄存器内容有：

 - CSR.CRMD 的 PLV=0，IE=0，DA=1，PG=0，DATF=0，DATM=0，WE=0；

 - CSR.EUEN 的 FPUen、VPUen、XVPUen、BTUen 均为 0；

 - CSR.MISC 中的所有可配置位均为 0；

 - CSR.ECFG 中的 VS 和 LIE 均为 0；

 - CSR.ESTAT 中 IS[1:0]均为 0；

 - CSR.RVACFG 中的 RDVA=0；

 - CSR.TCFG 的 En=0；

 - CSR.LLBCTL 的 KLO=0；

 - CSR.TLBRERA 的 IsTLBR=0；

 - CSR.ERRCTL 的 IsMERR=0；

 - 所有实现的 CSR.DMW 中的 PLV0~PLV3 均为 0；

 - 所有实现的 CSR.PMCFG 中除 EvCode 之外的所有可配置位均为 0；

 - 所有实现的数据断点控制 CSR 中的所有可配置位均为 0；

 - 所有实现的指令断点控制 CSR 中的所有可配置位均为 0；

 - CSR.DBG 中的 DS=0。
:::


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

假设我们现在使用4K页，或者2M页面，使用软件重填机制（硬件重填机制相似），   
访问一个虚拟地址为0xfffff8007898的地址。    

下面我们按照Kernel的视角来看如何处理。

我们还是以Linux内核代码为例说明。


假设内核有下面的汇编代码需要执行：
```asm
li.d  $t0, 0xfffff8007898
ld.d  $t1, $t0, 0
```

此时当执行指令``ld.d  $t1, $t0, 0``时，首先

CPU按照[<u>内部TLB查找流程</u>](#cpu_inner_tlb_lookup)看是否有对应的STLB和MTLB命中，
如果没有命中的话，直接抛出TLB重填例外。

## 情况1. 如果TLB中没有映射

如果CPU抛出TLB重填例外，此时CPU跳转到CSR.TLBRENTRY也就是``handle_tlb_refill``
函数执行，具体的分析看[<u>三级页表重填的示例代码</u>](#three_level_page_table_refill)。

:::{tip}
当触发 TLB 重填例外时，处理器硬件会进行如下操作：

❖将 CSR.CRMD 的 PLV、IE 分别存到 CSR.TLBRPRMD 的 PPLV、PIE 中，然后将 CSR.CRMD 的   
PLV 置为 0，IE 置为 0，DA 置为 1，PG 置为 0；

❖对于支持 Watch 功能的实现，还要将 CSR.CRMD 的 WE 存到 CSR.TLBRPRMD 的 PWE 中，然后    
将 CSR.CRMD 的 WE 置为 0；

❖将触发例外指令的 PC 的[GRLEN-1:2]位记录到 CSR.TLBRERA 的 ERA 域中，将 CSR.TLBRERA    
的 IsTLBR 置为 1；

❖将触发该例外的访存虚地址（如果是取指触发的则就是 PC）记录到 CSR.TLBRBADV 中，将虚地    
址的[PALEN-1:13]位记录到 CSR.TLBREHI 的 VPPN 域中；

❖跳转到 CSR.TLBRENTTRY 所配置的例外入口处取指。     
 

当软件执行 ERTN 指令从 TLB 重填例外执行返回时，处理器硬件会完成如下操作：    

❖将 CSR.TLBRPRMD 中的 PPLV、PIE 值恢复到 CSR.CRMD 的 PLV、IE 中；

❖对于支持 Watch 功能的实现，还要将 CSR.TLBRPRMD 中的 PWE 值恢复到 CSR.CRMD 的 WE 中；

❖将 CSR.CRMD 的 DA 置为 0，PG 置为 1；

❖将 CSR.TLBRERA 的 IsTLBR 置为 0；

❖跳转到 CSR.TLBRERA 所记录的地址处取指。

:::


:::{tip}
TLB软件重填使用频率高，因此代码都比较精简，一般情况下，代码都是通用的！可参看我们上个章节给出的例子，需要结合CSR.PWCH和CSR.PWCL寄存器。
:::

TLB重填完成后，会继续执行访存指令``ld.d  $t1, $t0, 0``

CPU按照[<u>内部TLB查找流程</u>](#cpu_inner_tlb_lookup)由于我们已经进行了TLB重填异常处理，因此我们
肯定会命中TLB表项。

然后我们继续分析下面的情况。

## 情况2. 如果页表项的V=0

如此CPU查找到TLB的表项中V=0，也就是说，此虚拟地址对应的物理页不存在，因此会抛出异常

此时区分访存的类型，抛出不同的异常处理：
  - FETCH : SignalException(PIF) #报取指操作页无效例外
  - LOAD:   SignalException(PIL) #报 load 操作页无效例外
  - STORE : SignalException(PIS) #报 store 操作页无效例外

按照我们当前的例子，我们是load指令出了异常，因此执行PIL异常处理。

按照上面的初始化流程[<u>异常初始化</u>](#exception_table_example)，此时CPU执行对应异常号为``EXCCODE_TLBL``的例外。

也就是执行函数handle_tlb_load，下面分析Linux中LoongArch有关的历程代码。

(example_handle_tlb_load)=
```text
SYM_CODE_START(handle_tlb_load)
   UNWIND_HINT_UNDEFINED
   csrwr    t0, EXCEPTION_KS0
   csrwr    t1, EXCEPTION_KS1
   csrwr    ra, EXCEPTION_KS2

   /*
    * The vmalloc handling is not in the hotpath.
    */
   csrrd    t0, LOONGARCH_CSR_BADV
   bltz     t0, vmalloc_load
   csrrd    t1, LOONGARCH_CSR_PGDL

vmalloc_done_load:
   /* Get PGD offset in bytes */
   bstrpick.d  ra, t0, PTRS_PER_PGD_BITS + PGDIR_SHIFT - 1, PGDIR_SHIFT
   alsl.d      t1, ra, t1, 3
#if CONFIG_PGTABLE_LEVELS > 3
   ld.d     t1, t1, 0
   bstrpick.d  ra, t0, PTRS_PER_PUD_BITS + PUD_SHIFT - 1, PUD_SHIFT
   alsl.d      t1, ra, t1, 3
#endif
#if CONFIG_PGTABLE_LEVELS > 2
   ld.d     t1, t1, 0
   bstrpick.d  ra, t0, PTRS_PER_PMD_BITS + PMD_SHIFT - 1, PMD_SHIFT
   alsl.d      t1, ra, t1, 3
#endif
   ld.d     ra, t1, 0

   /*
    * For huge tlb entries, pmde doesn't contain an address but
    * instead contains the tlb pte. Check the PAGE_HUGE bit and
    * see if we need to jump to huge tlb processing.
    */
   rotri.d     ra, ra, _PAGE_HUGE_SHIFT + 1
   bltz     ra, tlb_huge_update_load

   rotri.d     ra, ra, 64 - (_PAGE_HUGE_SHIFT + 1)
   bstrpick.d  t0, t0, PTRS_PER_PTE_BITS + PAGE_SHIFT - 1, PAGE_SHIFT
   alsl.d      t1, t0, ra, _PTE_T_LOG2

#ifdef CONFIG_SMP
smp_pgtable_change_load:
   ll.d     t0, t1, 0
#else
   ld.d     t0, t1, 0
#endif
   andi     ra, t0, _PAGE_PRESENT
   beqz     ra, nopage_tlb_load

   ori      t0, t0, _PAGE_VALID
#ifdef CONFIG_SMP
   sc.d     t0, t1, 0
   beqz     t0, smp_pgtable_change_load
#else
   st.d     t0, t1, 0
#endif
   tlbsrch
   bstrins.d   t1, zero, 3, 3
   ld.d     t0, t1, 0
   ld.d     t1, t1, 8
   csrwr    t0, LOONGARCH_CSR_TLBELO0
   csrwr    t1, LOONGARCH_CSR_TLBELO1
   tlbwr

   csrrd    t0, EXCEPTION_KS0
   csrrd    t1, EXCEPTION_KS1
   csrrd    ra, EXCEPTION_KS2
   ertn

#ifdef CONFIG_64BIT
vmalloc_load:
   la_abs      t1, swapper_pg_dir
   b     vmalloc_done_load
#endif

   /* This is the entry point of a huge page. */
tlb_huge_update_load:
#ifdef CONFIG_SMP
   ll.d     ra, t1, 0
#else
   rotri.d     ra, ra, 64 - (_PAGE_HUGE_SHIFT + 1)
#endif
   andi     t0, ra, _PAGE_PRESENT
   beqz     t0, nopage_tlb_load

#ifdef CONFIG_SMP
   ori      t0, ra, _PAGE_VALID
   sc.d     t0, t1, 0
   beqz     t0, tlb_huge_update_load
   ori      t0, ra, _PAGE_VALID
#else
   ori      t0, ra, _PAGE_VALID
   st.d     t0, t1, 0
#endif
   csrrd    ra, LOONGARCH_CSR_ASID
   csrrd    t1, LOONGARCH_CSR_BADV
   andi     ra, ra, CSR_ASID_ASID
   invtlb      INVTLB_ADDR_GFALSE_AND_ASID, ra, t1

   /*
    * A huge PTE describes an area the size of the
    * configured huge page size. This is twice the
    * of the large TLB entry size we intend to use.
    * A TLB entry half the size of the configured
    * huge page size is configured into entrylo0
    * and entrylo1 to cover the contiguous huge PTE
    * address space.
    */
   /* Huge page: Move Global bit */
   xori     t0, t0, _PAGE_HUGE
   lu12i.w     t1, _PAGE_HGLOBAL >> 12
   and      t1, t0, t1
   srli.d      t1, t1, (_PAGE_HGLOBAL_SHIFT - _PAGE_GLOBAL_SHIFT)
   or    t0, t0, t1

   move     ra, t0
   csrwr    ra, LOONGARCH_CSR_TLBELO0

   /* Convert to entrylo1 */
   addi.d      t1, zero, 1
   slli.d      t1, t1, (HPAGE_SHIFT - 1)
   add.d    t0, t0, t1
   csrwr    t0, LOONGARCH_CSR_TLBELO1

   /* Set huge page tlb entry size */
   addu16i.d   t0, zero, (CSR_TLBIDX_PS >> 16)
   addu16i.d   t1, zero, (PS_HUGE_SIZE << (CSR_TLBIDX_PS_SHIFT - 16))
   csrxchg     t1, t0, LOONGARCH_CSR_TLBIDX

   tlbfill

   addu16i.d   t0, zero, (CSR_TLBIDX_PS >> 16)
   addu16i.d   t1, zero, (PS_DEFAULT_SIZE << (CSR_TLBIDX_PS_SHIFT - 16))
   csrxchg     t1, t0, LOONGARCH_CSR_TLBIDX

   csrrd    t0, EXCEPTION_KS0
   csrrd    t1, EXCEPTION_KS1
   csrrd    ra, EXCEPTION_KS2
   ertn

nopage_tlb_load:
   dbar     0x700
   csrrd    ra, EXCEPTION_KS2
   la_abs      t0, tlb_do_page_fault_0
   jr    t0
SYM_CODE_END(handle_tlb_load)
```

过程如下：
 - 先保存handle_tlb_load使用的临时寄存器到CSR.SAVE[x]中。
 - 读出出错的地址从CSR.BADV寄存器。
 - 分析出错地址是内核地址还是用户地址空间，针对不同的历程，走不同的处理方法。
 - 从CSR.PGDL中得出全局页表基址，然后分析其PGD，PMD，PTE
 - 判断PTE是否存在，``andi $ra, $t0, _PAGE_PRESENT``, 如果不存在，跳转到nopage_tlb_load中执行。
 - nopage_tlb_load是对 tlb_do_page_fault的包装函数。
 - 执行函数LoongArch下的do_page_fault函数：
 ```c
   asmlinkage void __kprobes do_page_fault(struct pt_regs *regs,
            unsigned long write, unsigned long address)
   {
      irqentry_state_t state = irqentry_enter(regs);
   
      /* Enable interrupt if enabled in parent context */
      if (likely(regs->csr_prmd & CSR_PRMD_PIE))
         local_irq_enable();
   
      __do_page_fault(regs, write, address);
   
      local_irq_disable();
   
      irqentry_exit(regs, state);
   }
 ```
 - 进入内核的公共处理函数``fault = handle_mm_fault(vma, address, flags, regs)``执行：
 ```c
   static inline vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                unsigned long address, unsigned int flags,
                struct pt_regs *regs)
 ```

``handle_mm_fault`` 函数会执行我们上章节页表的遍历过程，分配物理页，设置对应的页表表项，然后返回。

## 情况3. 如果写操作页表项D=0
如果访存的是一个Store指令，比如``st.d  $t1, $t0, 0``时，store指令操作的虚地址在 TLB 中找到了匹配，且    V=1，且特权等级合规的项，但是该页 表项的 D 位为 0，将触发页修改例外PME例外。

PME按照我们上面的初始化，此时CPU执行对应异常号为``EXCCODE_TLBM``的例外。

也就是执行函数handle_tlb_modify，下面分析Linux中LoongArch有关PME的处理历程代码。

```
SYM_CODE_START(handle_tlb_modify)
   UNWIND_HINT_UNDEFINED
   csrwr    t0, EXCEPTION_KS0
   csrwr    t1, EXCEPTION_KS1
   csrwr    ra, EXCEPTION_KS2

   /*
    * The vmalloc handling is not in the hotpath.
    */
   csrrd    t0, LOONGARCH_CSR_BADV
   bltz     t0, vmalloc_modify
   csrrd    t1, LOONGARCH_CSR_PGDL

vmalloc_done_modify:
   /* Get PGD offset in bytes */
   bstrpick.d  ra, t0, PTRS_PER_PGD_BITS + PGDIR_SHIFT - 1, PGDIR_SHIFT
   alsl.d      t1, ra, t1, 3
#if CONFIG_PGTABLE_LEVELS > 3
   ld.d     t1, t1, 0
   bstrpick.d  ra, t0, PTRS_PER_PUD_BITS + PUD_SHIFT - 1, PUD_SHIFT
   alsl.d      t1, ra, t1, 3
#endif
#if CONFIG_PGTABLE_LEVELS > 2
   ld.d     t1, t1, 0
   bstrpick.d  ra, t0, PTRS_PER_PMD_BITS + PMD_SHIFT - 1, PMD_SHIFT
   alsl.d      t1, ra, t1, 3
#endif
   ld.d     ra, t1, 0

   /*
    * For huge tlb entries, pmde doesn't contain an address but
    * instead contains the tlb pte. Check the PAGE_HUGE bit and
    * see if we need to jump to huge tlb processing.
    */
   rotri.d     ra, ra, _PAGE_HUGE_SHIFT + 1
   bltz     ra, tlb_huge_update_modify

   rotri.d     ra, ra, 64 - (_PAGE_HUGE_SHIFT + 1)
   bstrpick.d  t0, t0, PTRS_PER_PTE_BITS + PAGE_SHIFT - 1, PAGE_SHIFT
   alsl.d      t1, t0, ra, _PTE_T_LOG2

#ifdef CONFIG_SMP
smp_pgtable_change_modify:
   ll.d     t0, t1, 0
#else
   ld.d     t0, t1, 0
#endif
   andi     ra, t0, _PAGE_WRITE
   beqz     ra, nopage_tlb_modify

   ori      t0, t0, (_PAGE_VALID | _PAGE_DIRTY | _PAGE_MODIFIED)
#ifdef CONFIG_SMP
   sc.d     t0, t1, 0
   beqz     t0, smp_pgtable_change_modify
#else
   st.d     t0, t1, 0
#endif
   tlbsrch
   bstrins.d   t1, zero, 3, 3
   ld.d     t0, t1, 0
   ld.d     t1, t1, 8
   csrwr    t0, LOONGARCH_CSR_TLBELO0
   csrwr    t1, LOONGARCH_CSR_TLBELO1
   tlbwr

   csrrd    t0, EXCEPTION_KS0
   csrrd    t1, EXCEPTION_KS1
   csrrd    ra, EXCEPTION_KS2
   ertn

#ifdef CONFIG_64BIT
vmalloc_modify:
   la_abs      t1, swapper_pg_dir
   b     vmalloc_done_modify
#endif

   /* This is the entry point of a huge page. */
tlb_huge_update_modify:
#ifdef CONFIG_SMP
   ll.d     ra, t1, 0
#else
   rotri.d     ra, ra, 64 - (_PAGE_HUGE_SHIFT + 1)
#endif
   andi     t0, ra, _PAGE_WRITE
   beqz     t0, nopage_tlb_modify

#ifdef CONFIG_SMP
   ori      t0, ra, (_PAGE_VALID | _PAGE_DIRTY | _PAGE_MODIFIED)
   sc.d     t0, t1, 0
   beqz     t0, tlb_huge_update_modify
   ori      t0, ra, (_PAGE_VALID | _PAGE_DIRTY | _PAGE_MODIFIED)
#else
   ori      t0, ra, (_PAGE_VALID | _PAGE_DIRTY | _PAGE_MODIFIED)
   st.d     t0, t1, 0
#endif
   csrrd    ra, LOONGARCH_CSR_ASID
   csrrd    t1, LOONGARCH_CSR_BADV
   andi     ra, ra, CSR_ASID_ASID
   invtlb      INVTLB_ADDR_GFALSE_AND_ASID, ra, t1

   /*
    * A huge PTE describes an area the size of the
    * configured huge page size. This is twice the
    * of the large TLB entry size we intend to use.
    * A TLB entry half the size of the configured
    * huge page size is configured into entrylo0
    * and entrylo1 to cover the contiguous huge PTE
    * address space.
    */
   /* Huge page: Move Global bit */
   xori     t0, t0, _PAGE_HUGE
   lu12i.w     t1, _PAGE_HGLOBAL >> 12
   and      t1, t0, t1
   srli.d      t1, t1, (_PAGE_HGLOBAL_SHIFT - _PAGE_GLOBAL_SHIFT)
   or    t0, t0, t1

   move     ra, t0
   csrwr    ra, LOONGARCH_CSR_TLBELO0

   /* Convert to entrylo1 */
   addi.d      t1, zero, 1
   slli.d      t1, t1, (HPAGE_SHIFT - 1)
   add.d    t0, t0, t1
   csrwr    t0, LOONGARCH_CSR_TLBELO1

   /* Set huge page tlb entry size */
   addu16i.d   t0, zero, (CSR_TLBIDX_PS >> 16)
   addu16i.d   t1, zero, (PS_HUGE_SIZE << (CSR_TLBIDX_PS_SHIFT - 16))
   csrxchg     t1, t0, LOONGARCH_CSR_TLBIDX

   tlbfill

   /* Reset default page size */
   addu16i.d   t0, zero, (CSR_TLBIDX_PS >> 16)
   addu16i.d   t1, zero, (PS_DEFAULT_SIZE << (CSR_TLBIDX_PS_SHIFT - 16))
   csrxchg     t1, t0, LOONGARCH_CSR_TLBIDX

   csrrd    t0, EXCEPTION_KS0
   csrrd    t1, EXCEPTION_KS1
   csrrd    ra, EXCEPTION_KS2
   ertn

nopage_tlb_modify:
   dbar     0x700
   csrrd    ra, EXCEPTION_KS2
   la_abs      t0, tlb_do_page_fault_1
   jr    t0
SYM_CODE_END(handle_tlb_modify)

```

其处理逻辑和上面的章节的[<u>handle_tlb_load</u>](#example_handle_tlb_load)主题逻辑差异不大，但是   
有几个需要注意下的是：
   - 将内存PTE读取后，首先要判断PTE属性是否可写``andi  $t0, $ra, _PAGE_WRITE``
   - 如果PTE不可写，直接走nopage_tlb_modify分支，
      ```asm
      nopage_tlb_modify:
         dbar     0x700
         csrrd    ra, EXCEPTION_KS2
         la_abs   t0, tlb_do_page_fault_1
      ```
      而nopage_tlb_modify实际上会调用LoongArch的__do_page_fault函数，这个函数会判断       
      这个Store操作是否合法，如果不合法的话，会发送信号，结束进程。

   - 假设这个PTE是可写的，而此时PTE没有置_PAGE_DIRTY，会将页表设置       
     ``ori  $t0, $t0, (_PAGE_VALID | _PAGE_DIRTY | _PAGE_MODIFIED)``
     写回到内存中。

   - 同时为了优化TLB重填，将PTE对应的奇数偶数页加载到了TLB中，避免了再一次进入TLB refill历程。

   - 最后ertn返回到出现异常的地方继续执行！

## 情况4. 如果权限不合法

假设上述的熟悉情况都正确，但是权限不能匹配，则此时执行如下的异常：

  - 页特权等级不合规例外（PPI）：访存操作的虚地址在 TLB 中找到了匹配且 V=1 的项，但是访问的特权等       
    级不合规，将触发该例外。特权等级不合规体现为，该页表项的 RPLV=0 且 CSR.CRMD.PLV 值大       
    于页表项中的 PLV；或是该页表项的 RPLV=1 且 CSR.CRMD.PLV 不等于页表项中的 PLV。       

  - 页不可读例外（PNR）：load 操作的虚地址在 TLB 中找到了匹配，且 V=1，且特权等级合规的项，但是该       
    页表项的 NR 位为 1，将触发该例外。

  - 页不可执行例外（PNX）：取指操作的虚地址在 TLB 中找到了匹配，且 V=1，且特权等级合规的项，但是       
    该页表项的 NX 位为 1，将触发该例外。


Linux对于这三种和内存管理相关的权限异常，使用了同一个例外处理函数：

```c
   [EXCCODE_TLBNR]      = handle_tlb_protect,
   [EXCCODE_TLBNX]      = handle_tlb_protect,
   [EXCCODE_TLBPE]      = handle_tlb_protect,
```

都是handle_tlb_protect处理函数，下面我们看handle_tlb_protect的具体内容。


```
SYM_CODE_START(handle_tlb_protect)
   UNWIND_HINT_UNDEFINED
   BACKUP_T0T1
   SAVE_ALL
   move     a0, sp
   move     a1, zero
   csrrd    a2, LOONGARCH_CSR_BADV
   REG_S    a2, sp, PT_BVADDR
   la_abs      t0, do_page_fault
   jirl     ra, t0, 0
   RESTORE_ALL_AND_RET
SYM_CODE_END(handle_tlb_protect)
```

上述的处理逻辑如下：
   - 保存所有出异常时的寄存器
   - 进入do_page_fault函数执行。

```c
asmlinkage void __kprobes do_page_fault(struct pt_regs *regs,
         unsigned long write, unsigned long address)
{
   irqentry_state_t state = irqentry_enter(regs);

   /* Enable interrupt if enabled in parent context */
   if (likely(regs->csr_prmd & CSR_PRMD_PIE))
      local_irq_enable();

   __do_page_fault(regs, write, address);

   local_irq_disable();

   irqentry_exit(regs, state);
}
```

``__do_page_fault``会判断操作的合法性。如果不合法，就会发送信号，然后结束进程。


## 如果使能硬件PTW

上述我们是假设使用软件TLB重填时的处理流程，因为我们需要考虑一些加速优化的方法。

但是如果使用硬件HPTW的话，我们处理的过程变得相对简单，如下初始化的时候的代码：

```c
   if (cpu_has_ptw) {
      exception_table[EXCCODE_TLBI] = handle_tlb_load_ptw;
      exception_table[EXCCODE_TLBL] = handle_tlb_load_ptw;
      exception_table[EXCCODE_TLBS] = handle_tlb_store_ptw;
      exception_table[EXCCODE_TLBM] = handle_tlb_modify_ptw;
   }
```

而这几个处理函数如下所示：

``handle_tlb_load_ptw``处理PIL异常

```asm
SYM_CODE_START(handle_tlb_load_ptw)
   UNWIND_HINT_UNDEFINED
   csrwr    t0, LOONGARCH_CSR_KS0
   csrwr    t1, LOONGARCH_CSR_KS1
   la_abs      t0, tlb_do_page_fault_0
   jr    t0
SYM_CODE_END(handle_tlb_load_ptw)
```

``handle_tlb_store_ptw``处理PIS异常

```asm
SYM_CODE_START(handle_tlb_store_ptw)
   UNWIND_HINT_UNDEFINED
   csrwr    t0, LOONGARCH_CSR_KS0
   csrwr    t1, LOONGARCH_CSR_KS1
   la_abs      t0, tlb_do_page_fault_1
   jr    t0
SYM_CODE_END(handle_tlb_store_ptw)
```


``handle_tlb_modify_ptw``处理PME异常
```asm
SYM_CODE_START(handle_tlb_modify_ptw)
   UNWIND_HINT_UNDEFINED
   csrwr    t0, LOONGARCH_CSR_KS0
   csrwr    t1, LOONGARCH_CSR_KS1
   la_abs      t0, tlb_do_page_fault_1
   jr    t0
SYM_CODE_END(handle_tlb_modify_ptw)
```

他们最后都调用了**do_page_fault**函数来进行页表的处理。

另外的异常情况PPI，PNR和PNX，和上面的一致，都是使用``handle_tlb_protect``处理函数。

