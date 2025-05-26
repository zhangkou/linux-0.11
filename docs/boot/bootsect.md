# bootsect.s 代码详解

`bootsect.s` 是 Linux 0.11 内核启动过程的第一个文件，由 BIOS 加载到内存地址 `0x7c00`。它的主要功能是将自身移动到 `0x90000`，然后加载 `setup.s` 和系统模块 (`system`) 到指定内存位置。

## 主要代码段和功能

### 1. 初始化和自我移动

*   **`.globl begtext, begdata, begbss, endtext, enddata, endbss`**: 定义全局符号，用于标记代码段、数据段和 BSS 段的开始和结束。
*   **`SETUPLEN = 4`**: 定义 `setup.s` 占用的扇区数 (4个扇区)。
*   **`BOOTSEG = 0x07c0`**: `bootsect.s` 被 BIOS 加载的初始段地址。
*   **`INITSEG = 0x9000`**: `bootsect.s` 将自身移动到的目标段地址。
*   **`SETUPSEG = 0x9020`**: `setup.s` 将被加载到的段地址 (紧随 `bootsect.s` 之后)。
*   **`SYSSEG = 0x1000`**: 系统模块 (`system`) 将被加载到的段地址 (`0x10000`)。
*   **`ENDSEG = SYSSEG + SYSSIZE`**: 系统模块加载的结束地址。`SYSSIZE` 定义了系统模块的最大大小 (0x3000 clicks = 196KB)。
*   **`ROOT_DEV = 0x306`**: 指定根文件系统所在的设备。这里 `0x306` 通常表示第二个硬盘的第一个分区。

代码开始于 `entry start`：
*   **`mov ax,#BOOTSEG; mov ds,ax`**: 设置数据段寄存器 (DS) 指向 `BOOTSEG`。
*   **`mov ax,#INITSEG; mov es,ax`**: 设置附加段寄存器 (ES) 指向 `INITSEG`。
*   **`mov cx,#256; sub si,si; sub di,di; rep; movw`**: 这段代码将 `bootsect.s` 自身 (256 个字，即 512 字节) 从 `DS:SI` (即 `0x07c0:0`) 复制到 `ES:DI` (即 `0x9000:0`)。
*   **`jmpi go,INITSEG`**: 跳转到 `INITSEG:go`，即新的 `bootsect.s` 的位置执行。

### 2. 设置堆栈和加载 `setup.s`

*   **`go: mov ax,cs; mov ds,ax; mov es,ax`**: 更新 DS 和 ES，使其指向当前的 CS (即 `INITSEG`)。
*   **`mov ss,ax; mov sp,#0xFF00`**: 设置堆栈段寄存器 (SS) 指向 `INITSEG`，堆栈指针 (SP) 设置为 `0xFF00` (在 `0x90000` 段内，地址为 `0x9FF00`)。这个位置是任意选择的，只要大于 512 字节（`bootsect.s`自身大小）即可。

*   **`load_setup:`**: 这部分代码负责加载 `setup.s` 模块。
    *   **`mov dx,#0x0000`**: 设置驱动器号 (0号驱动器) 和磁头号 (0号磁头)。
    *   **`mov cx,#0x0002`**: 设置起始扇区号 (2号扇区) 和磁道号 (0号磁道)。 `setup.s` 从第二个扇区开始。
    *   **`mov bx,#0x0200`**: 设置加载地址的偏移量，相对于 `INITSEG`，即 `0x90200`。
    *   **`mov ax,#0x0200+SETUPLEN`**: 设置 BIOS `int 0x13` 的功能号为 `0x02` (读扇区)，并指定读取 `SETUPLEN` (4) 个扇区。
    *   **`int 0x13`**: 调用 BIOS 磁盘服务中断。
    *   **`jnc ok_load_setup`**: 如果读取成功 (进位标志位 CF=0)，则跳转到 `ok_load_setup`。
    *   如果读取失败，则重置磁盘 (`mov ax,#0x0000; int 0x13`) 并重新尝试加载 `load_setup`。

### 3. 获取磁盘参数和打印信息

*   **`ok_load_setup:`**:
    *   **`mov dl,#0x00; mov ax,#0x0800; int 0x13`**: 调用 BIOS `int 0x13` 的功能号 `0x08` (获取驱动器参数)。
    *   **`mov ch,#0x00; seg cs; mov sectors,cx`**: 将返回的每磁道扇区数保存到 `sectors` 变量中。
    *   **`mov ax,#INITSEG; mov es,ax`**: 确保 ES 仍然指向 `INITSEG`。

*   **打印 "Loading system ..." 消息**:
    *   **`mov ah,#0x03; xor bh,bh; int 0x10`**: 读取当前光标位置。
    *   **`mov cx,#24; mov bx,#0x0007; mov bp,#msg1; mov ax,#0x1301; int 0x10`**: 调用 BIOS `int 0x10` 的功能号 `0x1301` (显示字符串并移动光标)。`msg1` 是要显示的消息。

### 4. 加载系统模块 (`system`)

*   **`mov ax,#SYSSEG; mov es,ax`**: 设置 ES 指向 `SYSSEG` (`0x1000`)，这是 `system` 模块要加载到的目标段地址。
*   **`call read_it`**: 调用 `read_it` 子程序来实际执行加载操作。
*   **`call kill_motor`**: 调用 `kill_motor` 子程序关闭软盘驱动器马达。

### 5. 确定根文件系统设备

