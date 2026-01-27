# 中断

LoongArch 架构支持线中断和消息中断两种中断形式，其中**线中断**为**必须实现**的中断形式，**消息中断为可选
择实现的中断形式**，是在线中断基础上的扩展。

## 线中断

LoongArch 架构下每个处理器核内部可记录 13 个线中断，分别是：
   - 1个核间中断（IPI）
   - 1个定时器中断（TI）
   - 1个性能监测计数溢出中断（PMI）
   - 8个硬中断（HWI0~HWI7）
       - HWI7
       - HWI6
       - HWI5
       - HWI4
       - HWI3
       - HWI2
       - HWI1
       - HWI0
   - 2个软中断（SWI0~SWI1）
       - SWI1
       - SWI0

所有的线中断都是**电平中断**，且都是**高电平**有效。

### 核间中断IPI

核间中断的中断输入来自于核外的中断控制器，其被处理器核采样记录在 CSR.ESTAT.IS[12]位。        


### 定时器中断

定时器中断的中断源来自于核内的恒定频率定时器。当恒定频率定时器倒计时至全 0 值时，该中断被        
置起。置起后的定时器中断被处理器核采样记录在 CSR.ESTAT.IS[11]位。

:::{tip}
清除定时器中断**需要通过软件**向CSR.TICLR 寄存器的 TI 位写 1 来完成。
:::


### 性能计数器中断

性能计数器溢出中断的中断源来自于核内的性能计数器。当任一个中断使能开启的性能计数器的计数        
值的第[63]位为1时， 该中断将被置起。 

置起后的性能计数器溢出中断被处理器核采样记录在        
CSR.ESTAT.IS[10]位。

:::{tip}
清除性能计数器溢出中断需要将引起中断的那个性能计数器的**第[63]位置为0**，或者**关闭该性能计数器**的中断使能。
:::


### 外部硬中断
硬中断的中断源来自于处理器核外部，其直接来源通常是核外的中断控制器。8个硬中断 HWI[7:0]被处        
理器核采样记录在 CSR.ESTAT.IS[9:2]位。

:::{tip}
清除外部硬中断必须让外部中断源清0，或者关闭对应的外部中断。
:::


### 内部软中断
软中断的中断源来自于处理器核内部，软件通过 CSR 指令对 CSR.ESTAT.IS[1:0]写 1 则置起软中断，

:::{tip}
写0 则清除软中断。
:::


### 中断号

中断在 CSR.ESTAT.IS 域中记录的位置的索引值也被称为中断号（Int Number）。

|对应CSR.ESTAT位域|读写|中断源|中断号|
|-|-|-|-|
|IS[0] |RW| SWI0 | 0  |
|IS[1] |RW| SWI1 | 1  |
|IS[2] |R | HWI0 | 2  |
|IS[3] |R | HWI1 | 3  |
|IS[4] |R | HWI2 | 4  |
|IS[5] |R | HWI3 | 5  |
|IS[6] |R | HWI4 | 6  |
|IS[7] |R | HWI5 | 7  |
|IS[8] |R | HWI6 | 8  |
|IS[9] |R | HWI7 | 9  |
|IS[10]|R | PMI  | 10 |
|IS[11]|R | TI   | 11 |
|IS[12]|R | IPI  | 12 |



## 消息中断

龙芯架构中每个逻辑处理器核内部可记录 256 个消息中断，可以包含处理器核外部输入的消息型核间        
中断和消息型硬中断。

一个处理器核内部 CSR.MSGIS0~CSR.MSGIS3 四个 64 位的 CSR 依次记录了从 0～255        
号消息中断有无被置起。

每个处理器核内部 256 个消息中断的具体来源由实现决定，软件开发人员需要参阅具体的芯片用户手册以获取相关信息。        



## 中断的优先级

### 线中断的优先级

同一时刻多个中断的响应采用固定优先级仲裁机制，中断号越大优先级越高。

:::{tip}
IPI 的优先级最高，TI 次之，SWI0 的优先级最低。

SWI0 < SWI1 < HWI0 < HWI1 < HWI2 < HWI3 < HWI4 < HWI5 < HWI6 < HWI7 < PMI < TI < IPI

:::


### 消息中断的优先级

龙芯架构每个逻辑处理器核内 256 个消息中断之间采用固定优先级，中断号越大的优先级越高。

