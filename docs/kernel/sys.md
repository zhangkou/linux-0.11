# kernel/sys.c 文件详解

`kernel/sys.c` 文件是 Linux 0.11 内核中实现一系列**杂项系统调用 (miscellaneous system calls)** 的地方。这些系统调用通常不属于文件系统、进程管理或内存管理等大的模块，但它们提供了重要的操作系统服务，如获取系统信息、设置用户/组ID、时间管理、进程组管理等。

这个文件中的函数通常以 `sys_` 开头，并直接对应一个系统调用号（在 `<linux/sys.h>` 和 `kernel/system_call.s` 中定义和关联）。

## 核心功能

该文件实现了以下系统调用：

1.  **未实现的系统调用桩 (Stubs)**:
    *   `sys_ftime()`: 获取更详细的时间信息 (返回 `-ENOSYS`)。
    *   `sys_break()`: (旧式) 改变数据段大小 (返回 `-ENOSYS`)。
    *   `sys_ptrace()`: 进程跟踪 (返回 `-ENOSYS`)。
    *   `sys_stty()`, `sys_gtty()`: (旧式) 设置/获取TTY属性 (返回 `-ENOSYS`)。
    *   `sys_rename()`: 重命名文件 (返回 `-ENOSYS`)。
    *   `sys_prof()`: 性能分析 (返回 `-ENOSYS`)。
    *   `sys_acct()`: 进程审计 (返回 `-ENOSYS`)。
    *   `sys_phys()`: (用途不明，可能与物理内存访问有关) (返回 `-ENOSYS`)。
    *   `sys_lock()`: 文件锁定 (返回 `-ENOSYS`)。
    *   `sys_mpx()`: (用途不明，可能与MPX协处理器有关) (返回 `-ENOSYS`)。
    *   `sys_ulimit()`: (旧式) 获取/设置用户资源限制 (返回 `-ENOSYS`)。
    *   这些系统调用在 Linux 0.11 中被定义了系统调用号，但没有提供具体实现，直接返回 `-ENOSYS` (Function not implemented)。
2.  **用户和组ID管理**:
    *   `sys_setregid(rgid, egid)`: 设置真实组ID (real GID) 和有效组ID (effective GID)。
    *   `sys_setgid(gid)`: 同时设置真实GID和有效GID为 `gid` (通过调用 `sys_setregid`)。
    *   `sys_setreuid(ruid, euid)`: 设置真实用户ID (real UID) 和有效用户ID (effective UID)。
    *   `sys_setuid(uid)`: 同时设置真实UID和有效UID为 `uid` (通过调用 `sys_setreuid`)。
3.  **时间管理**:
    *   `sys_time(tloc)`: 获取当前时间 (自Epoch以来的秒数)。
    *   `sys_stime(tptr)`: 设置系统时间 (需要超级用户权限)。
    *   `sys_times(tbuf)`: 获取进程的用户、系统CPU时间以及已终止子进程的CPU时间。
4.  **内存管理相关**:
    *   `sys_brk(end_data_seg)`: 改变数据段的上界 (break)，即调整堆的大小。
5.  **进程组和会话管理**:
    *   `sys_setpgid(pid, pgid)`: 设置指定进程 `pid` 的进程组ID为 `pgid`。
    *   `sys_getpgrp()`: 获取当前进程的进程组ID。
    *   `sys_setsid()`: 创建一个新的会话，并使当前进程成为新会话的领导者和新进程组的领导者。
6.  **系统信息**:
    *   `sys_uname(name)`: 获取当前系统的名称和版本信息 (如操作系统名、节点名、发行版、版本、硬件架构)。
7.  **文件模式创建屏蔽码**:
    *   `sys_umask(mask)`: 设置文件模式创建屏蔽码 (umask)，并返回旧的umask。

## 关键数据结构和宏

*   **`struct task_struct *current`**: 指向当前正在运行进程的 `task_struct` 结构体。这些系统调用大多会读取或修改 `current` 的成员。
*   **`struct tms`**: (在 `<sys/times.h>` 定义) 用于 `sys_times()`，存储CPU时间信息。
*   **`struct utsname`**: (在 `<sys/utsname.h>` 定义) 用于 `sys_uname()`，存储系统名称信息。
*   **`startup_time`**: 全局变量 (在 `kernel/sched.c` 定义)，存储系统启动时的Unix时间戳。
*   **`jiffies`**: 全局变量 (在 `kernel/sched.c` 定义)，系统启动以来的时钟滴答数。
*   **`HZ`**: 每秒时钟滴答数 (在 `<linux/sched.h>` 定义，通常为100)。
*   **`CURRENT_TIME`**: 宏 (在 `<linux/sched.h>` 定义)，计算当前Unix时间戳 (`startup_time + jiffies / HZ`)。
*   **`suser()`**: 宏 (在 `<linux/kernel.h>` 定义)，判断当前进程是否为超级用户 (`current->euid == 0`)。
*   **`verify_area(addr, size)`**: 函数 (在 `kernel/fork.c` 声明，实现在 `mm/memory.c`)，用于验证用户空间内存区域的可访问性。
*   **`get_fs_long(addr)`, `put_fs_long(val, addr)`**: 函数 (在 `<asm/segment.h>` 定义)，用于从/向用户空间 (`fs` 段) 读写长整型数据。

