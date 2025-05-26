# hdreg.h 文件详解

`include/linux/hdreg.h` 文件是 Linux 0.11 内核中用于定义 AT 兼容硬盘控制器 (Hard Disk Controller,通常指 Western Digital WD100x 系列或其兼容控制器，也称为IDE/ATA接口的早期版本) 相关寄存器地址、命令码和状态位等常量的头文件。这些定义对于内核中的硬盘驱动程序 (`hd.c`) 至关重要，驱动程序通过这些接口与硬盘硬件进行通信，实现数据的读写、寻道、复位、诊断等操作。

## 核心功能

1.  **定义硬盘控制器寄存器地址**: 定义了访问硬盘控制器各个主要寄存器的 I/O 端口地址。
2.  **定义状态寄存器位**: 定义了硬盘控制器状态寄存器 (`HD_STATUS`) 中各个位的含义。
3.  **定义错误寄存器位**: 定义了硬盘控制器错误寄存器 (`HD_ERROR`) 中各个位的含义。
4.  **定义硬盘命令码**: 定义了向硬盘控制器发送的各种操作命令的字节码。
5.  **定义分区表结构**: 定义了一个简化的分区表条目结构 `struct partition`，用于描述硬盘分区信息。

## 硬盘控制器寄存器地址

这些宏定义了 AT 硬盘控制器的主要 I/O 端口地址，也称为任务文件寄存器 (Task File Registers)。

*   `#define HD_DATA 0x1f0`: **数据寄存器 (Data Register)**。可读可写。用于CPU与控制器之间传输数据（扇区数据）。在写入命令参数时，此端口也可能用作其他控制寄存器 (如 `_CTL`，具体取决于上下文，但0.11中主要指数据传输)。
*   `#define HD_ERROR 0x1f1`: **错误寄存器 (Error Register)**。只读。当命令执行后状态寄存器的 `ERR_STAT` 位置位时，可以从此寄存器读取具体的错误代码。在写操作时，此端口地址用作**写预补偿寄存器 (Write Precompensation Register)** (`HD_PRECOMP`)。
*   `#define HD_NSECTOR 0x1f2`: **扇区数量寄存器 (Sector Count Register)**。可读可写。用于指定要读/写的扇区数量。
*   `#define HD_SECTOR 0x1f3`: **扇区号寄存器 (Sector Number Register)**。可读可写。用于指定操作的起始扇区号 (LBA模式下是地址的0-7位)。
*   `#define HD_LCYL 0x1f4`: **柱面低字节寄存器 (Cylinder Low Register)**。可读可写。用于指定操作的起始柱面号的低8位 (LBA模式下是地址的8-15位)。
*   `#define HD_HCYL 0x1f5`: **柱面高字节寄存器 (Cylinder High Register)**。可读可写。用于指定操作的起始柱面号的高8位 (LBA模式下是地址的16-23位)。
*   `#define HD_CURRENT 0x1f6`: **驱动器/磁头寄存器 (Drive/Head Register)**。可读可写。
    *   格式: `101DHhhhH` (早期 MFM/RLL 模式) 或 `1L10HHHH` (LBA 模式)。
    *   `D`: 驱动器选择 (0=主盘, 1=从盘)。
    *   `Hhhh` 或 `HHHH`: 磁头号 (LBA模式下是地址的24-27位)。
    *   `L`: LBA模式选择位 (1=LBA模式)。
*   `#define HD_STATUS 0x1f7`: **状态寄存器 (Status Register)**。只读。用于获取硬盘控制器的当前状态。在写操作时，此端口地址用作**命令寄存器 (Command Register)** (`HD_COMMAND`)。
*   `#define HD_PRECOMP HD_ERROR`: 将 `HD_PRECOMP` 定义为与 `HD_ERROR` 相同的地址 (0x1f1)，强调写操作时该端口的功能。
*   `#define HD_COMMAND HD_STATUS`: 将 `HD_COMMAND` 定义为与 `HD_STATUS` 相同的地址 (0x1f7)，强调写操作时该端口的功能。

