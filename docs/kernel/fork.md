# kernel/fork.c 文件详解

`kernel/fork.c` 文件是 Linux 0.11 内核中实现 `fork()` 系统调用（创建新进程）的核心辅助程序。它包含了创建新进程时复制父进程信息、设置新进程上下文、分配资源等关键逻辑。`fork()` 系统调用是 Unix-like 系统中创建新进程的主要方式，它创建一个与父进程几乎完全相同的子进程。

此文件中的核心函数是 `copy_process()`，它被 `sys_fork()`（在 `kernel/system_call.s` 中定义，是 `fork()` 的入口点）调用。此外，文件还包含 `verify_area()`（用于检查内存区域访问权限，但主要在 `fs/exec.c` 等处被更广泛使用其声明）和 `copy_mem()`（用于复制父进程的内存空间给子进程，核心是复制页表）。

## 核心功能

1.  **内存区域验证 (`verify_area`)**: 检查指定内存区域是否对当前进程可写（用于确保用户提供的缓冲区有效）。
2.  **内存复制 (`copy_mem`)**: 为新进程复制父进程的内存映像。在 Linux 0.11 中，这主要通过 `copy_page_tables()` 实现，采用**写时复制 (Copy-On-Write, COW)** 机制。即父子进程初始共享相同的物理内存页，只有当其中一方尝试写入共享页时，才会为写入方复制一个新的物理页。
3.  **进程创建核心逻辑 (`copy_process`)**:
    *   为新进程分配 `task_struct` 结构。
    *   复制父进程的 `task_struct` 内容到子进程。
    *   为子进程设置唯一的PID。
    *   初始化子进程的状态、调度信息（如 `counter`, `priority`）、信号、定时器等。
    *   设置子进程的TSS (任务状态段)，包括其内核栈指针 (`esp0`)、寄存器值（从父进程中断时的栈帧中获取）、LDT选择子等。
    *   如果父进程使用了数学协处理器，则保存父进程的FPU状态到子进程的TSS中。
    *   调用 `copy_mem()` 复制内存空间。
    *   复制父进程打开的文件描述符表 (`filp`)，并增加相应 `struct file` 的引用计数。
    *   增加父进程当前工作目录 (`pwd`)、根目录 (`root`) 和可执行文件 (`executable`) inode的引用计数。
    *   为子进程在GDT中设置新的TSS描述符和LDT描述符。
    *   将子进程状态设为 `TASK_RUNNING`，使其可以被调度。
4.  **查找空闲进程槽 (`find_empty_process`)**:
    *   为新进程查找一个未使用的 `task_struct` 在全局 `task[]` 数组中的槽位。
    *   分配一个新的、唯一的PID。

## 关键数据结构和宏

*   **`struct task_struct`**: 进程控制块，包含了进程的所有信息。`copy_process` 的核心就是填充这个结构。
*   **`task[NR_TASKS]`**: 全局进程表数组。
*   **`last_pid`**: 全局变量，用于生成新的PID。
*   **LDT (Local Descriptor Table)**: 每个进程拥有自己的LDT，定义了用户空间的段（代码段、数据/堆栈段）。`copy_process` 会为子进程设置LDT。
*   **TSS (Task State Segment)**: 每个进程拥有自己的TSS，用于保存任务切换时的CPU上下文。`copy_process` 会初始化子进程的TSS。
*   **页表**: `copy_page_tables()` 函数（在 `mm/memory.c` 中实现）用于复制页表，实现内存共享和写时复制。

## 主要函数详解

### `void verify_area(void * addr, int size)`

