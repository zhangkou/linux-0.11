# kernel/exit.c 文件详解

`kernel/exit.c` 文件是 Linux 0.11 内核中负责处理进程终止、资源释放以及父进程等待子进程结束（`wait` 功能）的核心部分。它包含了实现 `exit()`、`waitpid()` 和 `kill()` 这三个关键系统调用的主要逻辑。

## 核心功能

1.  **进程终止 (`do_exit`, `sys_exit`)**:
    *   释放进程占用的资源，如页表、打开的文件、inode引用（当前工作目录、根目录、可执行文件）。
    *   处理子进程的父进程指针（将其过继给init进程，即进程1）。
    *   如果进程是会话领导者且有关联的TTY，则清除TTY的进程组。
    *   如果进程是会话领导者，向会话中的所有进程发送 `SIGHUP` 信号。
    *   将进程状态设置为 `TASK_ZOMBIE`（僵尸态）。
    *   保存退出码。
    *   向父进程发送 `SIGCHLD` 信号。
    *   调用 `schedule()` 放弃CPU，不再返回。
2.  **发送信号 (`send_sig`, `sys_kill`)**:
    *   `send_sig()`: 内部辅助函数，向指定进程发送信号（设置其 `signal` 位图）。
    *   `sys_kill()`: 实现 `kill()` 系统调用，用于向指定PID的进程、指定进程组的进程或所有进程（pid=-1，但0.11中有限制）发送信号。它会进行权限检查。
3.  **等待子进程 (`sys_waitpid`)**:
    *   实现 `waitpid()` 系统调用，允许父进程等待其子进程的状态改变（终止或停止）。
    *   可以等待特定PID的子进程、特定进程组的子进程或任何子进程。
    *   支持 `WNOHANG` (非阻塞) 和 `WUNTRACED` (报告停止的子进程，0.11中此选项不完整) 选项。
    *   如果找到一个已终止的子进程 (僵尸进程)，则收集其退出状态，累加其用户和系统CPU时间到父进程的 `cutime` 和 `cstime`，释放该子进程的 `task_struct`，并返回子进程PID。
    *   如果子进程已停止，且指定了 `WUNTRACED`，则返回子进程PID和停止状态。
    *   如果没有符合条件的子进程已改变状态，则根据 `WNOHANG` 选项决定是立即返回0还是使父进程睡眠等待。
4.  **资源释放 (`release`)**:
    *   释放一个 `task_struct` 结构所占用的内存页，并将其从 `task[]` 数组中移除。
5.  **通知父进程 (`tell_father`)**:
    *   当进程退出时，向其父进程发送 `SIGCHLD` 信号。
    *   如果找不到父进程（理论上不应发生，除非是init进程退出或内核错误），则直接释放自身（这是一个不理想的处理，注释中提到应改为过继给init进程）。
6.  **会话终止 (`kill_session`)**:
    *   当一个会话领导者进程退出时，向该会话中的所有其他进程发送 `SIGHUP` 信号。

## 数据结构和宏

*   **`struct task_struct`**: 进程控制块，包含了进程状态、PID、父进程PID、信号位图、打开文件、LDT、TSS等所有进程相关信息。`exit.c` 中的函数大量操作此结构。
*   **`task[NR_TASKS]`**: 全局进程表数组。
*   **`SIGCHLD`**: 子进程状态改变时发送给父进程的信号。
*   **`SIGHUP`**: 连接断开信号，当会话领导者退出时发送给会话成员。
*   **`TASK_ZOMBIE`**: 僵尸进程状态。
*   **`TASK_STOPPED`**: 进程停止状态。
*   **`TASK_INTERRUPTIBLE`**: 可中断睡眠状态。
*   **`WUNTRACED`, `WNOHANG`**: `waitpid()` 的选项。

## 主要函数详解

### `void release(struct task_struct * p)`

*   **功能**: 彻底释放一个进程结构 `p`。
*   **步骤**:
    1.  空指针检查。
    2.  遍历 `task[]` 数组，找到指针 `p` 对应的条目。
    3.  将 `task[i]` 设为 `NULL`，表示该槽位空闲。
    4.  `free_page((long)p);`: 释放 `task_struct` 结构自身所占用的内存页（通常一个 `task_struct` 占用一页）。
    5.  `schedule();`: 调用调度器，因为当前进程列表已改变，可能需要重新调度。
    6.  如果找不到 `p`，则 `panic("trying to release non-existent task");`。

