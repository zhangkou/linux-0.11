# bitmap.c 文件详解

`bitmap.c` 文件是 Linux 0.11 内核中文件系统部分的核心组件之一，主要负责管理文件系统中的两种重要资源的分配与回收：**inode (索引节点)** 和 **block (数据块)**。它通过位图 (bitmap) 的方式来跟踪哪些 inode 和数据块是空闲的，哪些是已被使用的。

## 核心功能

该文件实现了以下核心功能：

1.  **数据块的分配与释放 (Block Management)**:
    *   `free_block(int dev, int block)`: 释放指定设备上的一个数据块。
    *   `new_block(int dev)`: 在指定设备上分配一个新的空闲数据块。
2.  **Inode 的分配与释放 (Inode Management)**:
    *   `free_inode(struct m_inode * inode)`: 释放一个 inode。
    *   `new_inode(int dev)`: 在指定设备上分配一个新的空闲 inode。

## 关键数据结构和宏

### 数据结构

*   **`struct super_block * sb`**: 超级块结构。它包含了文件系统的总体信息，包括指向 inode 位图 (`s_imap`) 和数据块位图 (`s_zmap`) 的指针。每个已挂载的文件系统在内存中都有一个超级块结构。
*   **`struct buffer_head * bh`**: 缓冲区头结构。位图本身也存储在磁盘的数据块中，并通过缓冲区高速缓存进行管理。`bh->b_data` 指向实际的位图数据。
*   **`struct m_inode * inode`**: 内存中的 inode 结构。

### 宏定义

文件开头定义了几个关键的内联汇编宏，用于高效地操作位图：

*   **`clear_block(addr)`**:
    *   功能：将指定地址 `addr` 开始的一个数据块（`BLOCK_SIZE` 字节）清零。
    *   实现：使用 `cld` (清除方向标志，使 `stos` 向前操作)，`rep` (重复执行)，`stosl` (将 `eax` 中的值存储到 `edi` 指向的地址，并根据方向标志增加/减少 `edi`)。`eax` 设置为0，`ecx` 设置为 `BLOCK_SIZE/4` (因为 `stosl` 一次操作4字节)，`edi` 设置为目标地址 `addr`。

*   **`set_bit(nr, addr)`**:
    *   功能：在指定地址 `addr` 处的位图中，将第 `nr` 位置1，并返回该位原来的值。
    *   实现：使用 `btsl %2,%3` (Bit Test and Set Long) 指令。该指令测试由 `%3` (内存地址 `*(addr)`) 和 `%2` (位号 `nr`) 指定的位，将其值存入进位标志 CF，然后将该位置1。`setb %%al` 根据 CF 的值设置 `al` (如果 CF=1 则 al=1, 否则 al=0)。函数返回 `al` 的值。

*   **`clear_bit(nr, addr)`**:
    *   功能：在指定地址 `addr` 处的位图中，将第 `nr` 位清零，并返回该位原来的值。
    *   实现：使用 `btrl %2,%3` (Bit Test and Reset Long) 指令。该指令测试指定位，将其值存入 CF，然后将该位清零。`setnb %%al` (Set Not Below, 等价于 Set if Carry Clear) 根据 CF 的值设置 `al` (如果 CF=0 则 al=1, 否则 al=0)。这里有点反直觉，如果位原来是1，`btrl`后CF=1，`setnb`使得`al=0`；如果位原来是0，`btrl`后CF=0，`setnb`使得`al=1`。所以返回1表示该位本来就是0。

