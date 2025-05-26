# inode.c 文件详解

`inode.c` 文件是 Linux 0.11 内核中管理文件系统 **inode (索引节点)** 的核心部分。Inode 是类Unix文件系统中用于存储文件元数据（如文件类型、权限、大小、时间戳、数据块位置等）的关键数据结构。此文件负责内存中 inode ( `m_inode` ) 的分配、释放、查找、读写（从磁盘同步）以及 inode 对应数据块的映射。

## 核心功能

1.  **内存 Inode 表管理**:
    *   定义全局内存 inode 表 `inode_table[NR_INODE]`。
    *   `get_empty_inode()`: 从 `inode_table` 中获取一个空闲的 `m_inode` 结构。
    *   `iget(dev, nr)`: 获取指定设备 `dev` 上编号为 `nr` 的 inode。如果该 inode 已在内存中，则增加其引用计数并返回；否则，从磁盘读取到空闲的 `m_inode` 中。
    *   `iput(inode)`: 释放对一个内存 inode 的引用。当引用计数减为0时，如果 inode 是脏的，则写回磁盘。如果链接计数也为0，则截断文件并释放 inode 及其占用的数据块。
2.  **Inode 磁盘同步**:
    *   `read_inode(inode)`: 从磁盘读取 inode 数据到内存中的 `m_inode` 结构。
    *   `write_inode(inode)`: 将内存中 `m_inode` 的内容（如果已修改）写回磁盘。
    *   `sync_inodes()`: 将所有脏的内存 inode 写回磁盘。
    *   `invalidate_inodes(dev)`: 使指定设备上的所有内存 inode 失效 (通常在卸载设备时调用)。
3.  **数据块映射**:
    *   `_bmap(inode, block, create)`: 内部核心函数，用于查找（或创建）文件逻辑块号 `block` 对应的物理磁盘块号。
    *   `bmap(inode, block)`: 获取文件逻辑块 `block` 对应的物理块号 (只读，不创建)。
    *   `create_block(inode, block)`: 获取文件逻辑块 `block` 对应的物理块号，如果不存在则创建新的物理块。
4.  **管道 Inode 管理**:
    *   `get_pipe_inode()`: 创建一个用于管道的特殊 inode。管道数据不存储在磁盘上，而是直接使用一个内存页。
5.  **并发控制**:
    *   `lock_inode(inode)` 和 `unlock_inode(inode)`: 对 inode 进行加锁和解锁，防止并发访问冲突。
    *   `wait_on_inode(inode)`: 等待 inode 解锁。

## 关键数据结构和宏

### `struct m_inode` (内存中的 inode 结构)

这是在 `<linux/fs.h>` 中定义的结构体，代表一个在内存中活动的 inode。它包含了磁盘 inode (`d_inode`) 的所有信息，以及一些内存中特有的状态：

*   `unsigned short i_mode`: 文件类型和权限。
*   `unsigned short i_uid`: 用户ID。
*   `unsigned short i_gid`: 组ID。
*   `unsigned short i_nlinks`: 硬链接计数。
*   `off_t i_size`: 文件大小 (字节)。
*   `time_t i_atime`, `i_mtime`, `i_ctime`: 访问时间、修改时间、inode状态改变时间。
*   `unsigned char i_dev`: inode 所在的设备号。
*   `unsigned short i_num`: inode 在磁盘上的编号。
*   `unsigned short i_count`: 内存中此 inode 的引用计数。
*   `unsigned char i_lock`: 锁定标志，用于互斥访问。
*   `unsigned char i_dirt`: 脏标志，如果为1，表示内存中的 inode 已被修改，需要写回磁盘。
*   `unsigned char i_pipe`: 管道标志，如果为1，表示此 inode 是一个管道。
*   `unsigned char i_mount`: 安装点标志，如果为1，表示此 inode 是一个文件系统的安装点。
*   `unsigned char i_seek`: (未使用)。
*   `unsigned char i_update`: (未使用)。
*   `struct task_struct * i_wait`: 指向等待此 inode 解锁的进程任务结构链表。
*   `unsigned short i_zone[9]`: 数据块区指针数组。
    *   `i_zone[0]` - `i_zone[6]`: 直接块指针 (7个)。
    *   `i_zone[7]`: 一级间接块指针。
    *   `i_zone[8]`: 二级间接块指针。

