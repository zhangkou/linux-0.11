# init/main.c 文件详解

`init/main.c` 文件是 Linux 0.11 内核的入口点和初始化流程的核心。在系统引导加载程序（如 `boot/bootsect.s` 和 `boot/setup.s`）完成硬件探测、内存布局确定并将内核映像加载到内存后，控制权最终会转移到 `init/main.c` 中的 `main()` 函数。这个文件负责执行一系列关键的初始化步骤，为多任务环境的运行和用户程序的执行做好准备，并最终创建init进程（进程0和进程1）。

## 核心功能

1.  **内核初始化序列的起点**: `main()` 函数是内核C代码的最高层和起始执行点。
2.  **内存管理初始化**: 根据 `setup.s` 探测到的内存大小，确定内核、缓冲区、主内存区域的划分，并调用 `mem_init()` 初始化物理内存管理。
3.  **设备驱动和子系统初始化**: 调用各个模块的初始化函数，如陷阱处理 (`trap_init`)、块设备 (`blk_dev_init`)、字符设备 (`chr_dev_init`)、TTY (`tty_init`)、硬盘 (`hd_init`)、软盘 (`floppy_init`) 等。
4.  **时间初始化**: `time_init()` 从CMOS实时时钟读取当前时间并设置系统启动时间 `startup_time`。
5.  **调度器初始化**: `sched_init()` 初始化任务数组、GDT中的TSS和LDT描述符等。
6.  **缓冲区初始化**: `buffer_init()` 初始化缓冲区高速缓存。
7.  **切换到用户模式**: `move_to_user_mode()` 将CPU从内核态切换到用户态，为创建第一个用户进程做准备。
8.  **创建init进程**:
    *   通过 `fork()` 创建进程1 (init进程)。
    *   进程0 (内核初始线程) 在完成初始化后，进入一个无限的 `pause()` 循环，成为空闲 (idle) 进程。
    *   进程1 (`init()`) 负责进一步的用户空间初始化，如执行 `/etc/rc` 脚本和启动登录shell。

## 内联系统调用宏

文件开头定义了一些内联的系统调用宏：
```c
static inline _syscall0(int,fork)
static inline _syscall0(int,pause)
static inline _syscall1(int,setup,void *,BIOS)
static inline _syscall0(int,sync)
```
*   **原因**: 注释中解释，为了避免在 `main()` 函数 `fork()` 之后使用栈（因为内核空间 `fork` 不会写时复制栈，直到 `execve`），关键的系统调用如 `fork` 和 `pause` 需要内联实现，以防止函数调用和返回时栈被弄乱。
*   **`_syscallX` 宏**: 这些宏在 `<unistd.h>` 中定义，用于通过 `int 0x80` 中断发起系统调用。

## 全局变量和外部函数

*   **`static char printbuf[1024];`**: 一个静态的打印缓冲区，供内核的 `printf` 函数使用。
*   **外部函数声明**: 声明了大量在其他内核模块中定义的初始化函数（如 `vsprintf`, `init`, `blk_dev_init` 等）和 `kernel_mktime` (用于将 `struct tm` 时间转换为 `time_t`)。
*   **`startup_time`**: 全局变量，存储系统启动时的Unix时间戳。

## 硬件信息宏

这些宏用于从内存特定地址（由 `boot/setup.s` 填充）读取硬件信息：
*   `#define EXT_MEM_K (*(unsigned short *)0x90002)`: 从地址 `0x90002` 读取扩展内存的大小 (KB)。
*   `#define DRIVE_INFO (*(struct drive_info *)0x90080)`: 从地址 `0x90080` 读取驱动器参数信息。`struct drive_info` 在此文件中仅简单定义为 `char dummy[32]`，实际结构可能在 `setup.s` 中更重要，或者这里的定义只是为了类型转换。
*   `#define ORIG_ROOT_DEV (*(unsigned short *)0x901FC)`: 从地址 `0x901FC` 读取原始的根设备号。

## `time_init()` 函数

*   **功能**: 初始化系统时间 `startup_time`。
*   **步骤**:
    1.  通过 `CMOS_READ(addr)` 宏从CMOS实时时钟 (RTC) 读取年、月、日、时、分、秒。
        *   `CMOS_READ` 宏通过向端口 `0x70` 输出地址，然后从端口 `0x71` 读取数据来实现。`0x80` 用于禁止NMI。
        *   循环读取直到两次读取的秒数相同，以避免跨秒读取导致数据不一致。
    2.  使用 `BCD_TO_BIN(val)` 宏将从CMOS读取到的BCD码转换为二进制数。
    3.  `time.tm_mon--`: 将月份从1-12调整为0-11 (C语言 `struct tm` 的约定)。
    4.  `startup_time = kernel_mktime(&time);`: 调用 `kernel_mktime` (在 `kernel/mktime.c` 中实现) 将 `struct tm` 结构转换为自Epoch以来的秒数，并存入全局变量 `startup_time`。

## `main(void)` 函数详解

这是内核的C语言入口点。注释强调 `void` 返回类型是正确的，因为启动例程就是这样假设的。

