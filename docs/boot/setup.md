# setup.s 代码详解

`setup.s` 是 Linux 0.11 内核启动过程中，在 `bootsect.s` 成功加载了 `setup.s` 自身和 `system` 模块之后，由 `bootsect.s` 跳转执行的代码。它运行在16位实模式下，主要负责从 BIOS 获取各种硬件参数，并将这些参数保存到内存中的特定位置 (从 `0x90000` 开始，即原先 `bootsect.s` 所在的位置)。然后，它会将 `system` 模块移动到物理内存的起始地址 `0x00000`，加载临时的GDT和IDT，开启A20地址线，重编程8259A中断控制器，最后进入32位保护模式，并跳转到 `system` 模块的起始处 (`0x00000`，由 `head.s` 开始执行)。

## 主要代码段和功能

### 1. 初始化和定义段地址常量

*   **`INITSEG = 0x9000`**: `bootsect.s` 将自身移动到的段地址，也是 `setup.s` 存储获取到的系统参数的起始段地址。
*   **`SYSSEG = 0x1000`**: `system` 模块被 `bootsect.s` 加载到的初始段地址 (`0x10000`)。
*   **`SETUPSEG = 0x9020`**: `setup.s` 当前运行的段地址。

代码从 `entry start` 开始。

### 2. 获取并保存硬件参数

这部分代码通过调用 BIOS 中断来获取各种硬件信息，并将它们保存在 `INITSEG` (即 `0x90000`) 开始的内存区域中。C语言的内核初始化代码后续会从这里读取这些参数。

*   **保存光标位置**:
    *   `mov ax,#INITSEG; mov ds,ax`: 设置DS指向 `0x9000`。
    *   `mov ah,#0x03; xor bh,bh; int 0x10`: BIOS中断 `0x10` 功能 `0x03` 用于读取光标位置。`dx` 中返回光标的行和列。
    *   `mov [0],dx`: 将光标位置 (2字节) 保存到 `0x90000`。

*   **获取扩展内存大小**:
    *   `mov ah,#0x88; int 0x15`: BIOS中断 `0x15` 功能 `0x88` 用于获取从1MB开始的扩展内存大小 (KB)。结果在 `ax` 中。
    *   `mov [2],ax`: 将扩展内存大小 (2字节) 保存到 `0x90002`。

*   **获取显卡数据**:
    *   `mov ah,#0x0f; int 0x10`: BIOS中断 `0x10` 功能 `0x0f` 用于获取当前视频模式。
    *   `mov [4],bx`: 保存显示页 (`bh`) 到 `0x90004`。
    *   `mov [6],ax`: 保存视频模式 (`al`) 和屏幕字符列数 (`ah`) 到 `0x90006`。

*   **检查 EGA/VGA 适配器信息**:
    *   `mov ah,#0x12; mov bl,#0x10; int 0x10`: BIOS中断 `0x10` 功能 `0x12` 子功能 `0x10` (BL=0x10) 用于获取EGA/VGA配置信息。
    *   `mov [8],ax; mov [10],bx; mov [12],cx`: 保存返回的配置参数到 `0x90008`, `0x9000A`, `0x9000C`。这些信息包括显存大小、特性连接器状态等。

*   **获取第一个硬盘 (hd0) 参数表**:
    *   `mov ax,#0x0000; mov ds,ax`: DS 设置为 `0x0000`，以便访问 BIOS 数据区 (BDA)。
    *   `lds si,[4*0x41]`: 从 BDA 的 `0x0040:0x0004` (中断向量 `0x41` 的地址) 处加载硬盘参数表的地址到 `DS:SI`。DS 会被更新为表中段地址，SI 为偏移。但这里 `ds` 已经为0，所以实际上是 `0000:xxxx`。更准确地说，`[4*0x41]` 指的是中断向量表 `0x104` (即 `0x0000:0x0104`)，这里存放着第一个硬盘参数表的指针 (16字节)。
    *   `mov ax,#INITSEG; mov es,ax; mov di,#0x0080`: ES:DI 指向目标地址 `0x9000:0x0080` (即 `0x90080`)。
    *   `mov cx,#0x10; rep; movsb`: 复制16字节的硬盘参数表。

