# 一些常用的库函数

## GLIBC

GLIBC（GNU C Library）是 GNU 项目为 Linux 系统开发的 C 标准库实现，是 Linux 生态中**最核心的系统运行库**，几乎所有应用程序和系统工具都直接或间接依赖它。以下是其核心解析：

---

### 一、核心功能与作用
1. **系统调用封装**  
   - 提供对 Linux 内核的系统调用接口（如文件操作 `open()`、进程控制 `fork()`），屏蔽底层硬件差异。
   - 实现 POSIX 标准接口（如线程 `pthread_create()`），确保跨 UNIX 系统兼容性。

2. **基础功能模块**  
   - **内存管理**：`malloc()`/`free()` 动态内存分配，支持多线程安全优化。
   - **字符串处理**：`strcpy()`、`strlen()` 等函数，优化字符串操作效率。
   - **国际化支持**：通过 `iconv` 实现字符集转换，支持多语言环境。

3. **动态链接支持**  
   - 作为动态链接器 `ld-linux.so` 的核心依赖，管理共享库加载（如 `libc.so.6`）。
   - 提供 `dlopen()`/`dlsym()` 接口，支持运行时动态加载插件或库。

---

### 二、架构与实现细节
1. **代码结构**  
   - **头文件**：位于 `/usr/include`，定义函数原型和宏（如 `stdio.h`）。
   - **静态库与动态库**：  
     - 静态库 `.a`：用于不依赖外部共享库的编译场景。  
     - 动态库 `.so`：如 `libc.so.6`，通过 `ldconfig` 管理版本和路径。

2. **线程与进程管理**  
   - 封装 Linux 线程（`pthread`）接口，支持多线程同步（互斥锁、条件变量）。
   - 实现进程间通信（IPC）机制，如管道、信号量和共享内存。

3. **安全与兼容性**  
   - 通过 `mprotect()` 实现内存保护，防止非法访问。
   - 兼容旧版 Linux 系统调用（如 `syscall()` 的封装），确保二进制向后兼容。

---

### 三、版本演进与升级风险
1. **版本历史**  
   - **libc5**：早期版本，不支持 ELF 共享库，国际化能力弱。
   - **glibc 2.x**：引入 ELF 支持、POSIX 兼容性，成为 Linux 主流标准。
   - **现代版本**（如 2.38）：优化性能（如内存分配算法）、增强安全（如堆溢出防护）。

2. **升级风险**  
   - **二进制兼容性**：新版本可能破坏旧程序的 ABI（应用二进制接口），导致崩溃。
   - **系统稳定性**：直接替换系统默认 glibc 可能引发段错误（如 `libc.so.6` 软链接错误）。
   - **解决方案**：  
     - 使用容器（Docker）隔离不同 glibc 版本。  
     - 通过 `patchelf` 修改程序的动态链接器路径，指向特定版本。

---

### 四、与其他运行库的对比
| **库**       | **特点**                                                                 | **适用场景**               |
|--------------|-------------------------------------------------------------------------|--------------------------|
| **glibc**    | GNU 标准实现，功能全面，兼容 POSIX，Linux 系统默认                        | 通用 Linux 应用开发        |
| **musl**     | 轻量级、静态链接友好，符合 ISO C 标准，兼容性较差                        | 嵌入式系统、Alpine Linux   |
| **MSVCRT**   | Windows 下的 C 运行库，依赖 Windows API                                  | Windows 应用开发           |

---

### 五、开发与调试工具
1. **调试工具**  
   - **gdb**：结合 glibc 的调试符号（`.debug` 文件），分析内存泄漏或崩溃。
   - **valgrind**：检测内存非法访问（需 glibc 的 `malloc` 钩子支持）。

2. **多版本管理**  
   - **glibc-all-in-one**：一键下载、编译多版本 glibc，支持版本共存。
   - **LD_LIBRARY_PATH**：临时指定程序使用的 glibc 库路径，避免污染系统环境。

---


## MUSL

### Musl C 库详解