*   **功能**: 验证从 `addr` 开始，长度为 `size` 的内存区域对于当前进程是否可写。主要用于检查用户空间指针的有效性。
*   **步骤**:
    1.  `start = (unsigned long) addr;`: 获取起始地址。
    2.  `size += start & 0xfff;`: 将 `size` 调整为包含起始地址页内偏移的总长度。
    3.  `start &= 0xfffff000;`: 将 `start` 向下对齐到页边界。
    4.  `start += get_base(current->ldt[2]);`: 将用户空间的线性地址 `start` 转换为相对于当前进程数据段基址的地址（虽然在0.11的平坦模型下，LDT[2]的基址通常是进程的起始线性地址，如 `nr * 0x4000000`，所以这一步是将逻辑段内地址转换为“绝对”线性地址，但如果 `get_base` 返回的是0，则无影响）。
        *   更准确地说，`get_base(current->ldt[2])` 获取的是当前进程数据段的线性基地址。`start` 是用户传入的指针，是用户段内的偏移。所以 `start + get_base(...)` 是该地址在整个线性地址空间中的位置。
    5.  **循环检查每一页**: `while (size > 0)`
        *   `size -= 4096;`: 每次处理一页。
        *   `write_verify(start);`: 调用 `write_verify` (通常在 `mm/memory.c` 中定义) 检查 `start` 指向的页面是否可写。`write_verify` 会尝试向该页写入一个字节（或仅检查页表项的写权限位），如果不可写，会触发页错误（缺页异常）。
        *   `start += 4096;`: 移动到下一页。
*   **注意**: 这个函数通过实际尝试“验证写”（可能导致缺页异常，由缺页处理程序处理）来确保区域的有效性。

### `int copy_mem(int nr, struct task_struct * p)`

*   **功能**: 为新进程 `p` (在 `task[nr]` 中) 复制父进程 (`current`) 的内存空间。
*   **步骤**:
    1.  获取父进程代码段和数据段的限长 (`code_limit`, `data_limit`) 及基址 (`old_code_base`, `old_data_base`)，这些信息从父进程的LDT中读取。
    2.  **检查I&D分离**: `if (old_data_base != old_code_base) panic("We don't support separate I&D");` Linux 0.11 不支持代码段和数据段分离的模式（即它们必须有相同的基址）。
    3.  **检查限长**: `if (data_limit < code_limit) panic("Bad data_limit");` 数据段限长不应小于代码段限长。
    4.  **计算新进程基址**: `new_data_base = new_code_base = nr * 0x4000000;`
        *   为新进程分配一个新的64MB线性地址空间块。`nr` 是新进程在 `task[]` 数组中的索引。每个进程被分配一个独立的64MB逻辑地址空间（这是0.11的设计）。
    5.  **设置子进程LDT**:
        *   `p->start_code = new_code_base;`: 记录子进程代码段起始地址。
        *   `set_base(p->ldt[1], new_code_base);`: 设置子进程LDT中代码段描述符的基址。
        *   `set_base(p->ldt[2], new_data_base);`: 设置子进程LDT中数据/堆栈段描述符的基址。
        *   (段限长通常与父进程相同，由 `*p = *current;` 复制而来，之后可能会调整)。
    6.  **复制页表**: `if (copy_page_tables(old_data_base, new_data_base, data_limit))`
        *   调用 `copy_page_tables()` (在 `mm/memory.c`) 将父进程的页表复制给子进程。
        *   `old_data_base`: 父进程线性地址空间的基址。
        *   `new_data_base`: 子进程线性地址空间的基址。
        *   `data_limit`: 要复制的地址空间范围限长。
        *   `copy_page_tables` 实现写时复制 (COW)：它会复制父进程的页目录和页表，但将所有用户数据页标记为只读。物理页面是共享的。当父或子进程尝试写入这些共享页时，会触发缺页异常，此时内核才真正为写入方分配一个新的物理页并复制内容。
    7.  **错误处理**: 如果 `copy_page_tables` 失败 (如内存不足以分配页表)，则调用 `free_page_tables` 清理已为子进程分配的页表，并返回 `-ENOMEM`。
    8.  返回0表示成功。

### `int copy_process(int nr, long ebp, ..., long ss)`

