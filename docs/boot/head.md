# head.s 代码详解

`head.s` 是 Linux 0.11 内核在 `setup.s` 之后执行的第一个32位代码。它在物理地址 `0x00000000` 开始执行，这个位置后续也将作为页目录表的起始地址。因此，`head.s` 的代码最终会被页目录表覆盖。

它的主要任务是进行一系列底层硬件的初始化，为C语言环境的内核主函数 `main` 的调用做好准备，核心包括：
1.  重新加载段寄存器。
2.  设置中断描述符表 (IDT)。
3.  设置全局描述符表 (GDT)。
4.  检查 A20 地址线是否开启。
5.  检测并初始化数学协处理器 (x87 FPU)。
6.  设置并启用分页机制 (Paging)。
7.  最后跳转到 `_main` 函数 (位于 `init/main.c`)。

## 主要代码段和功能

### 1. 初始化段寄存器和堆栈

*   **`_pg_dir:`**: 符号，标记页目录表的起始位置，物理地址为 `0x00000000`。
*   **`startup_32:`**: 32位代码的入口点。
    *   **`movl $0x10,%eax; mov %ax,%ds; mov %ax,%es; mov %ax,%fs; mov %ax,%gs`**: 将数据段 (DS)、附加段 (ES)、FS、GS 寄存器都设置为 `0x10`。这个值是 GDT 中内核数据段的选择子 (参见后续 `_gdt` 的定义，它是第二个描述符，索引为2，RPL=0，所以选择子是 `(2<<3)|0 = 0x10`)。
    *   **`lss _stack_start,%esp`**: 加载堆栈段寄存器 (SS) 和堆栈指针 (ESP)。`_stack_start` 是在 `kernel/sched.c` 中定义的 `user_stack` 数组的末尾，指向一个初始的内核堆栈。

### 2. 设置 IDT 和 GDT

*   **`call setup_idt`**: 调用 `setup_idt` 子程序初始化中断描述符表。
*   **`call setup_gdt`**: 调用 `setup_gdt` 子程序初始化全局描述符表。
*   **`movl $0x10,%eax; mov %ax,%ds; ...`**: 在 `setup_gdt` 中重新加载了 GDT 后，需要重新加载所有段寄存器 (CS 已在 `setup_gdt` 中的 `ret` 指令隐含重新加载)。
*   **`lss _stack_start,%esp`**: 再次加载堆栈，确保使用的是新的 GDT 下的堆栈段。

### 3. 检查 A20 地址线

*   **`xorl %eax,%eax; 1: incl %eax; movl %eax,0x000000; cmpl %eax,0x100000; je 1b`**: 这是一个简单的测试，用于确认 A20 地址线是否真的被启用。
    *   它将 `eax` 的值写入物理地址 `0x000000`。
    *   然后比较 `eax` 和物理地址 `0x100000` (1MB) 处的值。
    *   如果 A20 地址线没有开启，那么对 `0x100000` 的写操作会因为地址回绕 (wrap-around) 而实际写入 `0x000000`。这种情况下，`cmpl %eax,0x100000` 将会相等，导致无限循环 `je 1b`。
    *   如果 A20 开启，`0x000000` 和 `0x100000` 是两个不同的物理地址，比较会失败，循环结束。

### 4. 检测和初始化数学协处理器 (FPU)

*   **`movl %cr0,%eax; andl $0x80000011,%eax`**: 读取控制寄存器 CR0，并保留 PG (bit 31, Paging), PE (bit 0, Protection Enable), ET (bit 4, Extension Type) 位。
*   **`orl $2,%eax; movl %eax,%cr0`**: 设置 CR0 的 MP (bit 1, Monitor Coprocessor) 位。MP 位控制 `WAIT` 指令与 TS (Task Switched) 位之间的交互。
*   **`call check_x87`**: 调用 `check_x87` 子程序来检查和设置 FPU。

    *   **`check_x87:`**:
        *   **`fninit`**: 初始化 FPU。
        *   **`fstsw %ax`**: 将 FPU 的状态字存入 `ax`。
        *   **`cmpb $0,%al`**: 检查状态字的低字节是否为0。如果为0，表示存在 FPU (通常是 387 或更高)。
        *   **`je 1f`**: 如果没有 FPU (或 287)，则跳转到标签 `1f`。
        *   **`movl %cr0,%eax; xorl $6,%eax; movl %eax,%cr0`**: 如果没有 FPU，或者模拟 FPU，则清除 MP 位并设置 EM (bit 2, Emulate Coprocessor) 位，这样当遇到 FPU 指令时会产生异常，允许操作系统模拟 FPU 功能。
        *   **`1: .byte 0xDB,0xE4`**: 这是 `fsetpm` 指令 (set protected mode)，用于 287 协处理器，被 387 及更高版本忽略。

