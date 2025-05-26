# string.h 文件详解

`include/string.h` 文件是 Linux 0.11 内核以及标准 C 语言环境中用于定义字符串处理和内存区域操作函数的头文件。这些函数提供了复制、连接、比较、查找字符、计算长度等基本操作。

在 Linux 0.11 的这个版本中，一个显著的特点是这些字符串函数几乎都是通过**内联汇编 (inline assembly)** 实现的，并且经过了高度优化，以期达到最高的执行效率。Linus Torvalds 在注释中提到，这些实现直接在寄存器中完成操作，并广泛使用了 x86 的串操作指令，这使得代码“略显晦涩”，但快速且简洁。

## 核心功能

1.  **定义 `NULL` 和 `size_t`**: 如果尚未定义，则提供这些标准定义。
2.  **声明错误信息函数**: `strerror()`。
3.  **实现字符串操作函数 (内联汇编)**:
    *   复制: `strcpy`, `strncpy`
    *   连接: `strcat`, `strncat`
    *   比较: `strcmp`, `strncmp`
    *   查找字符: `strchr`, `strrchr`
    *   计算跨度: `strspn`, `strcspn` (计算一个字符串中第一个不在指定字符集内的字符的索引)
    *   查找子串: `strpbrk` (查找一个字符串中第一个出现在指定字符集内的字符), `strstr` (查找子串)
    *   计算长度: `strlen`
    *   分割字符串: `strtok` (注意：`strtok` 是不可重入的，因为它使用了一个静态内部变量 `___strtok`)
4.  **实现内存区域操作函数 (内联汇编)**:
    *   复制: `memcpy`, `memmove`
    *   比较: `memcmp`
    *   查找字符: `memchr`
    *   填充: `memset`

## 注释和实现特点

Linus Torvalds 的注释强调了以下几点：
*   **GCC 内联函数**: 所有字符串函数都定义为内联函数，依赖 GCC 编译器进行展开。
*   **段寄存器假设**: 假设 `ds=es=data space`，即数据段和附加段指向同一个普通数据区。这对于内核代码（`ds`和`es`指向内核数据空间）和行为正确的用户程序（`ds`和`es`指向用户数据空间）都是成立的。
*   **手动优化**: 函数经过了大量的手动优化，使用了寄存器和串指令。
*   **代码可读性**: 由于优化和直接使用汇编，代码可能不易理解。

## 宏和类型定义详解

### `NULL` 和 `size_t`

```c
#ifndef NULL
#define NULL ((void *) 0)
#endif

#ifndef _SIZE_T
#define _SIZE_T
typedef unsigned int size_t; // 注意：在 stddef.h 中通常是 unsigned long
#endif
```
*   如果 `NULL` 未定义，则定义为空指针常量 `(void *)0`。
*   如果 `_SIZE_T` 未定义（通常通过 `stddef.h` 定义），则将 `size_t` 定义为 `unsigned int`。这与 `stddef.h` 中通常将其定义为 `unsigned long` 有所不同，可能是特定于早期 Linux 0.11 内核或其编译环境的简化。`size_t` 用于表示对象的大小。

### `extern char * strerror(int errno);`

*   **功能**: 声明 `strerror` 函数，该函数根据错误码 `errno` 返回一个指向描述该错误的字符串的指针。其实现通常在C库中。

## 内联汇编函数详解 (部分示例)

这些函数的实现都大量使用了 x86 的串操作指令和寄存器。

### `extern inline char * strcpy(char * dest, const char *src)`

*   **功能**: 将源字符串 `src` (包括结尾的 `\0`) 复制到目标缓冲区 `dest`。
*   **汇编指令**:
    *   `cld`: 清除方向标志，使 `lodsb` 和 `stosb` 从低地址向高地址操作。
    *   `1: lodsb`: 从 `ds:[esi]` (src) 加载一个字节到 `al`，并增加 `esi`。
    *   `stosb`: 将 `al` 中的字节存储到 `es:[edi]` (dest)，并增加 `edi`。
    *   `testb %%al, %%al`: 测试刚复制的字节 `al` 是否为0 (即 `\0`)。
    *   `jne 1b`: 如果不是0，则循环回到标签 `1:` 继续复制下一个字节。
