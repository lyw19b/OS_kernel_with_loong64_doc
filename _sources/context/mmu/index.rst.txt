=========
内存管理
=========

本章节主要会涉及LoongArch与操作系统底层内存管理的相关内容。包括TLB组织结构，内存管理指令，多级页表等内容。

学习完本章节，你会掌握基本的技能，学会如下的技能：
  - LoongArch虚拟地址和物理地址管理
  - LoongArch存储访问类型
  - DMW直接映射窗的使用方法
  - TLB组织结构，包括表项格式与相应位的功能
  - 软件如何管理TLB，相关指令
  - 与内存管理相关的CSR寄存器，用法说明
  - TLB的初始化步骤
  - LoongArch的多级页表是如何组织的
  - 16KB，4KB，2MB页都是如何工作的
  - 与内存管理相关的异常和例外
  - 软件重填是什么，是否可以不用软件重填
  - Linux内核中针对内存管理相关的内容
  - RT-Thread中和内存管理相关的内容

我们会结合实际的例子，解释和说明LoongArch是如果管理内存的。最后会以Linux内核和RT-Thread内核为例，从初始化到异常管理展示LoongArch底层管理。


.. toctree::
  :maxdepth: 1

  tlb_struct.md
  dmw.md
  tlb_ptw.md
  page_table_in_kernel.md