#### 一、核心概念与架构
**Musl libc** 是 GNU 项目之外的轻量级 C 标准库实现，专注于 **高性能、低内存占用和代码简洁性**，遵循 ISO C 和 POSIX 标准。其核心特点包括：
1. **轻量化设计**  
   - 静态链接时最小仅需 **10KB**，动态链接后约 **50KB**，远低于 glibc 的数 MB 级体积。  
   - 无动态内存分配（如 `malloc` 需自行实现或通过第三方库），减少运行时开销。
2. **模块化架构**  
   - 分离基础库（`libc`）、数学库（`libm`）和动态链接器（`libdl`），实际在 OpenHarmony 中通过符号链接整合为单一库。  
   - 支持多架构（ARM64、x86_64、LoongArch 等），适配嵌入式设备和服务器场景。
3. **安全增强**  
   - 默认启用地址空间布局随机化（ASLR）和堆内存混淆（`mallocng`），防御缓冲区溢出等漏洞。  
   - 符号版本控制（Symbol Versioning）确保接口兼容性。

#### 二、与 glibc 的关键差异
| **特性**         | **Musl**                              | **glibc**                          |
|------------------|---------------------------------------|------------------------------------|
| **体积**         | 静态链接 10-50KB，动态 50KB+          | 静态链接 2MB+，动态 10MB+          |
| **许可证**       | MIT（商业友好）                       | LGPL（限制闭源分发）               |
| **兼容性**       | 部分 POSIX 扩展不支持（如 `gethostbyname`） | 完全兼容 Linux 系统调用和传统接口  |
| **线程模型**     | 独立线程栈（默认 80KB），无历史包袱   | 依赖 NPTL，兼容旧版线程库          |
| **动态加载**     | `dlopen` 无法卸载库（`dlclose` 无效） | 支持动态卸载和延迟绑定（Lazy Binding） |

#### 三、核心功能与实现
1. **标准库支持**  
   - **C11/C17 兼容**：完整实现线程、原子操作、Unicode 处理（`char16_t`/`char32_t`）。  
   - **数学函数优化**：使用硬件指令（如 SSE/AVX）加速浮点运算，避免 glibc 的软浮点回退。
2. **动态链接器（`ld-musl`）**  
   - **扁平化命名空间**：所有动态库直接加载到进程地址空间，避免 glibc 的 `rpath` 和 `LD_LIBRARY_PATH` 复杂性。  
   - **符号解析策略**：优先使用最新符号版本，避免旧版接口污染。
3. **国际化支持**  
   - 默认启用 `C.UTF-8` 区域设置，严格遵循 Unicode 标准，拒绝无效 UTF-8 序列。  
   - 通过 `iconv` 实现字符集转换，但仅支持有限编码（如 UTF-8/UCS-2，不支持 GBK）。

#### 四、应用场景与最佳实践
1. **嵌入式系统**  
   - **鸿蒙 OS**：作为默认 C 库，支持高安全性和低内存设备（如智能手表、IoT 终端）。  
   - **Alpine Linux**：替代 glibc 减少镜像体积（Docker 镜像可减小 50% 以上）。
2. **容器与云原生**  
   - 静态编译二进制文件，避免依赖宿主机的动态库，提升部署一致性。  
   - 结合 `musl-gcc` 工具链交叉编译，优化边缘计算节点性能。
3. **安全关键系统**  
   - 内存保护机制（如堆元数据混淆）降低漏洞利用风险，适用于金融、军工领域。

#### 五、开发与调试
1. **交叉编译**  
   - 使用 `musl-gcc` 工具链（如 `aarch64-linux-musl-gcc`）编译目标平台代码。  
   - 示例命令：  
     ```bash
     loongarch64-linux-musl-gcc -static hello.c -o hello
     ```
2. **调试工具**  
   - **动态日志**：通过 `param set musl.log.enable true` 开启运行时日志，定位内存分配或加载器异常。  
   - **Valgrind 替代**：由于 musl 无 `malloc` 钩子，需使用专用工具（如 `musl-valgrind`）检测内存泄漏。

#### 六、兼容性解决方案
1. **glibc 兼容层**  
   - 使用 `glibc-compat` 库提供 `pthread`、`dlfcn` 等接口的兼容实现。  
   - 示例：在 Alpine Linux 中安装 `glibc` 包以运行依赖 glibc 的软件。
