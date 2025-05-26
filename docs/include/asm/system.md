# system.h 文件详解

`include/asm/system.h` 文件是 Linux 0.11 内核中一个非常重要的底层头文件，专门为 x86 架构定义。它包含了一系列宏和内联汇编函数，用于执行与CPU控制、中断管理、任务状态段(TSS)和局部描述符表(LDT)设置等核心系统级操作。这些操作通常是保护模式下操作系统内核的特权功能。

## 核心功能

1.  **模式切换**:
    *   `move_to_user_mode()`: 从内核模式切换到用户模式。这是一个宏，通过构造一个 `iret` (中断返回) 指令的栈帧来实现。
2.  **中断控制**:
    *   `sti()`: 开中断 (Set Interrupt Flag)。
    *   `cli()`: 关中断 (Clear Interrupt Flag)。
3.  **空操作**:
    *   `nop()`: 执行一个空操作指令 (`nop`)。
4.  **中断返回**:
    *   `iret()`: 执行中断返回指令。
5.  **门描述符设置**:
    *   `_set_gate(gate_addr, type, dpl, addr)`: 设置门描述符（中断门或陷阱门）的底层宏。
    *   `set_intr_gate(n, addr)`: 设置中断描述符表 (IDT) 中的第 `n` 项为一个中断门。
    *   `set_trap_gate(n, addr)`: 设置 IDT 中的第 `n` 项为一个陷阱门。
    *   `set_system_gate(n, addr)`: 设置 IDT 中的第 `n` 项为一个系统门 (陷阱门，但DPL=3，允许用户态通过 `int n` 指令触发)。
6.  **段描述符设置 (通用方式)**:
    *   `_set_seg_desc(gate_addr, type, dpl, base, limit)`: (注意: 虽然参数名是 `gate_addr`，但这个宏用于设置通用的段描述符，如代码段或数据段，而不是门描述符)。它直接通过C语言位操作构造一个段描述符。
7.  **TSS 和 LDT 描述符设置**:
    *   `_set_tssldt_desc(n, addr, type)`: 设置任务状态段 (TSS) 或局部描述符表 (LDT) 描述符的底层宏，使用内联汇编。
    *   `set_tss_desc(n, addr)`: 将全局描述符表 (GDT) 的第 `n` 项设置为一个TSS描述符，指向地址 `addr`。
    *   `set_ldt_desc(n, addr)`: 将 GDT 的第 `n` 项设置为一个LDT描述符，指向地址 `addr`。

## 宏和函数定义详解

### `move_to_user_mode()`

*   **功能**: 从内核态切换到用户态。这是通过模拟一次中断返回 (`iret`) 实现的。
*   **定义**:
    ```c
    #define move_to_user_mode() \
    __asm__ ("movl %%esp,%%eax\n\t" \  // 1. 保存当前内核栈顶 esp 到 eax
        "pushl $0x17\n\t" \        // 2. 压入用户态 SS (0x17 - RPL=3 的数据段选择子)
        "pushl %%eax\n\t" \        // 3. 压入用户态 ESP (使用当前内核栈顶作为用户栈顶)
        "pushfl\n\t" \             // 4. 压入当前 EFLAGS
        "pushl $0x0f\n\t" \        // 5. 压入用户态 CS (0x0F - RPL=3 的代码段选择子)
        "pushl $1f\n\t" \          // 6. 压入返回地址 EIP (标签 1)
        "iret\n" \                 // 7. 执行中断返回，切换到用户态
        "1:\tmovl $0x17,%%eax\n\t" \  // 8. (iret返回后在用户态执行) 设置 eax 为用户数据段选择子
        "movw %%ax,%%ds\n\t" \     // 9. 设置 DS
        "movw %%ax,%%es\n\t" \     // 10. 设置 ES
        "movw %%ax,%%fs\n\t" \     // 11. 设置 FS
        "movw %%ax,%%gs" \         // 12. 设置 GS
        :::"ax")                   // eax 寄存器内容被改变
    ```
