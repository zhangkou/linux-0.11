# unistd.h 文件详解

`include/unistd.h` 文件是 Linux 0.11 内核以及遵循 POSIX (Portable Operating System Interface) 标准的Unix-like系统中一个非常核心且广泛使用的头文件。它的名字来源于 "Unix Standard"。该文件定义了大量与系统调用接口、POSIX标准符号常量、以及一些基本库函数原型相关的宏和声明。

`unistd.h` 可以看作是用户程序访问操作系统内核功能的**主要入口点清单**。它不仅声明了许多系统调用的C函数包装器原型，还定义了这些调用可能用到的各种常量（如文件描述符、lseek的定位方式、access的权限检查模式等）和POSIX特性宏。

## 核心功能

1.  **定义POSIX版本和特性宏**:
    *   `_POSIX_VERSION`: 指示所遵循的POSIX标准版本。
    *   其他 `_POSIX_...` 宏: 指示系统是否支持某些POSIX可选特性 (如作业控制、chown权限限制等)。
2.  **定义标准文件描述符**: `STDIN_FILENO`, `STDOUT_FILENO`, `STDERR_FILENO`。
3.  **定义 `NULL`**: 如果尚未定义。
4.  **定义常用常量**:
    *   `access()` 函数的模式常量: `F_OK`, `X_OK`, `W_OK`, `R_OK`。
    *   `lseek()` 函数的 `whence` (定位方式) 常量: `SEEK_SET`, `SEEK_CUR`, `SEEK_END`。
    *   `sysconf()` 和 `pathconf()` 使用的参数名常量 (尽管这些函数本身在0.11中可能未完全实现或未使用)。
5.  **包含其他相关头文件**: 如 `<sys/stat.h>`, `<sys/times.h>` 等，这些头文件定义了 `unistd.h` 中声明的某些函数所需的结构。
6.  **定义系统调用号 (`__NR_xxx`)**: 这一部分通常被 `__LIBRARY__` 宏条件编译，用于内核或C库内部，将每个系统调用映射到一个唯一的整数编号。
7.  **定义系统调用宏 (`_syscall0`, `_syscall1`, `_syscall2`, `_syscall3`)**: 这一部分也通常在 `__LIBRARY__` 宏下，用于C库中方便地生成调用 `int 0x80` 中断以执行系统调用的汇编代码存根。
8.  **声明大量系统调用和标准库函数的原型**: 这是该文件的主要部分，列出了用户程序可以调用的标准函数接口。

## 宏和常量定义详解

### POSIX 版本和特性宏

*   `#define _POSIX_VERSION 198808L`: 表示遵循1988年8月版的POSIX标准。这是一个占位符，表明了早期Linux对POSIX兼容性的努力。
*   `#define _POSIX_CHOWN_RESTRICTED`: (注释说明) 表示 `chown()` 操作通常受限，只有root用户可以更改文件所有者。
*   `#define _POSIX_NO_TRUNC`: (注释说明) 表示路径名不应被截断（但内核中可能有实际的长度限制 `NAME_LEN`）。
*   `#define _POSIX_VDISABLE '\0'`: 定义了用于禁用 `termios` 中特殊控制字符的值 (通常是 `\0`)。
*   `/*#define _POSIX_SAVED_IDS */`: (注释掉) 表示保存的用户ID和组ID特性可能尚未完全支持。
*   `/*#define _POSIX_JOB_CONTROL */`: (注释掉) 表示作业控制功能尚未完全实现。

### 标准文件描述符

*   `#define STDIN_FILENO 0`: 标准输入的文件描述符。
*   `#define STDOUT_FILENO 1`: 标准输出的文件描述符。
*   `#define STDERR_FILENO 2`: 标准错误输出的文件描述符。

### `access()` 模式常量

用于 `access()` 系统调用的 `mode` 参数，检查文件的可访问性。
*   `#define F_OK 0`: 检查文件是否存在。
*   `#define X_OK 1`: 检查文件是否可执行。
*   `#define W_OK 2`: 检查文件是否可写。
*   `#define R_OK 4`: 检查文件是否可读。
    (可以按位或组合使用，例如 `R_OK | W_OK`)

### `lseek()` 定位方式常量

用于 `lseek()` 系统调用的 `origin` (或 `whence`) 参数。
*   `#define SEEK_SET 0`: 从文件开头定位。
*   `#define SEEK_CUR 1`: 从当前文件指针位置定位。
*   `#define SEEK_END 2`: 从文件末尾定位。

### 系统配置常量 (`_SC_xxx`) 和路径配置常量 (`_PC_xxx`)