### 5. 跳转到 `after_page_tables`

*   **`jmp after_page_tables`**: 在完成上述初始化后，跳转到 `after_page_tables` 标签，该标签位于页表的物理定义之后，以继续执行分页设置等后续步骤。

### 6. 页表定义

*   **`.org 0x1000; pg0:`**: 将当前位置设置为物理地址 `0x1000`，并定义符号 `pg0`。这是第一个页表的开始位置。内核在这里定义了四个页表 (`pg0`, `pg1`, `pg2`, `pg3`)，每个页表大小为 4KB。
    *   `_pg_dir` (页目录) 位于 `0x0000`。
    *   `pg0` 位于 `0x1000`。
    *   `pg1` 位于 `0x2000`。
    *   `pg2` 位于 `0x3000`。
    *   `pg3` 位于 `0x4000`。
*   这四个页表 (`pg0` 到 `pg3`) 总共可以映射 `4 * 1024 * 4KB = 16MB` 的物理内存。每个页表包含 1024 个页表项 (PTE)，每个 PTE 4字节，指向一个 4KB 大小的物理页。

### 7. 临时软盘区域

*   **`.org 0x5000; _tmp_floppy_area: .fill 1024,1,0`**: 定义了一个 1KB 大小的临时区域 `_tmp_floppy_area`，起始于物理地址 `0x5000`。这个区域用于软盘驱动程序，在 DMA (直接内存访问) 无法直接访问目标缓冲区时作为临时中转。它需要对齐以避免跨越 64KB 边界。

### 8. 调用 `main` 函数前的准备

*   **`after_page_tables:`**:
    *   **`pushl $0; pushl $0; pushl $0`**: 将三个0压入堆栈。这些是传递给 `_main` 函数的参数 (通常是 `envp`, `argv`, `argc`，但内核 `main` 函数不使用它们，所以这里是占位符)。
    *   **`pushl $L6`**: 将标签 `L6` 的地址压入堆栈。这是 `_main` 函数的返回地址。
    *   **`pushl $_main`**: 将 `_main` 函数的地址压入堆栈。
    *   **`jmp setup_paging`**: 跳转到 `setup_paging` 子程序。注意这里不是直接 `call _main`，而是通过 `setup_paging` 之后的 `ret` 指令间接跳转到 `_main` (因为 `_main` 的地址已经在栈顶)。
*   **`L6: jmp L6`**: 这是一个死循环。如果 `_main` 函数意外返回，程序将在此处挂起。正常情况下，内核的 `main` 函数不会返回。

### 9. 默认中断处理程序 `ignore_int`

*   **`int_msg: .asciz "Unknown interrupt\n\r"`**: 未知中断发生时显示的错误消息。
*   **`ignore_int:`**: 这是一个通用的、默认的中断处理例程。
    *   保存所有通用寄存器 (`eax`, `ecx`, `edx`) 和段寄存器 (`ds`, `es`, `fs`)。
    *   设置 `ds`, `es`, `fs` 为内核数据段 (`0x10`)。
    *   将 `int_msg` 的地址压栈作为参数。
    *   **`call _printk`**: 调用内核的打印函数 `_printk` (定义在 `kernel/printk.c`) 来显示错误消息。
    *   恢复所有寄存器。
    *   **`iret`**: 中断返回指令。
    这个默认处理程序会被 `setup_idt` 用来填充整个 IDT 表。

### 10. `setup_idt` 子程序

*   **`setup_idt:`**: 初始化中断描述符表 (IDT)。
    *   **`lea ignore_int,%edx`**: `edx` 指向 `ignore_int` 的偏移地址。
    *   **`movl $0x00080000,%eax`**: `eax` 的高16位是段选择子 (0x0008，即内核代码段 CS)，低16位将被 `ignore_int` 的偏移地址填充。
    *   **`movw %dx,%ax`**: 将 `ignore_int` 的低16位偏移存入 `ax`。
    *   **`movw $0x8E00,%dx`**: `dx` 存储中断门描述符的属性：`P=1` (存在)，`DPL=0` (内核权限)，`Type=0xE` (32位中断门)。
    *   **`lea _idt,%edi`**: `edi` 指向 IDT 表的起始地址 (`_idt`)。
    *   **`mov $256,%ecx`**: 循环计数器，设置256个中断门。
    *   **`rp_sidt: movl %eax,(%edi); movl %edx,4(%edi); addl $8,%edi; dec %ecx; jne rp_sidt`**: 循环填充 IDT 表。每个门描述符8字节：
        *   低32位：`eax` (包含 `ignore_int` 的低16位偏移和段选择子的高16位)
        *   高32位：`edx` (包含 `ignore_int` 的高16位偏移和门属性)
    *   **`lidt idt_descr`**: 加载 IDT 寄存器 (IDTR)，使其指向 `idt_descr` 定义的 IDT。
    *   **`ret`**: 返回。

