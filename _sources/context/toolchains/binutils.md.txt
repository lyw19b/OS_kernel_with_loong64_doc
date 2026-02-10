# Binutils

GNU Binutils 是 GNU 项目下的**二进制工具集合**，用于**处理、分析和转换二进制文件**
（如目标文件、可执行文件、库文件等），是 Linux/Unix 系统开发流程中的核心工具链组件之一。它涵盖了从汇编、链接到二进制文件分析的完整环节，
支撑着 GCC 等编译器的底层运作，也是嵌入式开发、逆向工程、性能调优等场景的基础工具。

## GNU Binutils的相关说明

### **核心定位与背景**
GNU Binutils 的设计目标是**为 GNU 系统提供二进制文件处理能力**，最初是为了配合 GCC（GNU 编译器集合）使用，
解决目标文件的汇编、链接、符号管理等问题。随着发展，其功能逐渐扩展到二进制文件分析、转换、优化等多个领域，
成为 Linux 开发环境中不可或缺的工具集。  

Binutils 的核心依赖是**BFD（Binary File Descriptor）库**，该库提供了统一的接口来处理多种二进制文件格式（如 ELF、COFF、a.out 等），使得 Binutils 中的工具能跨格式工作。


### **重要组成部分与功能**
GNU Binutils 包含十余种工具，可分为**核心工具**（汇编、链接）、**辅助工具**（文件处理、符号分析）、
**跨平台工具**（Windows 兼容）三大类，以下是关键工具的详细说明：

#### **核心工具：汇编与链接**

- **`as`（GNU 汇编器）**：  
  将汇编语言源文件（如 `.s`、`.asm`）转换为**目标文件**（`.o`），是 GCC 编译流程中的第二步（第一步是预处理，第三步是链接）。
  `as` 支持多种架构（x86、ARM、RISC-V、LoongArch 等），其输出为目标文件中的机器码，供链接器使用。

- **`ld`（GNU 链接器）**：  
  将多个目标文件（`.o`）或库文件（`.a`、`.so`）组合成**可执行文件**或**共享库**。
  链接器的主要作用是解决符号引用（如函数调用、全局变量访问），分配内存地址，并处理重定位（Relocation）。
  `ld` 支持自定义链接脚本（Link Script），用于控制输出文件的布局（如代码段、数据段的位置）。
  
  后续章节我们会详细的介绍使用的例子。

- **`gold`（快速链接器）**：  
  `ld` 的替代工具，专为**ELF 格式**（Linux 下的标准可执行文件格式）设计，链接速度比传统 `ld` 快数倍。它优化了符号解析和数据布局，适合大型项目（如内核、浏览器）的快速构建，但目前仍处于 beta 测试阶段。


#### **辅助工具：二进制文件分析与处理**

- **`objdump`（目标文件信息显示）**：  
  显示目标文件或可执行文件的详细信息，包括**反汇编代码**、**段结构**（如 `.text`、`.data`、`.bss`）、
  **符号表**、**重定位条目**等。
  `objdump` 是逆向工程和调试的常用工具，例如通过 `-d` 选项可查看程序的机器码对应的汇编指令。

- **`nm`（符号表列举）**：  
  列出目标文件或库文件中的**符号信息**（如函数名、全局变量名），包括符号的地址、类型
  （如 `T` 表示代码段符号、`D` 表示数据段符号）。`nm` 常用于定位未定义符号
  （如链接错误中的 `undefined reference`）或分析库文件的导出符号。

- **`objcopy`（目标文件转换）**：  
  将目标文件从一种格式转换为另一种格式（如 ELF 转二进制、二进制转 S-record），
  或修改目标文件的段属性（如将 `.text` 段设置为只读）。例如，嵌入式开发中常用 `objcopy` 
  将 ELF 可执行文件转换为裸二进制文件（`.bin`），用于烧录到 Flash 中。

- **`readelf`（ELF 文件信息显示）**：  
  专门用于显示**ELF 格式**文件的信息（如 ELF 头、程序头表、节头表、符号表），比 `objdump` 更聚焦于 ELF 结构的细节。
  `readelf` 常用于分析 ELF 文件的布局（如代码段的位置、动态链接信息），
  是理解 Linux 可执行文件格式的关键工具。

- **`strip`（符号剥离）**：  
  从目标文件或可执行文件中**移除符号表、调试信息**（如 `.debug_*` 段），减小文件体积。`strip` 常用于发布版本的程序，避免泄露内部符号信息，或提高加载速度。

- **`ar`（归档工具）**：  
  创建、修改或提取**静态库**（`.a` 文件），静态库是多个目标文件的归档集合。    
  例如，`ar rv libfoo.a foo1.o foo2.o` 可将 `foo1.o` 和 `foo2.o` 打包成 `libfoo.a`，供链接器使用。

- **`ranlib`（归档索引生成）**：  
  为静态库生成**符号索引**（`.symtab` 段），加速链接器对静态库的符号查找。
  `ranlib` 通常与 `ar` 配合使用，例如 `ar rv libfoo.a foo1.o && ranlib libfoo.a`。

- **`addr2line`（地址转行号）**：  
  将程序中的**虚拟地址**转换为**源文件行号**，常用于调试（如 coredump 分析）。     
  例如，`addr2line -e a.out 0x400520` 可显示地址 `0x400520` 对应的源文件和行号。

- **`c++filt`（C++ 符号反混淆）**：  
  将 C++ 编译器生成的**混淆符号**（如 `_ZN3Foo3barEv`）
  转换为人类可读的形式（如 `Foo::bar()`），便于分析 C++ 程序的符号表。

- **`size`（段大小列举）**：  
  列出目标文件或库文件的**段大小**（如 `.text`、`.data`、`.bss`），帮助开发者了解程序的内存占用情况。   
  例如，`size a.out` 可显示各段的大小之和。