### `static inline int send_sig(long sig, struct task_struct * p, int priv)`

*   **功能**: 向进程 `p` 发送信号 `sig`。
*   **参数**:
    *   `sig`: 信号编号 (1-32)。
    *   `p`: 目标进程的 `task_struct` 指针。
    *   `priv`: 是否为特权发送 (1表示特权，如内核发送；0表示非特权，如用户通过`kill`发送)。
*   **步骤**:
    1.  检查 `p` 是否为空，`sig` 是否在有效范围内。如果无效，返回 `-EINVAL`。
    2.  **权限检查**:
        *   如果 `priv` 为真 (特权发送)，或者当前进程的有效用户ID `current->euid` 与目标进程的有效用户ID `p->euid` 相同，或者当前进程是超级用户 (`suser()`)，则允许发送。
        *   否则，返回 `-EPERM` (权限不足)。
    3.  `p->signal |= (1 << (sig - 1));`: 如果允许发送，则在目标进程 `p` 的 `signal` 位图中，将对应于信号 `sig` 的位置1。信号从1开始编号，位图从0开始索引，所以是 `sig-1`。
    4.  返回0表示成功。

### `static void kill_session(void)`

*   **功能**: 当当前进程（必须是会话领导者）退出时，向其会话内的所有其他进程发送 `SIGHUP` 信号。
*   **步骤**:
    1.  从 `task` 数组的末尾向前遍历。
    2.  `if (*p && (*p)->session == current->session)`: 如果找到一个存在的进程，并且其会话ID `(*p)->session` 与当前进程的会话ID `current->session` 相同。
    3.  `(*p)->signal |= 1<<(SIGHUP-1);`: 向该进程的信号位图添加 `SIGHUP`。

### `int sys_kill(int pid, int sig)`

*   **功能**: 实现 `kill()` 系统调用。
*   **参数**:
    *   `pid`: 指定目标进程或进程组。
        *   `pid > 0`: 向PID为 `pid` 的进程发送信号。
        *   `pid == 0`: 向与当前进程属于同一进程组的所有进程发送信号。
        *   `pid == -1`: 向所有有权限发送信号的进程发送信号（除了进程0和进程1）。Linux 0.11 的实现是向所有进程尝试发送（除了init进程外，对其他进程的发送受权限控制）。
        *   `pid < -1`: 向进程组ID为 `-pid` 的所有进程发送信号。
    *   `sig`: 要发送的信号。
*   **步骤**:
    1.  初始化 `retval = 0`。
    2.  根据 `pid` 的不同值，遍历 `task` 数组，找到目标进程：
        *   `if (!pid)`: `pid == 0` 的情况。目标是当前进程组。`priv` 参数为1（特权发送，因为是发送给同组的，但实际 `send_sig` 内部的权限检查仍会基于euid和suser）。
        *   `else if (pid > 0)`: 目标是特定PID。`priv` 参数为0（非特权）。
        *   `else if (pid == -1)`: 目标是所有可选进程。`priv` 参数为0。
        *   `else`: `pid < -1` 的情况。目标是特定进程组 `-pid`。`priv` 参数为0。
    3.  对每个找到的目标进程，调用 `send_sig(sig, *p, priv)`。
    4.  如果 `send_sig` 返回错误 `err`，则将 `retval` 设置为 `err`。这意味着如果向多个进程发送信号时，只要有一次失败，`sys_kill` 最终会返回一个错误码（通常是最后一次失败的错误码）。
    5.  返回 `retval`。

### `static void tell_father(int pid)`

*   **功能**: 向指定的父进程 `pid` 发送 `SIGCHLD` 信号，通知其子进程状态已改变（通常是子进程已终止）。
*   **步骤**:
    1.  如果 `pid` 非0 (即有父进程，不是init进程本身这种特殊情况)：
        *   遍历 `task` 数组，查找PID为 `pid` 的进程。
        *   如果找到，设置其 `signal` 位图中的 `SIGCHLD` 位。
        *   然后返回。
    2.  **特殊处理**: 如果 `pid` 为0，或者遍历完未找到父进程：
        *   打印错误信息 `"BAD BAD - no father found\n\r"`。
        *   `release(current);`: 直接释放当前进程。这是一个不正确的处理，注释也指出了这一点，正确的做法应该是将当前进程过继给init进程（PID为1）。

