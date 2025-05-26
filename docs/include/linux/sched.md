# sched.h 文件详解

`include/linux/sched.h` 是 Linux 0.11 内核中与进程调度 (scheduling) 和进程管理相关的最核心的头文件。它定义了进程的核心数据结构 `struct task_struct` (任务结构，也称进程控制块 PCB)，以及与进程状态、调度策略、信号处理、内存管理（LDT、TSS）、协处理器使用等相关的常量、宏和函数原型。

这个文件是理解 Linux 0.11 如何管理和调度进程的关键。

## 核心功能

1.  **定义系统常量**:
    *   `NR_TASKS`: 系统最大进程数。
    *   `HZ`: 系统时钟频率 (每秒产生的时钟中断次数)。
2.  **定义进程状态**: `TASK_RUNNING`, `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `TASK_ZOMBIE`, `TASK_STOPPED`。
3.  **定义核心数据结构**:
    *   `struct i387_struct`: 保存 FPU (浮点协处理器) 的上下文。
    *   `struct tss_struct`: 任务状态段 (TSS) 结构。
    *   `struct task_struct`: 进程控制块 (PCB)，包含了进程的所有信息。
4.  **声明外部全局变量**:
    *   `task[NR_TASKS]`: 全局进程表。
    *   `current`: 指向当前正在运行进程的 `task_struct`。
    *   `jiffies`: 系统启动以来的时钟滴答计数。
    *   `startup_time`: 系统启动的Unix时间戳。
5.  **声明核心调度和同步函数**:
    *   `sched_init()`: 调度器初始化。
    *   `schedule()`: 执行进程调度。
    *   `sleep_on()`, `interruptible_sleep_on()`, `wake_up()`: 进程睡眠和唤醒机制。
6.  **定义与GDT、LDT、TSS相关的宏**: 用于加载和管理这些段描述符和选择子。
7.  **定义进程切换宏**: `switch_to(n)`。
8.  **定义段基址和限长操作宏**: `set_base`, `set_limit`, `get_base`, `get_limit`。

## 宏和常量定义详解

### 系统参数

*   `#define NR_TASKS 64`: 系统允许的最大进程（任务）数量。
*   `#define HZ 100`: 系统时钟频率为100Hz，即每秒发生100次时钟中断。每次中断称为一个“滴答 (jiffy)”。

### 任务指针宏

*   `#define FIRST_TASK task[0]`: 指向任务数组的第一个任务（通常是init进程或idle进程的前身）。
*   `#define LAST_TASK task[NR_TASKS-1]`: 指向任务数组的最后一个任务。

### 进程状态 (`task_struct->state`)

*   `#define TASK_RUNNING 0`: 进程正在运行或在就绪队列中等待运行。
*   `#define TASK_INTERRUPTIBLE 1`: 进程处于可中断的睡眠状态，等待某个事件发生（如资源可用、信号）。可以被信号唤醒。
*   `#define TASK_UNINTERRUPTIBLE 2`: 进程处于不可中断的睡眠状态，通常用于等待硬件I/O完成。不能被信号唤醒，必须等待特定事件。
*   `#define TASK_ZOMBIE 3`: 僵尸进程。进程已终止，但其父进程尚未通过 `wait()` 读取其退出状态。进程的task_struct仍然保留。
*   `#define TASK_STOPPED 4`: 进程已停止（例如，被 `SIGSTOP` 或 `SIGTSTP` 信号暂停）。

### NULL 定义

*   `#ifndef NULL #define NULL ((void *) 0) #endif`

### `copy_page_tables()` 和 `free_page_tables()`

*   `extern int copy_page_tables(unsigned long from, unsigned long to, long size);`
    *   功能: 复制页表。通常在 `fork()` 时用于复制父进程的页表给子进程，或者实现写时复制。
    *   参数: `from` (源线性地址基址), `to` (目标线性地址基址), `size` (要复制的内存区域大小)。