- **`strings`（可打印字符串列举）**：  
  列出文件中的**可打印字符串**（如 ASCII 字符串），常用于分析二进制文件中的隐藏信息（如错误信息、版权信息）。


#### **跨平台工具：Windows 兼容**
- **`dlltool`（DLL 工具）**：  
  创建 Windows 动态链接库（`.dll`）的依赖文件（如 `.lib` 导入库），或生成 DLL 的导出表。    
  例如，`dlltool -e exports.o -l dll.lib dll.o` 可从 `dll.o` 生成 `dll.lib`，供 Windows 程序链接使用。

- **`windmc`（Windows 消息编译器）**：  
  编译 Windows 消息资源文件（`.mc`），生成消息表和资源头文件，用于国际化（i18n）开发。

- **`windres`（Windows 资源编译器）**：  
  编译 Windows 资源文件（`.rc`），生成目标文件（`.o`），用于嵌入图标、对话框等资源到可执行文件中。


### **应用场景**
GNU Binutils 的应用场景覆盖**软件开发的全流程**，以下是典型案例：

1. **嵌入式开发**：  
   嵌入式设备的资源有限（如 ROM/RAM 小），需用 `objcopy` 将 ELF 可执行文件转换为裸二进制文件（`.bin`），
   再用烧录工具写入 Flash；用 `nm` 分析静态库的符号，确保没有冗余代码。

2. **逆向工程**：  
   用 `objdump` 反汇编可执行文件，分析其机器码逻辑；用 `readelf` 查看 ELF 文件的动态链接信息（如依赖的共享库）；
   用 `strings` 查找二进制文件中的敏感字符串（如密码、URL）。

3. **性能调优**：  
   用 `gprof`（性能分析工具）分析程序的调用图，找出性能瓶颈；用 `strip` 移除调试信息，减小程序体积，提高加载速度。

4. **库开发**：  
   用 `ar` 和 `ranlib` 创建静态库（`.a`），用 `ld` 生成共享库（`.so`）；用 `nm` 检查库的导出符号，确保符合 API 设计规范。


下面我们针对不同的工具进行详细的案例说明使用。


## ld链接器

链接器就是将多个目标文件（`.o`）或库文件（`.a`、`.so`）组合成**可执行文件**或**共享库**。
链接器的主要作用是解决符号引用（如函数调用、全局变量访问），分配内存地址，并处理重定位（Relocation）。
`ld` 支持自定义链接脚本（Link Script），用于控制输出文件的布局（如代码段、数据段的位置）。

例如： 我们先有一个main.c 和 hello.c
```c
// main.c 
void call_say_hello();
void exit();

void main()
{
	call_say_hello();
	exit();
}

void _start(){
	main();
}
```

```c
// hello.c 
char *ptr = "hello LoongArch\n"; 

void call_say_hello() {
	__asm__ (
        "li.d $a7, 64   \t\n"
        "li.d $a0, 1   \t\n"
        "move $a1, %[wptr]   \t\n"
        "li.d $a2, %[count]   \t\n"
        "syscall 0x0"
        :   /*Output*/
        : [wptr] "r" (ptr),  /*Input*/
          [count] "I" (16)
        :"memory"
	);
}

void exit() {
	__asm__(
        "li.d $a7, 93 \t\n"
        "li.d $a0, 0 \t\n"
        "syscall 0x0 \t\n"
        :::"memory"
	);
}
```
其中hello.c中的内嵌汇编，我们在后面的章节会详细的介绍。

```bash
shell> loongarch64-linux-musl-gcc main.c -o main.o -c
shell> loongarch64-linux-musl-gcc hello.c -o hello.o -c
shell> loongarch64-linux-musl-ld main.o hello.o -o hello
```

我们使用``file hello``可以查看生成ELF的信息
```bash
shell> file hello
hello: ELF 64-bit LSB executable, LoongArch, version 1 (SYSV), statically linked, not stripped
```

然后可以执行
```bash
shell> ./hello
Hello LoongArch!
```

### LD的参数说明

下面我们详细说下ld链接器常使用的命令说明：
```bash
ld -o output hello.o -lc
```
将hello.o文件，生成output文件，hello.o中用到了C库中的函数，因此需要指定C库``-lc``
<!-- TODO: 什么是静态链接和动态链接 -->

具体的参数如下说明：

---
-e entry 或者
--entry=entry:
   使用entry显式的作为程序的入口地址，而不是使用默认的地址。
   在LoongArch中默认的地址是``0000000120000000``
   默认的输入数字是按照十进制来输入，比如``--entry=10000``是指十进制的10000。
   如果想用十六进制或者八进制，需要使用相应的前缀。
   比如：
   ```bash
   --entry=0x10000 #指的是16进制的0x10000
   --entry=010000  #指的是8进制的10000
   ```


---
-l namespec 或者
--library=namespec:
   增加一个.a或者.so按照namespec指定的列表。典型的通过库搜索路劲，去查找libnamespec.a或者libnamespec.so文件

---
-L searchdir
--library-path=searchdir
   Add path searchdir to the list of paths that ld will search for archive libraries and ld control scripts. You may use this option any number of times. The directories are searched in the order in which they are specified on the command line. Directories specified on the command line are searched before the default directories. All -L options apply to all -l options, regardless of the order in which the options appear. -L options do not affect how ld searches for a linker script unless -T option is specified.