*   **`find_first_zero(addr)`**:
    *   功能：在指定地址 `addr` 开始的位图中 (最大检查 8192 位，即一个块)，查找第一个为0的位，并返回其位号 (从0开始计数)。
    *   实现：
        1.  `cld`: 清方向标志。
        2.  `1: lodsl`: 从 `esi` (初始为 `addr`)加载一个双字 (4字节) 到 `eax`，`esi` 自动加4。
        3.  `notl %%eax`: 对 `eax` 按位取反。这样原来为0的位现在为1。
        4.  `bsfl %%eax,%%edx`: (Bit Scan Forward Long) 在 `eax` 中从低位到高位扫描第一个为1的位，将其索引存入 `edx`，并设置零标志ZF (如果找到则ZF=0，否则ZF=1)。
        5.  `je 2f`: 如果没找到1 (即原双字全为1，取反后全为0)，则ZF=1，跳转到 `2f`。
        6.  `addl %%edx,%%ecx`: 如果找到了，将位索引 `edx` 加到总位号计数器 `ecx` (初始为0)。
        7.  `jmp 3f`: 跳转到结束。
        8.  `2: addl $32,%%ecx`: 如果当前双字没有0位 (即取反后没有1位)，则将总位号 `ecx` 增加32。
        9.  `cmpl $8192,%%ecx`: 比较总位号是否达到8192 (一个块的总位数 `BLOCK_SIZE * 8 = 1024 * 8 = 8192`)。
        10. `jl 1b`: 如果还没达到8192位，则继续扫描下一个双字。
        11. `3:`: 结束，返回 `ecx` 中的位号。

## 主要函数详解

### `void free_block(int dev, int block)`

*   **功能**: 释放指定设备 `dev` 上的指定数据块 `block`。
*   **步骤**:
    1.  **获取超级块**: `sb = get_super(dev)`。如果设备不存在，则 `panic`。
    2.  **有效性检查**: 检查 `block` 是否在数据区范围内 (`sb->s_firstdatazone` 到 `sb->s_nzones - 1`)。如果无效，则 `panic`。
    3.  **缓冲区检查**: `bh = get_hash_table(dev, block)` 尝试从哈希表中查找该块是否在缓冲区中。
        *   如果块在缓冲区中 (`bh` 非空):
            *   检查引用计数 `bh->b_count`。如果 `!= 1`，表示该块可能仍被其他地方使用或存在错误，打印信息并返回 (不释放)。
            *   将缓冲区的 `b_dirt` (脏位) 和 `b_uptodate` (有效位) 清零。
            *   `brelse(bh)`: 释放该缓冲区头。
    4.  **计算位图中的相对块号**: `block -= sb->s_firstdatazone - 1`。位图中的块号是从数据区的第一个块开始计数的 (通常是第1个块，而不是第0个，因为第0个块是引导块)。
    5.  **清除位图中的对应位**:
        *   `sb->s_zmap[block/8192]`: 定位到该块号所在的位图块。`s_zmap` 是一个指向 `buffer_head` 指针的数组，每个 `buffer_head` 管理一个位图块。一个位图块管理 8192 个数据块。
        *   `clear_bit(block&8191, sb->s_zmap[block/8192]->b_data)`: 在该位图块的数据区 (`b_data`) 中，清除相对块号 `block % 8192` (即 `block & 8191`) 对应的位。
        *   如果 `clear_bit` 返回非零 (在Linux 0.11的实现中，`clear_bit` 如果该位原本就是0，会返回1)，表示该位已经被清除了，打印信息并 `panic`。
    6.  **标记位图块为脏**: `sb->s_zmap[block/8192]->b_dirt = 1`。因为位图已被修改，需要写回磁盘。

### `int new_block(int dev)`