*   `extern int free_page_tables(unsigned long from, unsigned long size);`
    *   功能: 释放指定线性地址范围 `from` 到 `from+size` 所占用的页表（不是物理内存页本身）。通常在进程退出或 `exec` 时调用。

## 核心数据结构定义

### `struct i387_struct`

*   **功能**: 保存 Intel 387 浮点协处理器 (FPU) 的上下文状态。
*   **成员**: 包括控制字 `cwd`, 状态字 `swd`, 标记字 `twd`, 指令指针 `fip`, 操作数指针 `foo`, 以及8个浮点寄存器 `st_space[20]` (每个寄存器10字节，共80字节)。

### `struct tss_struct` (任务状态段)

*   **功能**: 定义了 x86 架构的任务状态段 (TSS) 的结构。TSS 用于保存任务切换时CPU的完整上下文。在 Linux 0.11 中，每个进程都有一个TSS。
*   **主要成员**:
    *   `back_link`: 前一个任务的TSS选择子 (在硬件任务切换中使用，Linux 0.11主要用软件切换)。
    *   `esp0`, `ss0`: 0特权级 (内核态) 的栈指针和栈段选择子。当中断或系统调用从用户态进入内核态时，CPU会自动加载这两个值。
    *   `esp1`, `ss1`, `esp2`, `ss2`: (未使用) 1和2特权级的栈。
    *   `cr3`: 页目录基址寄存器 (PDBR)。
    *   `eip`, `eflags`, `eax`, `ecx`, `edx`, `ebx`, `esp`, `ebp`, `esi`, `edi`: 通用寄存器和程序状态。
    *   `es`, `cs`, `ss`, `ds`, `fs`, `gs`: 段寄存器选择子。
    *   `ldt`: 该任务的局部描述符表 (LDT) 的选择子。
    *   `trace_bitmap`: I/O许可位图偏移和调试陷阱标志 (T-bit)。
    *   `struct i387_struct i387`: 保存该任务的FPU状态。

### `struct task_struct` (进程控制块 PCB)

*   **功能**: 这是内核中最重要的结构体之一，用于描述一个进程（或任务）的所有信息。系统中的所有 `task_struct` 结构体都存储在 `task[]` 数组中。
*   **主要成员分组**:
    *   **调度信息**:
        *   `long state`: 进程当前的状态 (如 `TASK_RUNNING`, `TASK_INTERRUPTIBLE`)。
        *   `long counter`: 进程剩余的时间片计数。调度器会选择 `counter` 最大的就绪进程运行。当 `counter` 耗尽时，需要重新计算。
        *   `long priority`: 进程的静态优先级 (在0.11中，所有用户进程优先级相同，为15)。
    *   **信号处理**:
        *   `long signal`: 一个位图，表示进程已接收到但尚未处理的信号。
        *   `struct sigaction sigaction[32]`: 每个信号对应的处理动作（`struct sigaction` 在 `<signal.h>` 定义）。
        *   `long blocked`: 一个位图，表示被阻塞（屏蔽）的信号。
    *   **进程标识和关系**:
        *   `int exit_code`: 进程的退出码。
        *   `long pid`: 进程ID。
        *   `long father`: 父进程的PID (Linux 0.11 中实际存储的是 `task` 数组的索引，需要转换)。
        *   `long pgrp`: 进程组ID。
        *   `long session`: 会话ID。
        *   `long leader`: 是否是会话领导者 (通常是会话ID等于PID)。
    *   **用户和组ID**:
        *   `unsigned short uid, euid, suid`: 用户ID, 有效用户ID, 保存的设置用户ID。
        *   `unsigned short gid, egid, sgid`: 组ID, 有效组ID, 保存的设置组ID。
    *   **内存管理 (用户空间)**:
        *   `unsigned long start_code, end_code`: 代码段的起始和结束地址。
        *   `unsigned long end_data`: 数据段的结束地址 (BSS段紧随其后)。
        *   `unsigned long brk`: 进程当前的堆顶（break）。
        *   `unsigned long start_stack`: 用户栈的起始地址。
    *   **时间信息**:
        *   `long alarm`: 报警定时器的剩余滴答数。
        *   `long utime, stime`: 进程在用户态和内核态分别执行的时间 (单位：滴答)。
        *   `long cutime, cstime`: 子进程（已终止并被wait）在用户态和内核态执行的总时间。
        *   `long start_time`: 进程创建的时间 (相对于系统启动的 `startup_time`)。
    *   **协处理器使用**:
        *   `unsigned short used_math`: 标记进程是否使用了数学协处理器。
    *   **文件系统信息**:
        *   `int tty`: 控制终端的次设备号 (-1表示没有)。
        *   `unsigned short umask`: 文件创建模式屏蔽码。
        *   `struct m_inode * pwd`: 当前工作目录的 inode。
        *   `struct m_inode * root`: 根目录的 inode。
        *   `struct m_inode * executable`: 指向该进程对应的可执行文件的 inode (用于需求加载)。
        *   `unsigned long close_on_exec`: 一个位图，标记哪些文件描述符在 `exec` 时需要关闭。
        *   `struct file * filp[NR_OPEN]`: 进程打开的文件描述符表。
    *   **LDT 和 TSS**:
        *   `struct desc_struct ldt[3]`: 该进程的局部描述符表 (LDT)。包含3个描述符：
            *   `ldt[0]`: NULL 描述符。
            *   `ldt[1]`: 用户代码段描述符。
            *   `ldt[2]`: 用户数据段和堆栈段描述符。
        *   `struct tss_struct tss`: 该进程的任务状态段 (TSS)。