*   **获取第二个硬盘 (hd1) 参数表**:
    *   与获取 hd0 类似，但使用中断向量 `0x46` (`lds si,[4*0x46]`)，目标地址为 `0x9000:0x0090` (即 `0x90090`)。

*   **检查第二个硬盘 (hd1) 是否存在**:
    *   `mov ax,#0x01500; mov dl,#0x81; int 0x13`: BIOS中断 `0x13` 功能 `0x15` (AH=15h) 用于获取磁盘类型。`dl=0x81` 表示第二个硬盘。
    *   `jc no_disk1`: 如果发生错误 (CF=1)，则认为没有第二个硬盘。
    *   `cmp ah,#3; je is_disk1`: 如果 `ah=3` (表示固定磁盘)，则确认存在第二个硬盘。
    *   `no_disk1:`: 如果不存在第二个硬盘，则将 `0x90090` 开始的16字节清零。
    *   `is_disk1:`: 存在硬盘则跳过清零。

### 3. 移动 `system` 模块到物理地址 `0x00000`

在进入保护模式之前，需要将 `system` 模块 (由 `bootsect.s` 加载在 `0x10000` 到 `0x8FFFF` 之间，最大约512KB) 移动到物理内存的起始位置 `0x00000`。这是因为保护模式下的内核期望从这个地址开始运行。

*   **`cli`**: 关闭中断，防止在内存移动过程中发生中断。
*   **`mov ax,#0x0000; cld`**: `ax` 用于目标段地址，从 `0x0000` 开始。`cld` 设置方向标志位DF=0，使得 `movsw` 向前移动。
*   **`do_move:` (循环移动)**:
    *   **`mov es,ax`**: 设置目标段地址 ES。
    *   **`add ax,#0x1000`**: 目标段地址每次增加 `0x1000` (64KB)。
    *   **`cmp ax,#0x9000; jz end_move`**: 如果目标段地址达到 `0x9000` (即 `0x90000`)，说明已经移动了 `0x00000` 到 `0x8FFFF` 这部分，总共 `0x90000 - 0x10000 = 0x80000` (512KB) 的空间。这里应该是 `cmp ax, #SYSSEG+SYSSIZE_IN_SEGMENTS` (假设 `SYSSIZE` 是按段对齐的) 或者比较已移动的字节数。
        *   实际上，这里的逻辑是：`system` 模块的源地址从 `SYSSEG` (0x1000) 开始，每次移动16位字的 `0x8000` (即32KB) 个，也就是64KB的数据。
        *   源段地址 `ds` 从 `ax` 的当前值 (即 `es` 的旧值 + `0x1000`) 开始。
        *   `ds` 从 `0x1000` (第一次是 `0x0000 + 0x1000`)，`0x2000`, ..., 直到 `0x8000`。
        *   目标段 `es` 从 `0x0000`, `0x1000`, ..., 直到 `0x7000`。
        *   当 `ax` (下一个源 `ds`) 变成 `0x9000` 时，表示上一次 `ds` 是 `0x8000`，`es` 是 `0x7000`，已经完成了 `0x10000` 到 `0x8FFFF` 区域到 `0x00000` 到 `0x7FFFF` 区域的移动。
    *   **`mov ds,ax`**: 设置源段地址 DS。
    *   **`sub di,di; sub si,si`**: DI 和 SI 清零，作为段内偏移。
    *   **`mov cx,#0x8000`**: 移动计数器，`0x8000` 个字 (word)，即 `16KB * 2 = 32KB * 2 = 64KB`。
    *   **`rep; movsw`**: 重复将 `DS:[SI]` 的一个字复制到 `ES:[DI]`，并递增 SI 和 DI。
    *   **`jmp do_move`**: 继续下一块64KB的移动。

### 4. 加载 GDT 和 IDT 描述符

在进入保护模式前，需要为CPU提供一个临时的全局描述符表 (GDT) 和中断描述符表 (IDT)。