---
-M
--print-map
	打印在链接过程中，指定符号的内存排布

	```bash
	loongarch64-linux-musl-ld main.o hello.o -o hello -M=link.map
	```

	```
	There are no discarded input sections
    Memory Configuration
    Name             Origin             Length             Attributes
    *default*        0x0000000000000000 0xffffffffffffffff
    Linker script and memory map
    LOAD main.o
    LOAD hello.o
                    [!provide]                        PROVIDE (__executable_start = SEGMENT_START ("text-segment", 0x120000000))
                0x00000001200000e8                . = (SEGMENT_START ("text-segment", 0x120000000) + SIZEOF_HEADERS)
    .note.gnu.build-id
     *(.note.gnu.build-id)

     .text           0x00000001200000e8       0xb4
 	 *(.text.unlikely .text.*_unlikely .text.unlikely.*)
 	 *(.text.exit .text.exit.*)
 	 *(.text.startup .text.startup.*)
 	 *(.text.hot .text.hot.*)
 	 *(SORT_BY_NAME(.text.sorted.*))
 	 *(.text .stub .text.* .gnu.linkonce.t.*)
 	 .text          0x00000001200000e8       0x54 main.o
 	                0x00000001200000e8                main
 	                0x0000000120000114                _start
 	 .text          0x000000012000013c       0x60 hello.o
 	                0x000000012000013c                call_say_hello
 	                0x0000000120000174                exit
 	 *(.gnu.warning)
	```

<!-- https://www.sourceware.org/binutils/docs/ld.html -->


<!-- ## as
将汇编语言源文件（如 `.s`、`.asm`）转换为**目标文件**（`.o`），是 GCC 编译流程中的第二步（第一步是预处理，第三步是链接）。
  `as` 支持多种架构（x86、ARM、RISC-V、LoongArch 等），其输出为目标文件中的机器码，供链接器使用。

还是以上面的main.c 和 helloc
```bash
loongarch64-linux-musl-as test_main.S -o test_main.o

/home/airxs/local/loongarch64-linux-musl-gcc-15.2.0/bin/../lib/gcc/loongarch64-linux-musl/15.2.0/../../../../loongarch64-linux-musl/bin/as -v --traditional-format -mabi=lp64d -mrelax -o /tmp/ccPWOiX2.o /tmp/cc52QviI.s


```

### 常用的参数说明

---
-mabi=abi:
	指定使用哪一个abi，在LoongArch上，常见的有``lp64s``, ``lp64d``

 -->

## 如何正确的使用readelf

`readelf` 是 Linux 下用于分析 ELF（Executable and Linkable Format）格式文件的核心工具，能够显示 ELF 文件的头部信息、节区结构、符号表、重定位信息等。以下是其常用用法及关键选项解析：

---

### 基础用法与核心选项
#### 查看全部信息
```bash
readelf -a <elf文件>
```
- **等效选项**：`-h -l -S -s -r -d -V -A -I`  
- **作用**：综合显示文件头、程序头、节头、符号表、重定位表等所有关键信息。  
- **适用场景**：快速全面分析未知 ELF 文件结构。

#### 查看 ELF 文件头
```bash
readelf -h <elf文件>
```
- **关键字段**：
  - `Magic`：ELF 文件标识（`0x7F 45 4C 46`）。
  - `Class`：32 位（`ELF32`）或 64 位（`ELF64`）。
  - `Machine`：目标架构（如 `Intel 80386`、`ARM`）。
  - `Entry point address`：程序入口地址。
- **示例输出片段**：
  ```text
  ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           LoongArch
  Version:                           0x1
  Entry point address:               0x120000540
  Start of program headers:          64 (bytes into file)
  Start of section headers:          67672 (bytes into file)
  Flags:                             0x43, DOUBLE-FLOAT, OBJ-v1
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         8
  Size of section headers:           64 (bytes)
  Number of section headers:         24
  Section header string table index: 23

  ```

#### 查看节区头表（Sections）
```bash
readelf -S <elf文件>
```
- **输出内容**：节区名称、类型、大小、地址、权限等。
- **关键节区类型**：
  - `.text`：代码段（可执行指令）。
  - `.data`：已初始化全局变量。
  - `.bss`：未初始化全局变量。
  - `.debug_info`：调试信息。