### `INIT_TASK` 宏

*   **功能**: 用于静态初始化系统中第一个任务 (task[0]，即init进程的前身) 的 `task_struct` 结构。
*   **内容**: 为 `task_struct` 的所有字段提供初始值。例如：
    *   `state = 0` (TASK_RUNNING)。
    *   `counter = 15`, `priority = 15`。
    *   PID=0, father=-1 (无父进程)。
    *   LDT[1] (代码段) 和 LDT[2] (数据段) 的基址为0，限长为0x9FFFF (640KB - 1)。这是早期Linux假定用户空间在640KB内的设定。
    *   TSS的 `esp0` 指向 `PAGE_SIZE + (long)&init_task` (内核栈顶)，`ss0` 为内核数据段选择子 (0x10)，`cr3` 指向内核页目录 `pg_dir`，`ldt` 指向 `task[0]` 的LDT选择子。`i387` 为空。

## 全局变量声明

*   `extern struct task_struct *task[NR_TASKS];`: 全局进程表，是一个指向 `task_struct` 的指针数组。
*   `extern struct task_struct *last_task_used_math;`: 指向上一个使用过数学协处理器的任务。
*   `extern struct task_struct *current;`: 一个非常重要的指针，指向当前正在CPU上运行的进程的 `task_struct`。
*   `extern long volatile jiffies;`: 系统启动以来的时钟滴答数。`volatile` 确保编译器每次都从内存读取它。
*   `extern long startup_time;`: 系统启动时的Unix时间戳 (从1970年1月1日0时起的秒数)。

### `CURRENT_TIME` 宏

*   `#define CURRENT_TIME (startup_time + jiffies / HZ)`: 计算当前的Unix时间戳。

## 核心函数声明

*   `extern void add_timer(long jiffies, void (*fn)(void));`: 添加一个定时器，在指定的 `jiffies` 滴答后执行函数 `fn`。
*   `extern void sleep_on(struct task_struct ** p);`: 使当前进程在等待队列 `*p` 上进入不可中断睡眠。`*p` 会被设置为指向当前进程。
*   `extern void interruptible_sleep_on(struct task_struct ** p);`: 使当前进程在等待队列 `*p` 上进入可中断睡眠。
*   `extern void wake_up(struct task_struct ** p);`: _唤醒_在等待队列 `*p` 上睡眠的进程（将其状态设为 `TASK_RUNNING`）。`*p` 会被清零。

## GDT/LDT/TSS 相关宏

