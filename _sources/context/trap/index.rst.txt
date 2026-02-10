===============
中断与异常系统
===============

本章节主要介绍LoongArch架构下的中断与异常系统，包括异常的类型，不同配置下发生异常时的例外入口。

最后我们结合Linux内核中有关LoongArch的异常处理，举例说明如何使用中断和异常系统！

学习完本章节，你会掌握基本的技能，学会如下的技能：
  - LoongArch中断系统
  - 中断的打开与关闭
  - 中断和异常的处理流程
  - 中断和异常的入口地址怎么设置
  - 中断和异常怎么配置CSR.ECFG.VS
  - 几个常见的异常和中断的特殊说明： 比如定时器，TLB相关异常处理等
  - 对比下LoongArch与RISCV的异常类型
  - Linux的中断和异常处理过程是什么



.. toctree::
  :maxdepth: 1

  interrupt.md
  exception.md
  trap_in_kernel.md