*   **功能**: 在指定设备 `dev` 上分配一个新的空闲数据块。
*   **步骤**:
    1.  **获取超级块**: `sb = get_super(dev)`。如果设备不存在，则 `panic`。
    2.  **查找第一个0位**:
        *   `j = 8192`: 初始化 `j` 为一个大于最大可能位号的值。
        *   遍历数据块位图的所有块 (`sb->s_zmap[i]`, `i` 从 0 到 7，因为 `ZMAP_BLOCKS` 最多为8)。
        *   `if (bh=sb->s_zmap[i])`: 确保位图块存在。
        *   `if ((j=find_first_zero(bh->b_data))<8192) break;`: 在当前位图块 `bh->b_data` 中查找第一个0位。如果找到 (`j < 8192`)，则跳出循环。
    3.  **检查查找结果**:
        *   如果 `i >= 8` (所有位图块都检查完了) 或 `!bh` (当前位图块无效) 或 `j >= 8192` (未找到0位)，表示磁盘已满或出错，返回0。
    4.  **设置位图中的对应位**:
        *   `if (set_bit(j, bh->b_data)) panic("new_block: bit already set");`: 在找到的位图块 `bh` 中，将第 `j` 位置1。如果 `set_bit` 返回非零 (表示该位原本就是1)，则 `panic`。
        *   `bh->b_dirt = 1`: 标记位图块为脏。
    5.  **计算绝对块号**: `j += i*8192 + sb->s_firstdatazone - 1`。
        *   `i*8192`: 加上前面所有位图块管理的块数。
        *   `sb->s_firstdatazone - 1`: 转换成相对于设备起始的绝对块号。
    6.  **有效性检查**: `if (j >= sb->s_nzones) return 0;`: 如果计算出的块号超出了设备的总数据块数，则返回0 (表示失败)。
    7.  **获取并清理新分配的块**:
        *   `if (!(bh=getblk(dev,j))) panic("new_block: cannot get block");`: 为新分配的块 `j` 获取一个缓冲区。`getblk` 会返回一个锁定的缓冲区，如果块不在缓存中则会读取。
        *   `if (bh->b_count != 1) panic("new block: count is != 1");`: 新获取的块引用计数应为1。
        *   `clear_block(bh->b_data)`: 将该数据块的内容清零。
        *   `bh->b_uptodate = 1`: 标记缓冲区数据有效 (虽然是全0，但这是确定的状态)。
        *   `bh->b_dirt = 1`: 标记缓冲区为脏 (因为清零了)。
        *   `brelse(bh)`: 释放该缓冲区。
    8.  **返回块号**: 返回新分配的绝对块号 `j`。

### `void free_inode(struct m_inode * inode)`

*   **功能**: 释放一个指定的 inode。
*   **步骤**:
    1.  **空指针检查**: `if (!inode) return;`
    2.  **设备检查**: `if (!inode->i_dev) { memset(inode,0,sizeof(*inode)); return; }`: 如果 inode 没有关联设备 (可能是个无效或未使用的 inode)，则清空 inode 结构并返回。
    3.  **引用计数检查**: `if (inode->i_count > 1) { ... panic("free_inode"); }`: 如果 inode 的引用计数大于1，表示它仍被其他地方使用，`panic`。
    4.  **链接数检查**: `if (inode->i_nlinks) panic("trying to free inode with links");`: 如果 inode 的链接数不为0，表示仍有目录项指向它，此时不应直接释放 inode (通常应先由 `iput` 处理链接数)。这里直接 `panic` 说明预期 `i_nlinks` 应该已经是0了。
    5.  **获取超级块**: `if (!(sb = get_super(inode->i_dev))) panic(...);`
    6.  **Inode 号有效性检查**: `if (inode->i_num < 1 || inode->i_num > sb->s_ninodes) panic(...);`: Inode 号必须在 `1` 到 `sb->s_ninodes` 之间 (inode 0 不使用)。
    7.  **获取 inode 位图块**: `if (!(bh=sb->s_imap[inode->i_num>>13])) panic(...);`:
        *   `inode->i_num >> 13` (等价于 `inode->i_num / 8192`) 计算 inode 号所在的位图块索引。
        *   `sb->s_imap` 是指向 inode 位图块的 `buffer_head` 指针数组。
    8.  **清除位图中的对应位**:
        *   `if (clear_bit(inode->i_num&8191, bh->b_data)) printk(...);`: 在该位图块的数据区 (`b_data`) 中，清除 inode 号 `inode->i_num % 8192` (即 `inode->i_num & 8191`) 对应的位。
        *   如果 `clear_bit` 返回非零 (表示该位原本就是0)，打印提示信息。
    9.  **标记位图块为脏**: `bh->b_dirt = 1;`
    10. **清空内存中的 inode 结构**: `memset(inode,0,sizeof(*inode));` 将传入的 `m_inode` 结构体清零，表示它不再代表一个有效的 inode。

### `struct m_inode * new_inode(int dev)`