*   **`seg cs; mov ax,root_dev; cmp ax,#0; jne root_defined`**: 检查 `root_dev` 是否已经被预定义 (非零)。如果已定义，则直接跳转到 `root_defined`。
*   如果 `root_dev` 为零，则根据 BIOS 报告的每磁道扇区数来确定根设备：
    *   **`seg cs; mov bx,sectors`**: 获取之前保存的每磁道扇区数。
    *   **`mov ax,#0x0208`**: 默认为 `/dev/ps0` (1.2Mb 软盘，设备号 (2,8))。
    *   **`cmp bx,#15; je root_defined`**: 如果每磁道扇区数为 15，则使用此设备。
    *   **`mov ax,#0x021c`**: 否则，尝试 `/dev/PS0` (1.44Mb 软盘，设备号 (2,28)，注意原文注释笔误为PS0)。
    *   **`cmp bx,#18; je root_defined`**: 如果每磁道扇区数为 18，则使用此设备。
    *   **`undef_root: jmp undef_root`**: 如果无法确定设备类型 (例如，不是标准的15或18扇区/磁道)，则进入死循环。
*   **`root_defined: seg cs; mov root_dev,ax`**: 将最终确定的根设备号存入 `root_dev` 变量。

### 6. 跳转到 `setup.s`

*   **`jmpi 0,SETUPSEG`**: 无条件跳转到 `SETUPSEG:0` (即 `0x9020:0`)，开始执行 `setup.s` 中的代码。

## 子程序详解

### `read_it`

此子程序负责将 `system` 模块从磁盘加载到内存 `0x10000` 开始的位置。它尝试尽可能快地加载数据，一次加载整个磁道。

*   **`sread: .word 1+SETUPLEN`**: 已读取的当前磁道的扇区数。初始值为 `1+SETUPLEN` 是因为 `bootsect.s` 和 `setup.s` 占用了从磁道0开始的前几个扇区。
*   **`head: .word 0`**: 当前磁头号。
*   **`track: .word 0`**: 当前磁道号。

主要逻辑：
1.  **`mov ax,es; test ax,#0x0fff; die: jne die`**: 检查加载的段地址 `es` 是否是 64KB 对齐的。如果不是，则进入死循环。这是因为后续操作可能跨越 64KB 边界，导致寻址问题。
2.  **`rp_read:` (读取循环)**:
    *   **`mov ax,es; cmp ax,#ENDSEG; jb ok1_read; ret`**: 检查是否已经加载了所有数据 (当前段地址 `es` 是否小于 `ENDSEG`)。如果加载完成，则返回。
    *   计算本次可以读取的扇区数，确保不超过当前磁道的剩余扇区，并且不超过 64KB 边界。
    *   **`call read_track`**: 调用 `read_track` 读取计算出的扇区。
    *   更新 `sread` (已读扇区数)、`head` (磁头号)、`track` (磁道号)。
    *   更新 `bx` (段内偏移地址)。如果 `bx` 溢出 (超过 64KB)，则将 `es` 增加 `0x1000` (64KB)，并将 `bx` 清零。
    *   跳转回 `rp_read` 继续读取。

### `read_track`

此子程序使用 BIOS `int 0x13` 服务从磁盘读取指定数量的扇区。

*   参数通过寄存器传递：`dx` (磁道号和磁头号的组合)，`cx` (起始扇区号和磁道号的组合)，`ah=2` (BIOS读扇区功能号)，`al` (要读取的扇区数)，`dl` (驱动器号)，`dh` (磁头号)，`es:bx` (目标缓冲区地址)。
*   **`int 0x13`**: 调用 BIOS 磁盘中断。
*   **`jc bad_rt`**: 如果读取失败 (CF=1)，则跳转到 `bad_rt`。
*   **`bad_rt:`**: 重置磁盘控制器 (`mov ax,#0; mov dx,#0; int 0x13`)，然后重新尝试 `read_track`。

### `kill_motor`

此子程序用于关闭软盘驱动器的马达。

*   **`mov dx,#0x3f2; mov al,#0; outb`**: 向软盘控制器端口 `0x3f2`写入0，以关闭所有驱动器的马达。这是为了确保进入内核时软盘马达处于已知状态。

## 数据区

*   **`sectors: .word 0`**: 用于存储磁盘的每磁道扇区数。
*   **`msg1: .byte 13,10; .ascii "Loading system ..."; .byte 13,10,13,10`**: 启动时显示的提示信息。
*   **`.org 508`**: 指示编译器将后续内容放置在 512 字节启动扇区的第 508 字节处。
*   **`root_dev: .word ROOT_DEV`**: 存储根设备的设备号。
*   **`boot_flag: .word 0xAA55`**: 启动扇区的结束标记，BIOS 用它来验证这是一个有效的启动扇区。

## 总结

`bootsect.s` 是一个非常底层的引导加载程序。它完成了以下关键任务：
1.  将自身从 BIOS 加载的 `0x7C00` 移动到 `0x90000`，为后续加载留出空间。
2.  设置初始的堆栈。
3.  加载 `setup.s`模块到 `0x90200`。
4.  加载 `system` 核心模块到 `0x10000`。
5.  确定根文件系统设备。
6.  最后跳转到 `setup.s` 继续启动过程。

它的设计力求简单，直接使用 BIOS 中断进行磁盘操作，并包含基本的错误处理（如磁盘读取失败后重试）。
