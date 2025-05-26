# kernel/system_call.s 文件详解

`kernel/system_call.s` 文件是 Linux 0.11 内核中处理**系统调用 (system calls)**、**时钟中断 (timer interrupt)** 以及其他一些硬件中断（硬盘、软盘、并口）的底层汇编入口和处理流程。它是用户空间程序与内核空间服务之间，以及硬件中断与内核响应之间的关键桥梁。

此文件中的代码是用 x86 汇编语言编写的，负责：
1.  **系统调用总入口 (`_system_call`)**: 当用户程序执行 `int 0x80` 指令时，CPU会跳转到这里。它负责保存用户态上下文，调用相应的C语言系统调用处理函数（通过 `_sys_call_table`），并在返回前检查和处理信号，最后恢复用户态上下文。
2.  **时钟中断处理入口 (`_timer_interrupt`)**: 由8253定时器产生的时钟中断（IRQ0）会跳转到这里。它负责保存上下文，增加系统滴答计数 `_jiffies`，向8259A发送EOI（中断结束命令），调用C函数 `_do_timer`，并在返回前检查和处理信号。
3.  **其他硬件中断处理入口**:
    *   `_hd_interrupt`: 硬盘中断（IRQ14）处理入口。
    *   `_floppy_interrupt`: 软盘中断（IRQ6）处理入口。
    *   `_parallel_interrupt`: 并口中断（IRQ7）处理入口（非常简单，只是EOI后返回）。
4.  **特定CPU异常处理入口**:
    *   `_coprocessor_error`: IRQ13（FPU协处理器错误）的后续处理跳转点。
    *   `_device_not_available`: 设备不可用异常（int 7，通常是FPU相关的TS位导致）。
5.  **信号识别和处理的返回路径 (`ret_from_sys_call`)**: 无论是系统调用返回还是时钟中断返回，都会经过此路径。它负责检查当前进程是否有待处理的信号，如果有，则调用 `_do_signal`（在 `kernel/signal.c`）来处理信号，这可能导致用户态执行流转向信号处理函数。

## 核心数据和常量定义

文件开头定义了一些常量，用于方便地访问任务结构 (`task_struct`) 中的字段以及中断返回时栈帧中各寄存器的偏移量。

### 栈帧偏移量 (相对于 `ret_from_system_call` 中的 `esp`)

当从系统调用或中断返回时，内核栈上保存了用户态（或被中断的内核态）的上下文。
```assembly
EAX		= 0x00  ; 实际是 popl %eax 后的栈顶，是 pushl %eax 的结果
EBX		= 0x04  ; (原 system_call 中压栈的顺序是 edx, ecx, ebx)
ECX		= 0x08  ;  这里定义的偏移是 ret_from_sys_call 中 pop 的顺序
EDX		= 0x0C
FS		= 0x10
ES		= 0x14
DS		= 0x18
EIP		= 0x1C  ; 以下是 iret 指令要用的
CS		= 0x20
EFLAGS	= 0x24
OLDESP	= 0x28  ; 用户态 esp
OLDSS	= 0x2C  ; 用户态 ss
```
**注意**: 文件注释中描述的栈布局 (`0(%esp) - %eax` 等) 是 `ret_from_system_call` 标签处，在所有 `push` 完成后，`iret` 执行前的栈布局。而这里定义的 `EAX` 等常量更像是 `pop` 指令的偏移参考，或者是 `system_call` 入口处压栈的顺序。
更准确地说，`_system_call` 入口处压栈顺序是 `ds, es, fs, edx, ecx, ebx`，然后调用C函数，C函数返回值在 `eax`。`ret_from_sys_call` 恢复时，`eax` 是最先被 `push` (保存返回值) 和 `pop` 的。`iret` 使用的 `eip, cs, eflags, oldesp, oldss` 是CPU在中断或系统调用门发生时自动压栈的（或由内核代码如 `move_to_user_mode` 构造的）。

### `task_struct` 字段偏移量