## 主要系统调用实现详解

### `int sys_time(long * tloc)`

*   **功能**: 获取当前日历时间（自Epoch以来的秒数）。
*   **实现**:
    1.  `i = CURRENT_TIME;`: 计算当前时间。
    2.  `if (tloc)`: 如果用户提供了指针 `tloc`：
        *   `verify_area(tloc, 4);`: 验证用户空间地址 `tloc` 是否可写 (4字节)。
        *   `put_fs_long(i, (unsigned long *)tloc);`: 将时间值 `i` 写入用户空间。
    3.  返回时间值 `i`。

### `int sys_stime(long * tptr)`

*   **功能**: 设置系统时间。需要超级用户权限。
*   **实现**:
    1.  `if (!suser()) return -EPERM;`: 检查是否为超级用户。
    2.  `startup_time = get_fs_long((unsigned long *)tptr) - jiffies / HZ;`
        *   从用户空间 `tptr` 读取新的时间值 (秒数)。
        *   `jiffies / HZ` 是系统已运行的秒数。
        *   通过 `新时间 - 已运行秒数` 来反算出系统启动时对应的Epoch秒数，并更新全局变量 `startup_time`。

### `int sys_setuid(int uid)` 和 `int sys_setreuid(int ruid, int euid)`

*   **`sys_setuid(uid)`**: 调用 `sys_setreuid(uid, uid)`，同时设置真实用户ID (UID) 和有效用户ID (EUID) 为 `uid`。
*   **`sys_setreuid(ruid, euid)`**:
    *   **设置真实UID (`current->uid`)**:
        *   如果 `ruid > 0` (表示要修改)。
        *   允许的条件：当前EUID等于目标 `ruid`，或者当前UID等于目标 `ruid` (允许非特权用户将其UID改回其原始UID或当前的EUID)，或者是超级用户。
        *   否则返回 `-EPERM`。
    *   **设置有效UID (`current->euid`)**:
        *   如果 `euid > 0`。
        *   允许的条件：修改前的真实UID等于目标 `euid`，或者当前EUID等于目标 `euid`，或者是超级用户。
        *   否则，如果设置真实UID成功了，需要将其恢复 (`current->uid = old_ruid`)，然后返回 `-EPERM`。
    *   成功返回0。
    *   (Linux 0.11 未实现保存的 set-user-ID `suid` 的完整POSIX语义，但 `task_struct` 中有该字段)。

### `int sys_setgid(int gid)` 和 `int sys_setregid(int rgid, int egid)`

*   逻辑与 `setuid/setreuid` 非常相似，只是操作的是组ID (`gid`, `egid`, `sgid`)。
*   允许的条件：当前GID等于目标 `rgid/egid`，或当前EGID等于目标 `egid`，或当前SGID等于目标 `egid`（对于设置EGID），或者是超级用户。

### `int sys_times(struct tms * tbuf)`

*   **功能**: 获取当前进程及其已终止子进程的CPU时间统计。
*   **实现**:
    1.  `if (tbuf)`: 如果用户提供了 `tbuf` 指针：
        *   `verify_area(tbuf, sizeof *tbuf);`: 验证用户空间缓冲区的可写性。
        *   `put_fs_long(current->utime, (unsigned long *)&tbuf->tms_utime);`: 将当前进程的用户时间写入。
        *   `put_fs_long(current->stime, (unsigned long *)&tbuf->tms_stime);`: 写入系统时间。
        *   `put_fs_long(current->cutime, (unsigned long *)&tbuf->tms_cutime);`: 写入子进程用户时间。
        *   `put_fs_long(current->cstime, (unsigned long *)&tbuf->tms_cstime);`: 写入子进程系统时间。
    2.  返回当前的系统时钟滴答数 `jiffies`。

### `int sys_brk(unsigned long end_data_seg)`

