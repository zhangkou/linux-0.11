# kernel/traps.c 文件详解

`kernel/traps.c` 文件是 Linux 0.11 内核中负责处理CPU硬件陷阱 (traps) 和故障 (faults) 的C语言代码部分。当CPU在执行过程中遇到预定义的异常情况（如除零、无效操作码、段错误、页错误等）时，它会停止当前指令的执行，并根据中断描述符表 (IDT) 跳转到相应的处理程序。这些处理程序的底层入口点通常在 `kernel/asm.s` 中定义，它们负责保存现场并调用 `traps.c` 中的C函数来进行进一步的处理。

此文件的主要目的是：
1.  **为各种CPU异常提供C语言处理函数** (通常以 `do_` 开头，如 `do_divide_error`)。
2.  **初始化中断描述符表 (IDT)** (`trap_init()` 函数)，将这些C处理函数（或其汇编包装器）与对应的中断向量（0-255）关联起来。
3.  **打印调试信息并终止 offending 进程**: 大多数异常处理函数在当前版本 (0.11) 中主要是一个调试辅助工具，它们会调用 `die()` 函数打印出详细的CPU状态、寄存器信息、栈内容等，然后通常会通过 `do_exit(11)` (SIGSEGV信号) 来终止引发异常的进程。页错误 (`page_fault`) 是一个例外，它会调用 `page_exception()` (在 `mm/page.s` 或 `mm/memory.c` 中)，尝试通过按需调页来解决问题，而不是直接终止进程。

## 核心功能

*   **定义各种CPU异常的C语言处理函数**: 如 `do_divide_error`, `do_debug`, `do_nmi`, `do_int3` (断点), `do_overflow`, `do_bounds` (边界检查), `do_invalid_op` (无效操作码), `do_device_not_available` (设备不可用，如FPU), `do_double_fault` (双重故障), `do_coprocessor_segment_overrun` (协处理器段越限), `do_invalid_TSS` (无效TSS), `do_segment_not_present` (段不存在), `do_stack_segment` (栈段错误), `do_general_protection` (通用保护错误), `do_coprocessor_error` (协处理器错误), `do_reserved` (保留中断)。
*   **`die()` 函数**: 一个通用的错误报告和进程终止函数，被大多数 `do_` 系列异常处理函数调用。
*   **`trap_init()` 函数**: 初始化IDT，为CPU的前32个中断向量（0-31，主要对应CPU异常）以及一些其他特定中断（如协处理器错误IRQ13，并口中断）设置门描述符，将它们指向各自的处理程序。
*   **页错误处理入口**: `page_fault` 异常（中断14）被特殊处理，其门描述符指向 `page_exception`（在 `mm/page.s` 中定义），这是实现虚拟内存和按需调页的关键。

## 辅助宏和函数

### `get_seg_byte(seg, addr)` 和 `get_seg_long(seg, addr)`

这两个宏用于从指定的段 `seg` (段选择子) 和段内地址 `addr` 处读取一个字节或一个长字。它们通过临时修改 `fs` 段寄存器来实现。
```c
#define get_seg_byte(seg,addr) ({ \
register char __res; \
__asm__("push %%fs;mov %%ax,%%fs;movb %%fs:%2,%%al;pop %%fs" \
	:"=a" (__res):"0" (seg),"m" (*(addr))); \
__res;})

#define get_seg_long(seg,addr) ({ \
register unsigned long __res; \
__asm__("push %%fs;mov %%ax,%%fs;movl %%fs:%2,%%eax;pop %%fs" \
	:"=a" (__res):"0" (seg),"m" (*(addr))); \
__res;})
```
*   `push %%fs; mov %%ax, %%fs; ... pop %%fs`: 保存原 `fs`，将 `seg` (通过 `ax`) 加载到 `fs`，执行内存访问，然后恢复原 `fs`。
*   这允许内核代码访问任意段（通常是用户段）中的数据，主要用于 `die()` 函数中打印用户栈内容或指令。

### `_fs()`

*   `#define _fs() ({ register unsigned short __res; __asm__("mov %%fs,%%ax":"=a" (__res):); __res;})`
*   获取当前 `fs` 段寄存器的值。

### `do_exit(long code)`

*   外部函数声明，用于终止当前进程。在 `die()` 函数中被调用。

### `page_exception(void)`

*   外部函数（或汇编例程）声明，页错误（中断14）的处理入口。

## `die(char * str, long esp_ptr, long nr)` 函数

*   **功能**: 打印详细的错误信息和CPU状态，然后终止当前进程。
*   **参数**:
    *   `char * str`: 描述错误类型的字符串 (如 "divide error", "general protection")。
    *   `long esp_ptr`: 指向发生异常时内核栈的指针。这个栈上保存了被中断进程的寄存器状态（由 `asm.s` 中的汇编代码压栈）。
    *   `long nr`: 错误码 (对于某些异常，CPU会自动压入错误码；对于其他异常，`asm.s` 中会压入0)。
