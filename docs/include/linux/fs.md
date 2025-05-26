# fs.h 文件详解

`include/linux/fs.h` 文件是 Linux 0.11 内核中与文件系统相关的最核心的头文件之一。它定义了文件系统赖以运行的绝大多数重要数据结构、常量、宏以及许多外部函数的声明。这个文件是理解整个 Linux 0.11 文件系统实现的基础。

它包含了从底层块设备操作到高层文件操作所需的各种定义，如：设备号、操作常量、各种系统限制、缓冲区头 (`buffer_head`)、磁盘和内存中的 inode 结构 (`d_inode`, `m_inode`)、文件结构 (`file`)、超级块结构 (`super_block`, `d_super_block`) 以及目录项结构 (`dir_entry`)。

## 核心功能

1.  **定义设备主设备号**: 列出了系统中不同类型设备的主设备号。
2.  **定义 I/O 操作类型**: 如 `READ`, `WRITE`。
3.  **定义文件系统常量和限制**: 如 `NAME_LEN`, `ROOT_INO`, `NR_OPEN`, `NR_INODE`, `BLOCK_SIZE` 等。
4.  **定义核心数据结构**:
    *   `struct buffer_head`: 缓冲区高速缓存的头部结构。
    *   `struct d_inode`: 磁盘上 inode 的结构。
    *   `struct m_inode`: 内存中 inode 的结构。
    *   `struct file`: 打开文件表项的结构。
    *   `struct super_block`: 内存中超级块的结构。
    *   `struct d_super_block`: 磁盘上超级块的结构。
    *   `struct dir_entry`: 目录项的结构。
5.  **定义与管道相关的宏**: 用于操作管道 inode 中的头尾指针和大小。
6.  **声明外部全局变量**: 如 `inode_table`, `file_table`, `super_block` 等核心数据表。
7.  **声明外部函数原型**: 声明了文件系统中几乎所有重要函数的原型，这些函数分布在 `fs/` 目录下的各个 `.c` 文件中。

## 宏和常量定义详解

### 设备主设备号

注释中列出了 Minix 文件系统兼容的设备主设备号：
*   `0`: 未使用 (nodev)
*   `1`: `/dev/mem` (物理内存)
*   `2`: `/dev/fd` (软盘驱动器)
*   `3`: `/dev/hd` (硬盘驱动器)
*   `4`: `/dev/ttyx` (特定终端，如 /dev/tty1)
*   `5`: `/dev/tty` (当前控制终端)
*   `6`: `/dev/lp` (打印机)
*   `7`: 未命名管道

### `IS_SEEKABLE(x)`

*   **定义**: `#define IS_SEEKABLE(x) ((x)>=1 && (x)<=3)`
*   **功能**: 判断主设备号 `x` 对应的设备是否支持 `lseek` 定位操作。在 Linux 0.11 中，只有 `/dev/mem` (1), `/dev/fd` (2), `/dev/hd` (3) 被认为是可定位的。

### I/O 操作类型

*   `#define READ 0`
*   `#define WRITE 1`
*   `#define READA 2 /* read-ahead - don't pause */`: 预读操作，通常异步。
*   `#define WRITEA 3 /* "write-ahead" - silly, but somewhat useful */`: 预写操作。

### `MAJOR(a)` 和 `MINOR(a)`

*   `#define MAJOR(a) (((unsigned)(a))>>8)`: 从设备号 `a` (通常是一个 `dev_t` 类型，16位) 中提取主设备号 (高8位)。
*   `#define MINOR(a) ((a)&0xff)`: 从设备号 `a` 中提取次设备号 (低8位)。

### 文件系统常量

*   `#define NAME_LEN 14`: 文件名的最大长度。
*   `#define ROOT_INO 1`: 根目录的 inode 号通常为1。
*   `#define I_MAP_SLOTS 8`: 超级块中 inode 位图所占的 `buffer_head` 指针数组大小。
*   `#define Z_MAP_SLOTS 8`: 超级块中逻辑块位图所占的 `buffer_head` 指针数组大小。
*   `#define SUPER_MAGIC 0x137F`: Minix 文件系统的魔数，用于校验超级块。

### 系统资源限制

*   `#define NR_OPEN 20`: 每个进程能打开的最大文件描述符数量。
*   `#define NR_INODE 32`: 系统内存中 inode 表 (`inode_table`) 的大小。
*   `#define NR_FILE 64`: 系统打开文件表 (`file_table`) 的大小。
*   `#define NR_SUPER 8`: 系统超级块表 (`super_block`) 的大小。
*   `#define NR_HASH 307`: 缓冲区哈希表 (`hash_table`) 的大小 (一个素数)。
*   `#define NR_BUFFERS nr_buffers`: 系统中缓冲区头的总数，由 `nr_buffers` 全局变量决定 (在 `buffer.c` 中根据可用内存计算)。
*   `#define BLOCK_SIZE 1024`: 磁盘块的大小，单位字节。
*   `#define BLOCK_SIZE_BITS 10`: `BLOCK_SIZE` 的位数 (2^10 = 1024)。