2. **接口适配**  
   - 对不支持的函数（如 `gethostbyname`）封装自定义实现，或切换至 `musl` 兼容的替代库（如 `musl-nss`）。

#### 七、性能优化技巧
1. **编译优化**  
   - 启用 `-fno-stack-protector` 减少栈保护开销（需权衡安全性）。  
   - 使用 `-flto`（链接时优化）提升代码密度。
2. **内存管理**  
   - 预分配线程栈空间（`pthread_attr_setstacksize`），避免动态扩展导致的性能波动。  
   - 禁用不必要的 locale 支持（通过 `--disable-locale` 配置），减少初始化时间。

#### 八、生态与社区
- **开源协议**：MIT License，允许闭源商用，无专利限制。  
- **社区支持**：GitHub 仓库（https://github.com/bminor/musl）活跃，但企业级支持较少。  
- **工具链集成**：主流编译器（GCC、Clang）均提供 musl 交叉编译支持，但需手动配置。

---

### 总结
Musl C 库凭借轻量化、高安全性和模块化设计，成为嵌入式系统和容器化应用的首选。其与 glibc 的差异主要体现在兼容性、性能和资源占用上。开发者需根据场景权衡选择：**追求极致性能或资源受限时选 musl，需广泛兼容性时选 glibc**。对于混合环境，可通过兼容层或接口适配实现平滑过渡。



## memcpy

内存拷贝函数 memcpy ``void *memcpy(void *dst, const void *src, size_t n)``

**下面是C语言编写**：
```c
void *memcpy(void *restrict dest, const void *restrict src, size_t n)
{
	unsigned char *d = dest;
	const unsigned char *s = src;
	
	for (; n; n--) 
		*d++ = *s++;
	return dest;
}
```
-------------



**下面是优化的C语言编写**：
```c
void *memcpy(void *restrict dest, const void *restrict src, size_t n)
{
	unsigned char *d = dest;
	const unsigned char *s = src;

	typedef uint32_t __attribute__((__may_alias__)) u32;
	uint32_t w, x;

	for (; (uintptr_t)s % 4 && n; n--) *d++ = *s++;

	if ((uintptr_t)d % 4 == 0) {
		for (; n>=16; s+=16, d+=16, n-=16) {
			*(u32 *)(d+0) = *(u32 *)(s+0);
			*(u32 *)(d+4) = *(u32 *)(s+4);
			*(u32 *)(d+8) = *(u32 *)(s+8);
			*(u32 *)(d+12) = *(u32 *)(s+12);
		}
		if (n&8) {
			*(u32 *)(d+0) = *(u32 *)(s+0);
			*(u32 *)(d+4) = *(u32 *)(s+4);
			d += 8; s += 8;
		}
		if (n&4) {
			*(u32 *)(d+0) = *(u32 *)(s+0);
			d += 4; s += 4;
		}
		if (n&2) {
			*d++ = *s++; *d++ = *s++;
		}
		if (n&1) {
			*d = *s;
		}
		return dest;
	}

	if (n >= 32) switch ((uintptr_t)d % 4) {
	case 1:
		w = *(u32 *)s;
		*d++ = *s++;
		*d++ = *s++;
		*d++ = *s++;
		n -= 3;
		for (; n>=17; s+=16, d+=16, n-=16) {
			x = *(u32 *)(s+1);
			*(u32 *)(d+0) = (w >> 24) | (x << 8);
			w = *(u32 *)(s+5);
			*(u32 *)(d+4) = (x >> 24) | (w << 8);
			x = *(u32 *)(s+9);
			*(u32 *)(d+8) = (w >> 24) | (x << 8);
			w = *(u32 *)(s+13);
			*(u32 *)(d+12) = (x >> 24) | (w << 8);
		}
		break;
	case 2:
		w = *(u32 *)s;
		*d++ = *s++;
		*d++ = *s++;
		n -= 2;
		for (; n>=18; s+=16, d+=16, n-=16) {
			x = *(u32 *)(s+2);
			*(u32 *)(d+0) = (w >> 16) | (x << 16);
			w = *(u32 *)(s+6);
			*(u32 *)(d+4) = (x >> 16) | (w << 16);
			x = *(u32 *)(s+10);
			*(u32 *)(d+8) = (w >> 16) | (x << 16);
			w = *(u32 *)(s+14);
			*(u32 *)(d+12) = (x >> 16) | (w << 16);
		}
		break;
	case 3:
		w = *(u32 *)s;
		*d++ = *s++;
		n -= 1;
		for (; n>=19; s+=16, d+=16, n-=16) {
			x = *(u32 *)(s+3);
			*(u32 *)(d+0) = (w >> 8) | (x << 24);
			w = *(u32 *)(s+7);
			*(u32 *)(d+4) = (x >> 8) | (w << 24);
			x = *(u32 *)(s+11);
			*(u32 *)(d+8) = (w >> 8) | (x << 24);
			w = *(u32 *)(s+15);
			*(u32 *)(d+12) = (x >> 8) | (w << 24);
		}
		break;
	}
	if (n&16) {
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
	}
	if (n&8) {
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
	}
	if (n&4) {
		*d++ = *s++; *d++ = *s++; *d++ = *s++; *d++ = *s++;
	}
	if (n&2) {
		*d++ = *s++; *d++ = *s++;
	}
	if (n&1) {
		*d = *s;
	}
	return dest;
}
```
-------------