第255 号消息中断优先级最高，第 254 号消息中断次之，第 0 号消息中断优先级最低。

每个逻辑处理器核内部记录的消息中断只有其优先级不低于消息中断使能优先级门限值（记录在
CSR.MSGIE.PT 域）时，才可能被硬件进一步挑选并置起消息中断请求。


当一个处理器核内部同时有置起的消息中断请求和线中断请求时，消息中断请求的优先级高于线中断
请求。


## 中断的打开与关闭

1. 首先在控制与状态寄存器CSR.CRMD.IE域，使能全局的中断使能，高电平有效，即置1是打开全局中断。

:::{tip}
当触发例外时，硬件将该域的值置为 0，以确保陷入后屏蔽中断。例外处理程序决定      
重新开启中断响应时，需显式地将该位置 1。
:::

打开全局中断使能：

```asm
li.d  $t0, 0x04 // 加载 IE_MASK

csrxchg $t0, $t0, 0
```

关闭全局中断使能：

```asm
li.d  $t0, 0x04 // 加载 IE_MASK

csrxchg $zero, $t0, 0
```

使用内敛汇编如下：

```c
#define LOONGARCH_CSR_CRMD		0x0	/* Current mode info */
#define  CSR_CRMD_IE_SHIFT		2
#define  CSR_CRMD_IE			(0x1 << CSR_CRMD_IE_SHIFT)


static inline void arch_local_irq_enable(void)
{
	u32 flags = CSR_CRMD_IE;
	__asm__ __volatile__(
		"csrxchg %[val], %[mask], %[reg]\n\t"
		: [val] "+r" (flags)
		: [mask] "r" (CSR_CRMD_IE), [reg] "i" (LOONGARCH_CSR_CRMD)
		: "memory");
}

static inline void arch_local_irq_disable(void)
{
	u32 flags = 0;
	__asm__ __volatile__(
		"csrxchg %[val], %[mask], %[reg]\n\t"
		: [val] "+r" (flags)
		: [mask] "r" (CSR_CRMD_IE), [reg] "i" (LOONGARCH_CSR_CRMD)
		: "memory");
}

```


2. 例外配置CSR.ECFG寄存器中的LIE域，控制着局部中断的使能，高电平有效。

:::{tip}
这些局部中断使能位与 CSR.ESTAT 中 IS 域记录的 13 个中       
断源一一对应，每一位控制一个中断源。

CSR.ECFG[12:0] 与 CSR.ESTAT.IS[12:0] 一一对应！
:::


```c
#define LOONGARCH_CSR_ECFG		0x4	/* Exception config */

static inline void set_ecfg_lie_enable(u32 ecfg_flags)
{
	u32 flags = ecfg_flags;
	u32 mask  = ecfg_flags;
	__asm__ __volatile__(
		"csrxchg %[val], %[mask], %[reg]\n\t"
		: [val] "+r" (flags)
		: [mask] "r" (mask), [reg] "i" (LOONGARCH_CSR_ECFG)
		: "memory");
}
```


## 中断的入口地址

见下章节[<u>异常的入口地址选择</u>](#expection_entry_selection)内容描述。


## 中断的处理流程

当触发中断时，处理器硬件会进行如下操作：
  
  - ❖ 将 CSR.CRMD 的 PLV、IE 分别存到 CSR.PRMD 的 PPLV、PIE 中，然后将 CSR.CRMD 的 PLV 置      
	  为 0，IE 置为 0；
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.CRMD 的 WE 存到 CSR.PRMD 的 PWE 中，然后将      
      CSR.CRMD 的 WE 置为 0；
  
  - ❖ 将触发例外指令的 PC 值记录到 CSR.ERA 中；
  
  - ❖ 跳转到例外入口处取指。      
      当软件执行 ERTN 指令从普通例外执行返回时，处理器硬件会完成如下操作：
  
  - ❖ 将 CSR.PRMD 中的 PPLV、PIE 值恢复到 CSR.CRMD 的 PLV、IE 中；
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.PRMD 中的 PWE 值恢复到 CSR.CRMD 的 WE 中；
  
  - ❖ 跳转到 CSR.ERA 所记录的地址处取指。      
      针对上述硬件实现，软件在例外处理过程的中途如果需要开启中断，需要保存 CSR.PRMD 中的 PPLV、      
      PIE 等信息，并在例外返回前，将所保存的信息恢复到 CSR.PRMD 中。