### NULL 定义

*   `#ifndef NULL #define NULL ((void *) 0) #endif`: 如果 `NULL` 未定义，则定义为 `(void *)0`。

### 每块包含的条目数

*   `#define INODES_PER_BLOCK ((BLOCK_SIZE)/(sizeof (struct d_inode)))`: 每个磁盘块能容纳的 `d_inode` (磁盘inode) 数量。
*   `#define DIR_ENTRIES_PER_BLOCK ((BLOCK_SIZE)/(sizeof (struct dir_entry)))`: 每个磁盘块能容纳的 `dir_entry` (目录项) 数量。

### 管道相关宏

这些宏用于操作管道 inode (`m_inode` 结构，当 `i_pipe=1` 时) 中的伪 `i_zone` 字段来管理管道的环形缓冲区。管道的数据实际存储在一个内存页中，其地址存在 `inode.i_size` (非标准用法)。
*   `#define PIPE_HEAD(inode) ((inode).i_zone[0])`: 获取管道的头指针 (下一个可写位置的索引)，存储在 `i_zone[0]`。
*   `#define PIPE_TAIL(inode) ((inode).i_zone[1])`: 获取管道的尾指针 (下一个可读位置的索引)，存储在 `i_zone[1]`。
*   `#define PIPE_SIZE(inode) ((PIPE_HEAD(inode)-PIPE_TAIL(inode))&(PAGE_SIZE-1))`: 计算管道中当前数据量。使用 `& (PAGE_SIZE-1)` 实现回绕。
*   `#define PIPE_EMPTY(inode) (PIPE_HEAD(inode)==PIPE_TAIL(inode))`: 判断管道是否为空。
*   `#define PIPE_FULL(inode) (PIPE_SIZE(inode)==(PAGE_SIZE-1))`: 判断管道是否已满 (保留一个字节以区分空/满)。
*   `#define INC_PIPE(head) __asm__("incl %0\n\tandl $4095,%0"::"m" (head))`: 以原子方式增加管道头/尾指针 `head` 并处理回绕 (假设 `PAGE_SIZE` 为4096)。

## 核心数据结构定义

### `typedef char buffer_block[BLOCK_SIZE];`

*   定义了一个类型 `buffer_block`，它是一个大小为 `BLOCK_SIZE` 的字符数组，代表一个原始的磁盘数据块。

### `struct buffer_head`

*   **功能**: 缓冲区高速缓存中每个缓冲区的头部结构。它管理着一个从磁盘读入或待写入磁盘的数据块。
*   **主要成员**:
    *   `char * b_data`: 指向实际存储块数据的内存区域 (1KB)。
    *   `unsigned long b_blocknr`: 该缓冲区对应的物理块号。
    *   `unsigned short b_dev`: 设备号。如果为0，表示此 `buffer_head` 空闲。
    *   `unsigned char b_uptodate`: 数据有效标志。1表示数据与磁盘一致或是最新的。
    *   `unsigned char b_dirt`: 脏标志。1表示缓冲区内容已修改，需要写回磁盘。
    *   `unsigned char b_count`: 此缓冲区的引用计数。
    *   `unsigned char b_lock`: 锁定标志。1表示缓冲区正被操作，其他进程需等待。
    *   `struct task_struct * b_wait`: 指向等待此缓冲区解锁的进程队列。
    *   `struct buffer_head * b_prev`, `* b_next`: 哈希表拉链指针。
    *   `struct buffer_head * b_prev_free`, `* b_next_free`: 空闲链表指针 (双向循环)。

### `struct d_inode` (磁盘 Inode)

*   **功能**: 定义 inode 在磁盘上存储的格式。
*   **主要成员**:
    *   `unsigned short i_mode`: 文件类型和权限。
    *   `unsigned short i_uid`: 文件所有者用户ID。
    *   `unsigned long i_size`: 文件大小 (字节)。
    *   `unsigned long i_time`: 文件最后修改时间 (mtime)。
    *   `unsigned char i_gid`: 文件所有者组ID。
    *   `unsigned char i_nlinks`: 硬链接计数。
    *   `unsigned short i_zone[9]`: 数据块区指针。0-6为直接块，7为一级间接块，8为二级间接块。

### `struct m_inode` (内存 Inode)

