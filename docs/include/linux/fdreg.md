# fdreg.h 文件详解

`include/linux/fdreg.h` 文件是 Linux 0.11 内核中用于定义软盘控制器 (Floppy Disk Controller, FDC) 相关寄存器地址和命令常量的头文件。这些定义对于内核中的软盘驱动程序 (`floppy.c`) 至关重要，因为它需要通过这些寄存器和命令来与软盘硬件进行交互，实现数据的读取、写入、寻道、校准等操作。

该文件中的定义主要基于 IBM PC/AT 兼容机的标准软盘控制器（通常是 NEC µPD765 或其兼容芯片）。

## 核心功能

1.  **声明外部函数**: 声明了一些在软盘驱动中定义的、与软盘马达控制和选择相关的外部函数。
2.  **定义 FDC 寄存器地址**: 定义了访问 FDC 各个主要寄存器的 I/O 端口地址。
3.  **定义状态寄存器位**: 定义了 FDC 主状态寄存器 (MSR) 以及命令执行后返回的状态字节 (ST0-ST3) 中各个位的含义。
4.  **定义 FDC 命令码**: 定义了向 FDC 发送的各种操作命令的字节码。
5.  **定义 DMA 命令**: 定义了与 DMA (Direct Memory Access) 控制器配合进行软盘数据传输时使用的命令。

## 函数声明

文件开头声明了几个外部函数，这些函数通常在 `fs/floppy.c` 或类似的驱动代码中实现：

*   `extern int ticks_to_floppy_on(unsigned int nr);`: 计算距离软驱 `nr` 马达启动（或已启动）达到稳定状态所需的时间滴答数。
*   `extern void floppy_on(unsigned int nr);`: 开启指定软盘驱动器 `nr` 的马达。
*   `extern void floppy_off(unsigned int nr);`: 关闭指定软盘驱动器 `nr` 的马达。
*   `extern void floppy_select(unsigned int nr);`: 选择指定的软盘驱动器 `nr`。
*   `extern void floppy_deselect(unsigned int nr);`: 取消选择指定的软盘驱动器 `nr` (通常是关闭马达和选择信号)。

## FDC 寄存器地址

这些宏定义了 FDC 的主要 I/O 端口地址：

*   `#define FD_STATUS 0x3f4`: **主状态寄存器 (Main Status Register, MSR)**。只读。用于获取 FDC 的当前状态，如是否忙、数据方向、DMA模式等。
*   `#define FD_DATA 0x3f5`: **数据 FIFO 寄存器 (Data FIFO Register)**。可读可写。用于在命令阶段发送命令参数字节或命令码，在数据传输阶段读取或写入数据字节，在结果阶段读取状态字节。
*   `#define FD_DOR 0x3f2`: **数字输出寄存器 (Digital Output Register)**。只写。用于控制软盘驱动器的选择、马达的开关、FDC复位以及DMA和中断的使能。
*   `#define FD_DIR 0x3f7`: **数字输入寄存器 (Digital Input Register)**。只读。在某些系统中（如PS/2），这个地址用于读取磁盘更换信号。在标准PC/AT中，此地址也用作 `FD_DCR`。
*   `#define FD_DCR 0x3f7`: **数据速率选择寄存器 (Data Rate Control Register)**。只写。用于设置数据传输速率 (例如 250kbps, 500kbps)。

## 状态寄存器位定义

### 主状态寄存器 (MSR - 通过 `FD_STATUS` 读取)

*   `#define STATUS_BUSYMASK 0x0F`: 驱动器忙状态掩码。低4位分别对应驱动器0-3是否忙于寻道操作。
*   `#define STATUS_BUSY 0x10`: FDC 忙。如果置位，表示 FDC 正在执行一个命令，不能接受新命令。
*   `#define STATUS_DMA 0x20`: DMA 模式。如果为0，表示 FDC 处于 DMA 模式；如果为1，表示非 DMA 模式。Linux 0.11 通常使用 DMA 模式。
*   `#define STATUS_DIR 0x40`: 数据传输方向。如果为0，表示数据从 CPU/DMA -> FDC (写操作)；如果为1，表示数据从 FDC -> CPU/DMA (读操作)。
*   `#define STATUS_READY 0x80`: 数据寄存器就绪。如果为1，表示 `FD_DATA` 寄存器准备好进行数据传输 (读或写)。

### 状态字节 ST0 (命令结果的一部分)

*   `#define ST0_DS 0x03`: 驱动器选择掩码。低两位反映了当前操作的驱动器号。
*   `#define ST0_HA 0x04`: 磁头地址。反映了当前操作的磁头号。
*   `#define ST0_NR 0x08`: 未就绪 (Not Ready)。如果置位，表示驱动器未准备好或出现错误。
*   `#define ST0_ECE 0x10`: 设备检查错误 (Equipment Check Error)。通常表示磁道0信号故障或驱动器故障。
*   `#define ST0_SE 0x20`: 寻道结束 (Seek End)。表示寻道或重新校准操作完成。
*   `#define ST0_INTR 0xC0`: 中断代码掩码。高两位指示命令终止的原因：
    *   `00`: 正常终止。
    *   `01`: 异常终止。
    *   `10`: 命令无效。
    *   `11`: FDD 状态改变 (PS/2)。

### 状态字节 ST1 (命令结果的一部分)