### `struct d_inode` (磁盘上的 inode 结构)

这也是在 `<linux/fs.h>` 中定义的结构体，与 `m_inode` 的前一部分成员完全对应，代表 inode 在磁盘上存储的格式。`read_inode` 和 `write_inode` 就是在这两种结构之间复制数据。

### `struct inode_table[NR_INODE]`

*   全局内存 inode 表，是一个 `m_inode` 类型的数组，大小为 `NR_INODE` (通常为64)。系统所有活动的 inode 都存储在这里。

### 宏

*   `NR_INODE`: 系统内存中 inode 表的大小。
*   `INODES_PER_BLOCK`: 每个磁盘块能存储的 `d_inode` 数量 (`BLOCK_SIZE / sizeof(struct d_inode)`)。
*   `ROOT_INO`: 根目录的 inode 号 (通常为1)。
*   `PIPE_HEAD`, `PIPE_TAIL`: 用于管道 inode，获取管道数据缓冲区头尾指针的宏。

## 主要函数详解

### `static void read_inode(struct m_inode * inode)`

*   **功能**: 从磁盘读取 inode `inode->i_num` (在设备 `inode->i_dev` 上) 的数据，并填充到内存 `inode` 结构中。
*   **步骤**:
    1.  `lock_inode(inode);`: 锁定 inode。
    2.  获取超级块 `sb = get_super(inode->i_dev)`。
    3.  计算 inode 所在的磁盘块号 `block`:
        *   `block = 2 + sb->s_imap_blocks + sb->s_zmap_blocks + (inode->i_num - 1) / INODES_PER_BLOCK;`
        *   `2`: 跳过引导块和超级块。
        *   `sb->s_imap_blocks`: 跳过 inode 位图块。
        *   `sb->s_zmap_blocks`: 跳过数据区位图块。
        *   `(inode->i_num - 1) / INODES_PER_BLOCK`: 计算 inode 所在的 inode 表块的索引 (inode号从1开始)。
    4.  `if (!(bh = bread(inode->i_dev, block))) panic(...);`: 读取该 inode 表块到缓冲区。
    5.  **复制数据**: `*(struct d_inode *)inode = ((struct d_inode *)bh->b_data)[(inode->i_num - 1) % INODES_PER_BLOCK];`
        *   将缓冲区中对应位置的 `d_inode` 结构直接复制到 `m_inode` 结构的前面部分 (它们结构兼容)。
    6.  `brelse(bh);`: 释放缓冲区。
    7.  `unlock_inode(inode);`: 解锁 inode。

### `static void write_inode(struct m_inode * inode)`

*   **功能**: 将内存 `inode` 结构的内容写回磁盘上对应的 `d_inode`。
*   **步骤**:
    1.  `lock_inode(inode);`: 锁定 inode。
    2.  **有效性检查**: `if (!inode->i_dirt || !inode->i_dev) { unlock_inode(inode); return; }`: 如果 inode 不是脏的或没有关联设备，则无需写回。
    3.  获取超级块和计算 inode 所在块号 (同 `read_inode`)。
    4.  `if (!(bh = bread(inode->i_dev, block))) panic(...);`: 读取该 inode 表块到缓冲区。
    5.  **复制数据**: `((struct d_inode *)bh->b_data)[(inode->i_num - 1) % INODES_PER_BLOCK] = *(struct d_inode *)inode;`
        *   将内存 `m_inode` 结构的前面部分 (兼容 `d_inode` 的部分) 复制到缓冲区中对应 `d_inode` 的位置。
    6.  `bh->b_dirt = 1;`: 标记缓冲区为脏，以便后续写回磁盘。
    7.  `inode->i_dirt = 0;`: 清除内存 inode 的脏标志。
    8.  `brelse(bh);`: 释放缓冲区。
    9.  `unlock_inode(inode);`: 解锁 inode。

### `struct m_inode * iget(int dev, int nr)`

