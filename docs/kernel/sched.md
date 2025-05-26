# kernel/sched.c 文件详解

`kernel/sched.c` 是 Linux 0.11 内核的**核心调度程序**和进程管理的中枢。这个文件包含了进程调度的基本原语（如 `schedule()`、`sleep_on()`、`wake_up()`）、时钟中断处理、进程状态管理以及一些简单的系统调用实现（如 `getpid()`）。它是实现多任务操作系统的关键。

Linux 0.11 采用了一种简单但有效的基于优先级和时间片的调度算法。

## 核心功能

1.  **进程调度 (`schedule`)**:
    *   实现主要的调度逻辑，选择下一个要运行的进程。
    *   处理进程的报警信号 (`SIGALRM`)。
    *   唤醒因收到信号而变为可运行的、处于可中断睡眠状态的进程。
    *   当没有其他可运行进程时，选择任务0 (idle任务) 运行。
2.  **进程睡眠与唤醒**:
    *   `sleep_on(p)`: 使当前进程在等待队列 `p` 上进入不可中断睡眠。
    *   `interruptible_sleep_on(p)`: 使当前进程在等待队列 `p` 上进入可中断睡眠。
    *   `wake_up(p)`: 唤醒在等待队列 `p` 上睡眠的第一个进程。
3.  **时钟中断处理 (`do_timer`)**:
    *   由时钟中断处理程序 (`timer_interrupt`) 调用。
    *   更新当前进程的用户/系统CPU时间 (`utime`, `stime`)。
    *   处理内核定时器 (`timer_list`)。
    *   处理软盘马达关闭定时器 (`do_floppy_timer`)。
    *   递减当前进程的时间片 (`counter`)，如果耗尽则调用 `schedule()`。
4.  **任务和系统初始化 (`sched_init`)**:
    *   初始化任务0 (init_task) 的TSS和LDT描述符在GDT中。
    *   清空其他任务槽位在GDT中的描述符。
    *   清除EFLAGS寄存器的NT位 (Nested Task)。
    *   加载任务0的TR (Task Register) 和 LDTR (LDT Register)。
    *   设置8253定时器芯片以产生 `HZ` (100Hz) 的时钟中断。
    *   设置时钟中断门 (`timer_interrupt`) 和系统调用门 (`system_call`)。
5.  **FPU (数学协处理器) 状态管理 (`math_state_restore`)**:
    *   在任务切换时，保存上一个使用FPU任务的状态，并恢复当前任务的FPU状态（如果它之前使用过）。
6.  **软盘马达控制定时器**:
    *   `ticks_to_floppy_on()`, `floppy_on()`, `floppy_off()`, `do_floppy_timer()`: 一组与软盘马达相关的函数，通过定时器管理马达的开启和关闭，以节省能源和减少磨损。注释中提到这些本不应在内核核心调度文件中，但为了方便使用定时器而放在这里。
7.  **内核定时器列表 (`add_timer`, `timer_list`, `next_timer`)**:
    *   实现了一个简单的内核定时器机制，允许内核模块注册在指定时间后执行的函数。
8.  **简单系统调用实现**:
    *   `sys_pause()`: 使进程进入可中断睡眠。
    *   `sys_alarm()`: 设置进程的报警定时器。
    *   `sys_getpid()`, `sys_getppid()`, `sys_getuid()`, `sys_geteuid()`, `sys_getgid()`, `sys_getegid()`: 获取进程的各种ID。
    *   `sys_nice()`: 修改进程的优先级（时间片）。

## 关键数据结构和全局变量

*   **`struct task_struct`**: 进程控制块 (PCB)，定义在 `<linux/sched.h>`。`sched.c` 中包含了对它的初始化和管理。
*   **`union task_union { struct task_struct task; char stack[PAGE_SIZE]; }`**:
    *   将 `task_struct` 和一个页面大小的栈空间联合起来。这意味着每个进程的 `task_struct` 和其内核栈位于同一个4KB的内存页中。`task_struct` 在页的起始处，栈从页的末尾向下增长。
*   **`static union task_union init_task = {INIT_TASK,};`**:
    *   静态定义并初始化任务0 (init_task) 的 `task_union`。`INIT_TASK` 是在 `<linux/sched.h>` 中定义的宏，用于填充 `task_struct` 的初始值。
*   **`long volatile jiffies = 0;`**:
    *   系统启动以来的时钟滴答总数。`volatile` 确保每次访问都从内存读取。