这些是 `struct task_struct` 中一些常用字段的字节偏移量。
```assembly
state	= 0
counter	= 4
priority = 8
signal	= 12
sigaction = 16		# MUST be 16 (=len of sigaction in bytes)
blocked = (33*16)   # 32个sigaction结构，每个16字节，blocked在其后
```
*   `sigaction` 字段的偏移量是16，并且注释强调每个 `sigaction` 结构必须是16字节。
*   `blocked` (信号阻塞掩码) 的偏移量计算为 `33*16`，这暗示 `sigaction` 数组有32个元素，`blocked` 在这32个 `sigaction` 结构之后。

### `sigaction` 结构内偏移量

```assembly
sa_handler = 0
sa_mask = 4
sa_flags = 8
sa_restorer = 12
```
这些是 `struct sigaction` (在 `<signal.h>` 定义) 内部各成员的偏移量。

### `nr_system_calls = 72`

*   定义了系统调用的总数。用于检查系统调用号的有效性。

## 主要汇编例程详解

### `_system_call` (系统调用总入口)

当用户程序执行 `int 0x80` 时，CPU会跳转到由IDT的0x80项（系统门）指定的这个地址。

1.  **检查系统调用号**:
    *   `cmpl $nr_system_calls-1, %eax`: 将 `eax` 中的系统调用号与最大允许值比较。
    *   `ja bad_sys_call`: 如果号无效，跳转到 `bad_sys_call`。
2.  **保存段寄存器和参数寄存器**:
    *   `push %ds`, `push %es`, `push %fs`: 保存用户态的段寄存器。
    *   `pushl %edx`, `pushl %ecx`, `pushl %ebx`: 压入系统调用的参数 (通常C库将参数放入 `ebx`, `ecx`, `edx`)。
3.  **设置内核数据段**:
    *   `movl $0x10, %edx; mov %dx, %ds; mov %dx, %es;`: 将 `ds` 和 `es` 设置为内核数据段选择子 (0x10)。
    *   `movl $0x17, %edx; mov %dx, %fs;`: 将 `fs` 设置为用户数据段选择子 (0x17)。内核通过 `fs:` 前缀访问用户空间数据。
4.  **调用C处理函数**:
    *   `call _sys_call_table(,%eax,4)`: 根据 `eax` 中的系统调用号，在 `_sys_call_table` (定义在 `<linux/sys.h>`) 中找到对应的C函数地址并调用。`%eax,4` 表示将 `eax` 作为索引，每个表项4字节（函数指针大小）。
5.  **保存返回值**: `pushl %eax` 将C函数的返回值（现在在 `eax` 中）压栈。
6.  **检查是否需要调度**:
    *   `movl _current, %eax`: 获取当前进程的 `task_struct` 指针。
    *   `cmpl $0, state(%eax)`: 检查进程状态。如果非0 (例如，睡眠状态)，则跳转到 `reschedule`。
    *   `cmpl $0, counter(%eax)`: 检查进程时间片。如果为0，跳转到 `reschedule`。
7.  **返回路径**: 如果不需要调度，则执行 `ret_from_sys_call`。

### `bad_sys_call`

*   `movl $-1, %eax`: 将 `eax` (系统调用返回值) 设为 -1 (表示错误)。
*   `iret`: 直接中断返回。

### `reschedule`

*   `pushl $ret_from_sys_call`: 将 `ret_from_sys_call` 的地址压栈，作为 `_schedule` 函数返回后应该跳转到的地方。
*   `jmp _schedule`: 跳转到调度器函数 `_schedule` (在 `kernel/sched.c` 中定义)。

### `ret_from_sys_call` (从系统调用或时钟中断返回的公共路径)

1.  **检查是否为任务0或内核态中断**:
    *   `movl _current, %eax; cmpl _task, %eax; je 3f`: 如果当前是任务0 (`_task` 指向 `task[0]`)，则不处理信号，直接跳转到 `3f`。任务0是idle进程，不应接收或处理信号。
    *   `cmpw $0x0f, CS(%esp)`: 检查栈上保存的CS（代码段选择子）。如果不是用户代码段选择子 (0x0F, RPL=3)，说明中断发生在内核态。
    *   `jne 3f`: 如果是在内核态被中断（例如，时钟中断打断了内核代码执行），则不处理用户信号，跳转到 `3f`。
    *   `cmpw $0x17, OLDSS(%esp)`: 进一步检查，如果栈段也不是用户数据段(0x17)，也跳转。这是双重保险，确保只在从用户态返回时才处理用户信号。