*   **功能**: 创建一个新进程（子进程），它是当前进程 (`current`) 的副本。这是 `fork()` 系统调用的核心实现。
*   **参数**:
    *   `nr`: 新进程在 `task[]` 数组中的索引，由 `find_empty_process()` 分配。
    *   `ebp, edi, esi, gs, ..., ds`: 这些是父进程在执行 `int 0x80` 系统调用时，由 `system_call.s` 中的汇编代码压入内核栈的寄存器值。它们将用于初始化子进程的内核栈和TSS，使得子进程从系统调用返回时，看起来就像是从 `fork()` 调用返回一样。
    *   `eip, cs, eflags, esp, ss`: 这是中断发生时CPU自动压栈的用户态寄存器值，代表了父进程执行 `fork()` 时的状态。`eip` 是 `fork()` 之后的指令地址。这些值会成为子进程TSS中的初始用户态上下文。
*   **步骤**:
    1.  **分配 `task_struct`**: `p = (struct task_struct *) get_free_page();` 为新进程分配一页内存用于存储其 `task_struct`。如果失败，返回 `-EAGAIN`。
    2.  `task[nr] = p;`: 将新分配的 `task_struct` 放入全局任务数组。
    3.  **复制父进程信息**: `*p = *current;` 结构体整体复制。**注意**: 注释中提到这不包括管理栈（内核栈），因为子进程会有自己的内核栈（通常是 `task_struct` 所在页的末尾）。
    4.  **初始化子进程特定字段**:
        *   `p->state = TASK_UNINTERRUPTIBLE;`: 先设为不可中断，防止在未完全设置好时被调度。
        *   `p->pid = last_pid;`: 设置新的PID (由 `find_empty_process` 更新的 `last_pid`)。
        *   `p->father = current->pid;`: 设置父进程PID。
        *   `p->counter = p->priority;`: 初始化时间片。
        *   `p->signal = 0; p->alarm = 0; p->leader = 0;`: 清除信号、定时器，子进程不继承会话领导权。
        *   `p->utime = p->stime = 0; p->cutime = p->cstime = 0;`: 清零CPU时间统计。
        *   `p->start_time = jiffies;`: 设置启动时间。
    5.  **初始化子进程TSS (`p->tss`)**:
        *   `p->tss.back_link = 0;`
        *   `p->tss.esp0 = PAGE_SIZE + (long) p;`: 设置子进程的内核栈顶指针 (`esp0`)。当子进程从用户态进入内核态时，CPU会使用这个栈。它指向 `task_struct` 所在页的末尾。
        *   `p->tss.ss0 = 0x10;`: 内核数据段选择子。
        *   `p->tss.eip = eip; ... p->tss.gs = gs & 0xffff;`: 用从父进程中断栈中获取的寄存器值填充TSS中的对应字段。这些是子进程首次返回用户态时的初始CPU状态。
            *   对于段寄存器，只取低16位 (选择子)。
            *   `p->tss.eax = 0;`: **关键**: 子进程的 `eax` 被设置为0。这就是为什么 `fork()` 在子进程中返回0的原因。
        *   `p->tss.ldt = _LDT(nr);`: 设置子进程LDT在GDT中的选择子。`_LDT(nr)` (在 `<linux/sched.h>`) 计算出任务 `nr` 的LDT描述符的选择子。
        *   `p->tss.trace_bitmap = 0x80000000;`: I/O许可位图偏移量设为无效（超出TSS界限），表示没有I/O许可位图。
    6.  **处理FPU状态**:
        *   `if (last_task_used_math == current) __asm__("clts ; fnsave %0"::"m" (p->tss.i387));`
            *   如果父进程是最后一个使用数学协处理器的任务，则：
                *   `clts`: 清除CR0控制寄存器中的TS (Task Switched) 标志。
                *   `fnsave %0`: 将当前FPU的状态保存到子进程TSS的 `i387` 区域。这样子进程开始时就拥有父进程的FPU副本。
    7.  **复制内存**: `if (copy_mem(nr, p)) { ... return -EAGAIN; }` 调用 `copy_mem` 为子进程设置内存空间和页表。如果失败，则清理已分配的 `task_struct` 并返回错误。
    8.  **复制文件信息**:
        *   `for (i=0; i<NR_OPEN; i++) if (f = p->filp[i]) f->f_count++;`: 遍历父进程打开的文件描述符表（已复制到子进程 `p->filp`），对每个打开的文件，将其对应的 `struct file` 的引用计数 `f_count` 加1。这实现了父子进程共享打开的文件表项。
    9.  **增加inode引用计数**:
        *   `if (current->pwd) current->pwd->i_count++;`: 增加当前工作目录 inode 的引用计数。
        *   `if (current->root) current->root->i_count++;`: 增加根目录 inode 的引用计数。
        *   `if (current->executable) current->executable->i_count++;`: 增加可执行文件 inode 的引用计数。
        *   子进程会继承这些 inode 指针，所以需要增加它们的引用。
    10. **设置GDT中的TSS和LDT描述符**:
        *   `set_tss_desc(gdt + (nr << 1) + FIRST_TSS_ENTRY, &(p->tss));`
        *   `set_ldt_desc(gdt + (nr << 1) + FIRST_LDT_ENTRY, &(p->ldt));`
        *   在全局描述符表 (GDT) 中为子进程 `nr` 设置其TSS描述符和LDT描述符。
            *   `gdt`: GDT的基址。
            *   `(nr << 1)`: 每个任务在GDT中通常占用两个描述符（一个TSS，一个LDT）。`nr << 1` (即 `nr * 2`) 计算出相对于第一个任务（task0）的描述符组的偏移量。
            *   `FIRST_TSS_ENTRY` 和 `FIRST_LDT_ENTRY`: GDT中第一个TSS和LDT描述符的索引（在 `<linux/sched.h>` 中定义）。
            *   `&(p->tss)` 和 `&(p->ldt)`: 子进程TSS和LDT结构的线性地址。
            *   `set_tss_desc` 和 `set_ldt_desc` 宏 (在 `<asm/system.h>`) 用于填充这些描述符。
    11. **设置子进程状态为可运行**: `p->state = TASK_RUNNING;` **注意**: 注释强调这是最后一步，以防在设置过程中发生意外。
    12. **返回子进程PID**: `return last_pid;` (即子进程的PID)。这个返回值是给父进程的。子进程的返回值已在TSS中被设为0。