*   **功能**: 获取指定设备 `dev` 上编号为 `nr` 的 inode。这是访问 inode 的主要入口。
*   **步骤**:
    1.  `if (!dev) panic(...);`: 设备号不能为0。
    2.  `empty = get_empty_inode();`: 首先尝试获取一个空闲的 `m_inode` 结构，备用。
    3.  **在 `inode_table` 中查找**:
        *   遍历 `inode_table`。
        *   `if (inode->i_dev != dev || inode->i_num != nr) continue;`: 如果当前 `inode` 不是目标，继续查找。
        *   **找到匹配**:
            *   `wait_on_inode(inode);`: 等待 inode 解锁 (防止在检查期间 inode 状态改变)。
            *   `if (inode->i_dev != dev || inode->i_num != nr) { inode = inode_table; continue; }`: 再次检查，因为等待期间 inode 可能被重用。如果不再匹配，从头开始重新搜索。
            *   `inode->i_count++;`: 增加引用计数。
            *   **处理挂载点**: `if (inode->i_mount)`: 如果此 inode 是一个挂载点：
                *   查找对应的超级块 `super_block[i]` (其 `s_imount` 指向此 inode)。
                *   如果找到，则 `iput(inode)` 释放对当前挂载点 inode 的引用。
                *   更新 `dev` 为被挂载文件系统的设备号 (`super_block[i].s_dev`)，`nr` 更新为根 inode 号 (`ROOT_INO`)。
                *   从头开始在 `inode_table` 中搜索新的 `(dev, nr)` (即被挂载文件系统的根 inode)。
            *   如果不是挂载点，`if (empty) iput(empty);` 释放之前备用的空闲 inode。
            *   `return inode;`: 返回找到的 inode。
    4.  **未在内存中找到**: 如果遍历完 `inode_table` 仍未找到：
        *   `if (!empty) return (NULL);`: 如果之前未能获取到空闲的 `m_inode` 结构，则返回 `NULL` (通常不应发生，因为 `get_empty_inode` 会 `panic` 如果没有)。
        *   `inode = empty;`: 使用之前获取的空闲 `m_inode` 结构。
        *   `inode->i_dev = dev; inode->i_num = nr;`: 设置其设备号和 inode 号。
        *   `read_inode(inode);`: 从磁盘读取该 inode 的数据。
        *   `return inode;`: 返回新加载的 inode。

### `void iput(struct m_inode * inode)`

*   **功能**: 释放对 inode 的一次引用。
*   **步骤**:
    1.  空指针检查。
    2.  `wait_on_inode(inode);`: 等待 inode 解锁。
    3.  `if (!inode->i_count) panic(...);`: 如果引用计数已为0，则 `panic`。
    4.  **处理管道**: `if (inode->i_pipe)`:
        *   唤醒可能等待此管道的进程。
        *   `if (--inode->i_count) return;`: 引用计数减1。如果仍大于0 (管道的另一端仍在使用)，则返回。
        *   如果引用计数为0 (读写两端都关闭)，则 `free_page(inode->i_size)` 释放管道使用的内存页，并重置 inode 状态。
    5.  `if (!inode->i_dev) { inode->i_count--; return; }`: 如果 inode 没有设备号 (可能是已释放或无效的)，仅减少计数。
    6.  **处理块设备文件**: `if (S_ISBLK(inode->i_mode)) { sync_dev(inode->i_zone[0]); wait_on_inode(inode); }`: 如果是块设备文件，则同步其数据。
    7.  **`repeat:` 标签**:
        *   `if (inode->i_count > 1) { inode->i_count--; return; }`: 如果引用计数大于1，则减1并返回。
        *   **引用计数为1时**:
            *   `if (!inode->i_nlinks)`: 如果硬链接计数为0 (没有目录项指向此文件)：
                *   `truncate(inode);`: 截断文件 (释放所有数据块)。
                *   `free_inode(inode);`: 释放 inode (将其在磁盘位图中标记为空闲)。
                *   返回。
            *   `if (inode->i_dirt)`: 如果 inode 是脏的：
                *   `write_inode(inode);`: 将 inode 写回磁盘。
                *   `wait_on_inode(inode);`: 等待写操作完成。
                *   `goto repeat;`: 因为在 `write_inode` 中可能睡眠，inode 状态可能改变 (例如其他进程引用了它)，所以需要重新检查。
            *   `inode->i_count--;`: 最终，如果 inode 不脏且链接数不为0，则引用计数减为0。inode 结构保留在 `inode_table` 中，但不再被积极使用，可以被 `get_empty_inode` 重用。