这些常量用于 `sysconf()` 和 `pathconf()` 函数（这两个函数在Linux 0.11中可能未实现或不常用），用于查询系统或文件的可配置限制和特性。
*   例如: `_SC_ARG_MAX` (参数列表最大长度), `_SC_OPEN_MAX` (每进程最大打开文件数), `_PC_PATH_MAX` (路径最大长度), `_PC_NAME_MAX` (文件名最大长度)。

### 系统调用号 (`__NR_xxx`)

这部分被 `#ifdef __LIBRARY__` 包围，通常在构建C库时使用。
*   为每个系统调用定义了一个唯一的编号，例如 `#define __NR_exit 1`, `#define __NR_fork 2`。
*   这些编号在用户程序通过 `int 0x80` 中断发起系统调用时，通过 `eax` 寄存器传递给内核，内核使用此编号在 `sys_call_table` (定义在 `<linux/sys.h>`) 中查找并调用相应的内核处理函数。
*   Linux 0.11总共定义了 `__NR_setregid` (71) 为止的72个系统调用 (从0到71)。

### 系统调用宏 (`_syscall0` 到 `_syscall3`)

这部分也通常在 `__LIBRARY__` 宏下，用于C库。这些宏是用来生成实际发起系统调用的汇编代码的便捷方式。它们根据系统调用参数的数量（0到3个）来定义。

*   **示例 (`_syscall1`)**:
    ```c
    #define _syscall1(type,name,atype,a) \
    type name(atype a) \
    { \
    long __res; \
    __asm__ volatile ("int $0x80" \
        : "=a" (__res) \
        : "0" (__NR_##name),"b" ((long)(a))); \
    if (__res >= 0) \
        return (type) __res; \
    errno = -__res; \
    return -1; \
    }
    ```
    *   `type name(atype a)`: 定义了一个名为 `name` 的函数，它接受一个类型为 `atype` 的参数 `a`，返回类型为 `type`。
    *   `long __res;`: 用于存储系统调用的原始返回值 (在 `eax` 中)。
    *   `__asm__ volatile ("int $0x80" ...)`: 内联汇编，执行 `int $0x80` 中断。
        *   `"=a" (__res)`: 输出约束，将 `eax` (中断后的返回值) 赋给 `__res`。
        *   `"0" (__NR_##name)`: 输入约束，将系统调用号 `__NR_name` (通过 `##` 预处理器连接) 放入 `eax` (因为 `"0"` 表示与第一个输出操作数使用相同的寄存器)。
        *   `"b" ((long)(a))`: 输入约束，将第一个参数 `a` 放入 `ebx` 寄存器。
        *   类似地，`_syscall2` 会将第二个参数放入 `ecx`，`_syscall3` 会将第三个参数放入 `edx`。
    *   **返回值处理**:
        *   如果 `__res >= 0`，表示系统调用成功，返回 `(type)__res`。
        *   如果 `__res < 0`，表示系统调用失败，内核返回的是负的错误码 (如 `-EINVAL`)。此时，将 `errno` (全局错误变量) 设置为 `-__res` (即正的错误码)，并返回 `-1` (这是C库函数的标准错误返回方式)。

## 函数原型声明

文件末尾声明了大量的用户空间函数原型，这些函数是POSIX标准或Unix系统常用的接口。它们中的许多是系统调用的C库包装器。

例如：
*   `int access(const char * filename, mode_t mode);`
*   `int alarm(int sec);`
*   `int chdir(const char * filename);`
*   `int close(int fildes);`
*   `int execve(const char * filename, char ** argv, char ** envp);` (以及其他 `exec` 变体)
*   `volatile void exit(int status);`
*   `int fork(void);`
*   `int getpid(void);`
*   `int kill(pid_t pid, int signal);`
*   `int open(const char * filename, int flag, ...);`
*   `int read(int fildes, char * buf, off_t count);`
*   `int write(int fildes, const char * buf, off_t count);`
*   `pid_t wait(int * wait_stat);`
*   等等。

## `extern int errno;`

*   声明了全局错误变量 `errno`，用于在系统调用或库函数出错时存储错误码。

## 总结

`include/unistd.h` 是连接用户空间应用程序和 Linux 0.11 内核功能的关键桥梁。它定义了进行系统编程所需的各种标准常量、系统调用编号（供C库使用）以及大量系统调用和标准库函数的原型。通过包含此头文件，用户程序可以方便地调用操作系统提供的服务，如文件操作、进程管理、权限控制等。`_syscallX` 宏系列展示了C库如何将高级语言的函数调用转换为底层的 `int 0x80` 中断请求，是理解用户态到内核态切换和系统调用实现机制的重要部分。这个文件体现了早期Linux对POSIX标准的遵循和兼容性努力。