*   **功能**: 定义 inode 在内存中的活动结构。它包含了 `d_inode` 的所有信息，并增加了内存中特有的状态。
*   **主要成员 (除了 `d_inode` 的字段外)**:
    *   `struct task_struct * i_wait`: 等待此 inode 解锁的进程队列。
    *   `unsigned long i_atime`: 文件最后访问时间。
    *   `unsigned long i_ctime`: inode 状态最后改变时间。
    *   `unsigned short i_dev`: inode 所在的设备号。
    *   `unsigned short i_num`: inode 在磁盘上的编号。
    *   `unsigned short i_count`: 此内存 inode 的引用计数。
    *   `unsigned char i_lock`: 锁定标志。
    *   `unsigned char i_dirt`: 脏标志。
    *   `unsigned char i_pipe`: 是否为管道 inode。
    *   `unsigned char i_mount`: 是否为挂载点。
    *   `unsigned char i_seek`: (未使用)。
    *   `unsigned char i_update`: (未使用)。

### `struct file`

*   **功能**: 系统打开文件表 (`file_table`) 中的条目结构，代表一个打开的文件实例。
*   **主要成员**:
    *   `unsigned short f_mode`: 文件 inode 的模式 (类型和权限)。
    *   `unsigned short f_flags`: 打开文件时指定的标志 (如 `O_RDONLY`)。
    *   `unsigned short f_count`: 此 `file` 结构体的引用计数。
    *   `struct m_inode * f_inode`: 指向该文件对应的内存 inode。
    *   `off_t f_pos`: 当前文件的读写指针（偏移量）。

### `struct super_block` (内存超级块)

*   **功能**: 内存中活动文件系统的超级块结构。
*   **主要成员**:
    *   `s_ninodes`, `s_nzones`, `s_imap_blocks`, `s_zmap_blocks`, `s_firstdatazone`, `s_log_zone_size`, `s_max_size`, `s_magic`: 与磁盘超级块 `d_super_block` 对应的文件系统参数。
    *   `struct buffer_head * s_imap[8]`, `* s_zmap[8]`: 指向 inode 位图和数据块位图的缓冲区头指针数组。
    *   `unsigned short s_dev`: 设备号。
    *   `struct m_inode * s_isup`: 指向被覆盖文件系统（被挂载设备）的根 inode。
    *   `struct m_inode * s_imount`: 指向该文件系统被挂载到的目录的 inode。
    *   `unsigned long s_time`: 最后更新时间。
    *   `struct task_struct * s_wait`: 等待此超级块解锁的进程队列。
    *   `unsigned char s_lock`: 锁定标志。
    *   `unsigned char s_rd_only`: 只读标志。
    *   `unsigned char s_dirt`: 脏标志。

### `struct d_super_block` (磁盘超级块)

*   **功能**: 定义超级块在磁盘上存储的格式。与 `struct super_block` 的前一部分参数对应。

### `struct dir_entry` (目录项)

*   **功能**: 定义目录文件中每个条目的格式。
*   **主要成员**:
    *   `unsigned short inode`: 该目录项对应的 inode 编号。
    *   `char name[NAME_LEN]`: 文件名或目录名。

## 外部变量和函数声明

文件末尾使用 `extern` 关键字声明了许多在其他 `.c` 文件中定义的全局变量（主要是核心数据表）和函数原型。这使得包含此头文件的代码能够访问这些全局资源和调用这些核心文件系统函数。

例如：
*   `extern struct m_inode inode_table[NR_INODE];`
*   `extern struct file file_table[NR_FILE];`
*   `extern struct super_block super_block[NR_SUPER];`
*   `extern struct buffer_head * start_buffer;`
*   `extern int nr_buffers;`
*   以及大量的函数原型，如 `iget`, `iput`, `namei`, `bread`, `bmap`, `sync_inodes`, `mount_root` 等。

## 总结

`include/linux/fs.h` 是 Linux 0.11 文件系统的核心定义文件。它通过定义一系列关键的数据结构、常量和宏，构建了整个文件系统的框架。这些定义不仅描述了文件系统在磁盘上的布局（如 `d_inode`, `d_super_block`, `dir_entry`），也描述了它们在内存中的表示和管理方式（如 `m_inode`, `super_block`, `file`, `buffer_head`）。同时，它声明了文件系统操作所需的大量外部函数和全局表，是内核各个文件系统相关模块之间协作的接口基础。理解此文件是深入学习 Linux 0.11 文件系统实现不可或缺的一步。Okay, I have created `docs/include/linux/fs.md` with the detailed Chinese explanation for `include/linux/fs.h`.

Next, I will process `include/linux/hdreg.h`.
1. Create `docs/include/linux/hdreg.md`.
2. Read `include/linux/hdreg.h`.
3. Write the explanation into `docs/include/linux/hdreg.md`.

First, I'll read `include/linux/hdreg.h`.
