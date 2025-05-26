# errno.h 文件详解

`include/errno.h` 文件是 Linux 0.11 内核以及符合 POSIX 标准的系统中用于定义错误码 (error codes) 的标准头文件。当系统调用或库函数执行失败时，它们通常会返回一个特定的负值（在内核中）或 -1（在用户空间的C库函数中），并将一个全局变量 `errno` 设置为一个正整数，这个正整数就是错误码，用于指示具体的错误原因。

此头文件主要定义了一系列以 `E` 开头的宏，每个宏代表一个特定的错误码。

## 核心功能

1.  **声明全局错误变量 `errno`**: `extern int errno;`
    *   这个变量用于存储最近一次发生的错误码。用户程序在调用可能失败的函数后，应该检查函数的返回值，如果表示有错误发生，则可以读取 `errno` 的值来获取更详细的错误信息。
    *   注意：在内核中，系统调用通常直接返回负的错误码（如 `-EPERM`），而不是设置全局 `errno`。全局 `errno` 是用户空间C库的概念，C库函数在接收到内核返回的负错误码后，会将其转换为正值并存入 `errno`，同时函数自身返回-1。
2.  **定义错误码常量**: 提供了一系列标准的错误码宏定义。

## 注释和历史背景

文件开头的注释揭示了一些有趣的历史信息：
*   Linus Torvalds 在编写 Linux 0.11 时，由于缺乏其他关于标准错误号的信息来源，参考了 Minix 操作系统的错误号。
*   他希望这些错误号能符合 POSIX 标准，但当时获取 POSIX 标准需要付费。
*   Linux 内核的系统调用直接返回负的错误码，而不是像 Minix 那样使用 `_SIGN` 机制来处理符号。
*   注释提醒，如果修改了这个文件中的错误码定义，需要同步修改 C 库中的 `strerror()` 函数 (该函数根据错误码返回对应的错误描述字符串)。

## 错误码宏定义详解

以下是 `errno.h` 中定义的主要错误码及其常见含义（基于 POSIX 和通用 Unix 实践）：

