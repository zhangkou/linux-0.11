# kernel/signal.c 文件详解

`kernel/signal.c` 文件是 Linux 0.11 内核中负责处理信号的核心部分。它实现了信号的设置、发送（部分逻辑，`sys_kill` 在 `exit.c`）、信号处理函数的注册、以及在进程从内核态返回用户态前检查并处理待处理信号的逻辑。

信号是Unix-like系统中一种异步事件通知机制，允许内核或其他进程通知一个进程发生了某个特定事件（如非法内存访问、子进程终止、用户按下Ctrl-C等）。

## 核心功能

1.  **信号屏蔽码管理**:
    *   `sys_sgetmask()`: 获取当前进程的信号屏蔽码 (`current->blocked`)。
    *   `sys_ssetmask(newmask)`: 设置当前进程的信号屏蔽码。`SIGKILL` 不能被阻塞。
2.  **信号处理函数注册**:
    *   `sys_signal(signum, handler, restorer)`: 旧式的、简单的信号处理函数注册接口。它将信号 `signum` 的处理函数设置为 `handler`，并使用 `restorer` 作为信号处理完毕后的返回跳板。它默认设置了 `SA_ONESHOT` (处理一次后恢复默认) 和 `SA_NOMASK` (处理期间不阻塞当前信号) 标志。
    *   `sys_sigaction(signum, action, oldaction)`: POSIX标准的信号处理函数注册接口。允许更精细地控制信号处理行为，如设置信号处理期间的屏蔽码 (`sa_mask`) 和各种标志 (`sa_flags`)。
3.  **信号处理执行 (`do_signal`)**:
    *   这是在进程从内核态（通常是系统调用或中断处理完毕后）返回用户态之前调用的核心函数。
    *   检查当前进程是否有待处理且未被阻塞的信号。
    *   如果找到需要处理的信号，它会修改内核栈上保存的用户态上下文（主要是 `eip` 和 `esp`），使得进程返回用户态后不是执行原来的指令，而是跳转到注册的信号处理函数。
    *   在用户栈上为信号处理函数构建一个新的栈帧，包含信号编号、错误码（未使用）、上下文信息以及一个指向“信号恢复器” (`sa_restorer`) 的返回地址。
    *   更新进程的信号屏蔽码，加入 `sa_action->sa_mask` 中指定的以及当前信号本身（除非设置了 `SA_NOMASK`）。
4.  **辅助函数**:
    *   `save_old(from, to)`: 将内核空间的 `struct sigaction` 内容复制到用户空间。
    *   `get_new(from, to)`: 从用户空间将 `struct sigaction` 内容复制到内核空间。

## 数据结构和宏

*   **`struct sigaction`**: (定义在 `<signal.h>`)
    *   `sa_handler`: 指向信号处理函数的指针 (或 `SIG_DFL`, `SIG_IGN`)。
    *   `sa_mask`: 在执行信号处理函数期间需要额外阻塞的信号集。
    *   `sa_flags`: 修改信号处理行为的标志 (如 `SA_ONESHOT`, `SA_NOMASK`)。
    *   `sa_restorer`: 指向一个特殊的返回跳板函数的指针。当信号处理函数执行完毕后，会返回到这个 `sa_restorer`，它负责恢复原先的执行上下文（主要是通过 `sigreturn` 系统调用，尽管在0.11中这部分可能不完整或与现代Linux不同）。
*   **`current->sigaction[32]`**: 每个进程的 `task_struct` 中包含一个 `sigaction` 结构数组，存储了对每个信号（1-31）的处理方式。
*   **`current->signal`**: 进程收到的待处理信号的位图。
*   **`current->blocked`**: 进程当前阻塞的信号的位图。
*   **`SA_ONESHOT`**: 标志，表示信号处理函数执行一次后，该信号的处理方式恢复为默认。
*   **`SA_NOMASK`**: 标志，表示在执行当前信号的处理函数期间，不自动阻塞该信号本身。

## 主要函数详解

### `int sys_sgetmask()` 和 `int sys_ssetmask(int newmask)`

*   **功能**: 获取和设置当前进程的信号阻塞掩码 (`current->blocked`)。
*   `sys_sgetmask()`: 直接返回 `current->blocked`。
*   `sys_ssetmask(newmask)`:
    1.  保存旧的 `current->blocked` 值。
    2.  将 `current->blocked` 设置为 `newmask`，但会清除 `newmask` 中对应于 `SIGKILL` 的位（因为 `SIGKILL` 不能被阻塞）。`~(1<<(SIGKILL-1))` 创建一个除了 `SIGKILL` 位为0，其他位都为1的掩码。
    3.  返回旧的阻塞掩码。

### `static inline void save_old(char * from, char * to)` 和 `static inline void get_new(char * from, char * to)`