- **示例**：
  ```text
  There are 24 section headers, starting at offset 0x10858:
	Section Headers:
	  [Nr] Name              Type             Address           Offset
	       Size              EntSize          Flags  Link  Info  Align
	  [ 0]                   NULL             0000000000000000  00000000
	       0000000000000000  0000000000000000           0     0     0
	  [ 1] .interp           PROGBITS         0000000120000200  00000200
	       000000000000001e  0000000000000000   A       0     0     1
	  [ 2] .hash             HASH             0000000120000220  00000220
	       0000000000000038  0000000000000004   A       4     0     8
	  [ 3] .gnu.hash         GNU_HASH         0000000120000258  00000258
	       000000000000001c  0000000000000000   A       4     0     8
	  [ 4] .dynsym           DYNSYM           0000000120000278  00000278
	       00000000000000d8  0000000000000018   A       5     1     8
	  [ 5] .dynstr           STRTAB           0000000120000350  00000350
	       0000000000000090  0000000000000000   A       0     0     1
	  [ 6] .rela.dyn         RELA             00000001200003e0  000003e0
	       0000000000000090  0000000000000018   A       4     0     8
	  [ 7] .rela.plt         RELA             0000000120000470  00000470
	       0000000000000060  0000000000000018  AI       4    17     8
	  [ 8] .plt              PROGBITS         00000001200004d0  000004d0
	       0000000000000060  0000000000000010  AX       0     0     16
	  [ 9] .text             PROGBITS         0000000120000540  00000540
	       0000000000000200  0000000000000000  AX       0     0     32
	  [10] .rodata           PROGBITS         0000000120000740  00000740
	       000000000000000b  0000000000000000   A       0     0     8
	  [11] .eh_frame_hdr     PROGBITS         000000012000074c  0000074c
	       0000000000000014  0000000000000000   A       0     0     4
	  [12] .eh_frame         PROGBITS         0000000120000760  00000760
	       000000000000003c  0000000000000000   A       0     0     8
	  [13] .init_array       INIT_ARRAY       000000012001fdf0  0000fdf0
	       0000000000000008  0000000000000008  WA       0     0     8
	  [14] .fini_array       FINI_ARRAY       000000012001fdf8  0000fdf8
	       0000000000000008  0000000000000008  WA       0     0     8
	  [15] .dynamic          DYNAMIC          000000012001fe00  0000fe00
	       00000000000001b0  0000000000000010  WA       5     0     8
	  [16] .got              PROGBITS         000000012001ffb0  0000ffb0
	       0000000000000040  0000000000000008  WA       0     0     8
	  [17] .got.plt          PROGBITS         000000012001fff0  0000fff0
	       0000000000000030  0000000000000008  WA       0     0     8
	  [18] .sdata            PROGBITS         0000000120020020  00010020
	       0000000000000008  0000000000000000  WA       0     0     8
	  [19] .bss              NOBITS           0000000120020028  00010028
	       0000000000000038  0000000000000000  WA       0     0     8
	  [20] .comment          PROGBITS         0000000000000000  00010028
	       0000000000000012  0000000000000001  MS       0     0     1
	  [21] .symtab           SYMTAB           0000000000000000  00010040
	       0000000000000570  0000000000000018          22    42     8
	  [22] .strtab           STRTAB           0000000000000000  000105b0
	       00000000000001ed  0000000000000000           0     0     1
	  [23] .shstrtab         STRTAB           0000000000000000  0001079d
	       00000000000000bb  0000000000000000           0     0     1
	Key to Flags:
	  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
	  L (link order), O (extra OS processing required), G (group), T (TLS),
	  C (compressed), x (unknown), o (OS specific), E (exclude),
	  D (mbind), p (processor specific)
  ```

#### 查看符号表
```bash
readelf -s <elf文件>
```
- **输出内容**：符号名称、地址、大小、绑定类型（如 `GLOBAL`、`STATIC`）。
- **选项扩展**：
  - `--dyn-syms`：显示动态符号表（动态链接库相关）。
  - `--use-dynamic`：优先使用动态符号表而非静态符号表。
- **示例**：
  ```text
  Symbol table '.symtab' contains 16 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000001200000e8     0 SECTION LOCAL  DEFAULT    1 .text
     2: 00000001200001a0     0 SECTION LOCAL  DEFAULT    2 .rodata
     3: 00000001200001b8     0 SECTION LOCAL  DEFAULT    3 .eh_frame
     4: 0000000120010000     0 SECTION LOCAL  DEFAULT    4 .data
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 .comment
     6: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     7: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS hello.c
     8: 0000000120010000     8 OBJECT  GLOBAL DEFAULT    4 ptr
     9: 000000012000013c    56 FUNC    GLOBAL DEFAULT    1 call_say_hello
    10: 0000000120000114    40 FUNC    GLOBAL DEFAULT    1 _start
    11: 0000000120010008     0 NOTYPE  GLOBAL DEFAULT    4 __bss_start
    12: 00000001200000e8    44 FUNC    GLOBAL DEFAULT    1 main
    13: 0000000120010008     0 NOTYPE  GLOBAL DEFAULT    4 _edata
    14: 0000000120010008     0 NOTYPE  GLOBAL DEFAULT    4 _end
    15: 0000000120000174    40 FUNC    GLOBAL DEFAULT    1 exit

  ```

#### 5. **查看重定位信息**
```bash
readelf -r <elf文件>
```
- **作用**：显示程序中需要重定位的符号地址（如未定义函数或变量的占位符）。
- **典型场景**：调试链接错误（如 `undefined reference`）。
- 示例：
  ```text
	Relocation section '.rela.text' at offset 0x328 contains 4 entries:
	  Offset          Info           Type           Sym. Value    Sym. Name + Addend
	00000000000c  001000000047 R_LARCH_PCALA_HI2 0000000000000000 ptr + 0
	00000000000c  000000000064 R_LARCH_RELAX                        0
	000000000010  001000000048 R_LARCH_PCALA_LO1 0000000000000000 ptr + 0
	000000000010  000000000064 R_LARCH_RELAX                        0
	
	Relocation section '.rela.data' at offset 0x388 contains 1 entry:
	  Offset          Info           Type           Sym. Value    Sym. Name + Addend
	000000000000  000d00000002 R_LARCH_64        0000000000000000 .LC0 + 0
	
	Relocation section '.rela.eh_frame' at offset 0x3a0 contains 8 entries:
	  Offset          Info           Type           Sym. Value    Sym. Name + Addend
	00000000001c  000600000063 R_LARCH_32_PCREL  0000000000000000 L0^A + 0
	000000000020  000900000032 R_LARCH_ADD32     000000000000003c L0^A + 0
	000000000020  000600000037 R_LARCH_SUB32     0000000000000000 L0^A + 0
	00000000002f  000800000069 R_LARCH_ADD6      0000000000000034 L0^A + 0
	00000000002f  00070000006a R_LARCH_SUB6      000000000000000c L0^A + 0
	00000000003c  000a00000063 R_LARCH_32_PCREL  000000000000003c L0^A + 0
	000000000040  000b00000032 R_LARCH_ADD32     0000000000000064 L0^A + 0
	000000000040  000a00000037 R_LARCH_SUB32     000000000000003c L0^A + 0
  ```

---

### 二、高级用法

下面介绍一些在调试ELF文件和操作系统中常用的命令。

