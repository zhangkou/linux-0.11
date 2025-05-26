# sys.h 文件详解

`include/linux/sys.h` 文件是 Linux 0.11 内核中一个非常重要的头文件，它主要用于定义**系统调用表 (system call table)** 及其相关的函数原型。系统调用是用户空间程序请求内核执行特权操作的接口。当用户程序发起一个系统调用时，CPU会切换到内核态，并通过一个唯一的系统调用号在系统调用表中查找到对应的内核处理函数并执行。

这个文件有两个主要部分：
1.  **外部函数声明**: 声明了所有系统调用处理函数的原型。这些函数的命名通常以 `sys_` 开头，例如 `sys_open`, `sys_read` 等。
2.  **系统调用表定义**: 定义了一个名为 `sys_call_table` 的函数指针数组，该数组的每个元素指向一个系统调用处理函数。数组的索引即为系统调用号。

## 核心功能

1.  **提供系统调用处理函数的统一声明**: 使得内核的其他部分（如 `kernel/sys_call.s` 中的中断处理入口）可以引用这些函数。
2.  **构建系统调用分派表**: `sys_call_table` 是内核响应用户请求的核心，它将整数形式的系统调用号映射到实际的内核C函数地址。

## 函数原型声明详解

文件中声明了大量的 `extern int sys_xyz();` 函数，每一个都对应一个系统调用。以下是一些重要的系统调用及其功能的简要说明（详细功能通常在其各自的实现文件或相关头文件的文档中描述）：

*   `extern int sys_setup();`: 系统启动时由 `main.c` 调用，用于执行一些初始设置，如挂载根文件系统。它不是一个常规的用户可调用的系统调用。
*   `extern int sys_exit();`: 终止当前进程。
*   `extern int sys_fork();`: 创建一个新进程 (子进程)。
*   `extern int sys_read();`: 从文件描述符读取数据。
*   `extern int sys_write();`: 向文件描述符写入数据。
*   `extern int sys_open();`: 打开或创建文件。
*   `extern int sys_close();`: 关闭文件描述符。
*   `extern int sys_waitpid();`: 等待子进程状态改变。
*   `extern int sys_creat();`: 创建新文件（旧式接口，等同于特定参数的 `open`）。
*   `extern int sys_link();`: 创建硬链接。
*   `extern int sys_unlink();`: 删除文件名（如果链接数为0，则删除文件）。
*   `extern int sys_execve();`: 执行一个新程序。
*   `extern int sys_chdir();`: 改变当前工作目录。
*   `extern int sys_time();`: 获取当前时间 (Unix时间戳)。
*   `extern int sys_mknod();`: 创建特殊文件 (设备文件、FIFO)。
*   `extern int sys_chmod();`: 修改文件权限。
*   `extern int sys_chown();`: 修改文件所有者 (在0.11中也称 `sys_lchown`)。
*   `extern int sys_break();`: (未使用，旧式 `brk` 的接口)。
*   `extern int sys_stat();`: 获取文件状态 (通过路径名)。
*   `extern int sys_lseek();`: 定位文件读写指针。
*   `extern int sys_getpid();`: 获取当前进程ID。
*   `extern int sys_mount();`: 挂载文件系统。
*   `extern int sys_umount();`: 卸载文件系统。
*   `extern int sys_setuid();`: 设置用户ID。
*   `extern int sys_getuid();`: 获取用户ID。
*   `extern int sys_stime();`: 设置系统时间 (需要特权)。
*   `extern int sys_ptrace();`: 进程跟踪 (用于调试)。
*   `extern int sys_alarm();`: 设置报警定时器。
*   `extern int sys_fstat();`: 获取文件状态 (通过文件描述符)。
*   `extern int sys_pause();`: 使进程挂起直到收到信号。
*   `extern int sys_utime();`: 修改文件的访问和修改时间。
*   `extern int sys_stty();`, `extern int sys_gtty();`: (旧式) 设置和获取TTY属性。
*   `extern int sys_access();`: 检查文件访问权限。
*   `extern int sys_nice();`: 修改进程的nice值 (调度优先级)。
*   `extern int sys_ftime();`: (旧式) 获取日期和时间。
*   `extern int sys_sync();`: 将所有缓冲区数据写回磁盘。
*   `extern int sys_kill();`: 发送信号到进程。
*   `extern int sys_rename();`: (Linux 0.11 中未实现，通常返回错误)。
*   `extern int sys_mkdir();`: 创建目录。
*   `extern int sys_rmdir();`: 删除目录。
*   `extern int sys_dup();`: 复制文件描述符。
*   `extern int sys_pipe();`: 创建管道。
*   `extern int sys_times();`: 获取进程和子进程的执行时间。
*   `extern int sys_prof();`: (未使用，用于程序性能分析)。
*   `extern int sys_brk();`: 修改数据段大小 (堆的末尾)。
*   `extern int sys_setgid();`: 设置组ID。
*   `extern int sys_getgid();`: 获取组ID。
*   `extern int sys_signal();`: 设置信号处理函数 (旧式接口)。
*   `extern int sys_geteuid();`: 获取有效用户ID。
*   `extern int sys_getegid();`: 获取有效组ID。
*   `extern int sys_acct();`: 开启或关闭进程审计。
*   `extern int sys_phys();`: (未使用，物理地址映射相关)。
*   `extern int sys_lock();`: (未使用，文件锁定相关)。
*   `extern int sys_ioctl();`: 设备控制操作。
*   `extern int sys_fcntl();`: 文件控制操作。
*   `extern int sys_mpx();`: (未使用，MPX相关)。
*   `extern int sys_setpgid();`: 设置进程组ID。
*   `extern int sys_ulimit();`: (旧式) 获取或设置用户资源限制。
*   `extern int sys_uname();`: 获取系统名称和版本信息。
*   `extern int sys_umask();`: 设置文件模式创建屏蔽码。
*   `extern int sys_chroot();`: 改变根目录。
*   `extern int sys_ustat();`: 获取文件系统统计信息。
*   `extern int sys_dup2();`: 复制文件描述符到指定编号。
*   `extern int sys_getppid();`: 获取父进程ID。
*   `extern int sys_getpgrp();`: 获取当前进程的进程组ID。
*   `extern int sys_setsid();`: 创建新会话并设置进程组ID。
*   `extern int sys_sigaction();`: 设置信号处理函数 (POSIX 接口)。
*   `extern int sys_sgetmask();`: 获取当前信号屏蔽码 (旧式)。
*   `extern int sys_ssetmask();`: 设置当前信号屏蔽码 (旧式)。
*   `extern int sys_setreuid();`: 设置真实用户ID和有效用户ID。
*   `extern int sys_setregid();`: 设置真实组ID和有效组ID。