*   **约束**: `"S" (src)` (源地址放入 `esi`), `"D" (dest)` (目标地址放入 `edi`)。
*   **破坏寄存器**: `si`, `di`, `ax`。

### `extern inline int strcmp(const char * cs, const char * ct)`

*   **功能**: 比较两个字符串 `cs` 和 `ct`。
*   **返回值**:
    *   0: 如果 `cs == ct`。
    *   1: 如果 `cs > ct` (按字典序)。
    *   -1: 如果 `cs < ct`。
*   **汇编指令**:
    *   `cld`: 清方向。
    *   `1: lodsb`: 从 `cs` (`esi`) 加载字节到 `al`。
    *   `scasb`: 比较 `al` 和 `es:[edi]` (`ct`) 指向的字节，并根据结果设置标志位，增加 `edi`。
    *   `jne 2f`: 如果字节不相等，跳转到 `2f`。
    *   `testb %%al, %%al`: 测试刚比较的字节（来自 `cs`）是否为0。
    *   `jne 1b`: 如果不为0且之前相等，则继续比较下一个字节。
    *   `xorl %%eax, %%eax; jmp 3f`: 如果到达字符串末尾且所有字符都相等，则将 `eax` (返回值)设为0。
    *   `2: movl $1, %%eax; jl 3f; negl %%eax;`: 如果不相等，将 `eax` 设为1。如果 `cs` 中字符小于 `ct` 中字符 (通过 `jl` 判断，`scasb` 后 `SF!=OF`)，则 `eax` 保持1 (实际应为-1，`negl`后)。如果 `cs` 中字符大于 `ct` 中字符，`jl` 不跳转，`negl %%eax` 将1变为-1 (实际应为1)。这里的逻辑似乎与标准 `strcmp` 返回值约定（`cs > ct` 返回正，`cs < ct` 返回负）有些出入，需要仔细分析 `SF` 和 `OF` 标志与 `jl` 的关系。通常 `jl` (jump if less) 是在有符号比较后使用，`scasb` 的比较结果会影响标志位，进而影响 `jl`。
        *   更正理解：`scasb` 比较 `AL` 和 `ES:[DI]`。如果 `AL < ES:[DI]`，则 `CF=1` (如果 `SUB AL, ES:[DI]` 理解)。`jne` 后，如果不相等，则 `AL` 存储的是 `cs` 的字符，`ES:[DI-1]` 是 `ct` 的字符。比较 `AL` 和 `ES:[DI-1]`。
        *   Linux 0.11 `strcmp` 实际逻辑: 若 `cs` 字符 < `ct` 字符，eax = -1。若 `cs` 字符 > `ct` 字符，eax = 1。
*   **约束**: `"D" (cs)` (cs放入`edi`), `"S" (ct)` (ct放入`esi`)。注意这里 `D` 和 `S` 与 `strcpy` 的用法相反，`scasb` 是 `AL - ES:[EDI]`。
*   **破坏寄存器**: `si`, `di`。`ax` 是返回值。

### `extern inline int strlen(const char * s)`

*   **功能**: 计算字符串 `s` 的长度 (不包括结尾的 `\0`)。
*   **汇编指令**:
    *   `cld`: 清方向。
    *   `repne scasb`: (Repeat while Not Equal, Scan String Byte) 重复扫描 `es:[edi]` (s) 指向的字符串，每次将字节与 `al` (预设为0) 比较，直到找到0字节或者 `ecx` 减为0。`edi` 每次递增。
    *   `notl %0`: `ecx` 初始为 `0xffffffff` (-1)。`repne scasb` 结束后，`ecx` 的值是 `-1 - length`。`notl` 对 `ecx` 按位取反，结果是 `length`。
    *   `decl %0`: `ecx` 再减1，得到 `length - 1`。但由于 `scasb` 会多扫描一个 `\0`，所以 `not ecx` 后是 `len+1`，再 `dec` 后是 `len`。