#### 查看程序头表（Segments）
```bash
readelf -l <elf文件>
```
- **输出内容**：程序加载时内存映射的段（如 `LOAD`、`DYNAMIC`）。
- **关键字段**：
  - `Offset`：文件中的偏移量。
  - `Virtual Address`：加载到内存的虚拟地址。
  - `Flags`：权限（`R` 可读、`W` 可写、`X` 可执行）。
- **示例**：
  ```text
	Elf file type is EXEC (Executable file)
	Entry point 0x120000540
	There are 8 program headers, starting at offset 64
	
	Program Headers:
	  Type           Offset             VirtAddr           PhysAddr
	                 FileSiz            MemSiz              Flags  Align
	  PHDR           0x0000000000000040 0x0000000120000040 0x0000000120000040
	                 0x00000000000001c0 0x00000000000001c0  R      0x8
	  INTERP         0x0000000000000200 0x0000000120000200 0x0000000120000200
	                 0x000000000000001e 0x000000000000001e  R      0x1
	      [Requesting program interpreter: /lib/ld-musl-loongarch64.so.1]
	  LOAD           0x0000000000000000 0x0000000120000000 0x0000000120000000
	                 0x000000000000079c 0x000000000000079c  R E    0x10000
	  LOAD           0x000000000000fdf0 0x000000012001fdf0 0x000000012001fdf0
	                 0x0000000000000238 0x0000000000000270  RW     0x10000
	  DYNAMIC        0x000000000000fe00 0x000000012001fe00 0x000000012001fe00
	                 0x00000000000001b0 0x00000000000001b0  RW     0x8
	  GNU_EH_FRAME   0x000000000000074c 0x000000012000074c 0x000000012000074c
	                 0x0000000000000014 0x0000000000000014  R      0x4
	  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
	                 0x0000000000000000 0x0000000000000000  RW     0x10
	  GNU_RELRO      0x000000000000fdf0 0x000000012001fdf0 0x000000012001fdf0
	                 0x0000000000000210 0x0000000000000210  R      0x1
	
	 Section to Segment mapping:
	  Segment Sections...
	   00     
	   01     .interp 
	   02     .interp .hash .gnu.hash .dynsym .dynstr .rela.dyn .rela.plt .plt .text .rodata .eh_frame_hdr .eh_frame 
   	   03     .init_array .fini_array .dynamic .got .got.plt .sdata .bss 
   	   04     .dynamic 
   	   05     .eh_frame_hdr 
   	   06     
   	   07     .init_array .fini_array .dynamic .got
  ```

#### 查看调试信息
```bash
readelf --debug-dump=info <elf文件>
```
- **选项扩展**：
  - `-w`：显示 DWARF 调试信息（如行号、变量位置）。
  - `--dwarf-depth=N`：限制调试信息显示的嵌套深度。
- **适用场景**：逆向工程或调试崩溃问题。

- 示例：
	```text
	readelf --debug-dump=info test_main
	Contents of the .debug_info section:
	
	  Compilation Unit @ offset 0:
	   Length:        0xa6 (32-bit)
	   Version:       5
	   Unit Type:     DW_UT_compile (1)
	   Abbrev Offset: 0
	   Pointer Size:  8
	 <0><c>: Abbrev Number: 4 (DW_TAG_compile_unit)
	    <d>   DW_AT_producer    : (indirect string, offset: 0x25): GNU C23 15.2.0 -mabi=lp64d -march=la64v1.0 -mfpu=64 -msimd=none -mcmodel=normal -mtune=generic -g
	    <11>   DW_AT_language    : 29	(C11)
	    <12>   Unknown AT value: 90: 3
	    <13>   Unknown AT value: 91: 0x31647
	    <17>   DW_AT_name        : (indirect line string, offset: 0x2f): test_main.c
	    <1b>   DW_AT_comp_dir    : (indirect line string, offset: 0): /home/airxs/user/doc/loong64_os_works_doc/test
	    <1f>   DW_AT_low_pc      : 0x120000704
	    <27>   DW_AT_high_pc     : 0x3c
	    <2f>   DW_AT_stmt_list   : 0
	 <1><33>: Abbrev Number: 1 (DW_TAG_base_type)
	    <34>   DW_AT_byte_size   : 8
	    <35>   DW_AT_encoding    : 7	(unsigned)
	    <36>   DW_AT_name        : (indirect string, offset: 0x13): long unsigned int
	 <1><3a>: Abbrev Number: 1 (DW_TAG_base_type)
	    <3b>   DW_AT_byte_size   : 8
	    <3c>   DW_AT_encoding    : 5	(signed)
	    <3d>   DW_AT_name        : (indirect string, offset: 0x5): long int
	 <1><41>: Abbrev Number: 1 (DW_TAG_base_type)
	    <42>   DW_AT_byte_size   : 1
	    <43>   DW_AT_encoding    : 6	(signed char)
	    <44>   DW_AT_name        : (indirect string, offset: 0x87): char
	 <1><48>: Abbrev Number: 5 (DW_TAG_const_type)
	    <49>   DW_AT_type        : <0x41>
	 <1><4d>: Abbrev Number: 1 (DW_TAG_base_type)
	    <4e>   DW_AT_byte_size   : 8
	    <4f>   DW_AT_encoding    : 5	(signed)
	    <50>   DW_AT_name        : (indirect string, offset: 0): long long int
	 <1><54>: Abbrev Number: 1 (DW_TAG_base_type)
	    <55>   DW_AT_byte_size   : 8
	    <56>   DW_AT_encoding    : 4	(float)
	    <57>   DW_AT_name        : (indirect string, offset: 0x8c): double
	 <1><5b>: Abbrev Number: 6 (DW_TAG_subprogram)
	    <5c>   DW_AT_external    : 1
	    <5c>   DW_AT_name        : (indirect string, offset: 0xe): main
	    <60>   DW_AT_decl_file   : 1
	    <61>   DW_AT_decl_line   : 5
	    <62>   DW_AT_decl_column : 5
	    <63>   DW_AT_prototyped  : 1
	    <63>   DW_AT_type        : <0x98>
	    <67>   DW_AT_low_pc      : 0x120000704
	    <6f>   DW_AT_high_pc     : 0x3c
	    <77>   DW_AT_frame_base  : 1 byte block: 9c 	(DW_OP_call_frame_cfa)
	    <79>   DW_AT_call_all_tail_calls: 1
	    <79>   DW_AT_sibling     : <0x98>
	 <2><7d>: Abbrev Number: 2 (DW_TAG_formal_parameter)
	    <7e>   DW_AT_name        : (indirect string, offset: 0x93): argc
	    <82>   DW_AT_decl_file   : 1
	    <82>   DW_AT_decl_line   : 5
	    <82>   DW_AT_decl_column : 14
	    <83>   DW_AT_type        : <0x98>
	    <87>   DW_AT_location    : 2 byte block: 91 6c 	(DW_OP_fbreg: -20)
	 <2><8a>: Abbrev Number: 2 (DW_TAG_formal_parameter)
	    <8b>   DW_AT_name        : (indirect string, offset: 0x98): argv
	    <8f>   DW_AT_decl_file   : 1
	    <8f>   DW_AT_decl_line   : 5
	    <8f>   DW_AT_decl_column : 32
	    <90>   DW_AT_type        : <0x9f>
	    <94>   DW_AT_location    : 2 byte block: 91 60 	(DW_OP_fbreg: -32)
	 <2><97>: Abbrev Number: 0
	 <1><98>: Abbrev Number: 7 (DW_TAG_base_type)
	    <99>   DW_AT_byte_size   : 4
	    <9a>   DW_AT_encoding    : 5	(signed)
	    <9b>   DW_AT_name        : int
	 <1><9f>: Abbrev Number: 3 (DW_TAG_pointer_type)
	    <a0>   DW_AT_byte_size   : 8
	    <a0>   DW_AT_type        : <0xa4>
	 <1><a4>: Abbrev Number: 3 (DW_TAG_pointer_type)
	    <a5>   DW_AT_byte_size   : 8
	    <a5>   DW_AT_type        : <0x48>
	 <1><a9>: Abbrev Number: 0
```