*   `#define HD_CMD 0x3f6`: **设备控制寄存器 (Device Control Register) / 备用状态寄存器 (Alternate Status Register)**。
    *   **写操作 (Device Control Register)**: 用于复位控制器 (`SRST`位) 和禁止/允许中断 (`nIEN`位)。
    *   **读操作 (Alternate Status Register)**: 提供与 `HD_STATUS` 相同的状态位，但不清除中断请求。

## 状态寄存器 (`HD_STATUS`) 位定义

这些位在读取 `0x1f7` 端口时获得。

*   `#define ERR_STAT 0x01`: **错误标志 (Error)**。如果置位，表示前一个命令执行出错，具体的错误代码可以从 `HD_ERROR` 寄存器读取。
*   `#define INDEX_STAT 0x02`: **索引标志 (Index)**。驱动器每旋转一圈产生一个索引脉冲时置位 (不常用)。
*   `#define ECC_STAT 0x04`: **已纠正错误 (Corrected ECC)**。如果置位，表示发生了数据错误，但已被ECC (Error Correction Code) 逻辑纠正。
*   `#define DRQ_STAT 0x08`: **数据请求 (Data Request)**。如果置位，表示控制器准备好与主机进行数据传输（通过 `HD_DATA` 端口）。
*   `#define SEEK_STAT 0x10`: **寻道完成 (Seek Complete)**。如果置位，表示寻道操作已完成，磁头已到达目标磁道。
*   `#define WRERR_STAT 0x20`: **写故障 (Write Fault)**。如果置位，表示驱动器发生写故障。
*   `#define READY_STAT 0x40`: **驱动器就绪 (Drive Ready)**。如果置位，表示驱动器已准备好，能够响应命令。
*   `#define BUSY_STAT 0x80`: **控制器忙 (Busy)**。如果置位，表示控制器正在执行命令，不能接受新命令。在发送命令前通常需要等待此位清零。

## 命令寄存器 (`HD_COMMAND`) 值定义

这些是写入 `0x1f7` 端口的命令字节。这些命令通常被称为 "Winchester" 命令，因为它们源于 IBM PC/AT 的硬盘接口，而 "Winchester" 是 IBM 早期硬盘的一个代号。

*   `#define WIN_RESTORE 0x10`: **驱动器重新校准 (Restore)**。使磁头返回0磁道。也称为 `RECALIBRATE`。
*   `#define WIN_READ 0x20`: **读扇区 (Read Sector(s))**。从磁盘读取一个或多个扇区。
*   `#define WIN_WRITE 0x30`: **写扇区 (Write Sector(s))**。向磁盘写入一个或多个扇区。
*   `#define WIN_VERIFY 0x40`: **校验扇区 (Verify Sector(s))**。检查扇区是否可读且无CRC错误，但不传输数据。
*   `#define WIN_FORMAT 0x50`: **格式化磁道 (Format Track)**。对指定的磁道进行低级格式化。
*   `#define WIN_INIT 0x60`: **初始化驱动器参数 (Initialize Drive Parameters)**。在早期 MFM/RLL 控制器中使用，已废弃。应使用 `WIN_SPECIFY`。
*   `#define WIN_SEEK 0x70`: **寻道 (Seek)**。将磁头移动到指定的柱面/磁道。
*   `#define WIN_DIAGNOSE 0x90`: **执行驱动器诊断 (Execute Drive Diagnostic)**。控制器执行内部自检。
*   `#define WIN_SPECIFY 0x91`: **设置驱动器参数 (Set Drive Parameters)**。用于向控制器指定驱动器的几何参数（柱面数、磁头数等）。也称为 `INITIALIZE DRIVE PARAMETERS`。

## 错误寄存器 (`HD_ERROR`) 位定义

这些位在读取 `0x1f1` 端口时获得，前提是状态寄存器的 `ERR_STAT` 位为1。