**下面是基于LoongArch的汇编指令编写**：
```c
// 定义
#define SYM_FUNC_START(name) \
    .globl name;   \
    name: 

#define SYM_FUNC_END(name)

SYM_FUNC_START(memcpy)
	/*
	 * Some CPUs support hardware unaligned access
	 */
	// 常用型内存拷贝
	b __memcpy_generic
	
	// 快速的拷贝，注意需要硬件支持非对齐访问
	b __memcpy_fast
SYM_FUNC_END(memcpy)

/*
 * void *__memcpy_generic(void *dst, const void *src, size_t n)
 *
 * a0: dst
 * a1: src
 * a2: n
 */
SYM_FUNC_START(__memcpy_generic)
	move	$a3, $a0
	beqz	$a2, 2f

1:	ld.b	$t0, $a1, 0
	st.b	$t0, $a0, 0
	addi.d	$a0, $a0, 1
	addi.d	$a1, $a1, 1
	addi.d	$a2, $a2, -1
	bgt	    $a2, $zero, 1b

2:	move	$a0, $a3
	jr	$ra
SYM_FUNC_END(__memcpy_generic)

	.align	5
SYM_FUNC_START_NOALIGN(__memcpy_small)
	pcaddi	$t0, 8
	slli.d	$a2, $a2, 5
	add.d	$t0, $t0, $a2
	jr	$t0

	.align	5
0:	jr	$ra

	.align	5
1:	ld.b	$t0, $a1, 0
	st.b	$t0, $a0, 0
	jr	$ra

	.align	5
2:	ld.h	$t0, $a1, 0
	st.h	$t0, $a0, 0
	jr	$ra

	.align	5
3:	ld.h	$t0, $a1, 0
	ld.b	$t1, $a1, 2
	st.h	$t0, $a0, 0
	st.b	$t1, $a0, 2
	jr	$ra

	.align	5
4:	ld.w	$t0, $a1, 0
	st.w	$t0, $a0, 0
	jr	$ra

	.align	5
5:	ld.w	$t0, $a1, 0
	ld.b	$t1, $a1, 4
	st.w	$t0, $a0, 0
	st.b	$t1, $a0, 4
	jr	$ra

	.align	5
6:	ld.w	$t0, $a1, 0
	ld.h	$t1, $a1, 4
	st.w	$t0, $a0, 0
	st.h	$t1, $a0, 4
	jr	$ra

	.align	5
7:	ld.w	$t0, $a1, 0
	ld.w	$t1, $a1, 3
	st.w	$t0, $a0, 0
	st.w	$t1, $a0, 3
	jr	$ra

	.align	5
8:	ld.d	$t0, $a1, 0
	st.d	$t0, $a0, 0
	jr	$ra
SYM_FUNC_END(__memcpy_small)

/*
 * void *__memcpy_fast(void *dst, const void *src, size_t n)
 *
 * a0: dst
 * a1: src
 * a2: n
 */
SYM_FUNC_START(__memcpy_fast)
	sltui	$t0, $a2, 9
	bnez	$t0, __memcpy_small

	add.d	$a3, $a1, $a2
	add.d	$a2, $a0, $a2
	ld.d	$a6, $a1, 0
	ld.d	$a7, $a3, -8

	/* align up destination address */
	andi	$t1, $a0, 7
	sub.d	$t0, $zero, $t1
	addi.d	$t0, $t0, 8
	add.d	$a1, $a1, $t0
	add.d	$a5, $a0, $t0

	addi.d	$a4, $a3, -64
	bgeu	$a1, $a4, .Llt64

	/* copy 64 bytes at a time */
.Lloop64:
	ld.d	$t0, $a1, 0
	ld.d	$t1, $a1, 8
	ld.d	$t2, $a1, 16
	ld.d	$t3, $a1, 24
	ld.d	$t4, $a1, 32
	ld.d	$t5, $a1, 40
	ld.d	$t6, $a1, 48
	ld.d	$t7, $a1, 56
	addi.d	$a1, $a1, 64
	st.d	$t0, $a5, 0
	st.d	$t1, $a5, 8
	st.d	$t2, $a5, 16
	st.d	$t3, $a5, 24
	st.d	$t4, $a5, 32
	st.d	$t5, $a5, 40
	st.d	$t6, $a5, 48
	st.d	$t7, $a5, 56
	addi.d	$a5, $a5, 64
	bltu	$a1, $a4, .Lloop64

	/* copy the remaining bytes */
.Llt64:
	addi.d	$a4, $a3, -32
	bgeu	$a1, $a4, .Llt32
	ld.d	$t0, $a1, 0
	ld.d	$t1, $a1, 8
	ld.d	$t2, $a1, 16
	ld.d	$t3, $a1, 24
	addi.d	$a1, $a1, 32
	st.d	$t0, $a5, 0
	st.d	$t1, $a5, 8
	st.d	$t2, $a5, 16
	st.d	$t3, $a5, 24
	addi.d	$a5, $a5, 32

.Llt32:
	addi.d	$a4, $a3, -16
	bgeu	$a1, $a4, .Llt16
	ld.d	$t0, $a1, 0
	ld.d	$t1, $a1, 8
	addi.d	$a1, $a1, 16
	st.d	$t0, $a5, 0
	st.d	$t1, $a5, 8
	addi.d	$a5, $a5, 16

.Llt16:
	addi.d	$a4, $a3, -8
	bgeu	$a1, $a4, .Llt8
	ld.d	$t0, $a1, 0
	st.d	$t0, $a5, 0

.Llt8:
	st.d	$a6, $a0, 0
	st.d	$a7, $a2, -8

	/* return */
	jr	$ra
SYM_FUNC_END(__memcpy_fast)
```
-------------