#### 十六进制/字符串转储
```bash
# 十六进制查看指定节区内容
readelf -x .text <elf文件>
```
- **示例**：提取 `.text` 节的机器码：
  ```text
  Hex dump of section '.text':
  0x1200000e8 63c0ff02 6120c029 76000027 7640c002 c...a .)v..'v@..
  0x1200000f8 00440054 00780054 00004003 6120c028 .D.T.x.T..@.a .(
  0x120000108 76000026 6340c002 2000004c 63c0ff02 v..&c@.. ..Lc...
  0x120000118 6120c029 76000027 7640c002 ffc7ff57 a .)v..'v@.....W
  0x120000128 00004003 6120c028 76000026 6340c002 ..@.a .(v..&c@..
  0x120000138 2000004c 63c0ff02 7620c029 7640c002  ..Lc...v .)v@..
  0x120000148 ccf50718 8c010026 0b008103 04048003 .......&........
  0x120000158 85011500 06448003 00002b00 00004003 .....D....+...@.
  0x120000168 7620c028 6340c002 2000004c 63c0ff02 v .(c@.. ..Lc...
  0x120000178 7620c029 7640c002 0b748103 04001500 v .)v@...t......
  0x120000188 00002b00 00004003 7620c028 6340c002 ..+...@.v .(c@..
  0x120000198 2000004c                             ..L

  ```
---

```bash
# 字符串形式查看节区内容
readelf -p .rodata <elf文件>
```
- **示例**：提取 `.rodata` 节的字符串（一般都是字符串）：

  ```text
  shell> readelf -p .rodata hello
  String dump of section '.rodata':
  [     0]  Hello LoongArch!\n
  ```



#### 4. **架构特定信息**
```bash
readelf -A <elf文件>
```
- **输出内容**：CPU 架构扩展信息（如 SIMD 指令支持）。

---

### 三、实战示例
#### 1. **分析可执行文件**
```bash
# 查看入口地址和程序头
readelf -h /bin/ls
readelf -l /bin/ls

# 检查是否包含调试信息
readelf -S /bin/ls | grep debug
```

#### 2. **检查共享库依赖**
```bash
# 查看动态段中的依赖库
readelf -d libc.so.6 | grep NEEDED
```

#### 3. **定位符号地址**
```bash
# 查找函数 `printf` 的地址
readelf -s libc.so.6 | grep printf
```

---

### 四、与其他工具对比
| **工具** | **优势**                          | **局限性**                     |
|----------|----------------------------------|-------------------------------|
| `readelf` | 专注 ELF 结构，不依赖 BFD 库      | 功能较单一（如无反汇编能力）   |
| `objdump` | 支持反汇编、多架构解析            | 依赖 BFD 库，可能受其缺陷影响  |

---


## objdump

`objdump` 是 GNU Binutils 工具集中的核心命令行工具，用于解析和显示二进制文件（如 ELF、COFF 等格式）的详细信息，支持反汇编、符号表分析、段信息查看等功能。以下是其核心用法及典型场景：

---

### 一、基础用法与核心选项
#### 1. **查看文件头信息**
```bash
objdump -f <二进制文件>
```
- **作用**：显示文件格式、入口地址、架构等元信息。
- **示例输出**：
  ```text
  test:     file format elf64-x86-64
  architecture: i386:x86-64, flags 0x00000112:
  EXEC_P, HAS_SYMS, D_PAGED
  start address 0x0000000000401160
  ```