*   **步骤**:
    1.  `long * esp = (long *) esp_ptr;`: 将 `esp_ptr` 转换为长整型指针，方便访问栈上保存的各个值。
    2.  `printk("%s: %04x\n\r", str, nr & 0xffff);`: 打印错误类型和错误码（取低16位）。
    3.  **打印CPU寄存器状态**:
        *   `printk("EIP:\t%04x:%p\nEFLAGS:\t%p\nESP:\t%04x:%p\n", esp[1], esp[0], esp[2], esp[4], esp[3]);`
            *   `esp[0]`: 用户态 EIP (或异常发生时的EIP)。
            *   `esp[1]`: 用户态 CS (或异常发生时的CS)。
            *   `esp[2]`: 用户态 EFLAGS (或异常发生时的EFLAGS)。
            *   `esp[3]`: 用户态 ESP。
            *   `esp[4]`: 用户态 SS。
            *   (这些索引是相对于 `asm.s` 中 `iret` 指令执行前栈顶的偏移，`esp_ptr` 指向的是C函数的第一个参数，即指向所有保存寄存器的那个区域的指针，而C函数调用时，错误码和返回地址在其之上)。
            *   更准确地说，`esp_ptr` 是 `asm.s` 中 `lea 44(%esp), %edx` (对于 `no_error_code`) 或 `lea 44(%esp), %eax` (对于 `error_code`) 计算出的地址，这个地址指向了保存的 `eax` (对于 `no_error_code` 是原 `eax`，对于 `error_code` 是原 `ebx`)。因此，`esp[0]` 到 `esp[4]` 的解释需要对照 `asm.s` 中压栈的顺序和 `iret` 的栈帧结构。
            *   根据 `system_call.s` 注释的 `iret` 栈帧：`EIP` 在 `1C(%esp_iret)`，`CS` 在 `20(%esp_iret)`，`EFLAGS` 在 `24(%esp_iret)`，`OLDESP` 在 `28(%esp_iret)`，`OLDSS` 在 `2C(%esp_iret)`。如果 `die` 的 `esp_ptr` 参数是指向 `asm.s` 中 `pushl %edx` (或 `pushl %eax`) 那个参数的地址，那么 `esp[0]` 是 `EIP`，`esp[1]` 是 `CS`，`esp[2]` 是 `EFLAGS`，`esp[3]` 是 `ESP_user`，`esp[4]` 是 `SS_user`，这与printk的格式符 `%p` (指针) 和 `%04x` (16位段选择子) 的使用相符。
    4.  `printk("fs: %04x\n", _fs());`: 打印当前 `fs` 段寄存器的值。
    5.  `printk("base: %p, limit: %p\n", get_base(current->ldt[1]), get_limit(0x17));`: 打印当前进程LDT中代码段的基址 (`current->ldt[1]`) 和数据段的限长 (通过用户数据段选择子 `0x17` 获取)。
    6.  **打印用户栈内容 (如果SS是用户数据段)**:
        *   `if (esp[4] == 0x17)` (即用户SS是0x17):
        *   `printk("Stack: "); for (i=0; i<4; i++) printk("%p ", get_seg_long(0x17, i + (long *)esp[3])); printk("\n");`
            *   使用 `get_seg_long(0x17, ...)` 从用户栈 (段0x17，基地址是 `esp[3]`) 读取并打印前4个长字的内容。
    7.  `str(i); printk("Pid: %d, process nr: %d\n\r", current->pid, 0xffff & i);`:
        *   `str(i)`: (`store task register`) 将任务寄存器TR的值存入 `i`。TR的高13位是当前任务TSS在GDT中的索引。
        *   打印当前进程的PID (`current->pid`) 和任务号 (`0xffff & i`，TR的低16位，实际上是TSS选择子，其中索引部分需要右移3位)。
    8.  **打印用户代码段内容**:
        *   `for(i=0; i<10; i++) printk("%02x ", 0xff & get_seg_byte(esp[1], (i + (char *)esp[0]))); printk("\n\r");`
        *   使用 `get_seg_byte(esp[1], ...)` 从用户代码段 (段 `esp[1]` 即CS，基地址是 `esp[0]` 即EIP) 读取并打印EIP指向位置开始的10个字节（指令码）。
    9.  `do_exit(11); /* play segment exception */`: 调用 `do_exit` 终止当前进程，退出码11对应 `SIGSEGV` (段错误) 信号。

## `do_...` 系列异常处理函数