### 11. `setup_gdt` 子程序

*   **`setup_gdt:`**: 初始化全局描述符表 (GDT)。
    *   **`lgdt gdt_descr`**: 加载 GDT 寄存器 (GDTR)，使其指向 `gdt_descr` 定义的 GDT。
    *   **`ret`**: 返回。这个 `ret` 指令会比较特殊，因为它在 GDT 更改后执行，会使用新的代码段描述符重新加载 CS 寄存器，并刷新指令预取队列。

### 12. `setup_paging` 子程序

*   **`setup_paging:`**: 设置并启用分页机制。
    1.  **清空页目录和前四个页表**:
        *   **`movl $1024*5,%ecx`**: `ecx` 设置为 `1024*5`，因为页目录 (1KB，但按页表大小算4KB) + 四个页表 (每个4KB) 总共是 5 个 4KB 的页。这里是按 `long` (4字节) 为单位进行清零，所以是 1024 * 5 个 `long`。
        *   **`xorl %eax,%eax`**: `eax` 清零，用于 `stosl`。
        *   **`xorl %edi,%edi`**: `edi` 指向物理地址 `0x000000` (即 `_pg_dir` 的位置)。
        *   **`cld;rep;stosl`**: 清除方向标志位，然后重复将 `eax` (0) 存入 `es:[edi]` 并增加 `edi`，直到 `ecx` 为0。这将把从 `0x000000` 开始的 `5 * 4KB = 20KB` 内存清零。

    2.  **填充页目录项 (PDE)**:
        *   **`movl $pg0+7,_pg_dir`**: 设置页目录的第0项，使其指向第一个页表 `pg0`。`+7` 表示设置标志位：Present (P=1), Read/Write (R/W=1), User/Supervisor (U/S=1)。
        *   类似地设置页目录的第1、2、3项，分别指向 `pg1`, `pg2`, `pg3`。
        *   `_pg_dir` 位于 `0x0`，`_pg_dir+4` 是第二个PDE，以此类推。

    3.  **填充四个页表的页表项 (PTE) - 实现恒等映射**:
        *   **`movl $pg3+4092,%edi`**: `edi` 指向最后一个页表 (`pg3`) 的最后一个页表项 (PTE) 的地址。每个页表4KB，1024个PTE，每个PTE 4字节，所以最后一个PTE的偏移是 `1023 * 4 = 4092`。
        *   **`movl $0xfff007,%eax`**: `eax` 设置为 `0x00FFF000 + 7`。这是用于填充PTE的值。 `0x00FFF000` 是 16MB 内存的最后一个 4KB 页的基地址 (第 `4095` 页，即 `(16*1024*1024 / 4096 - 1) * 4096 = (4096-1)*4096 = 0xFFF000`)。`+7` 是标志位 (P=1, R/W=1, U/S=1)。
        *   **`std`**: 设置方向标志位，使得 `stosl` 向下 (从高地址到低地址)填充。
        *   **`1: stosl; subl $0x1000,%eax; jge 1b`**:
            *   `stosl`: 将 `eax` 的值 (物理页框地址 + 标志) 存入 `es:[edi]`，然后 `edi` 减4。
            *   `subl $0x1000,%eax`: 将 `eax` 中的物理页框地址减去 `0x1000` (4KB)，准备下一个PTE。
            *   `jge 1b`: 如果 `eax` (物理页框地址部分) 仍然大于等于0，则继续填充。
            *   这个循环从最后一个页表 (`pg3`) 的最后一个PTE开始，向上填充所有四个页表的PTE，直到第一个页表 (`pg0`) 的第一个PTE。每个PTE都将对应的虚拟页映射到相同的物理页 (恒等映射)。例如，虚拟地址 `0x000000` 映射到物理地址 `0x000000`，虚拟地址 `0x001000` 映射到物理地址 `0x001000`，以此类推，直到 `16MB - 4KB` (即 `0x00FFF000`)。

    4.  **加载页目录基址寄存器 (CR3) 并启用分页**:
        *   **`xorl %eax,%eax; movl %eax,%cr3`**: 将 `_pg_dir` 的物理地址 (0) 加载到 CR3 寄存器。CR3 保存页目录表的物理基地址。
        *   **`movl %cr0,%eax; orl $0x80000000,%eax; movl %eax,%cr0`**: 读取 CR0，设置其最高位 PG (bit 31) 为1，然后写回 CR0。这将正式启用分页机制。
        *   **`ret`**: 返回。这条 `ret` 指令非常关键。它会导致 CPU 清空指令预取队列，因为在分页启用后，之前的线性地址可能不再映射到相同的物理地址。然后它会从堆栈中弹出 `_main` 的地址 (之前 `pushl $_main` 推入的) 并跳转到那里执行。此时，内核已经在分页模式下运行。