1.  **中断仍被禁止**: 此时CPU中断仍处于关闭状态。
2.  **获取硬件信息**:
    *   `ROOT_DEV = ORIG_ROOT_DEV;`: 设置全局根设备号。
    *   `drive_info = DRIVE_INFO;`: 复制驱动器信息。
3.  **内存大小确定与划分**:
    *   `memory_end = (1<<20) + (EXT_MEM_K<<10);`: 计算总物理内存大小。`1MB + (扩展内存KB * 1024)`。
    *   `memory_end &= 0xfffff000;`: 将总内存大小向下对齐到4KB页边界。
    *   `if (memory_end > 16*1024*1024) memory_end = 16*1024*1024;`: 限制最大可用内存为16MB (Linux 0.11的限制)。
    *   根据总内存大小，确定用于高速缓冲区的内存大小 `buffer_memory_end`：
        *   如果总内存 > 12MB，缓冲区为 4MB。
        *   如果总内存 > 6MB，缓冲区为 2MB。
        *   否则，缓冲区为 1MB。
    *   `main_memory_start = buffer_memory_end;`: 主内存（用于页表、进程等）的起始地址在缓冲区之后。
4.  **RAMDISK 初始化 (可选)**:
    *   `#ifdef RAMDISK main_memory_start += rd_init(main_memory_start, RAMDISK*1024); #endif`: 如果定义了 `RAMDISK` 宏 (通常在Makefile中配置)，则调用 `rd_init()` 初始化虚拟盘，并调整主内存起始地址。
5.  **内存管理初始化**:
    *   `mem_init(main_memory_start, memory_end);`: 初始化物理内存页管理，标记哪些页面可用。
6.  **核心模块初始化**:
    *   `trap_init();`: 初始化陷阱门和中断门 (IDT的设置，在 `kernel/traps.c`)。
    *   `blk_dev_init();`: 块设备请求队列初始化 (在 `kernel/blk_drv/ll_rw_blk.c`)。
    *   `chr_dev_init();`: 字符设备初始化 (可能为空或简单占位，具体驱动在后续)。
    *   `tty_init();`: TTY子系统初始化 (在 `drivers/char/tty_io.c`)。
    *   `time_init();`: 初始化系统时间 `startup_time` (如上所述)。
    *   `sched_init();`: 调度器初始化 (在 `kernel/sched.c`)，包括初始化任务0的TSS和LDT，设置GDT中的相关描述符。
    *   `buffer_init(buffer_memory_end);`: 初始化缓冲区高速缓存 (在 `fs/buffer.c`)。
    *   `hd_init();`: 硬盘驱动程序初始化 (在 `drivers/block/hd.c`)。
    *   `floppy_init();`: 软盘驱动程序初始化 (在 `drivers/block/floppy.c`)。
7.  **开中断**: `sti();` 允许CPU响应中断。
8.  **切换到用户模式**: `move_to_user_mode();`
    *   这个宏 (在 `<asm/system.h>`) 通过构造一个 `iret` 指令的栈帧，将CPU从内核态切换到用户态。此时CS指向用户代码段，SS指向用户数据段，EIP指向 `iret` 后紧随的指令。栈仍然是内核栈，但CPU特权级已改变。
9.  **创建进程1 (init进程)**:
    *   `if (!fork()) { init(); }`:
        *   调用内联的 `fork()` 系统调用。
        *   对于子进程 (进程1，`fork()` 返回0)，它会调用 `init()` 函数。
        *   对于父进程 (进程0)，`fork()` 返回子进程的PID (即1)。
10. **进程0成为空闲进程**:
    *   `for(;;) pause();`: 父进程 (进程0) 进入一个无限循环，不断调用 `pause()`。
    *   注释解释：对于任务0，`pause()` 的行为特殊。它不会真的等待信号，而是让调度器检查是否有其他可运行的任务。如果没有，`pause()` 会返回，任务0继续循环。这使得任务0在没有其他任务运行时占用CPU，成为空闲 (idle) 进程。

## `static int printf(const char *fmt, ...)` 函数

*   **功能**: 内核版本的简单 `printf` 实现。
*   **步骤**:
    1.  `va_list args; va_start(args, fmt);`: 初始化可变参数列表。
    2.  `write(1, printbuf, i = vsprintf(printbuf, fmt, args));`:
        *   `vsprintf(printbuf, fmt, args)`: 将格式化字符串和参数列表通过 `vsprintf` (在 `kernel/vsprintf.c`) 转换为最终的字符串，并存入静态缓冲区 `printbuf`。返回字符串长度 `i`。
        *   `write(1, printbuf, i)`: 调用 `write` 系统调用 (文件描述符1，即标准输出，在内核初始化后通常指向控制台TTY) 将 `printbuf` 中的内容输出。
    3.  `va_end(args);`: 结束可变参数处理。
    4.  返回写入的字符数 `i`。

## `init(void)` 函数

这是进程1执行的函数，负责更高层次的系统初始化和用户环境的建立。