文件中为每种CPU异常（除了页错误）都定义了一个 `do_...` 函数，例如：
*   `void do_double_fault(long esp, long error_code)`
*   `void do_general_protection(long esp, long error_code)`
*   `void do_divide_error(long esp, long error_code)`
*   ... 等等。

这些函数的结构都非常相似：它们都直接调用 `die()` 函数，将特定的错误描述字符串、从汇编代码传递过来的内核栈指针 `esp` (实际上是 `esp_ptr`，指向保存的寄存器区域) 和错误码 `error_code` 传递给 `die()`。

**`do_int3` 的特殊性**:
`do_int3` (断点中断 `int 3` 的处理) 的参数列表与众不同，它接收了所有通用寄存器和段寄存器的值作为独立参数。它会打印这些寄存器的值，但**不会调用 `die()`**，也不会终止进程。这使得 `int 3` 可以被用作调试断点，打印状态后程序可以继续（如果调试器允许）。

**`do_coprocessor_error` 的特殊性**:
`if (last_task_used_math != current) return;`
如果最后一个使用数学协处理器的任务不是当前任务，则直接返回。这可能是为了避免处理并非由当前任务直接引起的、陈旧的或无关的协处理器错误。如果错误确实与当前任务相关，则调用 `die()`。

## `trap_init(void)` 函数

*   **功能**: 初始化中断描述符表 (IDT)，为各种陷阱和故障设置门描述符。
*   **步骤**:
    1.  `set_trap_gate(vector, &handler_function_name);`: 为每个CPU异常（中断向量0-16，除了15）设置一个陷阱门。陷阱门在被调用时不会自动关中断。
        *   `set_trap_gate(0, &divide_error);`
        *   `set_trap_gate(1, &debug);`
        *   `set_trap_gate(2, &nmi);`
        *   ...
        *   `set_trap_gate(14, &page_fault);` (页错误处理程序 `page_fault` 指向 `mm/page.s` 中的 `_page_exception`)
        *   `set_trap_gate(16, &coprocessor_error);`
    2.  **设置系统门 (`int 3`, `int 4`, `int 5`)**:
        *   `set_system_gate(3, &int3);`
        *   `set_system_gate(4, &overflow);`
        *   `set_system_gate(5, &bounds);`
        *   系统门是一种特殊的陷阱门，其DPL (描述符特权级) 设置为3，允许用户态程序通过 `int` 指令（如 `int 3`）主动触发这些中断。
    3.  **为保留中断设置默认处理**:
        *   `set_trap_gate(15, &reserved);`
        *   `for (i=17; i<48; i++) set_trap_gate(i, &reserved);`
        *   将中断向量15以及17到47（这些通常是硬件中断IRQ1到IRQ15映射到的向量，减去已明确设置的）都指向 `reserved` 处理函数。
    4.  **设置特定硬件中断的门**:
        *   `set_trap_gate(45, &irq13);` (IRQ13 + 32 = 45，FPU协处理器错误中断)。`irq13` 是在 `kernel/asm.s` 中定义的汇编入口。
        *   `set_trap_gate(39, &parallel_interrupt);` (IRQ7 + 32 = 39，并口中断)。`parallel_interrupt` 也是汇编入口。
    5.  **初始化8259A可编程中断控制器 (PIC)**:
        *   `outb_p(inb_p(0x21) & 0xfb, 0x21);`: 解除对IRQ2（从PIC级联）在主PIC (端口0x21) 上的屏蔽 (清第2位)。`0xfb` = `11111011b`。
        *   `outb(inb_p(0xA1) & 0xdf, 0xA1);`: 解除对IRQ13（FPU错误，对应从PIC的第5个输入，即位5）在从PIC (端口0xA1) 上的屏蔽 (清第5位)。`0xdf` = `11011111b`。
        *   这确保了FPU错误中断可以被正确接收。其他硬件中断（如时钟、键盘、硬盘、软盘）的屏蔽位通常在它们各自的初始化函数中或在需要时解除。

## 总结

`kernel/traps.c` 是Linux 0.11内核异常和中断处理机制的C语言层核心。它通过 `trap_init()` 配置IDT，将CPU的内部异常和部分外部硬件中断（通过 `asm.s` 中的底层包装）引导到相应的C处理函数。在0.11版本中，大多数异常处理（除了页错误和调试断点）的最终结果是调用 `die()` 函数，该函数会打印详细的调试信息并终止出错的进程。这反映了早期内核错误处理能力相对有限，主要侧重于信息报告和防止系统崩溃。`die()` 函数提供的详细状态转储对于内核开发者调试这些底层错误非常有价值。对页错误中断的特殊处理（指向 `page_exception`）则是实现虚拟内存和按需调页的关键入口。