*   **`long startup_time = 0;`**:
    *   系统启动时的Unix时间戳 (从1970年1月1日00:00:00 UTC到系统启动时的秒数)，在 `init/main.c` 的 `time_init()` 中设置。
*   **`struct task_struct *current = &(init_task.task);`**:
    *   指向当前正在运行进程的 `task_struct` 的指针，初始指向任务0。
*   **`struct task_struct *last_task_used_math = NULL;`**:
    *   指向上一个使用了数学协处理器的任务。
*   **`struct task_struct * task[NR_TASKS] = {&(init_task.task), };`**:
    *   全局进程表数组，大小为 `NR_TASKS` (64)。`task[0]` 初始化为指向 `init_task.task`，其他条目在 `sched_init()` 中初始化为 `NULL`。
*   **`long user_stack [ PAGE_SIZE>>2 ] ;`**:
    *   一个页面大小的数组，用作初始用户栈（似乎是为任务0或早期启动阶段准备的，后续用户进程会有自己的栈）。
*   **`struct { long * a; short b; } stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };`**:
    *   `stack_start.a` 指向 `user_stack` 的末尾（栈顶）。
    *   `stack_start.b` (0x10) 是内核数据段选择子，可能用于初始化任务0的某些堆栈信息。
*   **`wait_motor[4]`, `mon_timer[4]`, `moff_timer[4]`, `current_DOR`**: 软盘马达控制相关的静态全局变量。
*   **`struct timer_list timer_list[TIME_REQUESTS], * next_timer`**: 内核定时器列表及其头指针。`TIME_REQUESTS` (64) 是最大定时器请求数。

## 宏定义

*   `#define _S(nr) (1<<((nr)-1))`: 将信号编号 `nr` 转换为对应的位掩码。
*   `#define _BLOCKABLE (~(_S(SIGKILL) | _S(SIGSTOP)))`: 定义一个位图，表示所有可被阻塞的信号（即除了 `SIGKILL` 和 `SIGSTOP` 之外的所有信号）。
*   `#define LATCH (1193180/HZ)`: 计算8253定时器通道0的计数器初始值。1193180 Hz 是8253定时器的输入时钟频率，`HZ` 是期望的中断频率。

## 主要函数详解

### `void math_state_restore()`

*   **功能**: 在任务切换后，恢复当前任务的数学协处理器 (FPU) 状态，并保存上一个使用FPU任务的状态。
*   **逻辑**:
    1.  如果上一个使用FPU的任务就是当前任务，则直接返回。
    2.  `__asm__("fwait");`: 等待上一个FPU指令完成。
    3.  如果 `last_task_used_math` 非空 (即之前有任务使用了FPU)，则 `__asm__("fnsave %0"::"m" (last_task_used_math->tss.i387));` 将其FPU状态保存到其TSS的 `i387` 区域。
    4.  `last_task_used_math = current;`: 更新标记，当前任务是最后一个使用FPU的。
    5.  如果 `current->used_math` 为真 (表示当前任务之前曾使用过FPU，并且其状态已保存在其TSS中)，则 `__asm__("frstor %0"::"m" (current->tss.i387));` 从当前任务TSS中恢复FPU状态。
    6.  否则 (当前任务首次使用FPU或之前未使用)，则 `__asm__("fninit"::);` 初始化FPU，并将 `current->used_math` 置1。

### `void schedule(void)`