*   **解释**:
    1.  将当前内核栈指针 `esp` 保存到 `eax`。
    2.  将用户数据段选择子 (0x17) 压栈，作为 `iret` 恢复后的 `ss`。
    3.  将之前保存的内核栈指针 (现在是 `eax` 中的值) 压栈，作为 `iret` 恢复后的 `esp`。这意味着用户态将和内核态共享同一个栈的起始区域，这在简单的系统中是可能的，但通常不推荐。
    4.  将当前的 `eflags` 寄存器压栈。
    5.  将用户代码段选择子 (0x0F) 压栈，作为 `iret` 恢复后的 `cs`。
    6.  将标签 `1:` 的地址压栈，作为 `iret` 恢复后的 `eip` (指令指针)。
    7.  `iret` 指令执行：从栈中依次弹出 EIP, CS, EFLAGS, ESP, SS，并切换到 CS 指定的特权级（用户态）。CPU 开始执行标签 `1:` 处的指令。
    8.  在用户态，标签 `1:` 处的代码首先将用户数据段选择子 (0x17) 加载到 `eax`。
    9.  然后用 `ax` (eax的低16位) 初始化 `ds`, `es`, `fs`, `gs` 段寄存器。

### `sti()` 和 `cli()`

*   `#define sti() __asm__ ("sti"::)`: 执行 `sti` (Set Interrupt Flag) 汇编指令，开启CPU对可屏蔽中断的响应。
*   `#define cli() __asm__ ("cli"::)`: 执行 `cli` (Clear Interrupt Flag) 汇编指令，禁止CPU对可屏蔽中断的响应。

### `nop()`

*   `#define nop() __asm__ ("nop"::)`: 执行 `nop` (No Operation) 汇编指令，该指令不执行任何操作，通常用于短延时或对齐。

### `iret()`

*   `#define iret() __asm__ ("iret"::)`: 执行 `iret` (Interrupt Return) 汇编指令，用于从中断或异常处理程序返回到被中断的代码。它会从栈中恢复 EIP, CS, EFLAGS，有时还会恢复 ESP 和 SS (取决于是否发生特权级切换)。

### 门描述符设置宏

门描述符（中断门、陷阱门、调用门、任务门）是IDT或GDT中的条目，用于定义中断、异常或特定调用的入口点。一个门描述符是8字节。

*   `_set_gate(gate_addr, type, dpl, addr)`:
    *   **功能**: 设置位于 `gate_addr` (IDT或GDT中某个门描述符的起始地址) 的门描述符。
    *   **参数**:
        *   `gate_addr`: 门描述符的内存地址。
        *   `type`: 门的类型 (如14代表中断门，15代表陷阱门)。
        *   `dpl`: 描述符特权级 (Descriptor Privilege Level, 0-3)。
        *   `addr`: 门处理程序的入口点线性地址。
    *   **实现**: 通过内联汇编将门描述符的各个字段（包括处理程序地址 `addr` 的低16位和高16位，段选择子 `0x0008` 即内核代码段，以及类型、DPL、存在位P=1等属性）写入 `gate_addr` 指向的8字节内存中。
        *   `0x0008`: 内核代码段选择子。
        *   `0x8000+(dpl<<13)+(type<<8)`: 构造门描述符高4字节中的Access Rights Byte和Type/S/DPL/P字段。`0x8000` 对应 P=1 (存在位)。
*   `set_intr_gate(n, addr)`:
    *   将IDT表 (`idt`) 中的第 `n` 项设置为一个**中断门**。
    *   类型为14 (中断门)，DPL为0 (内核级)。
    *   当中断门被调用时，IF标志位（中断允许标志）会被自动清零（关中断）。
*   `set_trap_gate(n, addr)`:
    *   将IDT表 (`idt`) 中的第 `n` 项设置为一个**陷阱门**。
    *   类型为15 (陷阱门)，DPL为0 (内核级)。
    *   陷阱门被调用时，IF标志位不会自动改变。
*   `set_system_gate(n, addr)`:
    *   将IDT表 (`idt`) 中的第 `n` 项设置为一个**系统门**。
    *   类型为15 (陷阱门)，DPL为3 (用户级)。
    *   这允许用户模式的程序通过 `int n` 指令（例如 `int 0x80`）来触发这个门，从而进入内核执行相应的处理程序。