*   **功能**: 这两个函数用于在内核空间和用户空间之间复制 `struct sigaction` 结构。
*   `save_old(from_kernel, to_user)`: 将内核中的 `struct sigaction` (`from_kernel`) 复制到用户空间地址 `to_user`。它会调用 `verify_area` 检查用户空间地址的有效性，然后逐字节使用 `put_fs_byte` 复制。
*   `get_new(from_user, to_kernel)`: 将用户空间地址 `from_user` 处的 `struct sigaction` 复制到内核中的 `to_kernel`。它逐字节使用 `get_fs_byte` 读取。

### `int sys_signal(int signum, long handler, long restorer)`

*   **功能**: 旧式的信号处理设置接口，兼容早期的Unix。
*   **参数**:
    *   `signum`: 信号编号 (1-31，且不能是 `SIGKILL`)。
    *   `handler`: 用户提供的信号处理函数地址 (或 `SIG_DFL`, `SIG_IGN`)。
    *   `restorer`: 用户提供的信号恢复函数的地址（跳板）。
*   **步骤**:
    1.  检查 `signum` 是否有效。无效则返回-1。
    2.  构造一个临时的 `struct sigaction tmp`：
        *   `tmp.sa_handler = (void (*)(int)) handler;`
        *   `tmp.sa_mask = 0;` (不额外阻塞其他信号)
        *   `tmp.sa_flags = SA_ONESHOT | SA_NOMASK;` (处理一次后恢复默认，且处理期间不阻塞当前信号)
        *   `tmp.sa_restorer = (void (*)(void)) restorer;`
    3.  保存旧的 `sa_handler`: `handler = (long) current->sigaction[signum-1].sa_handler;`
    4.  将新的 `tmp` 结构赋给 `current->sigaction[signum-1]`。
    5.  返回旧的 `handler` 地址。

### `int sys_sigaction(int signum, const struct sigaction * action, struct sigaction * oldaction)`

*   **功能**: POSIX标准的信号处理设置接口。
*   **参数**:
    *   `signum`: 信号编号 (1-31，且不能是 `SIGKILL`)。
    *   `action`: 指向用户空间提供的 `struct sigaction` 结构，包含新的处理方式。如果为 `NULL`，则不改变当前处理方式。
    *   `oldaction`: 指向用户空间的 `struct sigaction` 结构，用于保存旧的处理方式。如果为 `NULL`，则不保存。
*   **步骤**:
    1.  检查 `signum` 是否有效。无效则返回-1。
    2.  `tmp = current->sigaction[signum-1];`: 保存当前进程对该信号的旧的 `sigaction` 设置。
    3.  `get_new((char *) action, (char *) (signum-1+current->sigaction));`: 如果 `action` 非空，则从用户空间 `action` 指向的地址将新的 `sigaction` 设置复制到内核 `current->sigaction[signum-1]`。
    4.  `if (oldaction) save_old((char *) &tmp, (char *) oldaction);`: 如果 `oldaction` 非空，则将之前保存的旧 `sigaction` 设置 `tmp` 复制到用户空间 `oldaction` 指向的地址。
    5.  **处理 `sa_mask`**:
        *   `if (current->sigaction[signum-1].sa_flags & SA_NOMASK)`: 如果设置了 `SA_NOMASK` 标志，则 `sa_mask` 保持用户提供的值（或者如果 `sys_signal` 方式调用，则为0）。
        *   `else current->sigaction[signum-1].sa_mask |= (1<<(signum-1));`: 否则（默认行为），将当前信号 `signum` 自身也加入到 `sa_mask` 中，意味着在执行该信号的处理函数期间，同类型的信号会被自动阻塞。
    6.  返回0表示成功。

### `void do_signal(long signr, long eax, ..., long ss)`

*   **功能**: 这是实际执行信号处理的核心函数。它在从系统调用或中断返回到用户态之前被调用（通常由 `system_call.s` 中的 `ret_from_sys_call` 或类似汇编代码在检查到 `current->signal` 非零时调用）。
*   **参数**:
    *   `signr`: 要处理的信号编号 (1-31)。
    *   `eax, ebx, ..., ds`: 这些是当前进程在进入内核时保存的**内核栈**上的寄存器值。`do_signal` 会修改这些值，特别是用户态的 `eip`, `cs`, `eflags`, `esp`, `ss`，以便返回用户态时能跳转到信号处理函数。
    *   `eip, cs, eflags`: 用户态的指令指针、代码段、标志寄存器。
    *   `esp, ss`: 指向用户态栈顶的指针 (`unsigned long * esp`) 和用户态栈段选择子。