*   **功能**: 内核的调度器主函数，选择下一个要运行的进程并进行上下文切换。
*   **逻辑**:
    1.  **处理信号和唤醒**:
        *   遍历 `task` 数组 (从后向前，跳过任务0)。
        *   检查每个任务 `*p`：
            *   **处理闹钟**: `if ((*p)->alarm && (*p)->alarm < jiffies)`: 如果任务设置了闹钟 (`alarm` 非0) 且闹钟时间已到 (`alarm < jiffies`)，则向该任务发送 `SIGALRM` 信号 (`(*p)->signal |= (1<<(SIGALRM-1));`) 并清除闹钟 (`(*p)->alarm = 0;`)。
            *   **唤醒可中断睡眠进程**: `if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) && (*p)->state == TASK_INTERRUPTIBLE)`:
                *   `(*p)->signal`: 进程收到的信号位图。
                *   `(*p)->blocked`: 进程阻塞的信号位图。
                *   `_BLOCKABLE`: 所有可阻塞信号的掩码。
                *   `~(_BLOCKABLE & (*p)->blocked)`: 计算出当前对该进程有效的、未被阻塞的信号掩码。
                *   如果进程有未被阻塞的信号，并且它当前处于 `TASK_INTERRUPTIBLE` 状态，则将其状态置为 `TASK_RUNNING`，使其可以被调度。
    2.  **选择下一个进程 (调度器核心算法)**:
        *   `while (1)`: 无限循环，直到选出一个进程。
        *   `c = -1; next = 0;`: `c` 用于记录当前找到的最佳进程的 `counter` 值，`next` 是其在 `task` 数组中的索引。
        *   遍历 `task` 数组 (从 `NR_TASKS-1` 到 `1`，任务0是idle任务，不参与此轮选择):
            *   `if (!*--p) continue;`: 跳过空任务槽。
            *   `if ((*p)->state == TASK_RUNNING && (*p)->counter > c)`: 如果任务处于 `TASK_RUNNING` 状态，并且其时间片 `counter` 大于当前已找到的最大 `counter` 值 `c`。
            *   `c = (*p)->counter, next = i;`: 更新 `c` 和 `next`。
        *   `if (c) break;`: 如果找到了一个 `counter > 0` 的就绪进程 (`c` 不为-1且不为0)，则跳出 `while(1)` 循环，准备切换到 `task[next]`。
        *   **重新计算所有进程的 counter (如果所有就绪进程的 counter 都已耗尽)**:
            *   `for(p = &LAST_TASK ; p > &FIRST_TASK ; --p) if (*p) (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;`
            *   遍历所有任务（除任务0外）。
            *   将每个任务的 `counter` 值减半 (`>>1`)，然后加上其静态优先级 `priority` (对于用户任务通常是15)。这使得I/O消耗型任务（它们经常睡眠，`counter` 不会很快耗尽，所以减半影响小）和长时间未运行的任务（`counter` 可能已接近0）有机会获得更高的动态优先级。
            *   然后重新进入 `while(1)` 循环进行选择。
    3.  **切换到选中的进程**: `switch_to(next);`
        *   调用 `<linux/sched.h>` 中定义的 `switch_to` 宏，执行实际的上下文切换到 `task[next]`。

### `void sleep_on(struct task_struct **p)`

*   **功能**: 使当前进程在等待队列 `*p` 上进入**不可中断**睡眠。
*   **参数 `p`**: 指向一个 `struct task_struct *` 类型的指针。这个指针通常是一个等待队列的头。
*   **逻辑**:
    1.  `if (!p) return;`: 如果队列指针无效，则返回。
    2.  `if (current == &(init_task.task)) panic("task[0] trying to sleep");`: 任务0 (idle任务) 不能睡眠。
    3.  `tmp = *p;`: 保存等待队列头 `*p` 原来的值 (可能是 `NULL` 或上一个等待者)。
    4.  `*p = current;`: 将当前进程设置为新的等待队列头（或者说，将当前进程链入等待队列，这里实现的是单元素等待队列，后来的会覆盖先来的，但通常配合 `wake_up` 的行为）。
    5.  `current->state = TASK_UNINTERRUPTIBLE;`: 设置当前进程状态为不可中断睡眠。
    6.  `schedule();`: 调用调度器，放弃CPU。
    7.  **唤醒后**: 当其他地方调用 `wake_up(p)` 使得 `*p` 被清空且当前进程被唤醒后，`schedule()` 会返回到这里（或者说，切换回当前进程时，从这里继续）。
    8.  `if (tmp) tmp->state = 0;`: 如果之前队列头 `tmp` 中保存了一个任务，则将其状态设为 `TASK_RUNNING` (0)。这看起来像是试图唤醒“上一个”等待者，但 `wake_up` 的实现是直接唤醒 `*p` 指向的任务并将其设为 `NULL`。这里的逻辑可能是为了处理一种特定的链式唤醒或确保旧的等待者也被考虑，但通常 `sleep_on`/`wake_up` 的配对行为是 `wake_up` 唤醒 `*p`，然后 `*p` 应该被设为 `NULL`。
        *   **更准确的理解**: 这里的 `sleep_on` 实现的是一个简单的“替换式”等待队列。当一个任务 `A` 调用 `sleep_on(&wait_queue)` 时，`wait_queue` 指向 `A`。如果之后任务 `B` 也调用 `sleep_on(&wait_queue)`，则 `wait_queue` 会指向 `B`，而 `A` 的指针（原 `*p` 的值）被存在 `tmp` 中。当 `wake_up(&wait_queue)` 被调用时，它会唤醒 `B` (当前的 `*p`)，并将 `wait_queue` 设为 `NULL`。当 `B` 被调度器唤醒后从 `schedule()` 返回，它会执行 `if (tmp) tmp->state=0;`，此时 `tmp` 里存的是 `A`。所以 `B` 醒来后会尝试唤醒 `A`。这形成了一个链式的唤醒，但不是一个真正的队列，因为 `wake_up` 只唤醒头，而头会唤醒它之前被覆盖的那个。