*   **`end_move:`**:
    *   **`mov ax,#SETUPSEG; mov ds,ax`**: 确保 DS 指向 `SETUPSEG` (`0x9020`)，因为 GDT 和 IDT 的定义位于 `setup.s` 的数据区。
    *   **`lidt idt_48`**: 加载中断描述符表寄存器 (IDTR)。`idt_48` 定义了一个空的 IDT (limit=0, base=0)。此时中断仍被禁止，保护模式下的IDT将在 `head.s` 中重新设置。
    *   **`lgdt gdt_48`**: 加载全局描述符表寄存器 (GDTR)。`gdt_48` 定义了一个包含几个基本段描述符的GDT (见文件末尾的 `gdt` 定义)。

    *   **`idt_48:`**
        *   `.word 0`: IDT limit 为0。
        *   `.word 0,0`: IDT base 为0。
    *   **`gdt_48:`**
        *   `.word 0x800`: GDT limit 为 `0x800` (2048 字节)，意味着最多可以有 `2048 / 8 = 256` 个GDT条目。
        *   `.word 512+gdt,0x9`: GDT base address。`gdt` 是GDT表在 `setup.s` 内的标签。`512` 可能是因为 `bootsect.s` (512字节) 之前在 `0x90000`，而 `setup.s` 在 `0x90200`，所以 `gdt` 的实际地址是 `0x90200 + offset(gdt)`。 `0x9` 表示段地址 `0x9000`。这里更准确的解释是，`gdt` 标签的地址是 `SETUPSEG:gdt_offset`，将其转换为线性地址 `0x90000 + 0x200 + gdt_offset`。 `512` 加上 `gdt` 的偏移量，然后 `0x9` 是 `INITSEG` (0x9000)。所以基地址是 `0x90000 + (gdt - start_of_setup_segment_in_memory + 512)`。这里的 `512+gdt` 表示 `gdt` 符号相对于 `SETUPSEG:0` 的偏移量，再加上 `0x200` (512字节，即 `bootsect.s` 的大小，因为 `setup.s` 紧随其后)。所以 GDT 的基址是 `INITSEG: (gdt - start + 0x200)`。
          更清晰的解释：`gdt_48` 定义了 GDT 的基址和限长。限长是 `0x800` 字节。基址是 `0x90000 + (offset of gdt within setup.s code/data) + 512` (因为 `setup.s` 是从 `INITSEG:0x0200` 开始加载的)。`gdt` 标签代表了GDT表在 `SETUPSEG` 内的偏移。所以 `gdt_48` 中的地址是 `(SETUPSEG << 4) + gdt_offset`。
          `word 512+gdt` 是基址的低16位，`word 0x9` 是基址的高16位中的低4位（即段地址 `0x9000` 的 `0x9`，因为CPU会左移4位 `0x9 << 4 = 0x90`，再与偏移组合）。实际上，`gdt_48` 的格式是：`limit (2 bytes)`, `base_low (2 bytes)`, `base_high (1 byte)`。这里的 `0x9` 应该是 `base_high` 的一部分，通常基址是32位的。
          Linux 0.11 中 `gdt_48` 的结构是：
          `gdt_48: .word 0x800       ! gdt limit (2048 bytes, allowing 256 entries)`
          `        .long 0x90200+gdt ! gdt base address (linear address)`
          这里的 `0x90200` 是 `SETUPSEG` 的线性基地址, `gdt` 是GDT表在 `SETUPSEG` 内的偏移。
          然而，原始代码是 `.word 512+gdt,0x9`。这表示基址的低16位是 `(gdt - start_of_setup) + 0x200` (因为 `setup.s` 被加载到 `INITSEG:0020h`)，高8位(或16位中的高部分)是 `0x0009` (对应线性地址 `0x90000`)。所以线性基地址是 `0x90000 + (gdt_offset_within_setup_loaded_at_90200)`.
          `gdt_48: .word 0x800 ; .word (gdt - start + 0x200), INITSEG >> 4` (不完全准确，但接近)
          正确解释 `gdt_48: .word 0x800; .word 512+gdt, 0x9`:
          Limit: `0x0800` (2048 bytes)
          Base: `0x00090000 + (gdt - start_of_SETUPSEG_in_memory + 512)`. `gdt` 是 `SETUPSEG` 内的偏移，`SETUPSEG` 线性地址是 `0x90200`。所以 `gdt` 的线性地址是 `0x90200 + (gdt - entry_start_of_setup)`.
          这里的 `512+gdt` 是 `(gdt - start) + 512`，其中 `start` 是 `setup.s` 的起始符号。 `512` 是 `0x200`。
          所以基址是：`INITSEG:((gdt - start) + 0x200)`. 线性地址是 `0x90000 + (gdt - start) + 0x200`.