*   `#define ST1_MAM 0x01`: 地址标记丢失 (Missing Address Mark)。找不到扇区ID的地址标记。
*   `#define ST1_WP 0x02`: 写保护 (Write Protect)。试图向写保护的磁盘写入数据。
*   `#define ST1_ND 0x04`: 无数据 (No Data)。找不到请求的扇区，或扇区ID字段的CRC校验错误。
*   `#define ST1_OR 0x10`: 超限 (OverRun)。CPU未能及时从FDC读取数据或向FDC写入数据，导致数据丢失。
*   `#define ST1_CRC 0x20`: CRC 错误。数据字段或ID字段的CRC校验失败。
*   `#define ST1_EOC 0x80`: 柱面结束 (End Of Cylinder)。在读/写操作中，遇到了磁道上的最后一个扇区。

### 状态字节 ST2 (命令结果的一部分)

*   `#define ST2_MAM 0x01`: 地址标记丢失 (Missing Address Mark)。同ST1中的定义，但在数据扫描命令中出现时意义不同。
*   `#define ST2_BC 0x02`: 坏柱面 (Bad Cylinder)。通常表示在ID字段中记录的柱面号与期望的不符。
*   `#define ST2_SNS 0x04`: 扫描未满足 (Scan Not Satisfied)。在扫描命令中，未找到满足比较条件的数据。
*   `#define ST2_SEH 0x08`: 扫描相等命中 (Scan Equal Hit)。在扫描命令中，找到了数据等于比较条件的扇区。
*   `#define ST2_WC 0x10`: 错误柱面 (Wrong Cylinder)。在ID字段中发现一个与磁道内容不一致的柱面号。
*   `#define ST2_CRC 0x20`: 数据字段CRC错误。在数据字段中发生CRC校验错误。
*   `#define ST2_CM 0x40`: 控制标记 (Control Mark)。读取到一个被删除数据地址标记的扇区。

### 状态字节 ST3 (通常由 `FD_SENSEI` 命令返回)

*   `#define ST3_HA 0x04`: 磁头地址。当前选中的磁头号。
*   `#define ST3_TZ 0x10`: 磁道0信号 (Track Zero)。如果为1，表示磁头当前位于0磁道。
*   `#define ST3_WP 0x40`: 写保护。当前选中的驱动器是写保护的。

## FDC 命令码 (`FD_COMMAND`)

这些是写入 `FD_DATA` 寄存器的命令字节，用于指示 FDC 执行特定操作。

*   `#define FD_RECALIBRATE 0x07`: **重新校准命令**。使磁头移动到0磁道。
*   `#define FD_SEEK 0x0F`: **寻道命令**。将磁头移动到指定的磁道。
*   `#define FD_READ 0xE6`: **读数据命令**。从磁盘读取数据。
    *   `0xE6` (或 `0xC6` for write): `MT` (bit 7, Multi-Track) = 1, `MFM` (bit 6, MFM mode) = 1, `SK` (bit 5, Skip deleted) = 1 for read.
*   `#define FD_WRITE 0xC5`: **写数据命令**。向磁盘写入数据。
    *   `0xC5`: `MT`=1, `MFM`=1. `SK` (bit 5) = 0 for write.
*   `#define FD_SENSEI 0x08`: **感知中断状态命令 (Sense Interrupt Status)**。用于清除FDC的中断状态并获取ST0状态字节，通常在中断处理程序中调用。
*   `#define FD_SPECIFY 0x03`: **指定参数命令**。用于设置磁头步进速率 (SRT)、磁头卸载时间 (HUT) 和磁头加载时间 (HLT) 以及DMA模式。

## DMA 命令

这些命令用于设置 DMA 控制器（如8237A）以配合 FDC 进行数据传输。

*   `#define DMA_READ 0x46`: 设置 DMA 通道进行读操作 (从 FDC 到内存)。
    *   `0x46` = `01000110b`: `01` (写内存), `00` (地址不变), `0` (非自动初始化), `0` (地址增加), `10` (通道2选择), `00` (按需模式)。
*   `#define DMA_WRITE 0x4A`: 设置 DMA 通道进行写操作 (从内存到 FDC)。
    *   `0x4A` = `01001010b`: `01` (读内存), `00` (地址不变), `0` (非自动初始化), `0` (地址增加), `10` (通道2选择), `00` (按需模式)。
    *   注意：这里的模式位设置，特别是读写方向，是指DMA控制器相对于内存的操作。`DMA_READ` 表示DMA从I/O设备读取数据并写入内存。`DMA_WRITE` 表示DMA从内存读取数据并写入I/O设备。

## 总结

`include/linux/fdreg.h` 文件为 Linux 0.11 的软盘驱动程序提供了一套完整的与 FDC 硬件交互所需的常量定义。它详细列出了 FDC 的寄存器端口、各种状态位的含义以及核心的操作命令码。这些宏使得驱动程序的代码更具可读性，并且将硬件相关的细节集中在一个地方，方便查阅和（如果需要）修改。理解这些定义是深入分析 Linux 0.11 软盘驱动工作原理的基础。Okay, I have created `docs/include/linux/fdreg.md` with the detailed Chinese explanation for `include/linux/fdreg.h`.

Next, I will process `include/linux/fs.h`.
1. Create `docs/include/linux/fs.md`.
2. Read `include/linux/fs.h`.
3. Write the explanation into `docs/include/linux/fs.md`.

First, I'll read `include/linux/fs.h`.