### `void interruptible_sleep_on(struct task_struct **p)`

*   **功能**: 使当前进程在等待队列 `*p` 上进入**可中断**睡眠。
*   **逻辑**: 与 `sleep_on` 非常相似，主要区别在于：
    *   `current->state = TASK_INTERRUPTIBLE;`: 设置状态为可中断。
    *   **唤醒后的处理**:
        ```c
        if (*p && *p != current) { // 如果等待队列头不是NULL且不是当前进程 (说明在schedule期间，*p 被其他任务通过 wake_up 修改了)
            (**p).state=0;        // 唤醒 *p 指向的任务 (通常是新加入的等待者)
            goto repeat;          // 当前任务应该继续睡眠，重新把自己加入等待队列
        }
        *p=NULL;                  // 如果 *p 是当前进程或 NULL (表示当前进程被正常 wake_up)，则将队列头清空
        if (tmp)
            tmp->state=0;         // 同样尝试唤醒之前被覆盖的等待者 tmp
        ```
        这里的 `repeat` 逻辑是为了处理当一个进程被信号唤醒，但它原本等待的条件可能尚未满足，或者等待队列的头已被其他进程占据的情况。它试图重新将自己置于睡眠状态。

### `void wake_up(struct task_struct **p)`

*   **功能**: 唤醒在等待队列 `*p` 上的第一个（也是唯一一个，根据 `sleep_on` 的实现）进程。
*   **逻辑**:
    1.  `if (p && *p)`: 如果队列指针有效且队列头 `*p` 非空（即有进程在等待）。
    2.  `(**p).state = 0;`: 将等待进程的状态设置为 `TASK_RUNNING` (0)。
    3.  `*p = NULL;`: 将队列头清空，表示没有进程在等待了。

### `void do_timer(long cpl)`

*   **功能**: 时钟中断的核心C处理函数，由汇编例程 `timer_interrupt` 调用。
*   **参数 `cpl`**: Current Privilege Level (当前特权级)。如果为0，表示中断时CPU在内核态；如果为3，表示在用户态。
*   **逻辑**:
    1.  **处理蜂鸣器**: `if (beepcount) if (!--beepcount) sysbeepstop();` (如果蜂鸣器计数器非零，则减一；如果减到零，则停止蜂鸣)。
    2.  **更新CPU时间统计**:
        *   `if (cpl) current->utime++; else current->stime++;`
        *   如果中断发生在用户态 (`cpl` 通常是3，这里用非0判断)，则增加当前进程的用户时间 `utime`。
        *   否则 (中断发生在内核态)，增加当前进程的系统时间 `stime`。
    3.  **处理内核定时器**:
        *   `if (next_timer)`: 如果有注册的定时器。
        *   `next_timer->jiffies--;`: 将链表头定时器的剩余时间减1。
        *   `while (next_timer && next_timer->jiffies <= 0)`: 处理所有已到期的定时器。
            *   保存函数指针 `fn = next_timer->fn;`。
            *   将 `next_timer->fn` 设为 `NULL` (标记为空闲或已处理)。
            *   `next_timer = next_timer->next;`: 移动到下一个定时器。
            *   `(fn)();`: 执行到期定时器的处理函数。
    4.  **处理软盘马达定时器**: `if (current_DOR & 0xf0) do_floppy_timer();` (如果任何软盘马达是开启的)。
    5.  **处理当前进程时间片**:
        *   `if ((--current->counter) > 0) return;`: 当前进程的时间片 `counter` 减1。如果仍大于0，则函数返回，当前进程继续运行。
        *   `current->counter = 0;`: 如果时间片耗尽。
        *   `if (!cpl) return;`: **关键**: 如果中断发生在内核态 (`cpl` 为0)，即使时间片耗尽，也不进行调度，直接返回。这允许内核代码执行完毕，避免在内核临界区中被抢占。
        *   `schedule();`: 如果中断发生在用户态且时间片耗尽，则调用调度器 `schedule()`。

### `void sched_init(void)`