*   **约束**: `"D" (s)` (s放入`edi`), `"a" (0)` (`al`设为0), `"0" (0xffffffff)` (`ecx` 初始为-1，这里`"0"`表示与第一个输出操作数相同的约束，但用于输入，即`ecx`也作为输入)。
*   **破坏寄存器**: `di`。`cx` 是返回值。

### `extern inline void * memcpy(void * dest, const void * src, int n)`

*   **功能**: 从源地址 `src` 复制 `n` 个字节到目标地址 `dest`。与 `asm/memory.h` 中的版本类似。
*   **汇编指令**:
    *   `cld`: 清方向。
    *   `rep movsb`: 重复 `n` (`ecx`) 次，将 `ds:[esi]` 的字节复制到 `es:[edi]`，并递增 `esi`, `edi`。
*   **约束**: `"c" (n)` (n放入`ecx`), `"S" (src)` (src放入`esi`), `"D" (dest)` (dest放入`edi`)。
*   **破坏寄存器**: `cx`, `si`, `di`。

### `extern inline void * memmove(void * dest, const void * src, int n)`

*   **功能**: `memcpy` 的安全版本，能够正确处理源和目标区域重叠的情况。
*   **实现**:
    *   `if (dest < src)`: 如果目标地址在源地址之前（或者无重叠且从低到高复制安全），则使用与 `memcpy` 相同的前向复制 (`cld; rep; movsb`)。
    *   `else`: 如果目标地址在源地址之后且区域可能重叠，则需要从后向前复制以避免数据被覆盖。
        *   `std`: 设置方向标志，使串操作从高地址向低地址进行。
        *   `rep movsb`: 此时 `esi` 应指向 `src+n-1`，`edi` 指向 `dest+n-1`。
        *   约束中 ` "S" (src+n-1), "D" (dest+n-1)` 正是为此设置的。
*   **注意**: `std` 执行后方向标志会保持设置，如果后续有其他串操作依赖 `cld`，需要显式清除。不过，由于这些是内联函数，其影响通常局限于函数本身。

### `extern inline void * memset(void * s, char c, int count)`

*   **功能**: 将内存区域 `s` 的前 `count` 个字节设置为字符 `c`。
*   **汇编指令**:
    *   `cld`: 清方向。
    *   `rep stosb`: 重复 `count` (`ecx`) 次，将 `al` (字符 `c`) 存入 `es:[edi]` (s)，并递增 `edi`。
*   **约束**: `"a" (c)` (c放入`al`), `"D" (s)` (s放入`edi`), `"c" (count)` (count放入`ecx`)。
*   **破坏寄存器**: `cx`, `di`。

## `___strtok` 变量

*   `extern char * ___strtok;`
*   这是为 `strtok` 函数使用的静态（或全局，此处声明为外部）指针。`strtok` 在首次调用时会保存传入的字符串指针，在后续调用（第一个参数为 `NULL`）时会从这个保存的指针处继续查找。这使得 `strtok` 是不可重入的，即不能在多线程环境中安全使用，也不能在同一时间对多个不同字符串进行交错的 `strtok` 操作。

## 总结

`include/string.h` 在 Linux 0.11 中提供了一套高度优化的字符串和内存操作函数。通过直接使用内联汇编和x86的串处理指令，这些函数旨在提供最佳性能。虽然代码因此变得较为底层和平台相关，但在当时追求极致效率的内核代码中，这种做法是常见的。理解这些函数的汇编实现有助于深入了解底层的数据操作方式，同时也提醒我们早期内核开发中对性能的极致追求以及与硬件的紧密结合。这些函数的接口（参数和返回值）大多遵循了标准C库的约定。