*   **步骤**:
    1.  `struct sigaction * sa = current->sigaction + signr - 1;`: 获取该信号对应的 `sigaction` 结构。
    2.  `sa_handler = (unsigned long) sa->sa_handler;`: 获取信号处理函数的地址。
    3.  **处理 `SIG_IGN` (忽略)**: `if (sa_handler == 1) return;` 如果 `sa_handler` 是 `SIG_IGN` (值为1)，则直接返回，不做任何处理。
    4.  **处理 `SIG_DFL` (默认)**: `if (!sa_handler)` (即 `sa_handler` 为0，即 `SIG_DFL`)
        *   `if (signr == SIGCHLD) return;`: 对于 `SIGCHLD`，默认动作是忽略，所以直接返回。
        *   `else do_exit(1 << (signr - 1));`: 对于其他信号，默认动作通常是终止进程。调用 `do_exit()`，退出码中包含了导致终止的信号信息（将信号编号对应的位置1）。
    5.  **准备执行用户提供的信号处理函数**:
        *   `if (sa->sa_flags & SA_ONESHOT) sa->sa_handler = NULL;`: 如果设置了 `SA_ONESHOT`，则将该信号的处理函数重置为 `NULL` (即 `SIG_DFL`)，表示下次再收到此信号时执行默认动作。
        *   `*(&eip) = sa_handler;`: **关键**: 修改内核栈上保存的用户态 `eip` 的值为信号处理函数的地址。这样，当 `iret` 执行时，CPU会跳转到信号处理函数。
        *   `longs = (sa->sa_flags & SA_NOMASK) ? 7 : 8;`: 计算需要在用户栈上构建的栈帧包含多少个长字 (4字节)。
            *   如果设置了 `SA_NOMASK` (处理期间不阻塞当前信号，也不需要在栈上传递旧的 `blocked` 掩码)，则栈帧包含7个长字。
            *   否则，包含8个长字 (包括旧的 `blocked` 掩码)。
        *   `*(&esp) -= longs * 4;`: **关键**: 在用户栈上预留空间。修改内核栈上保存的用户态 `esp` 的值，使其向下移动 `longs * 4` 字节。
        *   `verify_area(esp, longs * 4);`: 验证新的用户栈顶区域是否可写。
        *   `tmp_esp = esp;`: `tmp_esp` 指向新的用户栈顶。
        *   **在用户栈上构建信号处理函数的栈帧 (从高地址到低地址)**:
            *   `put_fs_long((long) sa->sa_restorer, tmp_esp++);`: 压入信号恢复器 (`sa_restorer`) 的地址。这是信号处理函数返回后应该跳转到的地址。
            *   `put_fs_long(signr, tmp_esp++);`: 压入信号编号 `signr`，作为信号处理函数的第一个参数。
            *   `if (!(sa->sa_flags & SA_NOMASK)) put_fs_long(current->blocked, tmp_esp++);`: 如果没有 `SA_NOMASK`，则压入当前进程在进入信号处理前的信号阻塞掩码 `current->blocked`。
            *   `put_fs_long(eax, tmp_esp++); ... put_fs_long(old_eip, tmp_esp++);`: 依次压入 `eax`, `ecx`, `edx`, `eflags`, `old_eip` (原始的用户态 `eip`)。这些值可以被 `sa_restorer` (通过 `sigreturn` 系统调用) 用来恢复信号处理前的上下文。
                *   注意：这里压入的 `eax`, `ecx`, `edx` 是内核栈上保存的进入内核时的值，并非用户态的。`eflags` 和 `old_eip` 是用户态的。
    6.  **更新进程的信号阻塞掩码**:
        *   `current->blocked |= sa->sa_mask;`: 将 `sigaction` 结构中指定的 `sa_mask` (以及通常包含当前信号 `signr` 本身，除非有 `SA_NOMASK`) 加入到当前进程的阻塞信号集中。这确保了在执行信号处理函数期间，这些信号不会再次中断它。

## 总结

`kernel/signal.c` 负责Linux 0.11中信号机制的核心实现。它提供了设置信号处理行为（`sys_signal`, `sys_sigaction`）和管理信号屏蔽（`sys_sgetmask`, `sys_ssetmask`）的系统调用接口。最关键的部分是 `do_signal` 函数，它在内核即将返回用户空间时被调用，负责检查是否有待处理的信号。如果有，它会巧妙地修改内核栈上保存的用户态返回地址 (`eip`) 和栈指针 (`esp`)，强迫用户进程在返回后首先执行指定的信号处理函数。同时，它会在用户栈上构建一个新的栈帧，包含信号处理所需的信息以及一个特殊的返回地址（信号恢复器 `sa_restorer`），以便信号处理函数结束后能够正确恢复现场（通常通过 `sigreturn` 系统调用，其实现未在此文件中）。这个过程是实现异步信号处理的关键。