*   `#define MARK_ERR 0x01`: **地址标记错误 (Address Mark Not Found, AMNF)**。找不到扇区头部的地址标记。
*   `#define TRK0_ERR 0x02`: **0磁道未找到错误 (Track 0 Not Found, T0NF)**。在重新校准操作后，未能定位到0磁道。
*   `#define ABRT_ERR 0x04`: **命令中止错误 (Aborted Command, ABRT)**。命令因某种原因（如参数无效、设备故障等）被中止。
*   `#define ID_ERR 0x10`: **ID未找到错误 (ID Not Found, IDNF)**。找不到请求扇区的ID字段，或扇区头CRC错误。
*   `#define ECC_ERR 0x40`: **不可纠正的ECC错误 (Uncorrectable ECC Error)**。数据在传输中发生错误，且ECC无法纠正。也可能是 **数据CRC错误 (UNC)**。
*   `#define BBD_ERR 0x80`: **坏块检测到错误 (Bad Block Detected, BBD)**。扇区被标记为坏块。

## `struct partition` 结构体

该文件还定义了一个 `struct partition` 结构体，用于表示从硬盘主引导记录 (MBR) 中读取的分区表信息。

```c
struct partition {
	unsigned char boot_ind;		/* 0x80 - active (unused in Linux 0.11) */
	unsigned char head;		    /* 起始磁头号 (CHS) */
	unsigned char sector;		/* 起始扇区号 (CHS, sector in bits 0-5, cyl high 2 bits in 6-7) */
	unsigned char cyl;		    /* 起始柱面号低8位 (CHS) */
	unsigned char sys_ind;		/* 分区类型字节 (如 0x81 Minix, 0x83 Linux) */
	unsigned char end_head;		/* 结束磁头号 (CHS) */
	unsigned char end_sector;	/* 结束扇区号 (CHS) */
	unsigned char end_cyl;		/* 结束柱面号低8位 (CHS) */
	unsigned int start_sect;	/* 分区的起始扇区号 (LBA, 从0开始计数) */
	unsigned int nr_sects;		/* 分区占用的总扇区数 */
};
```
*   **`boot_ind`**: 引导指示符。`0x80` 表示活动分区（可引导），`0x00` 表示非活动。Linux 0.11 本身不直接使用它来选择引导分区。
*   `head`, `sector`, `cyl`: 分区起始的CHS (Cylinder/Head/Sector) 地址。`sector` 字段的低6位是扇区号(1-63)，高2位是柱面号的高2位。
*   `sys_ind`: 系统指示符，表示分区的类型。例如，`0x81` 代表 Minix 文件系统，`0x83` 代表 Linux ext2/ext3/ext4 等。
*   `end_head`, `end_sector`, `end_cyl`: 分区结束的CHS地址。
*   `start_sect`: 分区的起始绝对扇区号（LBA寻址方式），从磁盘的第一个扇区（扇区0）开始计数。这是Linux内核更常用的分区定位方式。
*   `nr_sects`: 该分区包含的总扇区数量。

硬盘驱动程序会读取MBR中的分区表（通常有4个这样的条目），并用这个结构来解析每个分区的信息。

## 总结

`include/linux/hdreg.h` 文件为 Linux 0.11 的硬盘驱动程序提供了一套完整的与 AT 硬盘控制器硬件进行交互所需的常量和结构定义。它详细列出了控制器的寄存器端口、命令集、状态和错误位的含义，以及用于解析分区表的结构。这些宏和定义使得驱动程序的代码更具可读性，并将硬件相关的细节集中在一个地方，方便查阅和理解。这是分析 Linux 0.11 硬盘驱动工作原理的基础。Okay, I have created `docs/include/linux/hdreg.md` with the detailed Chinese explanation for `include/linux/hdreg.h`.

Next, I will process `include/linux/head.h`.
1. Create `docs/include/linux/head.md`.
2. Read `include/linux/head.h`.
3. Write the explanation into `docs/include/linux/head.md`.

First, I'll read `include/linux/head.h`.
