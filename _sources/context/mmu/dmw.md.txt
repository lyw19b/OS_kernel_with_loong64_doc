(csr-dmw)=
# 直接映射翻译模式

LoongArch的直接映射地址翻译模式是一个比较特殊的映射模式，相比于其他的架构X86，ARM，RISC-V等。   
它是将一个很大的虚拟地址空间线性的映射到了一个同等大小的物理空间（这里并没有要求这个物理地址空间都存在）。

下面我们介绍下DMW寄存器，因为它控制和设置一些访问的属性，下面我们主要以LoongArch64为主。

## CSR.DMW[0-3]

我们这里主要讲一下CSR.DMW0，其他的CSR.DMW1、CSR.DMW2和CSR.DMW3是与CSR.DMW0相似。

DMW寄存器宽度是64位，主要分三个域： PLV域，MAT域和VSEG域，如下所示，

|**位域**| **名称** | **属性** | **描述** |
|-|-|-|-|
|0 | PLV0 | 可读可写 | 1：表示在特权等级PLV0下使用，0表示不使用 |
|1 | PLV1 | 可读可写 | 1：表示在特权等级PLV1下使用，0表示不使用 |
|2 | PLV2 | 可读可写 | 1：表示在特权等级PLV2下使用，0表示不使用 |
|3 | PLV3 | 可读可写 | 1：表示在特权等级PLV3下使用，0表示不使用 |
|5:4| MAT | 可读可写 | 表示访问该命中DMW的内存类型，比如是Cached还是UnCached等|
|59:6 | 保留域| 只读 | 读返回0，软件不可写 | 
|63:60|VSEG | 可读可写 | 映射虚拟地址的[63:60]，用于匹配虚拟地址 |

:::{caution}
如果我们设置了一个地址A，然后这个地址既落在了DMW内，也有相应的页表映射，此时我们以DMW为主。    
也就是说CPU首先访问DMW，返回如果命中的话，就不会再去查找TLB了。
:::


可参考官方指令手册查看具体的寄存器内容。

:::{caution}
CSR.DMW**0**和CSR.DWM**1**: 此两个配置寄存器窗可以同时用于 **取指** 和 **Load、Store** 。    
CSR.DMW**2**和CSR.DWM**3**: 此两个配置寄存器窗只用于 **Load、Store**。不能用于取指。
:::


## 操作系统中如何使用

下面我们举一个简单的例子，来说明一下如何使用DMW直接地址映射。

假设DMW0寄存器的内容是0x9000000000000011

也就是说： PLV0 = 1， MAT = 2'b01, VSEG = 4'b1001

此时，如果虚拟地址VA[63:60] = 4'b1001时，并且当前的 PLV = 0，也就是特权级最高。

此时0x9000000000000000 - 0x9000FFFFFFFFFFFF 这段地址的就会映射到 0x0 - 0xFFFFFFFFFFFF

:::{tip}
> 算法就是: 将虚拟地址最高四位去掉，剩下的地址按照PALEN的扩展规则来决定。
:::

此时访问内存的类型由当前匹配的MAT决定。

### DMW的优点

优点在于，直接设置一次，可以将很大一块内存，甚至将全部的物理内存映射到一个DMW中，避免了像页管理那样，频繁的
访问TLB，设置相关属性等。能够极大的降低映射的复杂度，访问速度很快。

### DMW的缺点

缺点在于，缺乏灵活性，将整个的物理内存空间设置成了同样的访问属性，要么可缓存Cached的，要么不能缓存UnCached的。
确定总是相对的。


### 代码示例如何在内核中使用

```c
#define DMW_PABITS	48

// DMW Register Context
#define CSR_DMW0_PLV0		1 << 0
#define CSR_DMW0_VSEG		0x8000
#define CSR_DMW0_BASE		(CSR_DMW0_VSEG << DMW_PABITS)
#define CSR_DMW0_INIT		(CSR_DMW0_BASE | CSR_DMW0_PLV0)

#define CSR_DMW1_PLV0		1 << 0
#define CSR_DMW1_MAT		1 << 4
#define CSR_DMW1_VSEG		0x9000
#define CSR_DMW1_BASE		(CSR_DMW1_VSEG << DMW_PABITS)
#define CSR_DMW1_INIT		(CSR_DMW1_BASE | CSR_DMW1_MAT | CSR_DMW1_PLV0)

// CSR DMW Defined
#define LOONGARCH_CSR_DMWIN0		0x180	/* 64 direct map win0: MEM & IF */
#define LOONGARCH_CSR_DMWIN1		0x181	/* 64 direct map win1: MEM & IF */
#define LOONGARCH_CSR_DMWIN2		0x182	/* 64 direct map win2: MEM */
#define LOONGARCH_CSR_DMWIN3		0x183	/* 64 direct map win3: MEM */

// Code here.

// 1. UC, PLV0, 0x8000 xxxx xxxx xxxx
li.d		t0, CSR_DMW0_INIT	
csrwr		t0, LOONGARCH_CSR_DMWIN0
// 2. CA, PLV0, 0x9000 xxxx xxxx xxxx
li.d		t0, CSR_DMW1_INIT	
csrwr		t0, LOONGARCH_CSR_DMWIN1

```