#### 2. **反汇编代码**
```bash
objdump -d <二进制文件>       # 反汇编可执行代码段（.text）
objdump -D <二进制文件>       # 反汇编所有段（包括数据段）
objdump -d -j .text <文件>    # 仅反汇编指定段（如 .text）
```
- **示例输出**：
  ```text
  0000000000401160 <main>:
    401160:       55                      push   %rbp
    401161:       48 89 e5                mov    %rsp,%rbp
    401164:       48 83 ec 10             sub    $0x10,%rsp
  ```

#### 3. **查看符号表**
```bash
objdump -t <二进制文件>    # 静态符号表（类似 nm -s）
objdump -T <二进制文件>    # 动态符号表（仅共享库）
```
- **输出字段**：地址、符号类型（如 `T` 表示代码段符号）、绑定方式（`g` 全局，`l` 局部）。

#### 4. **查看段信息**
```bash
objdump -h <二进制文件>    # 显示节头信息（Size、VMA、Offset 等）
```
- **关键字段**：
  - `Idx`：节索引。
  - `Name`：节名称（如 `.text`, `.data`）。
  - `Size`：节大小（字节）。
  - `VMA`：虚拟内存地址。

#### 5. **显示完整内容**
```bash
objdump -s <二进制文件>    # 以十六进制/ASCII 显示节内容
objdump -s -j .rodata <文件>  # 查看只读数据段（如字符串常量）
```
- **示例**：提取 `.rodata` 中的字符串：
  ```text
  Contents of section .rodata:
    402000 48656c6c6f20576f726c6421    Hello World!
  ```

---

### 二、高级用法
#### 1. **结合源代码反汇编**
```bash
objdump -S <二进制文件>    # 混合显示源代码与汇编指令（需编译时加 -g）
```
- **输出示例**：
  ```text
  0000000000401160 <main>:
  #include <stdio.h>
  int main() {
    401160:       55                      push   %rbp
    401161:       48 89 e5                mov    %rsp,%rbp
    printf("Hello World!\n");
    401164:       48 8d 3d e5 ff ff ff    lea    0xffffffffffffffe5(%rip),%rdi
  }
  ```

#### 2. **指定架构与字节序**
```bash
objdump -d --architecture=arm <ARM二进制文件>  # 分析 ARM 架构代码
objdump -d -EB <文件>          # 指定大端模式（默认小端）
```

#### 3. **重定位信息分析**
```bash
objdump -r <二进制文件>    # 显示重定位条目（如未解析的函数地址）
```
- **输出示例**：
  ```text
  RELOCATION RECORDS FOR [.text]:
  OFFSET   TYPE              VALUE
  00000012 R_X86_64_PC32     add-0x0000000000000004
  ```

#### 4. **动态链接信息**
```bash
objdump -T <共享库>        # 查看动态符号表（如依赖的库函数）
```
- **示例输出**：
  ```text
  DYNAMIC SYMBOL TABLE:
    0000000000000000  w   DF *UND*  0000000000000000  GLIBC_2.2.5 printf
  ```

#### 5. **地址范围过滤**
```bash
objdump -d --start-address=0x400500 --stop-address=0x400520 <文件>
```
- **作用**：仅显示指定地址范围内的反汇编代码。

---

### 三、典型应用场景
#### 1. **逆向工程**
- **分析恶意软件**：通过 `-d` 反汇编可疑程序，定位关键函数。
- **提取 Shellcode**：使用 `-s` 导出 `.text` 段的原始机器码。

#### 2. **编译器调试**
- **验证汇编优化**：对比 `-O0` 和 `-O2` 编译选项生成的汇编代码差异。
- **调试链接错误**：通过 `-r` 检查未解析的重定位符号。

#### 3. **嵌入式开发**
- **验证交叉编译结果**：分析 ARM 目标文件的 `.text` 段是否符合预期。
- **内存布局分析**：通过 `-h` 检查 `.bss` 和 `.data` 段的大小。

#### 4. **安全研究**
- **检测漏洞利用**：通过 `-d` 分析缓冲区溢出攻击的触发点。
- **绕过反调试**：解析混淆代码中的真实指令。

---

### 四、与其他工具对比
| **工具** | **优势**                          | **局限性**                     |
|----------|----------------------------------|-------------------------------|
| `objdump` | 支持多架构、反汇编与符号分析一体化 | 无调试功能（如断点、单步执行） |
| `gdb`    | 动态调试能力强                    | 无法直接查看二进制结构         |
| `readelf`| 专注 ELF 文件解析                 | 不支持反汇编                   |

---

### 五、注意事项
1. **调试信息依赖**：使用 `-S` 或 `-l` 需编译时添加 `-g` 选项。
2. **架构匹配**：分析非 x86 文件时需指定 `--architecture`（如 `arm`、`riscv`）。
3. **性能问题**：对大型文件（如内核镜像）使用 `-D` 可能耗时较长。
4. **权限要求**：分析系统文件（如 `/lib` 下的库）可能需要 `sudo`。

---

### 六、实战示例
#### 示例 1：分析崩溃地址
```bash
objdump -d myprogram | grep -A20 "0x400526"  # 定位崩溃地址附近的代码
```

#### 示例 2：提取加密密钥
```bash
objdump -s -j .data myapp | grep "ENCRYPT_KEY"  # 从数据段提取硬编码密钥
```

#### 示例 3：自动化脚本
```bash
#!/bin/bash
for file in *.so; do
  echo "Checking $file..."
  objdump -T $file | grep "GLIBC_2.28"  # 检测 glibc 版本依赖
done
```

---

通过灵活组合选项，`objdump` 可以成为二进制分析的瑞士军刀。对于开发者，掌握其核心功能能显著提升调试和逆向工程效率。


## objcopy