### `int do_exit(long code)`

*   **功能**: 进程退出的核心处理函数。执行所有必要的清理工作，并将进程转为僵尸态。
*   **参数**: `code` 是进程的退出状态（包含了退出码和可能的终止信号信息）。
*   **步骤**:
    1.  **释放内存页表**:
        *   `free_page_tables(get_base(current->ldt[1]), get_limit(0x0f));`: 释放用户代码段占用的页表。`0x0f` 是用户代码段选择子。
        *   `free_page_tables(get_base(current->ldt[2]), get_limit(0x17));`: 释放用户数据/堆栈段占用的页表。`0x17` 是用户数据段选择子。
    2.  **处理子进程 (过继给init)**:
        *   遍历 `task` 数组。
        *   `if (task[i] && task[i]->father == current->pid)`: 如果找到当前进程的子进程：
            *   `task[i]->father = 1;`: 将该子进程的父进程ID设为1 (init进程)。
            *   `if (task[i]->state == TASK_ZOMBIE)`: 如果这个子进程已经是僵尸态，那么它的原父进程（即当前退出的进程）还没来得及 `wait()` 它。现在它被过继给init，init进程需要得到通知来回收它。
            *   `(void) send_sig(SIGCHLD, task[1], 1);`: 向init进程 (`task[1]`) 发送 `SIGCHLD` 信号（以特权方式）。
    3.  **关闭打开的文件**:
        *   `for (i=0 ; i<NR_OPEN ; i++) if (current->filp[i]) sys_close(i);`: 遍历当前进程的文件描述符表，关闭所有已打开的文件。
    4.  **释放inode引用**:
        *   `iput(current->pwd); current->pwd = NULL;`: 释放对当前工作目录inode的引用。
        *   `iput(current->root); current->root = NULL;`: 释放对根目录inode的引用。
        *   `iput(current->executable); current->executable = NULL;`: 释放对可执行文件inode的引用。
    5.  **处理TTY**:
        *   `if (current->leader && current->tty >= 0) tty_table[current->tty].pgrp = 0;`: 如果当前进程是会话领导者并且拥有一个控制TTY，则将该TTY的进程组设为0（表示没有前台进程组）。
    6.  **处理数学协处理器**:
        *   `if (last_task_used_math == current) last_task_used_math = NULL;`: 如果当前进程是最后一个使用FPU的进程，则清除该标记。
    7.  **处理会话**:
        *   `if (current->leader) kill_session();`: 如果当前进程是会话领导者，则调用 `kill_session()` 向会话中的所有其他进程发送 `SIGHUP`。
    8.  **设置僵尸状态**:
        *   `current->state = TASK_ZOMBIE;`
        *   `current->exit_code = code;`: 保存退出状态码。
    9.  **通知父进程**: `tell_father(current->father);`
    10. **放弃CPU**: `schedule();` 调用调度器，选择下一个进程运行。当前进程（现在是僵尸）不会再被调度执行。
    11. `return (-1);`: 这行代码理论上不会被执行到，因为 `schedule()` 不会返回到这里。它的存在是为了抑制编译器关于函数没有返回值的警告。

### `int sys_exit(int error_code)`

*   **功能**: 实现 `exit()` 系统调用。
*   **参数**: `error_code` 是用户程序调用 `exit()` 时提供的退出码 (通常是0-255)。
*   **实现**: `return do_exit((error_code & 0xff) << 8);`
    *   将用户提供的 `error_code` (取低8位，确保在0-255范围) 左移8位。这是因为在 `waitpid` 返回的状态中，正常退出的退出码存储在高字节。低字节用于区分是正常退出还是因信号终止/停止。
    *   调用 `do_exit()` 执行实际的退出流程。

### `int sys_waitpid(pid_t pid, unsigned long * stat_addr, int options)`