## memmove
内存移动 ``void *memmove(void *dest, const void *src, size_t n)``

**下面是基础C语言编写**：
```c
void *memmove(void *dest, const void *src, size_t n)
{
	char *d = dest;
	const char *s = src;
	for (; n; n--) 
		*d++ = *s++;
	return dest;
}

```
-------------


**下面是C语言优化编写**：
```c
void *memmove(void *dest, const void *src, size_t n)
{
	char *d = dest;
	const char *s = src;

	if (d==s) return d;
	if ((uintptr_t)s-(uintptr_t)d-n <= -2*n) return memcpy(d, s, n);

	if (d<s) {
#ifdef __GNUC__
		if ((uintptr_t)s % sizeof(size_t) == (uintptr_t)d % sizeof(size_t)) {
			while ((uintptr_t)d % sizeof(size_t)) {
				if (!n--) return dest;
				*d++ = *s++;
			}
			for (; n>=sizeof(size_t); n-=sizeof(size_t), d+=sizeof(size_t), s+=sizeof(size_t)) *(WT *)d = *(WT *)s;
		}
#endif
		for (; n; n--) *d++ = *s++;
	} else {
#ifdef __GNUC__
		if ((uintptr_t)s % sizeof(size_t) == (uintptr_t)d % sizeof(size_t)) {
			while ((uintptr_t)(d+n) % sizeof(size_t)) {
				if (!n--) return dest;
				d[n] = s[n];
			}
			while (n>=sizeof(size_t)) n-=sizeof(size_t), *(WT *)(d+n) = *(WT *)(s+n);
		}
#endif
		while (n) n--, d[n] = s[n];
	}

	return dest;
}

```
-------------

