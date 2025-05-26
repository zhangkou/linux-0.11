# a.out.h 文件详解

`a.out.h` 文件是 Linux 0.11 内核以及早期 Unix 系统中用于定义 **a.out (assembler output)** 可执行文件格式的头文件。这种格式是早期编译器和链接器生成可执行程序时使用的一种标准二进制格式。该头文件主要定义了 `struct exec` 结构体，用于描述 a.out 文件的头部信息，以及一系列相关的宏，用于解析文件头中的字段、计算各段（文本段、数据段等）在文件和内存中的偏移及地址。

尽管现代 Linux 系统主要使用 ELF (Executable and Linkable Format) 格式，但在 Linux 0.11 的时代，a.out 是主要的执行文件格式。

## 核心数据结构

### `struct exec`

这是 `a.out.h` 中最核心的结构体，用于描述 a.out 可执行文件的头部。

```c
struct exec {
  unsigned long a_magic;	/* 魔数，使用宏 N_MAGIC() 等访问 */
  unsigned a_text;		/* 文本段（代码段）的长度，单位字节 */
  unsigned a_data;		/* 初始化数据段的长度，单位字节 */
  unsigned a_bss;		/* 未初始化数据段 (BSS) 的长度，单位字节 */
  unsigned a_syms;		/* 文件中符号表数据的长度，单位字节 */
  unsigned a_entry;		/* 程序执行的入口地址（虚拟地址） */
  unsigned a_trsize;	/* 文本段重定位信息的长度，单位字节 */
  unsigned a_drsize;	/* 数据段重定位信息的长度，单位字节 */
};
```

*   `a_magic`: 一个魔数，用于标识文件的类型。常见的魔数值在下面会介绍。
*   `a_text`: 代码段的大小。
*   `a_data`: 已初始化数据段的大小。这部分数据会直接从可执行文件中加载到内存。
*   `a_bss`: 未初始化数据段的大小。这部分数据在可执行文件中不占空间，但在加载到内存时会分配空间并初始化为零。
*   `a_syms`: 符号表的大小。符号表包含了程序中定义的函数、变量等符号的信息，主要用于调试和链接。
*   `a_entry`: 程序加载到内存后的执行入口点。
*   `a_trsize`: 文本段的重定位信息大小。
*   `a_drsize`: 数据段的重定位信息大小。重定位信息用于在加载时修正代码或数据中的地址引用。

## 重要宏和常量

### 魔数 (Magic Numbers)

用于 `a_magic` 字段，标识可执行文件的具体类型和加载方式。

*   `OMAGIC (0407)`: 表示目标文件或“旧式”的纯可执行文件。文本段和数据段在加载后是连续的，并且数据段是可写的。
*   `NMAGIC (0410)`: 表示纯可执行文件 (NetBSD Magic)。文本段是只读的，数据段紧随其后，并且与文本段在内存中是页对齐的。
*   `ZMAGIC (0413)`: 表示需求分页的可执行文件 (Sun Magic)。这是 Linux 0.11 中主要支持的格式。文本段和数据段都是页对齐的。文本段只读。程序加载时，只有头部被读入，实际的代码和数据页在访问时通过缺页异常按需从磁盘加载。

### 访问和校验魔数的宏

*   `N_MAGIC(exec)`: 从 `struct exec` 变量 `exec` 中提取 `a_magic` 字段的值。
*   `N_BADMAG(x)` 和 `_N_BADMAG(x)`: 检查给定的 `struct exec` 变量 `x` 中的魔数是否是已知的 `OMAGIC`, `NMAGIC`, `ZMAGIC` 之一。如果不是，则认为魔数无效。

### 段偏移和地址计算宏

这些宏用于根据 `struct exec` 头部信息计算文件中各个段（文本、数据、符号表等）的偏移量，以及加载到内存后的虚拟地址。

*   **文件内偏移宏**:
    *   `N_TXTOFF(x)`: 计算文本段在文件中的起始偏移。对于 `ZMAGIC` 格式，由于头部可能占用整个 `SEGMENT_SIZE` (通常是一个块，如1024字节)，所以文本段从 `_N_HDROFF((x)) + sizeof(struct exec)` 开始。对于其他格式，通常紧随 `struct exec` 头部之后，即 `sizeof(struct exec)`。
    *   `N_DATOFF(x)`: 数据段在文件中的起始偏移，即 `N_TXTOFF(x) + x.a_text`。
    *   `N_TRELOFF(x)`: 文本段重定位信息在文件中的起始偏移。
    *   `N_DRELOFF(x)`: 数据段重定位信息在文件中的起始偏移。
    *   `N_SYMOFF(x)`: 符号表在文件中的起始偏移。
    *   `N_STROFF(x)`: 字符串表（用于存储符号名）在文件中的起始偏移，紧随符号表之后。