*   **功能**: 实现 `waitpid()` 系统调用。
*   **步骤**:
    1.  `verify_area(stat_addr, 4);`: 验证用户提供的 `stat_addr` 指针是否有效（用于存储4字节的退出状态）。
    2.  **`repeat:` 标签 (循环等待)**:
        a.  `flag = 0;`: `flag` 用于标记是否找到了任何子进程（无论其状态如何）。
        b.  **遍历进程表 (从后向前)**: `for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)`
            *   跳过空槽位 (`!*p`) 和当前进程 (`*p == current`)。
            *   跳过不是当前进程子进程的进程 (`(*p)->father != current->pid`)。
            *   **根据 `pid` 参数筛选目标子进程**:
                *   `pid > 0`: 只关心PID为 `pid` 的子进程。
                *   `pid == 0`: 只关心与当前进程同组的子进程。
                *   `pid == -1`: 关心任何子进程 (默认行为)。
                *   `pid < -1`: 只关心进程组ID为 `-pid` 的子进程。
            *   **检查子进程状态 `(*p)->state`**:
                *   `case TASK_STOPPED`: 如果子进程已停止：
                    *   `if (!(options & WUNTRACED)) continue;`: 如果没有设置 `WUNTRACED` 选项，则忽略停止的子进程，继续寻找。
                    *   `put_fs_long(0x7f, stat_addr);`: 向用户空间 `stat_addr` 写入 `0x7f` (表示因信号停止，但停止信号在高字节，这里只设低字节为0x7f)。正确做法应是将停止信号放入高字节，低字节设为0x7F。Linux 0.11这里可能不完全符合POSIX。
                    *   `return (*p)->pid;`: 返回停止的子进程PID。
                *   `case TASK_ZOMBIE`: 如果子进程是僵尸态（已终止）：
                    *   `current->cutime += (*p)->utime; current->cstime += (*p)->stime;`: 将子进程的用户时间和系统时间累加到父进程的 `cutime` 和 `cstime`。
                    *   `flag = (*p)->pid; code = (*p)->exit_code;`: 保存子进程PID和退出状态码。
                    *   `release(*p);`: 彻底释放该僵尸子进程的 `task_struct`。
                    *   `put_fs_long(code, stat_addr);`: 将子进程的退出状态码写入用户空间。
                    *   `return flag;`: 返回已终止子进程的PID。
                *   `default`: 其他状态（如 `TASK_RUNNING`, `TASK_INTERRUPTIBLE`），设置 `flag = 1` 表示至少有一个符合条件的子进程存在但尚未终止/停止，然后 `continue` 检查下一个。
        c.  **处理遍历完后的情况**:
            *   `if (flag)` (表示至少有一个符合条件的子进程存在，但它尚未终止或停止):
                *   `if (options & WNOHANG) return 0;`: 如果设置了 `WNOHANG` 选项，则非阻塞，立即返回0。
                *   `current->state = TASK_INTERRUPTIBLE; schedule();`: 否则，将当前父进程设为可中断睡眠，并调用调度器。
                *   `if (!(current->signal &= ~(1<<(SIGCHLD-1)))) goto repeat;`: 当父进程被唤醒时：
                    *   首先清除可能收到的 `SIGCHLD` 信号（因为 `waitpid` 本身就在处理子进程状态改变事件）。
                    *   如果除了 `SIGCHLD` 外没有其他信号导致唤醒 (`current->signal` 为0)，则 `goto repeat` 重新尝试查找子进程。
                    *   否则 (被其他信号唤醒)，返回 `-EINTR` (系统调用被中断)。
            *   `return -ECHILD;`: 如果 `flag` 为0 (表示没有找到任何符合 `pid` 条件的子进程，或者所有子进程都已通过 `wait` 被处理)，则返回 `-ECHILD` (无子进程)。

## 总结

`kernel/exit.c` 文件是Linux 0.11进程模型中生命周期管理的后端。它处理了进程如何“死亡”（`do_exit`），如何释放其占用的核心资源，以及如何通知其他相关进程（父进程、会话成员、子进程）。同时，它也实现了父进程如何“感知”和“回收”子进程的死亡（`sys_waitpid`）。`sys_kill` 则提供了进程间发送信号的基本机制，这是进程间异步通信和控制的重要手段。这些功能的正确和高效实现对于一个多任务操作系统的稳定性和可靠性至关重要。文件中对资源（如页表、文件描述符、inode引用）的细致清理，以及对子进程和会话成员状态的妥善处理，都体现了操作系统内核在进程管理方面的复杂性。