2.  **信号处理逻辑**:
    *   `movl signal(%eax), %ebx`: 获取当前进程的待处理信号位图 `current->signal` 到 `ebx`。
    *   `movl blocked(%eax), %ecx; notl %ecx; andl %ebx, %ecx;`: 获取阻塞信号位图 `current->blocked` 到 `ecx`，取反 `ecx` (得到未阻塞的信号掩码)，然后与 `ebx` (待处理信号) 按位与。结果 `ecx` 中现在是“当前需要立即处理的信号”位图。
    *   `bsfl %ecx, %ecx`: (Bit Scan Forward Long) 在 `ecx` 中查找第一个被置位的位 (即最低位的待处理信号)，将其索引 (0-31) 存回 `ecx`。
    *   `je 3f`: 如果没有找到置位的位 (ZF=1)，即没有需要立即处理的信号，则跳转到 `3f`。
    *   `btrl %ecx, %ebx; movl %ebx, signal(%eax);`: 从 `current->signal` (在 `ebx` 中) 中清除该信号位 (表示该信号即将被处理)，并写回。
    *   `incl %ecx`: 将信号索引 (0-31) 转换为信号编号 (1-32)。
    *   `pushl %ecx`: 将信号编号压栈作为 `_do_signal` 的参数。
    *   `call _do_signal`: 调用C函数 `_do_signal` (在 `kernel/signal.c`)。`_do_signal` 会修改内核栈上保存的用户态EIP和ESP，使得 `iret` 后跳转到信号处理函数。
    *   `popl %eax`: 清理 `_do_signal` 的参数（信号编号）。
3.  **`3:` 标签 (恢复寄存器并返回)**:
    *   `popl %eax`: 弹出系统调用或 `_do_signal` (如果被调用) 的返回值，或者之前保存的 `eax`。
    *   `popl %ebx`, `popl %ecx`, `popl %edx`: 恢复通用寄存器。
    *   `pop %fs`, `pop %es`, `pop %ds`: 恢复段寄存器。
    *   `iret`: 中断返回。恢复 EIP, CS, EFLAGS, 用户ESP, 用户SS，返回到用户态（或被中断的内核态代码）。

### `_timer_interrupt` (时钟中断处理入口)

1.  **保存寄存器**: `push %ds`, `push %es`, `push %fs`, `pushl %edx`, `pushl %ecx`, `pushl %ebx`, `pushl %eax`。
2.  **设置内核数据段**: 同 `_system_call`。
3.  **增加系统滴答**: `incl _jiffies`。
4.  **发送EOI到8259A**: `movb $0x20, %al; outb %al, $0x20`。
5.  **获取CPL**: `movl CS(%esp), %eax; andl $3, %eax`。从栈上保存的CS中提取当前特权级 (RPL)。
6.  **调用 `_do_timer`**: `pushl %eax` (压入CPL作为参数)；`call _do_timer` (在 `kernel/sched.c`)。
7.  **清理参数**: `addl $4, %esp`。
8.  **跳转到返回路径**: `jmp ret_from_sys_call`。时钟中断后也需要检查信号和可能的调度。

### 其他中断和异常入口

*   **`_coprocessor_error` (IRQ13的后续)** 和 `_device_not_available` (int 7):
    *   这两个入口点都用于处理与数学协处理器 (FPU) 相关的问题。
    *   它们首先保存所有通用寄存器和段寄存器。
    *   设置内核数据段。
    *   压入 `ret_from_sys_call` 的地址作为返回点。
    *   然后跳转到相应的C处理函数：`_math_error` (可能是 `do_coprocessor_error` 的别名或包装) 或 `_math_state_restore` / `_math_emulate`。
    *   `_device_not_available` 中会清除CR0的TS位，并根据EM位决定是恢复FPU状态还是模拟FPU。
