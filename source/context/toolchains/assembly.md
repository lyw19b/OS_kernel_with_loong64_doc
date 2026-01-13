# 汇编代码与链接脚本

## 与LoongArch相关的汇编伪指令

### move指令

### li.d/li.w指令

### 地址加载指令

la
la.abs

## 链接脚本

GNU LD 链接脚本是控制链接器行为的核心配置文件，通过定义内存布局、段分配规则和符号映射，实现对最终可执行文件或库的精确控制。以下是其核心语法和用法详解：

---

### 一、基础语法结构
#### 1. **MEMORY 坽块**  
定义物理内存区域及其属性，格式为：  
```text
MEMORY
{
  名称 (属性) : ORIGIN = 起始地址, LENGTH = 大小
}
```
- **属性**：`r`（可读）、`w`（可写）、`x`（可执行）、`a`（可分配）、`i`（已初始化）等，用 `!` 反转属性。  
- **示例**：  
  ```text
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 2M   # Flash存储器，只读可执行
  RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 128K # RAM，可读写执行
  ```

#### 2. **SECTIONS 块**  
定义段的放置规则，格式为：  
```text
SECTIONS
{
  段名称 : [AT(加载地址)] [ALIGN(对齐值)] {
    内容规则
  } >目标内存区域
}
```
- **关键概念**：  
  - **VMA（Virtual Memory Address）**：运行时地址。  
  - **LMA（Load Memory Address）**：加载地址（默认与 VMA 相同）。  
- **示例**：  
  ```text
  .text : {
    *(.text)       # 合并所有输入文件的 .text 段
    KEEP(*(.init)) # 保留初始化段不被优化
  } >FLASH AT>FLASH # 运行在 Flash，加载地址也为 Flash
  ```

---

### 二、核心命令与操作
#### 1. **段内容控制**
- **通配符匹配**：  
  - `*(.text)`：所有输入文件的 `.text` 段。  
  - `lib*.a:(.text)`：所有 `lib` 开头的库文件的 `.text` 段。  
- **排除规则**：  
  ```text
  *(EXCLUDE_FILE(boot.o) .text)  # 排除 boot.o 的 .text 段
  ```

#### 2. **符号操作**
- **符号赋值**：  
  ```text
  _start = 0x8000000;        # 定义入口地址
  data_start = ADDR(.data);  # 获取 .data 段地址
  ```  
- **PROVIDE 关键字**：  
  ```text
  PROVIDE(etext = .);        # 若未定义 etext，则设为当前地址
  ```

#### 3. **内存对齐与填充**
- **对齐**：  
  ```text
  . = ALIGN(4);              # 对齐到 4 字节边界
  .text : ALIGN(16) { ... }  # 段起始地址对齐到 16 字节
  ```
- **填充**：  
  ```text
  .bss : {
    *(.bss)
    FILL(0x90)               # 用 0x90 填充剩余空间
  }
  ```

#### 4. **动态表达式**
- **SIZEOF(section)**：获取段大小。  
- **ADDR(section)**：获取段起始地址。  
- **NEXT(address)**：返回对齐后的下一个地址。  
  ```text
  .data : {
    *(.data)
    _data_end = . + SIZEOF(.data);  # 计算 .data 结束地址
  }
  ```

---

### 三、高级功能
#### 1. **多内存区域管理**
通过 `MEMORY` 定义多个区域，结合 `>region` 指定段存放位置：  
```text
MEMORY
{
  ROM (rx)  : ORIGIN = 0x0, LENGTH = 32K
  RAM (rwx) : ORIGIN = 0x8000, LENGTH = 128K
}

SECTIONS
{
  .text : { *(.text) } >ROM
  .data : { *(.data) } >RAM AT>ROM  # 数据从 ROM 加载到 RAM
}
```

#### 2. **段叠加与覆盖**
- **OVERLAY**：允许段重叠加载（需配合 `INSERT` 命令）。  
- **PHDRS**：定义程序头表（如 ELF 的加载段）。

#### 3. **错误检查**
- **ASSERT**：条件不满足时终止链接。  
  ```text
  ASSERT(_end <= 0x20000000, "内存溢出")
  ```
- **NOCROSSREFS**：禁止段间交叉引用。  
  ```text
  NOCROSSREFS(.text .data)  # .text 和 .data 不可相互引用
  ```

---

### 四、典型应用场景
#### 1. **嵌入式开发**
- **Flash-RAM 分离**：将代码段（.text）放在 Flash，数据段（.data）从 Flash 加载到 RAM：  
  ```text
  SECTIONS
  {
    .text : { *(.text) } >FLASH
    .data : { *(.data) } >RAM AT>FLASH
    .bss  : { *(.bss) } >RAM
  }
  ```
- **启动代码**：通过 `KEEP(*(.init))` 保留初始化代码。

#### 2. **库文件优化**
- **隐藏符号**：使用 `HIDDEN(symbol)` 限制符号可见性。  
- **强制分配**：`FORCE_COMMON_ALLOCATION` 为未初始化符号分配空间。

#### 3. **安全加固**
- **反调试**：通过 `PHDRS` 修改程序头，隐藏调试信息。  
- **代码混淆**：重命名符号或调整段顺序。

---

### 五、实战示例
#### 示例 1：简单链接脚本
```text
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  .text : {
    KEEP(*(.isr_vector))  # 保留中断向量表
    *(.text)
  } >FLASH

  .data : {
    *(.data)
  } >RAM AT>FLASH

  .bss : {
    *(.bss)
    *(COMMON)
  } >RAM
}
```

#### 示例 2：动态数据加载
```text
SECTIONS
{
  .rodata : {
    *(.rodata)
  } >FLASH

  .data : {
    _data_load = LOADADDR(.data);  # 记录加载地址
    _data_start = ADDR(.data);     # 记录运行地址
    *(.data)
  } >RAM AT>_data_load
}
```

---

### 六、调试与验证
1. **查看段信息**：  
   ```bash
   arm-none-eabi-objdump -h program.elf  # 显示段布局
   ```
2. **生成映射文件**：  
   ```bash
   arm-none-eabi-ld -T linker.ld -M=map.txt main.c
   ```
3. **验证符号地址**：  
   ```bash
   nm program.elf | grep main  # 检查 main 函数地址
   ```

---

### 七、总结
GNU LD 链接脚本通过 **MEMORY** 和 **SECTIONS** 两大核心块，结合符号操作、对齐填充和动态表达式，实现了对程序内存布局的完全控制。其典型应用包括嵌入式系统的 Flash-RAM 分离、库文件优化和安全加固。掌握链接脚本是进行底层开发和性能调优的关键技能。


### 以LoongArch为例说明链接脚本