### 13. IDT 和 GDT 的描述符和表定义

*   **`idt_descr: .word 256*8-1; .long _idt`**: IDT 描述符。
    *   `.word 256*8-1`: IDT 表的大小限制 (limit)，256个条目，每个8字节。
    *   `.long _idt`: IDT 表的32位线性基地址。
*   **`gdt_descr: .word 256*8-1; .long _gdt`**: GDT 描述符。
    *   `.word 256*8-1`: GDT 表的大小限制。
    *   `.long _gdt`: GDT 表的32位线性基地址。
*   **`_idt: .fill 256,8,0`**: IDT 表的实际存储空间，256个条目，每个8字节，初始填充为0。后续由 `setup_idt` 填充。
*   **`_gdt: .quad ...`**: GDT 表的实际存储空间。
    *   **`_gdt[0]: .quad 0x0000000000000000`**: 第一个描述符必须是 NULL 描述符。
    *   **`_gdt[1]: .quad 0x00c09a0000000fff`**: 内核代码段描述符 (索引1，选择子0x08)。
        *   Base=0x00000000, Limit=0x000FFFFF (乘以4K页粒度后是4GB，但这里实际是16MB，因为`fff`是段限长低12位，高4位在另一个字节) -> 实际Limit是 `0xFFFFF * 1 = 1MB` * 16 = 16MB (G=1, D=1, Limit=0xFFFF -> 4GB, 这里是16MB，需要仔细看GDT各项)。
        *   `0x00c0fa0000000fff` (示例，与Linux0.11不同) -> Base=0, Limit=0xfffff (1MB), G=1 (页粒度, limit*4K), P=1, DPL=0, S=1, Type=Code, A=0, R=1, C=0.
        *   Linux 0.11 GDT: `0x00c09a0000000fff`
            *   Limit (low 16 bits): `0x0fff` (与 `0000` 组合)
            *   Base (low 16 bits): `0x0000`
            *   Base (mid 8 bits): `0x00`
            *   Access byte: `0x9a` (P=1, DPL=00, S=1, Type=1010 -> Execute/Read Code, Accessed=0)
            *   Granularity/Limit (high 4 bits): `0xc0` (G=1 (limit is in 4KB pages), D=1 (32-bit default), AVL=0, Limit (high 4 bits)=0)
            *   Base (high 8 bits): `0x00`
            *   So, Base = `0x00000000`. Limit = `0x000FF` (来自 `0x0fff`的低8位和`0xc0`的低4位组合的 `0F`，然后 `0xFF` 来自 `0xfff`的高八位) -> 实际是 `0x000FFFFF` (1MB) * 4KB (G=1) = 4GB。但这里注释是16Mb。
            *   Linux 0.11 的GDT项 `0x00c09a0000000fff` 表示：
                Base = `0x00000000`
                Limit = `0x000fffff` (即 `0xfff` 和 `0x0f` 来自 `0xcf` 中的 `f`)，由于 `G=1`，所以实际 Limit 是 `0xfffff * 4KB = 1MB * 4KB = 4GB`。然而，注释和实际使用中，内核段被限制在16MB。这是通过 `setup.s` 中加载GDT时设置的 limit 实现的，或者说，这里的 `0xfff` 配合 `G=0` 时是 `0xfff` 字节，`G=1` 时是 `0xfff`页。
                Linux 0.11的 `head.s` GDT条目 `0x00c09a0000000fff` (代码段) 和 `0x00c0920000000fff` (数据段) 都定义了基址为0，界限为4GB（因为G=1, limit=0xfffff）。但实际上内核只使用了前面16MB。
    *   **`_gdt[2]: .quad 0x00c0920000000fff`**: 内核数据段描述符 (索引2，选择子0x10)。与代码段类似，可读写。
    *   **`_gdt[3]: .quad 0x0000000000000000`**: 临时用的，未使用。
    *   **`.fill 252,8,0`**: 预留了剩余 GDT 空间，用于后续的 LDT (局部描述符表) 和 TSS (任务状态段)。

## 总结

`head.s` 是内核启动的关键阶段，它完成了从16位实模式到32位保护模式的转换后的核心初始化工作：
*   建立了基本的内存管理模型 (GDT)。
*   建立了中断处理机制的框架 (IDT)。
*   确保了关键的硬件特性 (A20 地址线，FPU) 已正确设置。
*   最重要的是，它初始化并启用了分页机制，这是现代操作系统内存管理的基础。
*   最后，它将控制权安全地交给了C语言编写的内核主函数 `_main`。

这段代码虽然简短，但涉及了大量的处理器架构底层细节，是理解操作系统启动过程的重要部分。它的执行标志着系统真正进入了受保护的、具备现代内存管理能力的运行环境。
