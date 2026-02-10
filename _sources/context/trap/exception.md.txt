# 例外

## 例外的优先级

例外优先级遵循两个基本原则：
 - 其一，中断的优先级高于例外；
 - 其二，对于例外，取指阶段检测出的优先级最高，       
        译码阶段检测出的优先级次之，       
        执行阶段检测出的优先级再次之。             

对于取指阶段检测出的例外：取指 Watch 例外优先级最高，取指地址错例外优先级次之，取指 TLB 相      
关例外优先级再次之，取指的机器错误例外优先级最低。      
译码阶段可检测出的例外彼此互斥，故无需考虑其间的优先级。      

执行阶段仅访存指令或同时触发多种例外，       

其优先级从高到低依次为：       
地址错例外(ADE) >        
要求地址对齐的访存指令因地址不对齐而产生的地址对齐错例外(ALE) >        
边界约束检查错例外1(BCE) >        
TLB 相关的例外 >        
允许地址非对齐的访存指令因地址跨越了不同存储访问类型的两个页时而产生的地址对齐错例外       
(ALE)。

不过，这里有个细节需要特别说明一下，即对于要求地址对齐的访存指令，如果它在地址不对齐的       
情况下恰好跨越了两个属性不同的访问区域，那么如果该指令在低地址所在访问区域内满足 ADE 例外判定      
条件，那么该指令触发 ADE 例外，但是，如果该指令仅在高地址所在访问区域内满足 ADE 例外判定条件，      
那么该指令触发 ALE 例外而非 ADE 例外。      


## 例外的入口地址

见下章节[<u>异常的入口地址选择</u>](#expection_entry_selection)内容描述。


## 例外的处理过程

### 普通例外的处理过程

当触发普通例外时，处理器硬件会进行如下操作：
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


### TLB 重填例外硬件处理过程

当触发 TLB 重填例外时，处理器硬件会进行如下操作：

  - ❖ 将 CSR.CRMD 的 PLV、IE 分别存到 CSR.TLBRPRMD 的 PPLV、PIE 中，然后将 CSR.CRMD 的       
	  PLV 置为 0，IE 置为 0，DA 置为 1，PG 置为 0；       
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.CRMD 的 WE 存到 CSR.TLBRPRMD 的 PWE 中，然后       
	  将 CSR.CRMD 的 WE 置为 0；       
  
  - ❖ 将触发例外指令的 PC 的[GRLEN-1:2]位记录到 CSR.TLBRERA 的 ERA 域中，将 CSR.TLBRERA       
      的 IsTLBR 置为 1；
  
  - ❖ 将触发该例外的访存虚地址（如果是取指触发的则就是 PC）记录到 CSR.TLBRBADV 中，将虚地       
      址的[PALEN-1:13]位记录到 CSR.TLBREHI 的 VPPN 域中；       
  
  - ❖ 跳转到 CSR.TLBRENTTRY 所配置的例外入口处取指。       
      当软件执行 ERTN 指令从 TLB 重填例外执行返回时，处理器硬件会完成如下操作：       
  
  - ❖ 将 CSR.TLBRPRMD 中的 PPLV、PIE 值恢复到 CSR.CRMD 的 PLV、IE 中；       
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.TLBRPRMD 中的 PWE 值恢复到 CSR.CRMD 的 WE 中；       
  
  - ❖ 将 CSR.CRMD 的 DA 置为 0，PG 置为 1；
  
  - ❖ 将 CSR.TLBRERA 的 IsTLBR 置为 0；
  
  - ❖ 跳转到 CSR.TLBRERA 所记录的地址处取指。


### 机器错误例外硬件处理过程

当触发机器错误例外时，处理器硬件会进行如下操作：
  
  - ❖ 将 CSR.CRMD 的 PLV、IE、DA、PG、DATF、DATM 分别存到 CSR.MERRCTL 的 PPLV、PIE、      
      PDA、PPG、PDATF、PDATM 中，然后将 CSR.CRMD 的 PLV 置为 0，IE 置为 0，DA 置为 1，PG      
      置为 0，DATF 置为 0，DATM 置为 0；      
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.CRMD 的 WE 存到 CSR.MERRCTL 的 PWE 中，然后将       
      CSR.CRMD 的 WE 置为 0；       
  
  - ❖ 将触发例外指令的 PC 记录到 CSR.MERRERA 中；
  
  - ❖ 将 CSR.MERRCTL 的 IsMERR 位置为 1；
  
  - ❖ 将校验的具体错误信息记录到 CSR.MERRINFO1 和 CSR.MERRINFO2 中；
  
  - ❖ 跳转到 CSR.MERRENTRY 所配置的例外入口处取指。             
      当软件执行 ERTN 指令从机器错误例外执行返回时，处理器硬件会完成如下操作：
  
  - ❖ 将 CSR.MERRCTL 中的 PPLV、PIE、PDA、PPG、PDATF、PDATM 值恢复到 CSR.CRMD 的 PLV、       
      IE、DA、PG、DATF、DATM 中；
  
  - ❖ 对于支持 Watch 功能的实现，还要将 CSR.MERRCTL 中的 PWE 值恢复到 CSR.CRMD 的 WE 中；
  
  - ❖ 将 CSR.MERRCTL 的 IsMERR 位置为 0；
  
  - ❖ 跳转到 CSR.MERRERA 所记录的地址处取指。