**下面是基于LoongArch的汇编指令编写**：

```c
// 定义
#define SYM_FUNC_START(name) \
    .globl name;   \
    name: 

#define SYM_FUNC_END(name)


SYM_FUNC_START(memmove)
	blt	$a0, $a1, __memcpy	/* dst < src, memcpy */
	blt	$a1, $a0, __rmemcpy	/* src < dst, rmemcpy */
	jr	$ra			/* dst == src, return */
SYM_FUNC_END(memmove)

SYM_FUNC_START(__rmemcpy)
	/*
	 * Some CPUs support hardware unaligned access
	 */

	// 常用型内存拷贝
	b __rmemcpy_generic
	
	// 快速的拷贝，注意需要硬件支持非对齐访问
	b __rmemcpy_fast

SYM_FUNC_END(__rmemcpy)

/*
 * void *__rmemcpy_generic(void *dst, const void *src, size_t n)
 *
 * a0: dst
 * a1: src
 * a2: n
 */
SYM_FUNC_START(__rmemcpy_generic)
	move	$a3, $a0
	beqz	$a2, 2f

	add.d	$a0, $a0, $a2
	add.d	$a1, $a1, $a2

1:	ld.b	$t0, $a1, -1
	st.b	$t0, $a0, -1
	addi.d	$a0, $a0, -1
	addi.d	$a1, $a1, -1
	addi.d	$a2, $a2, -1
	bgt	    $a2, $zero, 1b

2:	move	$a0, $a3
	jr	$ra
SYM_FUNC_END(__rmemcpy_generic)

/*
 * void *__rmemcpy_fast(void *dst, const void *src, size_t n)
 *
 * a0: dst
 * a1: src
 * a2: n
 */
SYM_FUNC_START(__rmemcpy_fast)
	sltui	$t0, $a2, 9
	bnez	$t0, __memcpy_small

	add.d	$a3, $a1, $a2
	add.d	$a2, $a0, $a2
	ld.d	$a6, $a1, 0
	ld.d	$a7, $a3, -8

	/* align up destination address */
	andi	$t1, $a2, 7
	sub.d	$a3, $a3, $t1
	sub.d	$a5, $a2, $t1

	addi.d	$a4, $a1, 64
	bgeu	$a4, $a3, .Llt64

	/* copy 64 bytes at a time */
.Lloop64:
	ld.d	$t0, $a3, -8
	ld.d	$t1, $a3, -16
	ld.d	$t2, $a3, -24
	ld.d	$t3, $a3, -32
	ld.d	$t4, $a3, -40
	ld.d	$t5, $a3, -48
	ld.d	$t6, $a3, -56
	ld.d	$t7, $a3, -64
	addi.d	$a3, $a3, -64
	st.d	$t0, $a5, -8
	st.d	$t1, $a5, -16
	st.d	$t2, $a5, -24
	st.d	$t3, $a5, -32
	st.d	$t4, $a5, -40
	st.d	$t5, $a5, -48
	st.d	$t6, $a5, -56
	st.d	$t7, $a5, -64
	addi.d	$a5, $a5, -64
	bltu	$a4, $a3, .Lloop64

	/* copy the remaining bytes */
.Llt64:
	addi.d	$a4, $a1, 32
	bgeu	$a4, $a3, .Llt32
	ld.d	$t0, $a3, -8
	ld.d	$t1, $a3, -16
	ld.d	$t2, $a3, -24
	ld.d	$t3, $a3, -32
	addi.d	$a3, $a3, -32
	st.d	$t0, $a5, -8
	st.d	$t1, $a5, -16
	st.d	$t2, $a5, -24
	st.d	$t3, $a5, -32
	addi.d	$a5, $a5, -32

.Llt32:
	addi.d	$a4, $a1, 16
	bgeu	$a4, $a3, .Llt16
	ld.d	$t0, $a3, -8
	ld.d	$t1, $a3, -16
	addi.d	$a3, $a3, -16
	st.d	$t0, $a5, -8
	st.d	$t1, $a5, -16
	addi.d	$a5, $a5, -16

.Llt16:
	addi.d	$a4, $a1, 8
	bgeu	$a4, $a3, .Llt8
	ld.d	$t0, $a3, -8
	st.d	$t0, $a5, -8

.Llt8:
	st.d	$a6, $a0, 0
	st.d	$a7, $a2, -8

	/* return */
	jr	$ra
SYM_FUNC_END(__rmemcpy_fast)
```
-------------