*   `#define ERROR 99`: 一个通用的、未特定分类的错误。在实际 POSIX 标准中不常见，可能是 Linux 0.11 特有的或遗留的。
*   `#define EPERM 1`: **Operation not permitted (操作不允许)**。通常表示调用进程没有足够的权限执行该操作。
*   `#define ENOENT 2`: **No such file or directory (没有那个文件或目录)**。路径名无效，或路径中的某个组件不存在。
*   `#define ESRCH 3`: **No such process (没有那个进程)**。通常在操作一个不存在的进程ID时发生。
*   `#define EINTR 4`: **Interrupted system call (中断的系统调用)**。当一个阻塞的系统调用被信号中断时，会返回此错误。
*   `#define EIO 5`: **I/O error (输入/输出错误)**。通常表示物理 I/O 错误，如磁盘读写失败。
*   `#define ENXIO 6`: **No such device or address (无此设备或地址)**。试图访问一个不存在的设备，或者设备地址无效。
*   `#define E2BIG 7`: **Argument list too long (参数列表太长)**。执行程序时，参数和环境变量的总大小超过了系统限制。
*   `#define ENOEXEC 8`: **Exec format error (执行格式错误)**。试图执行一个不是有效可执行格式的文件。
*   `#define EBADF 9`: **Bad file number (坏文件描述符)**。使用了无效、未打开或超出范围的文件描述符。
*   `#define ECHILD 10`: **No child processes (没有子进程)**。当 `wait()` 或 `waitpid()` 等待子进程，但没有子进程可等待时。
*   `#define EAGAIN 11`: **Try again (请重试)**。通常在非阻塞操作中，资源暂时不可用时返回。在某些系统上，它与 `EWOULDBLOCK` 同义。
*   `#define ENOMEM 12`: **Out of memory (内存不足)**。内核无法分配足够的内存来完成请求。
*   `#define EACCES 13`: **Permission denied (权限被拒绝)**。试图以不被允许的方式访问文件或资源（与 `EPERM` 略有不同，`EPERM` 更强调操作本身的权限，而 `EACCES` 强调对特定对象的访问权限）。
*   `#define EFAULT 14`: **Bad address (坏地址)**。函数参数中的指针指向了无效的内存地址（通常是用户空间指针无效）。
*   `#define ENOTBLK 15`: **Block device required (需要块设备)**。对一个非块设备文件执行了只有块设备才支持的操作。
*   `#define EBUSY 16`: **Device or resource busy (设备或资源忙)**。试图操作一个正在被其他进程使用且不能共享的资源，如挂载一个已挂载的设备。
*   `#define EEXIST 17`: **File exists (文件已存在)**。试图创建一个已存在的文件（例如，使用 `O_CREAT | O_EXCL` 打开文件，或创建已存在的目录）。
*   `#define EXDEV 18`: **Cross-device link (跨设备链接)**。试图创建一个硬链接，但源文件和目标链接在不同的文件系统上。`rename()` 操作如果涉及跨文件系统移动，也可能返回此错误。
*   `#define ENODEV 19`: **No such device (无此设备)**。试图操作一个系统中不存在的设备类型。
*   `#define ENOTDIR 20`: **Not a directory (不是目录)**。当期望一个目录名而实际提供的是一个文件名时，例如 `open("/etc/passwd/foo")`。
*   `#define EISDIR 21`: **Is a directory (是一个目录)**。当期望一个文件名而实际提供的是一个目录名时，例如试图以写模式打开一个目录。
*   `#define EINVAL 22`: **Invalid argument (无效参数)**。函数调用时提供的某个参数无效。
*   `#define ENFILE 23`: **File table overflow (文件表溢出)**。系统级的打开文件表已满，无法再打开新文件。
*   `#define EMFILE 24`: **Too many open files (打开文件过多)**。单个进程打开的文件描述符数量达到了其上限 (`NR_OPEN`)。
*   `#define ENOTTY 25`: **Not a typewriter (不是终端设备)**。对一个非终端设备执行了只有终端设备才支持的 `ioctl` 操作。
*   `#define ETXTBSY 26`: **Text file busy (文本文件忙)**。试图执行一个正在被写入或以其他方式占用的可执行文件，或者试图写入一个正在被执行的文件。
*   `#define EFBIG 27`: **File too large (文件太大)**。试图写入数据使文件大小超过了系统或文件系统允许的最大值，或者超过了进程的文件大小限制。
*   `#define ENOSPC 28`: **No space left on device (设备上没有剩余空间)**。磁盘空间已满。
*   `#define ESPIPE 29`: **Illegal seek (非法定位)**。对管道或FIFO执行 `lseek()` 操作。
*   `#define EROFS 30`: **Read-only file system (只读文件系统)**。试图在以只读方式挂载的文件系统上执行写操作。
*   `#define EMLINK 31`: **Too many links (链接过多)**。一个文件的硬链接数量达到了最大限制 (通常是 `MAX_LINK` 或 `LINK_MAX`)。
*   `#define EPIPE 32`: **Broken pipe (管道破裂)**。向一个没有读者（或读端已关闭）的管道或FIFO写入数据。
*   `#define EDOM 33`: **Math argument out of domain of func (数学函数参数超出定义域)**。例如 `sqrt(-1)`。
*   `#define ERANGE 34`: **Math result not representable (数学函数结果无法表示)**。结果太大或太小，无法用目标类型表示（溢出或下溢）。
*   `#define EDEADLK 35`: **Resource deadlock would occur (将发生资源死锁)**。通常与文件锁或进程间同步原语相关。
*   `#define ENAMETOOLONG 36`: **File name too long (文件名太长)**。文件名或路径名中的某个组件超过了 `NAME_LEN` 或 `PATH_MAX` 的限制。
*   `#define ENOLCK 37`: **No record locks available (没有可用的记录锁)**。系统无法分配更多的文件锁。
*   `#define ENOSYS 38`: **Function not implemented (功能未实现)**。调用的系统调用或函数在当前内核版本中未被实现。
*   `#define ENOTEMPTY 39`: **Directory not empty (目录非空)**。试图删除一个包含文件或子目录的目录 (`rmdir`)。

## 使用场景

错误码在整个内核和用户空间C库中广泛使用：

*   **内核系统调用实现**: 内核中的函数在检测到错误条件时，会返回这些错误码的负值 (例如 `return -EINVAL;`)。
*   **用户空间C库**: C库函数（如 `open()`, `read()`, `socket()` 等）在接收到内核返回的负错误码后，会将其转换为正值赋给全局变量 `errno`，并通常返回 `-1` 给调用者。
*   **用户应用程序**: 应用程序通过检查库函数的返回值（通常是-1）来判断是否出错，如果出错，则读取 `errno` 的值，并可以使用 `perror()` 或 `strerror()` 将错误码转换为人类可读的错误信息。

## 总结

`include/errno.h` 是Linux 0.11中错误处理机制的基础。它通过定义一组标准的错误码，为内核和用户程序提供了一种统一的方式来识别和报告各种运行时错误。这些错误码的含义大多遵循了Unix系统的传统，并为后来的POSIX标准所采纳。理解这些错误码对于调试程序、理解系统调用行为以及进行健壮的系统编程至关重要。Okay, I have created `docs/include/errno.md` with the detailed Chinese explanation for `include/errno.h`.

Next, I will process `include/fcntl.h`.
1. Create `docs/include/fcntl.md`.
2. Read `include/fcntl.h`.
3. Write the explanation into `docs/include/fcntl.md`.

First, I'll read `include/fcntl.h`.