## 系统调用表 `sys_call_table`

*   **定义**: `fn_ptr sys_call_table[] = { ... };`
    *   `fn_ptr`: 是一个函数指针类型，在 `sched.h` 中定义为 `typedef int (*fn_ptr)();`，表示一个返回 `int` 并且参数未指定的函数指针。
    *   `sys_call_table[]`: 这是一个函数指针数组。
*   **内容**: 数组的每个元素都是一个指向前面声明的 `sys_` 函数的指针。数组的顺序非常重要，因为系统调用号 (通常由用户程序在 `eax` 寄存器中传递) 就是这个数组的索引。
    *   例如，`sys_call_table[0]` 是 `sys_setup`，`sys_call_table[1]` 是 `sys_exit`，依此类推。
*   **使用**: 当用户程序执行 `int 0x80` (Linux 0.11 的系统调用中断) 时，中断处理程序 (在 `kernel/sys_call.s` 中) 会：
    1.  从 `eax` 寄存器获取系统调用号 `nr`。
    2.  检查 `nr` 是否在有效范围内 (小于 `sys_call_table` 的大小)。
    3.  如果有效，则调用 `call sys_call_table(,%eax,4)` (大致是 `call sys_call_table[eax*4]`) 来执行相应的内核函数。
    4.  系统调用处理函数的返回值会存入 `eax`，然后返回到用户空间。

## 总结

`include/linux/sys.h` 文件是 Linux 0.11 内核系统调用机制的核心声明文件。它通过声明所有系统调用处理函数的原型，并定义一个包含这些函数指针的 `sys_call_table` 数组，为内核提供了一个从系统调用号到具体内核服务例程的映射。这个表是用户空间与内核空间之间受控交互的主要桥梁。该文件中列出的系统调用种类和数量，反映了 Linux 0.11 所能提供的基本操作系统服务范围。Okay, I have created `docs/include/linux/sys.md` with the detailed Chinese explanation for `include/linux/sys.h`.

Next, I will process `include/linux/tty.h`.
1. Create `docs/include/linux/tty.md`.
2. Read `include/linux/tty.h`.
3. Write the explanation into `docs/include/linux/tty.md`.

First, I'll read `include/linux/tty.h`.