## memset

将内存范围全部使用特定的值填充。

**下面是基础C语言编写**：
```c
void *memset(void *dest, int c, size_t n)
{
	unsigned char *s = dest;

	for (; n; n--, s++) 
		*s = c;
	return dest;
}
```
-------------

**下面是C语言优化编写**：
```c
void *memset(void *dest, int c, size_t n)
{
	unsigned char *s = dest;
	size_t k;

	/* Fill head and tail with minimal branching. Each
	 * conditional ensures that all the subsequently used
	 * offsets are well-defined and in the dest region. 
	 */

	if (!n) return dest;
	s[0] = c;
	s[n-1] = c;
	if (n <= 2) return dest;
	s[1] = c;
	s[2] = c;
	s[n-2] = c;
	s[n-3] = c;
	if (n <= 6) return dest;
	s[3] = c;
	s[n-4] = c;
	if (n <= 8) return dest;

	/* Advance pointer to align it at a 4-byte boundary,
	 * and truncate n to a multiple of 4. The previous code
	 * already took care of any head/tail that get cut off
	 * by the alignment. */

	k = -(uintptr_t)s & 3;
	s += k;
	n -= k;
	n &= -4;

	typedef uint32_t __attribute__((__may_alias__)) u32;
	typedef uint64_t __attribute__((__may_alias__)) u64;

	u32 c32 = ((u32)-1)/255 * (unsigned char)c;

	/* In preparation to copy 32 bytes at a time, aligned on
	 * an 8-byte bounary, fill head/tail up to 28 bytes each.
	 * As in the initial byte-based head/tail fill, each
	 * conditional below ensures that the subsequent offsets
	 * are valid (e.g. !(n<=24) implies n>=28). */

	*(u32 *)(s+0) = c32;
	*(u32 *)(s+n-4) = c32;
	if (n <= 8) return dest;
	*(u32 *)(s+4) = c32;
	*(u32 *)(s+8) = c32;
	*(u32 *)(s+n-12) = c32;
	*(u32 *)(s+n-8) = c32;
	if (n <= 24) return dest;
	*(u32 *)(s+12) = c32;
	*(u32 *)(s+16) = c32;
	*(u32 *)(s+20) = c32;
	*(u32 *)(s+24) = c32;
	*(u32 *)(s+n-28) = c32;
	*(u32 *)(s+n-24) = c32;
	*(u32 *)(s+n-20) = c32;
	*(u32 *)(s+n-16) = c32;

	/* Align to a multiple of 8 so we can fill 64 bits at a time,
	 * and avoid writing the same bytes twice as much as is
	 * practical without introducing additional branching. */

	k = 24 + ((uintptr_t)s & 4);
	s += k;
	n -= k;

	/* If this loop is reached, 28 tail bytes have already been
	 * filled, so any remainder when n drops below 32 can be
	 * safely ignored. */

	u64 c64 = c32 | ((u64)c32 << 32);
	for (; n >= 32; n-=32, s+=32) {
		*(u64 *)(s+0) = c64;
		*(u64 *)(s+8) = c64;
		*(u64 *)(s+16) = c64;
		*(u64 *)(s+24) = c64;
	}
	return dest;
}
```
-------------


**下面是基于LoongArch的汇编指令编写**：

```c
void *memset(void *dest, int c, size_t n)
```