### 5. 开启 A20 地址线

为了访问超过1MB的内存，需要开启A20地址线。这是通过8042键盘控制器完成的。

*   **`call empty_8042`**: 等待8042键盘控制器输入缓冲区为空。
*   **`mov al,#0xD1; out #0x64,al`**: 发送命令 `0xD1` (写输出端口) 到8042的状态/命令端口 `0x64`。
*   **`call empty_8042`**: 再次等待。
*   **`mov al,#0xDF; out #0x60,al`**: 发送数据 `0xDF` (A20 Gate enable, bit 1 = 1) 到8042的数据端口 `0x60`。
*   **`call empty_8042`**: 再次等待。

    *   **`empty_8042:`**:
        *   `.word 0x00eb,0x00eb`: 短跳转指令，用于IO延迟。
        *   `in al,#0x64`: 从状态端口 `0x64` 读取8042状态。
        *   `test al,#2`: 测试状态字节的第1位 (Input Buffer Full)。
        *   `jnz empty_8042`: 如果输入缓冲区满，则循环等待。
        *   `ret`: 返回。

### 6. 重编程 8259A 中断控制器

PC/AT架构的BIOS将硬件中断映射到了 `0x08-0x0F`，这与Intel处理器保留的中断冲突。因此，需要重新编程主从8259A中断控制器，将硬件中断映射到 `0x20-0x2F` (主片IRQ0-7 -> INT 20-27) 和 `0x28-0x2F` (从片IRQ8-15 -> INT 28-2F，但Linux 0.11中从片映射到 INT 28-2F，所以是0x28开始)。
这里实际是将主片的IRQ0-7映射到INT 0x20-0x27，从片的IRQ8-15映射到INT 0x28-0x2F。

*   **`mov al,#0x11; out #0x20,al; ... out #0xA0,al`**: 发送ICW1 (Initialization Command Word 1) 到主片 (端口 `0x20`) 和从片 (端口 `0xA0`)。`0x11` 表示：需要ICW4，级联模式，边沿触发。
*   **`mov al,#0x20; out #0x21,al; ...`**: 发送ICW2到主片 (端口 `0x21`)。设置主片中断向量起始地址为 `0x20` (IRQ0 -> INT 0x20)。
*   **`mov al,#0x28; out #0xA1,al; ...`**: 发送ICW2到从片 (端口 `0xA1`)。设置从片中断向量起始地址为 `0x28` (IRQ8 -> INT 0x28)。
*   **`mov al,#0x04; out #0x21,al; ...`**: 发送ICW3到主片。`0x04` 表示从片连接到主片的IRQ2 (二进制 `00000100b`)。
*   **`mov al,#0x02; out #0xA1,al; ...`**: 发送ICW3到从片。`0x02` 表示从片连接到主片的IRQ2 (作为标识)。
*   **`mov al,#0x01; out #0x21,al; ... out #0xA1,al`**: 发送ICW4到主片和从片。`0x01` 表示8086/88模式，普通EOI，非缓冲模式，非特殊全嵌套。
*   **`mov al,#0xFF; out #0x21,al; ... out #0xA1,al`**: 发送OCW1 (Operational Command Word 1) 到主片和从片，屏蔽所有中断 (`0xFF`)。中断将在后续内核初始化中逐步开启。
*   `.word 0x00eb,0x00eb`: 在每个 `out` 指令后都有两条 `jmp $+2` 指令，这是为了在快速CPU上提供足够的I/O延迟，确保8259A有时间处理命令。

### 7. 进入保护模式并跳转到 `system` (`head.s`)