`objcopy` 是 GNU Binutils 中的核心工具，用于**复制、转换和修改二进制文件**（如 ELF、COFF、SREC 等格式），支持跨格式转换、节区操作、符号处理等功能。以下是其核心用法及典型场景：

---

### 一、基础用法与核心选项
#### 1. **格式转换**
将 ELF 文件转换为纯二进制文件（去除符号和重定位信息）：
```bash
objcopy -O binary input.elf output.bin
```
- **关键选项**：`-O <格式>` 指定输出格式（如 `binary`、`srec`、`elf32-i386`）。
- **应用场景**：生成固件烧录文件或嵌入资源到代码中。

#### 2. **提取或删除节区**
- **提取特定节**（如 `.text`）：
  ```bash
  objcopy -j .text -O binary input.elf output.text.bin
  ```
- **删除冗余节**（如 `.comment` 和 `.note`）：
  ```bash
  objcopy -R .comment -R .note input.elf output.cleaned.elf
  ```
- **适用场景**：减小文件体积或提取关键代码段。

#### 3. **剥离符号信息**
- **完全剥离符号**：
  ```bash
  objcopy -S input.elf output.stripped.elf
  ```
- **保留特定符号**：
  ```bash
  objcopy -K main -K global_var input.elf output.elf
  ```
- **应用场景**：保护代码隐私或优化调试信息。

---

### 二、高级功能
#### 1. **修改节属性**
- **重命名节**：
  ```bash
  objcopy --rename-section .data=.mydata input.elf output.elf
  ```
- **调整段地址**：
  ```bash
  objcopy --change-section-address .text=0x8000 input.elf output.elf
  ```
- **应用场景**：兼容不同硬件地址映射或修复段布局。

#### 2. **符号操作**
- **重命名符号**：
  ```bash
  objcopy --redefine-sym old_func=new_func input.elf output.elf
  ```
- **弱化符号**（避免链接冲突）：
  ```bash
  objcopy --weaken-symbol global_var input.elf output.elf
  ```
- **应用场景**：解决符号命名冲突或适配动态链接库。

#### 3. **填充与对齐**
- **填充段间空隙**：
  ```bash
  objcopy --gap-fill=0xFF input.elf output.elf
  ```
- **填充到指定地址**：
  ```bash
  objcopy --pad-to=0x1000 input.elf output.elf
  ```
- **应用场景**：ROM 编程或内存对齐优化。

#### 4. **处理二进制数据**
- **反转字节序**：
  ```bash
  objcopy --reverse-bytes=4 input.bin output.bin
  ```
- **截取特定字节**：
  ```bash
  objcopy -b 1 -i 4 input.bin output.bin  # 每4字节取第1字节
  ```
- **应用场景**：处理特定硬件协议的二进制数据。

---

### 三、参数详解
| **选项**                | **功能**                                                                 |
|-------------------------|-------------------------------------------------------------------------|
| `-I <格式>`             | 指定输入文件格式（如 `elf64-x86-64`）                                   |
| `-O <格式>`             | 指定输出文件格式（如 `srec`）                                           |
| `-F <格式>`             | 强制输入/输出格式一致（不转换）                                         |
| `-j <节名>`             | 仅处理指定节（等效于 `--only-section`）                                 |
| `-R <节名>`             | 删除指定节                                                              |
| `--set-start <地址>`    | 设置程序入口地址                                                        |
| `--change-section-lma`  | 修改段的加载地址（LMA）                                                 |
| `--strip-unneeded`      | 仅保留重定位需要的符号                                                  |
| `-v`                    | 显示详细操作日志                                                        |

---

### 四、典型应用场景
#### 1. **嵌入式开发**
- **生成二进制固件**：
  ```bash
  arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
  ```
- **提取 Bootloader**：
  ```bash
  objcopy -j .bootloader -O binary input.elf bootloader.bin
  ```

#### 2. **逆向工程**
- **分析可执行文件结构**：
  ```bash
  objcopy -O srec program.exe program.srec  # 转换为 SREC 格式便于分析
  ```
- **修复损坏文件**：
  ```bash
  objcopy --remove-section .corrupted input.exe output.exe
  ```

#### 3. **安全加固**
- **移除调试符号**：
  ```bash
  objcopy -S -g input.elf output.elf
  ```
- **混淆符号名称**：
  ```bash
  objcopy --redefine-sym _start=main input.elf output.elf
  ```

---

### 五、常见问题与解决
#### 问题1：转换后文件无法运行
- **原因**：删除了关键节（如 `.text` 或 `.data`）。
- **解决**：避免使用 `-R` 删除必要节，或通过 `objdump -h` 检查节依赖。

#### 问题2：格式转换失败
- **原因**：输入/输出格式不兼容（如 ELF 转 Windows PE）。
- **解决**：使用中间格式（如 `elf32-littlearm` → `srec` → `hex`）。

#### 问题3：符号重命名无效
- **原因**：未指定全局符号或格式不支持。
- **解决**：使用 `--globalize-symbol` 或检查目标格式是否允许符号修改。

---

### 六、与其他工具对比
| **工具** | **优势**                          | **局限性**                     |
|----------|----------------------------------|-------------------------------|
| `objcopy` | 支持跨格式转换和节级操作          | 无法反汇编或调试               |
| `readelf` | 专注 ELF 文件分析                 | 不支持格式转换                 |
| `strip`   | 仅剥离符号和调试信息              | 功能单一，无法修改节内容       |

---

### 七、总结
`objcopy` 是二进制文件处理的瑞士军刀，适用于：
- **嵌入式开发**：生成固件、提取代码段。
- **安全加固**：移除敏感信息、混淆符号。
- **逆向工程**：分析文件结构、修复损坏文件。

通过灵活组合选项（如 `-O binary -R .comment -S`），开发者可以高效完成二进制文件的转换与优化。