CSR_DMW0_INIT: CSR_DMW0_PLV0为1表示只有特权级0才能使用。CSR_DMW0_BASE表示将虚拟地址0X8000 xxxx xxxx xxxx
映射到0Xxxxx xxxx xxxx上， 此时访问0X8000 xxxx xxxx xxxx的内存类型MAT=0，表示是UnCached的。

``csrwr		t0, LOONGARCH_CSR_DMWIN0`` 此时将内容写入DMW0寄存器中。


CSR_DMW1_INIT: CSR_DMW0_PLV0为1表示只有特权级0才能使用。CSR_DMW1_BASE表示将虚拟地址0X9000 xxxx xxxx xxxx
映射到0Xxxxx xxxx xxxx上，并且MAT=1，表示通过虚拟地址0X9000 xxxx xxxx xxxx访问时，内存类型是Cached，可缓存的。

:::{tip}
注意上述这样做的目的在于，如果我们访问正常的内存(即需要经过Cache的)，可以通过地址0x9000 xxxx xxxx xxxx访问，
如果我们想要访问设备的内存（即不需要经过Cached, UnCached类型），可以通过0x8000 xxxx xxxx xxxx访问。

假设我们访问物理地址``0x1ff0_1f00``这个地址的内容：

如果是设备内存的话，我们使用虚拟地址``0x800000001ff01f00``来访问。

如果是正常的内存，我们使用虚拟地址0x``900000001ff01f00``来访问。

如果我们访问正常物理地址``0x20000000``这个地址的内容：

我们可以使用``0x9000000020000000``来访问（推荐的方式），其实也可以使用``0x8000000020000000``来访问，读取的内存值是
一致的，唯一的区别是他们访问的速度不一致，前者会放到Cache中，而后者直接读内存或者写回到内存，导致性能低下。

:::


:::{note}
CSR.DMW直接映射模式有几点需要注意：     

1. DMW0/1 可以用于取指（Fetch），也可以访存（Load/Store）；而DMW2/3 只能用于访存（Load/Store）。    
	
2. 虚拟地址多次命中DMW.x的行为是不确定的。比如下面的代码：
   ```asm
   # 向DMW0写入PLV0，Cache，VSEG=8
   li.d  $t0, 0x8000 0000 0000 0011
   csrwr $t0, 0x180 // DMW0

   # 向DMW1写入PLV0/PLV3，UnCache，VSEG=8
   li.d  $t0, 0x8000 0000 0000 0009
   csrwr $t0, 0x181 // DMW1
   ```
   此时如果使用0x8000 xxxx xxxx xxxx访问时，结果行为会不预期，禁止这样使用。

3. 可以使用多个DMW映射到同一个块地址空间。比如我们上面将``0x8000 xxxx xxxx xxxx``和
   ``0x9000 xxxx xxxx xxxx`` 映射到了``xxxx xxxx xxxx``，可以通过不同的虚拟地址使用
   不同的内存属性进行访问。

4. 主要DMW的使用很方便，但是仅仅限定于特权态，在Linux中PLV0是开启的。如果说你将PLV3用户态也设置成了1   
   的话，这就导致，几乎全部的物理内存都可以被用户态的程序访问，这是不允许、禁止的。    
   除非不故意设置成用户态可访问！！！。

5. 在Linux内核中，通常使用``0x9000 xxxx xxxx xxxx``作为可缓存Cached内存访问的虚拟地址，
   使用``0x8000 xxxx xxxx xxxx``作为设备的访问虚拟地址。

   至于为什么用``0x9000``和``0x8000``是因为Loongson之前的很多固件使用这个地址，因此沿用了
   这个地址空间段。

   你可以不按照Linux的使用规约，自己定义别的，也是可以的，比如VSEG=4'b0110或者VSEG=4'b1110，
   也是合法的。

:::

如何地址划分，划分了怎么处理等等

使用csrrd/csrwr/csrxchg指令操作

汇编代码的初始话流程是什么？