*   **功能**: 在指定设备 `dev` 上分配一个新的空闲 inode。
*   **步骤**:
    1.  **获取空闲 inode 结构**: `if (!(inode=get_empty_inode())) return NULL;`: 从系统维护的 `inode_table` 中获取一个空闲的 `m_inode` 结构体。如果获取不到，返回 `NULL`。
    2.  **获取超级块**: `if (!(sb = get_super(dev))) panic("new_inode with unknown device");`
    3.  **查找第一个0位 (类似 `new_block`)**:
        *   遍历 inode 位图的所有块 (`sb->s_imap[i]`)。
        *   `if ((j=find_first_zero(bh->b_data))<8192) break;`: 在当前位图块中查找第一个0位。
    4.  **检查查找结果**:
        *   `if (!bh || j >= 8192 || j+i*8192 > sb->s_ninodes)`: 如果未找到0位，或者找到的 inode 编号 (`j + i*8192`) 超出了总 inode 数 (`sb->s_ninodes`)，则释放之前获取的空闲 `m_inode` 结构 (`iput(inode)`) 并返回 `NULL`。
    5.  **设置位图中的对应位**:
        *   `if (set_bit(j,bh->b_data)) panic("new_inode: bit already set");`: 将找到的位在位图块中置1。
        *   `bh->b_dirt = 1`: 标记位图块为脏。
    6.  **初始化新的 inode 结构**:
        *   `inode->i_count = 1;`: 引用计数置1。
        *   `inode->i_nlinks = 1;`: 链接数置1 (新创建的文件/目录至少有一个链接)。
        *   `inode->i_dev = dev;`: 设置设备号。
        *   `inode->i_uid = current->euid;`: 设置用户ID为当前进程的有效用户ID。
        *   `inode->i_gid = current->egid;`: 设置组ID为当前进程的有效组ID。
        *   `inode->i_dirt = 1;`: 标记 inode 为脏 (需要写回磁盘)。
        *   `inode->i_num = j + i*8192;`: 计算并设置 inode 的绝对编号。
        *   `inode->i_mtime = inode->i_atime = inode->i_ctime = CURRENT_TIME;`: 设置修改时间、访问时间、创建时间为当前时间。
        *   其他字段如 `i_mode`, `i_size`, `i_zone` 等会在之后根据文件类型和内容具体设置。
    7.  **返回 inode**: 返回初始化好的 `m_inode` 结构指针。

## 算法和机制

*   **位图管理**: 使用位图来表示 inode 和数据块的空闲状态。每个位对应一个 inode 或数据块。位为0表示空闲，位为1表示已分配。
*   **块式位图存储**: 位图本身也存储在磁盘块中，并通过 `s_imap` (inode map) 和 `s_zmap` (zone map, 即 block map) 在超级块中进行管理。每个位图块通常是1KB (`BLOCK_SIZE`)，可以管理 `1024 * 8 = 8192` 个 inode 或数据块。
*   **线性搜索**: `find_first_zero` 宏通过线性扫描位图块中的每个双字 (32位) 来查找第一个0位。
*   **两级查找**:
    1.  首先遍历位图块数组 (如 `sb->s_imap` 或 `sb->s_zmap`)，找到第一个包含0位的位图块。
    2.  然后在该位图块内使用 `find_first_zero` 查找具体的0位。
*   **错误处理**: 代码中大量使用了 `panic()` 来处理严重错误，例如在尝试释放一个不存在的设备上的块，或位图状态与预期不符等情况。对于某些可预期的失败 (如磁盘满)，则返回0或 `NULL`。

## 总结

`bitmap.c` 是 Minix 文件系统在 Linux 0.11 中实现的关键部分，它通过简单而有效的位图机制，提供了对 inode 和数据块这两种核心文件系统资源的分配和回收功能。这些功能是文件创建、删除、扩展等操作的基础。文件中的内联汇编宏展示了早期 Linux 内核对性能优化的关注。I have created `docs/fs/bitmap.md` and written the detailed explanation for `fs/bitmap.c`.
Now I'll proceed with `fs/block_dev.c`.
1. Create `docs/fs/block_dev.md`.
2. Read `fs/block_dev.c`.
3. Write the explanation into `docs/fs/block_dev.md`.

First, I'll read `fs/block_dev.c`.