*   **内存中地址宏**:
    *   `N_TXTADDR(x)`: 文本段加载到内存后的起始虚拟地址。在 Linux 0.11 中，通常为 `0` (用户空间的起始地址)。
    *   `_N_SEGMENT_ROUND(x)`: 将地址 `x` 向上舍入到 `SEGMENT_SIZE` 的整数倍。`SEGMENT_SIZE` 在 Linux 0.11 中被定义为 1024 字节。
    *   `N_DATADDR(x)`: 数据段加载到内存后的起始虚拟地址。
        *   对于 `OMAGIC`，数据段紧随文本段之后：`N_TXTADDR(x) + x.a_text`。
        *   对于 `NMAGIC` 和 `ZMAGIC`，数据段在文本段之后，并向上舍入到下一个 `SEGMENT_SIZE` 边界：`_N_SEGMENT_ROUND(N_TXTADDR(x) + x.a_text)`。
    *   `N_BSSADDR(x)`: BSS 段在内存中的起始虚拟地址，即 `N_DATADDR(x) + x.a_data`。

### `PAGE_SIZE` 和 `SEGMENT_SIZE`

*   `PAGE_SIZE`: 内存页面的大小，在此文件中被定义为 4096 字节（尽管 Minix 通常使用1KB的块，但内存分页可能是4KB）。
*   `SEGMENT_SIZE`: 段对齐的大小。在 Linux 0.11 的 a.out 实现中，它被定义为 1024 字节。这影响了 `ZMAGIC` 格式下文本段的起始偏移和数据段的对齐。

### 符号表项结构 `struct nlist`

定义了符号表中每个条目的格式：

```c
struct nlist {
  union {
    char *n_name;         // 符号名指针 (在链接器内部使用)
    struct nlist *n_next; // 指向下一个符号 (用于哈希表等)
    long n_strx;          // 符号名在字符串表中的偏移量 (文件中的形式)
  } n_un;
  unsigned char n_type;   // 符号类型
  char n_other;           // 通常未使用或其他信息
  short n_desc;           // 描述信息 (通常用于调试)
  unsigned long n_value;  // 符号的值 (通常是地址)
};
```

### 符号类型常量 (部分)

用于 `n_type` 字段，描述符号的类型和属性。

*   `N_UNDF (0)`: 未定义符号。
*   `N_ABS (2)`: 绝对符号 (值是常数，不因重定位改变)。
*   `N_TEXT (4)`: 文本段符号。
*   `N_DATA (6)`: 数据段符号。
*   `N_BSS (8)`: BSS 段符号。
*   `N_FN (15)`: 文件名符号 (用于调试)。
*   `N_EXT (1)`: 外部符号标志 (按位或到类型中，表示符号是全局可见的)。
*   `N_TYPE (036)`: 用于屏蔽类型位的掩码。
*   `N_STAB (0340)`: 调试符号标志位掩码 (表示符号是 stab 调试信息)。

### 重定位信息结构 `struct relocation_info`

描述了一个需要进行重定位的条目。a.out 文件通常包含文本段和数据段的重定位信息区。

```c
struct relocation_info {
  int r_address;            // 需要重定位的地址 (段内偏移)
  unsigned int r_symbolnum:24; // 符号索引 (如果 r_extern=1) 或段类型 (如果 r_extern=0)
  unsigned int r_pcrel:1;     // 是否是 PC 相对寻址
  unsigned int r_length:2;    // 要修改的字段长度 (0->byte, 1->word, 2->long)
  unsigned int r_extern:1;    // 是否是外部符号 (基于符号表) 或基于段的重定位
  unsigned int r_pad:4;       // 未使用，应为0
};
```

## 总结

`a.out.h` 文件为 Linux 0.11 系统提供了解析和加载 a.out 格式可执行文件的标准定义。通过 `struct exec` 结构体和一系列宏，内核能够理解文件的组织结构（如代码段、数据段、BSS段的大小和位置），并正确地将其映射到内存中执行。对 `ZMAGIC` 格式的支持是实现需求分页（按需加载页面）的关键，这有助于提高程序启动速度和内存利用效率。虽然 a.out 格式已被 ELF 取代，但理解 `a.out.h` 有助于了解早期 Linux 内核的程序加载机制和二进制文件接口。Okay, I have created `docs/include/a.out.md` with the detailed Chinese explanation for `include/a.out.h`.

Next, I will process `include/asm/io.h`.
1. Create `docs/include/asm/io.md`.
2. Read `include/asm/io.h`.
3. Write the explanation into `docs/include/asm/io.md`.

First, I'll read `include/asm/io.h`.