```c
// 定义
#define SYM_FUNC_START(name) \
    .globl name;   \
    name: 

#define SYM_FUNC_END(name)


SYM_FUNC_START(memset)
	/*
	 * Some CPUs support hardware unaligned access
	 */

	// 常用型内存设置
	b __memset_generic
	
	// 快速的设置，注意需要硬件支持非对齐访问
	b __memset_fast
SYM_FUNC_END(memset)

/*
 * void *__memset_generic(void *s, int c, size_t n)
 *
 * a0: s
 * a1: c
 * a2: n
 */
SYM_FUNC_START(__memset_generic)
	move	$a3, $a0
	beqz	$a2, 2f

1:	st.b	$a1, $a0, 0
	addi.d	$a0, $a0, 1
	addi.d	$a2, $a2, -1
	bgt	    $a2, $zero, 1b

2:	move	$a0, $a3
	jr	$ra
SYM_FUNC_END(__memset_generic)

/*
 * void *__memset_fast(void *s, int c, size_t n)
 *
 * a0: s
 * a1: c
 * a2: n
 */
SYM_FUNC_START(__memset_fast)
	/* fill a1 to 64 bits */
	bstrins.d $a1, $a1, 15, 8
	bstrins.d $a1, $a1, 31, 16
	bstrins.d $a1, $a1, 63, 32

	sltui	$t0, $a2, 9
	bnez	$t0, .Lsmall

	add.d	$a2, $a0, $a2
	st.d	$a1, $a0, 0

	/* align up address */
	addi.d	$a3, $a0, 8
	bstrins.d	$a3, $zero, 2, 0

	addi.d	$a4, $a2, -64
	bgeu	$a3, $a4, .Llt64

	/* set 64 bytes at a time */
.Lloop64:
	st.d	$a1, $a3, 0
	st.d	$a1, $a3, 8
	st.d	$a1, $a3, 16
	st.d	$a1, $a3, 24
	st.d	$a1, $a3, 32
	st.d	$a1, $a3, 40
	st.d	$a1, $a3, 48
	st.d	$a1, $a3, 56
	addi.d	$a3, $a3, 64
	bltu	$a3, $a4, .Lloop64

	/* set the remaining bytes */
.Llt64:
	addi.d	$a4, $a2, -32
	bgeu	$a3, $a4, .Llt32
	st.d	$a1, $a3, 0
	st.d	$a1, $a3, 8
	st.d	$a1, $a3, 16
	st.d	$a1, $a3, 24
	addi.d	$a3, $a3, 32

.Llt32:
	addi.d	$a4, $a2, -16
	bgeu	$a3, $a4, .Llt16
	st.d	$a1, $a3, 0
	st.d	$a1, $a3, 8
	addi.d	$a3, $a3, 16

.Llt16:
	addi.d	$a4, $a2, -8
	bgeu	$a3, $a4, .Llt8
	st.d	$a1, $a3, 0

.Llt8:
	st.d	$a1, $a2, -8

	/* return */
	jr	$ra

	.align	4
.Lsmall:
	pcaddi	$t0, 4
	slli.d	$a2, $a2, 4
	add.d	$t0, $t0, $a2
	jr	$t0

	.align	4
0:	jr	$ra

	.align	4
1:	st.b	$a1, $a0, 0
	jr	$ra

	.align	4
2:	st.h	$a1, $a0, 0
	jr	$ra

	.align	4
3:	st.h	$a1, $a0, 0
	st.b	$a1, $a0, 2
	jr	$ra

	.align	4
4:	st.w	$a1, $a0, 0
	jr	$ra

	.align	4
5:	st.w	$a1, $a0, 0
	st.b	$a1, $a0, 4
	jr	$ra

	.align	4
6:	st.w	$a1, $a0, 0
	st.h	$a1, $a0, 4
	jr	$ra

	.align	4
7:	st.w	$a1, $a0, 0
	st.w	$a1, $a0, 3
	jr	$ra

	.align	4
8:	st.d	$a1, $a0, 0
	jr	$ra
SYM_FUNC_END(__memset_fast)
```
-------------


以上的函数都给出具体的例子与说明，或者参考的代码等等

可以结合向量指令操作。