这些宏用于操作GDT中的TSS和LDT描述符，以及加载相应的CPU寄存器。

*   `#define FIRST_TSS_ENTRY 4`: GDT中第一个TSS描述符的索引 (0=NULL, 1=内核代码, 2=内核数据, 3=系统调用TSS(未使用), 4=任务0的TSS)。
*   `#define FIRST_LDT_ENTRY (FIRST_TSS_ENTRY + 1)`: GDT中第一个LDT描述符的索引 (紧随第一个TSS之后)。
*   `#define _TSS(n) ((((unsigned long) n) << 4) + (FIRST_TSS_ENTRY << 3))`: 计算任务 `n` (即 `task[n]`) 的TSS选择子。
    *   `n` 是任务在 `task[]` 数组中的索引。
    *   `FIRST_TSS_ENTRY << 3`: GDT中第0个任务（task[0]）的TSS描述符的起始偏移量（字节）。
    *   `((unsigned long) n) << 4`: 每个任务的TSS和LDT在GDT中占两个描述符（16字节）。所以 `n*16` 是任务 `n` 的TSS描述符相对于第0个任务的TSS描述符的偏移。
    *   实际上，Linux 0.11中GDT的布局是：TSS0, LDT0, TSS1, LDT1, ... 所以任务 `n` 的TSS索引是 `FIRST_TSS_ENTRY + 2*n`，LDT索引是 `FIRST_LDT_ENTRY + 2*n`。
    *   因此，`_TSS(n)` 更准确的理解是： `(( (FIRST_TSS_ENTRY) + 2*(n) ) << 3 )` (TI=0, RPL=0)。
    *   `_LDT(n)` 类似地计算任务 `n` 的LDT选择子。
*   `#define ltr(n) __asm__("ltr %%ax"::"a" (_TSS(n)))`: 加载任务寄存器 (TR)，使其指向任务 `n` 的TSS。
*   `#define lldt(n) __asm__("lldt %%ax"::"a" (_LDT(n)))`: 加载LDT寄存器 (LDTR)，使其指向任务 `n` 的LDT。
*   `#define str(n)`: (Store Task Register) 读取当前TR的值（选择子），并从中计算出当前任务的索引 `n`。

### `switch_to(n)` 宏

*   **功能**: 实现进程上下文切换到任务 `n` (即 `task[n]`)。
*   **实现**:
    1.  `cmpl %%ecx, _current`: 比较要切换到的任务 `task[n]` (`ecx` 寄存器中) 是否就是当前任务 `_current`。如果是，则跳转到 `1f` 直接结束。
    2.  `movw %%dx, %1`: `%1` 是 `__tmp.b`，`dx` 中是任务 `n` 的TSS选择子 `_TSS(n)`。这是 `ljmp` 指令的操作数的一部分。
    3.  `xchgl %%ecx, _current`: 原子地将 `task[n]` (在 `ecx` 中) 与全局变量 `_current` 交换。`_current` 现在指向新任务，`ecx` 中是旧任务。
    4.  `ljmp %0`: 执行长跳转 (`ljmp segment_selector:offset`) 到新任务的TSS。
        *   `%0` 是 `__tmp.a`。`__tmp` 的设置 (`struct {long a,b;}`) 是用来构造 `ljmp` 指令的操作数。`ljmp` 到一个TSS选择子会触发硬件任务切换（如果TSS描述符类型指示硬件切换），或者在Linux 0.11中，这更像是一个技巧，通过 `ljmp` 来改变 `cs:eip`，并配合后续的栈和寄存器恢复（或者依赖TSS的自动加载）。
        *   Linux 0.11 的 `switch_to` 主要依赖于 `ljmp` 到TSS选择子。这个 `ljmp` 会导致CPU加载新任务的TSS中保存的所有寄存器状态（包括 `eip`, `esp`, 段寄存器, `cr3` 等），从而完成上下文切换。新任务从其TSS中保存的 `eip` 处开始执行。
    5.  `cmpl %%ecx, _last_task_used_math`: (切换回来后，`ecx` 中是旧任务) 比较旧任务是否是上一个使用FPU的任务。
    6.  `jne 1f`: 如果不是，则跳转。
    7.  `clts`: 如果是，则清除CR0控制寄存器中的TS (Task Switched) 标志。TS标志在每次任务切换时由硬件设置。如果新任务要使用FPU，而TS位已设置，会产生一个设备不可用异常（int 7），此时内核会保存旧任务的FPU状态，恢复新任务的FPU状态，然后清除TS位。这里的 `clts` 是在判断如果“我们切换到的这个任务（现在是current）就是上次用FPU的那个，那么就清除TS”，这似乎是为了避免不必要的FPU异常。
    8.  `1:`: 结束点。