*   **功能**: 改变进程数据段的上界（break）。用于动态内存分配（如 `malloc` 的底层实现）。
*   **实现**:
    1.  检查请求的新break值 `end_data_seg` 是否合法：
        *   必须大于等于代码段的结束地址 (`current->end_code`)。
        *   必须小于用户栈的起始地址减去一个安全余量 (`current->start_stack - 16384`)，以防止堆栈碰撞。
    2.  如果合法，则更新 `current->brk = end_data_seg;`。
    3.  返回当前（可能已更新的）`current->brk` 值。
    *   **注意**: 这个 `sys_brk` 只是更新了 `task_struct` 中的记录。实际的内存页分配和页表建立是在发生缺页异常时，由 `do_no_page()` (在 `mm/memory.c`) 根据 `current->brk` 的值来处理的。

### `int sys_setpgid(int pid, int pgid)`

*   **功能**: 设置进程 `pid` 的进程组ID为 `pgid`。
*   **实现**:
    1.  如果 `pid == 0`，则目标进程是当前进程。
    2.  如果 `pgid == 0`，则目标进程组ID是当前进程的PID。
    3.  遍历 `task` 数组查找PID为 `pid` 的进程。
    4.  **权限和条件检查**:
        *   目标进程不能是会话领导者 (`task[i]->leader`)。
        *   目标进程必须与当前进程在同一个会话中 (`task[i]->session != current->session`)。
        *   (POSIX标准还要求：如果 `pid` 是当前进程，则不能将自己移出当前会话；如果 `pid` 是子进程且尚未 `exec`，则允许；其他情况需要更复杂的权限)。Linux 0.11的检查相对简单。
    5.  如果检查通过，设置 `task[i]->pgrp = pgid;`。
    6.  成功返回0，找不到进程返回 `-ESRCH`，权限不足返回 `-EPERM`。

### `int sys_setsid(void)`

*   **功能**: 创建一个新的会话，并使当前进程成为新会话的领导者和新进程组的领导者。
*   **实现**:
    1.  如果当前进程已经是会话领导者 (`current->leader`) 并且不是超级用户，则返回 `-EPERM` (因为只有非会话领导者才能创建新会话，除非是超级用户可能有一些特殊情况，但这里逻辑是阻止已是leader的普通用户)。
    2.  `current->leader = 1;`: 标记当前进程为会话领导者。
    3.  `current->session = current->pgrp = current->pid;`: 将会话ID和进程组ID都设置为当前进程的PID。
    4.  `current->tty = -1;`: 使新会话脱离原来的控制终端。
    5.  返回新的进程组ID (即当前PID)。

### `int sys_uname(struct utsname * name)`

*   **功能**: 获取系统名称和版本信息。
*   **实现**:
    1.  定义一个静态的 `struct utsname thisname`，其中硬编码了系统信息：
        *   `sysname`: "linux .0" (注意，版本号0.11不在这里，可能在 `version` 字段)
        *   `nodename`: "nodename" (默认节点名，实际应由系统配置)
        *   `release`: "release " (应为内核发行版号，如 "0.11")
        *   `version`: "version " (应为更详细的内核版本信息)
        *   `machine`: "machine " (应为硬件架构，如 "i386")
        *   (这些硬编码的值在实际系统中会被更动态或配置的值替换)
    2.  如果 `name` 指针为空，返回 `-ERROR` (应为 `-EFAULT`)。
    3.  `verify_area(name, sizeof *name);`: 验证用户空间缓冲区的可写性。
    4.  逐字节将 `thisname` 的内容复制到用户空间的 `name` 指向的结构中。
    5.  返回0。

### `int sys_umask(int mask)`

*   **功能**: 设置当前进程的文件模式创建屏蔽码 (umask)，并返回旧的umask。
*   **实现**:
    1.  `int old = current->umask;`: 保存旧的umask。
    2.  `current->umask = mask & 0777;`: 将新的umask设置为 `mask` 的低9位 (权限位)。
    3.  返回 `old`。

## 总结

`kernel/sys.c` 文件是 Linux 0.11 中一系列不方便归类到其他主要内核模块（如`fs/`, `mm/`, `kernel/sched.c`）但又非常必要的系统调用的集合地。它处理了用户/组ID管理、基本的时间获取与设置、进程的堆内存调整 (`brk`)、进程组和会话管理、系统信息查询 (`uname`)以及文件创建模式屏蔽 (`umask`) 等功能。同时，它也为许多在 `<linux/sys.h>` 中声明但尚未实现的系统调用提供了一个返回 `-ENOSYS` 的占位实现，这表明了内核仍在开发和完善中。这些系统调用共同丰富了内核向用户程序提供的服务接口。