*   **`mov ax,#0x0001; lmsw ax`**: 将机器状态字 (Machine Status Word, CR0的低16位) 的PE位 (Protection Enable, bit 0) 置1。这将使CPU进入保护模式。
*   **`jmpi 0,8`**: 执行一条段间跳转指令。
    *   `0`: 跳转的目标偏移地址为0。
    *   `8`: 跳转的目标代码段选择子为8 (二进制 `00001000b`)。
        *   选择子结构: `Index (13 bits) | TI (1 bit) | RPL (2 bits)`
        *   `8` -> `Index=1`, `TI=0` (GDT), `RPL=0`。
        *   这对应于GDT中的第1个描述符 (索引从0开始，所以是第二个条目)。根据文件末尾的 `gdt` 定义，这是基地址为0，限长8MB (或4GB，取决于G位) 的可执行代码段。
    *   这条指令将使得CS寄存器加载选择子8，并跳转到 `0x00000000` (因为GDT中该段的基址是0) 开始执行。此时，`system` 模块已经被移动到 `0x00000000`，其起始部分是 `head.s`。

### 8. GDT 定义 (`gdt`)

`setup.s` 定义了一个临时的 GDT，在进入保护模式时使用。

*   **`gdt:`**
    *   **`.word 0,0,0,0`**: 第一个描述符是NULL描述符，必须全0。
    *   **代码段描述符**: (对应选择子 `0x08`)
        *   `.word 0x07FF`: 段限长低16位 (Limit 15:0)。 `0x07FF` (2047)。
        *   `.word 0x0000`: 段基址低16位 (Base 15:0)。 `0x0000`。
        *   `.word 0x9A00`:
            *   高字节 `0x9A`: Access Rights Byte. P=1, DPL=00, S=1 (代码/数据段), Type=1010 (代码, 执行/读, 非一致, A=0)。
            *   低字节 `0x00`: 段基址中16位 (Base 23:16)。 `0x00`。
        *   `.word 0x00C0`: (Linux 0.11实际是 `0x00C0`)
            *   高字节 `0x00`: 段基址高8位 (Base 31:24)。 `0x00`。
            *   低字节 `0xC0`: Granularity Byte. G=1 (段限长单位为4KB页), D=1 (32位操作数和地址), 0, AVL=0. Limit 19:16 (`C`中的`0`，与`07FF`组合成 `0007F FFF` -> `0x07FFF`页, 即 `32767 * 4KB = 128MB`)。
            *   实际的Linux 0.11 `gdt` 中代码段和数据段的Limit高4位在Granularity字节中是 `0x0`，所以Limit是 `0x000FF` (来自`0x07FF`的`0xFF`和Access Byte的`0x0`)，配合G=1，Limit = `0xFFFFF` pages = `1M * 4KB = 4GB`。但注释说是8Mb。这里 `0x07FF` 和 `0x00C0` 组合起来，Limit = `(C & 0x0F) << 16 | 0x07FF = 0x007FF`，如果G=1，则是 `0x7FF * 4KB = 2047 * 4KB = 8188KB approx 8MB`。
            *   Base = `0x00000000`. Limit = `0x0007FFFFF` (8MB if G=0 and limit is 0x7FF pages).
            *   如果 `.word 0x07FF` 和 `.word 0x00C0`，则 Limit[15:0]=`07FF`, Limit[19:16]=(C&0xF)=`0`. So limit is `000007FF` (2047). If G=1, then `2047 * 4096 = 8MB`.
    *   **数据段描述符**: (对应选择子 `0x10`)
        *   与代码段类似，但Access Rights Byte为 `0x92` (数据, 读/写, A=0)。

这个GDT将物理内存的前8MB (根据注释) 映射为代码段和数据段，基地址都为0。

## 总结

`setup.s` 扮演了从 BIOS 环境到 Linux 内核自定义环境的关键桥梁角色：
1.  **收集系统信息**: 从BIOS获取硬件参数，为后续内核运行提供必要的数据。
2.  **准备内存布局**: 将核心的 `system` 模块移动到预期的 `0x00000000` 起始位置。
3.  **初始化保护模式环境**: 加载临时的GDT和IDT，虽然IDT是空的，但GDT定义了内核运行所需的基本段。
4.  **硬件配置**: 开启A20地址线以访问全部内存，并重新配置中断控制器以避免冲突，为内核的中断处理打下基础。
5.  **模式切换**: 最后，通过修改CR0寄存器，将CPU从实模式切换到32位保护模式，并跳转到 `head.s` (位于 `system` 模块的开始处)，开始真正的32位内核代码执行。

`setup.s` 的执行是 Linux 0.11 启动过程中非常重要的一步，它完成了所有进入保护模式前必要的底层设置。