### 段基址和限长操作宏

这些宏用于从段描述符（特别是LDT中的描述符）中获取或设置32位的基地址和20位的界限。段描述符是8字节。

*   `#define PAGE_ALIGN(n) (((n)+0xfff)&0xfffff000)`: 将地址 `n` 向上对齐到4KB页边界。

*   `_set_base(addr, base)` 和 `set_base(ldt, base)`:
    *   将 `ldt` (一个 `desc_struct` 结构体，即8字节描述符) 的基地址字段设置为 `base`。
    *   基地址在描述符中分散在多个位置：字节2,3 (低16位)，字节4 (中8位)，字节7 (高8位)。
    *   `_set_base` 通过内联汇编将 `base` 的相应部分写入到 `addr` 指向的描述符的这些字节中。

*   `_set_limit(addr, limit)` 和 `set_limit(ldt, limit)`:
    *   将 `ldt` 的界限字段设置为 `limit`。
    *   注意：`set_limit` 的第二个参数 `limit` 是实际的字节限长，在传递给 `_set_limit` 之前会进行处理：`(limit-1) >> 12`。这是因为如果描述符的G位（粒度位）为1，则界限是以4KB页为单位的。所以 `(limit-1)` 先得到最大偏移，然后 `>>12` 转换为4KB页数。
    *   界限在描述符中分散在字节0,1 (低16位) 和字节6的低4位 (高4位)。
    *   `_set_limit` 通过内联汇编将处理后的 `limit` 值写入这些位置。

*   `_get_base(addr)` 和 `get_base(ldt)`:
    *   从 `ldt` 描述符中提取并组合出32位的基地址。

*   `get_limit(segment)`:
    *   使用 `lsll` (Load Segment Limit) 汇编指令从指定的段选择子 `segment` 获取该段的字节限长。
    *   `lsll %1, %0`: 将选择子 `%1` (segment) 的段限长加载到寄存器 `%0` (`__limit`)。
    *   `incl %0`: `lsll` 返回的是最后一个可访问字节的偏移（即 `limit_in_bytes - 1` 如果G=0，或者 `(limit_in_pages * 4096) - 1` 如果G=1），所以加1得到实际的字节大小。

## 总结

`include/linux/sched.h` 是 Linux 0.11 内核的神经中枢之一。它通过 `struct task_struct` 定义了进程的完整画像，包含了调度、信号、内存、文件系统、权限、定时器等几乎所有与进程相关的信息。围绕 `task_struct`，此文件还定义了进程状态、初始化宏 (`INIT_TASK`)、全局进程表和当前进程指针。更进一步，它提供了进程调度 (`schedule`, `sleep_on`, `wake_up`)、上下文切换 (`switch_to`)、以及与底层硬件（TSS, LDT, GDT, FPU）交互所需的宏和函数原型。这些定义和接口共同构成了 Linux 0.11 多任务操作系统的核心调度与管理框架。Okay, I have created `docs/include/linux/sched.md` with the detailed Chinese explanation for `include/linux/sched.h`.

Next, I will process `include/linux/sys.h`.
1. Create `docs/include/linux/sys.md`.
2. Read `include/linux/sys.h`.
3. Write the explanation into `docs/include/linux/sys.md`.

First, I'll read `include/linux/sys.h`.