### `int find_empty_process(void)`

*   **功能**: 查找 `task[]` 数组中的一个空闲槽位以创建新进程，并为新进程分配一个唯一的PID。
*   **步骤**:
    1.  **分配新PID**:
        *   `repeat:` 标签。
        *   `if ((++last_pid) < 0) last_pid = 1;`: 全局变量 `last_pid` 递增。如果溢出（变为负数），则重置为1。PID 0 通常为idle/sched进程保留。
        *   **检查PID是否已存在**: `for(i=0 ; i<NR_TASKS ; i++) if (task[i] && task[i]->pid == last_pid) goto repeat;` 遍历 `task` 数组，如果新生成的 `last_pid` 已被某个活动进程使用，则 `goto repeat` 重新生成。
    2.  **查找空闲 `task` 槽位**:
        *   `for(i=1 ; i<NR_TASKS ; i++) if (!task[i]) return i;`: 从索引1开始遍历 `task` 数组 (task[0]是init进程的前身，通常不通过此方式分配)。如果找到一个 `task[i]` 为 `NULL` 的槽位，则返回该索引 `i`。这个索引将作为新进程在 `task[]` 数组中的编号，也用于计算其LDT和TSS在GDT中的位置。
    3.  **无空闲槽位**: `return -EAGAIN;` 如果遍历完都找不到空闲槽位，表示系统进程数已达上限 (`NR_TASKS`)，返回错误。

## 总结

`kernel/fork.c` 是Linux 0.11实现多任务的核心。`copy_process` 函数细致地完成了创建新进程所需的所有步骤：从分配和初始化 `task_struct`，复制父进程状态（寄存器、内存、打开文件、目录引用），到为子进程设置独立的TSS和LDT，并将其置于可调度状态。写时复制 (COW) 机制的运用（通过 `copy_page_tables`）是提高 `fork()` 效率的关键。`find_empty_process` 则保证了新进程PID的唯一性和 `task` 数组槽位的有效分配。这个文件深刻体现了早期操作系统内核在进程复制和资源管理方面的设计与挑战。