### `static int _bmap(struct m_inode * inode, int block, int create)`

*   **功能**: 核心的数据块映射函数。根据文件 `inode` 的逻辑块号 `block`，查找或创建对应的物理磁盘块号。
*   **参数**:
    *   `inode`: 文件的 inode。
    *   `block`: 逻辑块号 (从0开始)。
    *   `create`: 标志位。如果为1，则当逻辑块不存在时，会分配新的物理块；如果为0，则不创建。
*   **返回值**: 物理块号。如果失败或 (在 `create=0` 时) 块不存在，则返回0。
*   **步骤 (处理直接块、一级间接块、二级间接块)**:
    1.  **有效性检查**: `block < 0` 或 `block >= 7+512+512*512` (超出最大支持的块数) 则 `panic`。
    2.  **直接块 (0-6)**: `if (block < 7)`
        *   `if (create && !inode->i_zone[block])`: 如果需要创建且该直接块指针为空。
        *   `if (inode->i_zone[block] = new_block(inode->i_dev))`: 分配一个新物理块，并将其地址存入 `i_zone[block]`。
        *   更新 `inode->i_ctime` 和 `inode->i_dirt`。
        *   返回 `inode->i_zone[block]`。
    3.  **一级间接块 (7-518)**: `block -= 7; if (block < 512)`
        *   `if (create && !inode->i_zone[7])`: 如果需要创建且一级间接块本身 (`i_zone[7]`) 不存在，则分配一个新块作为一级间接块。
        *   `if (!inode->i_zone[7]) return 0;`: 如果一级间接块不存在 (且不创建)，返回0。
        *   `if (!(bh = bread(inode->i_dev, inode->i_zone[7]))) return 0;`: 读取一级间接块到缓冲区。
        *   `i = ((unsigned short *) (bh->b_data))[block];`: 从一级间接块中获取逻辑块 `block` (相对一级间接块的偏移) 对应的物理块号 `i`。
        *   `if (create && !i)`: 如果需要创建且该物理块号为空。
        *   `if (i = new_block(inode->i_dev))`: 分配新物理块 `i`。
        *   `((unsigned short *) (bh->b_data))[block] = i; bh->b_dirt = 1;`: 将新物理块号写入一级间接块的相应条目，并标记缓冲区为脏。
        *   `brelse(bh); return i;`
    4.  **二级间接块 (519 - 262662)**: `block -= 512;`
        *   类似地，先处理二级间接块本身 (`i_zone[8]`) 的分配和读取。
        *   然后从二级间接块中找到对应的一级间接块指针 `i` (`block >> 9` 是在一级间接块数组中的索引)。
        *   再读取这个一级间接块，从中找到最终的物理块指针 (`block & 511` 是在二级寻址的一级间接块中的索引)。
        *   过程中如果需要创建，则分配新块并更新相应的间接块和 inode。
        *   返回最终的物理块号 `i`。

## 总结

`inode.c` 是 Linux 0.11 文件系统的心脏，它负责管理文件的核心元数据——inode。通过内存 inode 表、磁盘同步机制、引用计数以及复杂但有效的数据块映射逻辑 (支持直接、一级间接、二级间接块)，它实现了文件的创建、查找、读写、删除等操作的基础。inode 的锁机制提供了基本的并发控制。这个文件体现了早期Unix文件系统设计的精髓，并为Linux文件系统的后续发展奠定了坚实的基础。Okay, I have created `docs/fs/inode.md` with the detailed Chinese explanation for `fs/inode.c`.

Next, I will process `fs/ioctl.c`.
1. Create `docs/fs/ioctl.md`.
2. Read `fs/ioctl.c`.
3. Write the explanation into `docs/fs/ioctl.md`.

First, I'll read `fs/ioctl.c`.