*   **功能**: 初始化调度相关的子系统。在 `init/main.c` 中被调用。
*   **步骤**:
    1.  `if (sizeof(struct sigaction) != 16) panic(...)`: 检查 `struct sigaction` 的大小是否为16字节，这是一个硬性要求，可能与栈操作或二进制兼容性有关。
    2.  **设置任务0的TSS和LDT描述符**:
        *   `set_tss_desc(gdt + FIRST_TSS_ENTRY, &(init_task.task.tss));`
        *   `set_ldt_desc(gdt + FIRST_LDT_ENTRY, &(init_task.task.ldt));`
        *   `FIRST_TSS_ENTRY` (4) 和 `FIRST_LDT_ENTRY` (5) 是任务0的TSS和LDT在GDT中的索引。
        *   `set_tss_desc` 和 `set_ldt_desc` 宏 (在 `<asm/system.h>`) 用于填充GDT条目。
    3.  **清空其他任务的GDT槽位**:
        *   `p = gdt + 2 + FIRST_TSS_ENTRY;` (指向任务1的TSS描述符位置，每个任务占2个GDT条目：TSS和LDT)。
        *   `for(i=1; i<NR_TASKS; i++) { task[i] = NULL; p->a=p->b=0; p++; p->a=p->b=0; p++; }`
        *   将 `task` 数组中除 `task[0]` 外的所有条目设为 `NULL`。
        *   将GDT中对应于任务1到 `NR_TASKS-1` 的TSS和LDT描述符都清零。
    4.  **清除EFLAGS的NT位**: `__asm__("pushfl ; andl $0xffffbfff,(%esp) ; popfl");`
        *   NT (Nested Task) 标志位如果被设置，`iret` 指令会尝试进行任务切换到 `back_link` 指向的TSS。这里清除它是为了避免不期望的任务切换。
    5.  **加载任务0的TR和LDTR**:
        *   `ltr(0);`: 加载任务寄存器TR，使其指向GDT中任务0的TSS。0是任务号。
        *   `lldt(0);`: 加载LDT寄存器LDTR，使其指向GDT中任务0的LDT。
    6.  **设置8253定时器 (PIT)**:
        *   `outb_p(0x36, 0x43);`: 向模式控制寄存器 `0x43` 写入 `0x36` (二进制 `00110110b`)。
            *   选择通道0 (`00`)。
            *   访问模式：先低字节后高字节 (`11`)。
            *   工作模式：方式3 (方波发生器 `011`)。
            *   BCD/二进制模式：0 (16位二进制计数器)。
        *   `outb_p(LATCH & 0xff, 0x40);`: 向通道0数据端口 `0x40` 写入 `LATCH` 值的低字节。`LATCH` 是 `1193180 / HZ`。
        *   `outb(LATCH >> 8, 0x40);`: 写入 `LATCH` 值的高字节。
        *   这样设置后，定时器通道0会以 `HZ` (100Hz) 的频率产生中断。
    7.  **设置时钟中断门**:
        *   `set_intr_gate(0x20, &timer_interrupt);`: 将IDT的第 `0x20` 项 (IRQ0，即时钟中断) 设置为一个中断门，其处理程序入口点是 `timer_interrupt` (在 `kernel/system_call.s` 中定义的汇编例程)。
        *   `outb(inb_p(0x21) & ~0x01, 0x21);`: 解除对IRQ0在8259A中断控制器中的屏蔽 (清主PIC的IMR寄存器的第0位)。
    8.  **设置系统调用门**:
        *   `set_system_gate(0x80, &system_call);`: 将IDT的第 `0x80` 项设置为一个系统门（陷阱门，DPL=3），其处理程序入口点是 `system_call` (在 `kernel/system_call.s` 中定义的汇编例程)。这使得用户程序可以通过 `int 0x80` 指令发起系统调用。

## 总结

`kernel/sched.c` 是 Linux 0.11 实现多任务调度、进程同步与通信（通过信号和等待队列）、以及时间管理（时钟中断、闹钟、内核定时器）的核心。`schedule()` 函数中的调度算法虽然简单（基于 `counter` 和 `priority`），但奠定了后续更复杂调度器的基础。`sleep_on`/`wake_up` 提供了基本的进程阻塞和唤醒原语。`do_timer` 精确地处理了时钟中断的各项任务，包括更新进程时间、时间片递减和定时器触发。`sched_init` 则完成了调度系统和相关硬件（定时器、中断控制器）的初始化，为整个系统的多任务运行铺平了道路。文件中还包含了一些特定于硬件（如FPU、软盘）的辅助代码，反映了早期内核设计的紧凑性。