### 段描述符设置宏 (`_set_seg_desc`)

*   `_set_seg_desc(gate_addr, type, dpl, base, limit)`:
    *   **功能**: 设置位于 `gate_addr` (GDT或LDT中某个段描述符的起始地址) 的段描述符。
    *   **参数**:
        *   `gate_addr`: 指向8字节段描述符的指针 (类型为 `unsigned int *`，所以 `gate_addr` 指向低4字节，`gate_addr+1` 指向高4字节)。
        *   `type`: 段类型 (如代码段、数据段)。
        *   `dpl`: 描述符特权级。
        *   `base`: 段的32位基地址。
        *   `limit`: 段的20位界限。
    *   **实现**: 直接通过C语言的位操作来构造段描述符的两个32位部分，并写入到 `*gate_addr` 和 `*(gate_addr+1)`。
        *   `0x00408000`: 这部分包含了G=0 (字节粒度), D=1 (32位段), P=1 (存在) 等固定属性。实际的type会通过 `(type<<8)` 加入。Linux 0.11 GDT中代码段和数据段通常设置G=1（页粒度），D=1，所以这个宏可能用于LDT中的段，或者GDT中某些特殊段。
        *   注意：这个宏的实现方式与GDT中常见的代码/数据段（如0xC09A00000000FFFF中的 `0xC0` 代表 G=1, D=1）的粒度位(G)设置不同。它设置的是G=0。

### TSS 和 LDT 描述符设置宏

TSS和LDT描述符也是GDT中的条目，它们指向TSS或LDT结构。

*   `_set_tssldt_desc(n, addr, type_str)`:
    *   **功能**: 设置GDT中第 `n` 个描述符（每个描述符8字节）为一个TSS或LDT描述符。
    *   **参数**:
        *   `n`: 指向GDT中目标描述符的起始地址 (char *)。
        *   `addr`: TSS或LDT结构的线性基地址。
        *   `type_str`: 描述符的类型字节字符串 (如 "0x89" 代表TSS, "0x82" 代表LDT)。
    *   **实现**: 通过内联汇编将TSS/LDT描述符的各个字段（基地址 `addr`，界限103 (0x67)，类型字节 `type_str`，P=1，DPL=0，G=0等）写入到 `n` 指向的8字节内存中。
        *   `movw $104, %1`: 设置段界限为104字节 - 1 = 103字节。TSS最小为104字节。LDT的界限是实际LDT表的大小减1。
        *   `movb $type, %4`: 设置类型字节。
*   `set_tss_desc(n, addr)`:
    *   调用 `_set_tssldt_desc` 将GDT表项 `n` 设置为一个**TSS描述符** (类型0x89，表示386可用的TSS)。
*   `set_ldt_desc(n, addr)`:
    *   调用 `_set_tssldt_desc` 将GDT表项 `n` 设置为一个**LDT描述符** (类型0x82)。

## 总结

`include/asm/system.h` 文件是Linux 0.11内核进行底层系统控制和初始化的基石。它提供的宏封装了许多特权指令和复杂的数据结构设置（如门描述符、段描述符），使得内核的其他部分（如启动代码 `head.s`、中断处理初始化 `traps.c`、进程调度 `sched.c`）能够以一种相对简洁的方式配置CPU的保护模式环境、处理中断和异常、以及管理任务和内存分段。这些宏的正确使用对于操作系统的稳定运行至关重要。Okay, I have created `docs/include/asm/system.md` with the detailed Chinese explanation for `include/asm/system.h`.

All files in `include/asm/` are now processed.
- `docs/include/asm/io.md`
- `docs/include/asm/memory.md`
- `docs/include/asm/segment.md`
- `docs/include/asm/system.md`

Next, I will process `include/const.h`.
1. Create `docs/include/const.md`.
2. Read `include/const.h`.
3. Write the explanation into `docs/include/const.md`.

First, I'll read `include/const.h`.