*   **`_hd_interrupt` (硬盘中断)** 和 `_floppy_interrupt` (软盘中断):
    *   保存寄存器，设置内核数据段。
    *   向相应的8259A控制器（主片或从片）发送EOI。
    *   通过 `xchgl` 指令与全局函数指针（`_do_hd` 或 `_do_floppy`，在驱动中设置）交换，获取实际的C处理函数地址并调用。如果指针为空，则调用一个意外中断处理函数。
    *   恢复寄存器并 `iret`。
*   **`_parallel_interrupt` (并口中断)**:
    *   非常简单，只保存 `eax`，发送EOI到主8259A，恢复 `eax`，然后 `iret`。这表明Linux 0.11对并口中断的处理非常有限，可能只是应答一下。

### `_sys_execve` 和 `_sys_fork`

*   这两个是特殊的系统调用入口点，因为它们在C函数 (`_do_execve`, `_copy_process`) 返回后，其栈帧结构或返回路径可能与普通系统调用不同，或者它们需要特殊的参数传递方式（如 `_sys_execve` 直接从栈上传递 `eip` 给 `_do_execve`）。
*   `_sys_execve`:
    *   `lea EIP(%esp), %eax`: 将栈上保存的 `eip` (用户态 `eip`) 的地址加载到 `eax`。
    *   `pushl %eax`: 将此地址压栈作为 `_do_execve` 的第一个参数。
    *   `call _do_execve`: 调用C函数。
    *   `addl $4, %esp`: 清理参数。
    *   `ret`: **注意**: 这里是 `ret` 而不是 `iret`。因为 `execve` 成功后不会返回到调用者，而是切换到新程序的执行。如果 `execve` 失败，`_do_execve` 会直接返回错误码到 `_system_call` 的通用返回路径，然后通过 `iret` 返回用户态。此处的 `ret` 可能是针对 `_do_execve` 内部如果需要直接修改返回地址到用户态的情况，或者是一个永不应到达的路径。更常见的 `execve` 实现是，如果成功，`_do_execve` 不返回，它会修改当前进程的上下文并“变形”成新程序，然后通过修改内核栈上的 `eip` 等值，使得最终的 `iret` 直接跳转到新程序的入口。
*   `_sys_fork`:
    *   `call _find_empty_process`: 调用C函数查找空闲进程槽，返回值（新任务索引或错误）在 `eax` 中。
    *   `testl %eax, %eax; js 1f`: 如果 `eax` 为负 (错误)，则跳转到 `1f` 直接返回。
    *   保存 `gs, esi, edi, ebp`。
    *   `pushl %eax`: 将新任务索引 `nr` 压栈作为 `_copy_process` 的第一个参数。
    *   `call _copy_process`: 调用C函数。其参数是 `nr` 和从 `system_call` 传递下来的用户寄存器值（通过栈传递）。
    *   `addl $20, %esp`: 清理 `_copy_process` 的参数（5个long）。
    *   `1: ret`: `fork` 的汇编包装器通过 `ret` 返回到 `_system_call` 中的 `call _sys_call_table(...)` 之后。子进程的 `eax` 已在 `_copy_process` 中被设为0，父进程的 `eax` 是子进程的PID。

## 总结

`kernel/system_call.s` 是Linux 0.11操作系统内核与用户程序、硬件中断交互的底层枢纽。它通过精心设计的汇编代码，实现了：
*   一个统一的系统调用入口 (`_system_call`)，负责上下文保存、参数传递、调用C处理函数、错误处理、调度以及信号检查和分发。
*   时钟中断 (`_timer_interrupt`) 的处理，驱动着系统的时间统计、进程调度和定时器服务。
*   其他关键硬件中断（硬盘、软盘、协处理器）的初步处理和EOI发送。
*   CPU异常的捕获和到C语言处理函数的引导。

这个文件中的 `ret_from_sys_call` 路径尤其重要，因为它是在每次系统调用或时钟中断返回用户态之前必须执行的代码，确保了信号的及时处理和进程的按需调度。这些汇编例程的正确性和效率直接影响到整个操作系统的稳定性和性能。它是理解Linux内核如何响应外部事件和内部请求的核心部分。