1.  **`setup((void *) &drive_info);`**: 调用 `setup` 系统调用。
    *   这个系统调用的内核实现是 `sys_setup()` (在 `kernel/blk_drv/hd.c` 或相关块设备初始化中)。
    *   它的目的是利用 `drive_info` (从BIOS获取的硬盘参数) 和其他可能的硬件信息来进一步配置块设备，特别是硬盘和软盘。它可能会读取分区表等。
2.  **设置标准I/O**:
    *   `(void) open("/dev/tty0", O_RDWR, 0);`: 以读写方式打开控制台设备 `/dev/tty0`。这将成为进程1的标准输入 (文件描述符0)。
    *   `(void) dup(0);`: 复制文件描述符0到下一个可用的描述符 (即1)。标准输出现在也指向 `/dev/tty0`。
    *   `(void) dup(0);`: 再次复制文件描述符0到下一个可用的描述符 (即2)。标准错误现在也指向 `/dev/tty0`。
    *   至此，进程1的标准输入、标准输出、标准错误都已重定向到 `/dev/tty0`。
3.  **打印系统信息**:
    *   `printf("%d buffers = %d bytes buffer space\n\r", NR_BUFFERS, NR_BUFFERS*BLOCK_SIZE);`
    *   `printf("Free mem: %d bytes\n\r", memory_end - main_memory_start);`
4.  **执行 `/etc/rc` 脚本**:
    *   `if (!(pid = fork()))`: 创建一个子进程来执行 `/etc/rc`。
        *   子进程中：
            *   `close(0);`: 关闭标准输入 (因为 `/etc/rc` 通常不需要输入)。
            *   `if (open("/etc/rc", O_RDONLY, 0)) _exit(1);`: 以只读方式打开 `/etc/rc`，将其作为新的标准输入 (文件描述符0)。如果打开失败，子进程退出码为1。
            *   `execve("/bin/sh", argv_rc, envp_rc);`: 执行 `/bin/sh` 来解释 `/etc/rc` 脚本。
                *   `argv_rc[] = { "/bin/sh", NULL };` (参数：执行sh)
                *   `envp_rc[] = { "HOME=/", NULL };` (环境变量：HOME设为根目录)
            *   `_exit(2);`: 如果 `execve` 失败，子进程退出码为2。
    *   父进程 (进程1) 中：
        *   `if (pid > 0) while (pid != wait(&i)) /* nothing */;`: 等待执行 `/etc/rc` 的子进程结束。
5.  **循环创建登录Shell**:
    *   `while (1)`: 进入一个无限循环。
        *   `if ((pid = fork()) < 0) { printf("Fork failed in init\r\n"); continue; }`: 创建一个新的子进程。如果 `fork` 失败，打印错误并继续循环。
        *   **子进程 (新的登录Shell会话)**:
            *   `if (!pid)`:
                *   `close(0); close(1); close(2);`: 关闭从父进程继承来的标准文件描述符。
                *   `setsid();`: 创建一个新的会话，使当前子进程成为新会话的领导者和新进程组的领导者。这将使其脱离原来的控制终端（如果有的话）。
                *   `(void) open("/dev/tty0", O_RDWR, 0); (void) dup(0); (void) dup(0);`: 重新打开 `/dev/tty0` 作为标准输入、标准输出和标准错误。这确保了新的登录会话有自己的控制终端。
                *   `_exit(execve("/bin/sh", argv, envp));`: 执行 `/bin/sh` (登录shell)。
                    *   `argv[] = { "-/bin/sh", NULL };` (参数：以登录shell方式执行sh，通常通过在 `argv[0]` 前加 `-` 实现)。
                    *   `envp[] = { "HOME=/usr/root", NULL };` (环境变量：HOME设为 `/usr/root`，超级用户的HOME目录)。
        *   **父进程 (进程1)**:
            *   `while (1) if (pid == wait(&i)) break;`: 等待刚刚创建的登录shell子进程结束。
            *   `printf("\n\rchild %d died with code %04x\n\r", pid, i);`: 打印子进程的PID和退出状态。
            *   `sync();`: 调用 `sync` 系统调用，将所有缓冲区数据写回磁盘，确保文件系统一致性。
            *   然后循环继续，准备创建下一个登录shell (如果前一个退出了)。
6.  `_exit(0);`: 理论上 `init` 函数 (进程1) 不会退出。如果它退出了，表示系统出现了严重问题。`_exit` 是一个直接终止进程的系统调用，与 `exit()` 不同（后者会执行一些清理，如调用 `atexit` 注册的函数）。

## 总结

`init/main.c` 是 Linux 0.11 内核的“指挥中心”，它编排了系统从最原始的硬件状态到能够运行多任务和用户程序的环境的整个初始化过程。`main()` 函数负责底层的硬件和服务初始化，然后通过创建进程0（空闲进程）和进程1（init进程）来启动多任务环境。`init()` 函数（由进程1执行）则接管了用户空间的初始化，通过执行 `/etc/rc` 脚本和循环创建登录shell，最终将系统呈现给用户。这个文件清晰地展示了操作系统启动时从内核态到用户态的过渡，以及第一个用户进程如何被创建并承担起系统后续管理任务的过程。内联系统调用的使用和对栈的谨慎处理也反映了早期内核开发的特定约束。
