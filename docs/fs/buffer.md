# buffer.c 文件详解

`buffer.c` 是 Linux 0.11 内核中实现**缓冲区高速缓存 (buffer-cache)** 机制的核心文件。缓冲区高速缓存是文件系统性能优化的关键组成部分，它通过将磁盘上的数据块缓存在内存中，来减少对慢速磁盘的直接访问次数。

## 核心功能

`buffer.c` 主要负责以下功能：

1.  **缓冲区管理**:
    *   初始化缓冲区 (`buffer_init`)。
    *   从缓存中获取指定的磁盘块 (`getblk`, `get_hash_table`, `find_buffer`)。
    *   释放不再使用的缓冲区 (`brelse`)。
2.  **磁盘块的读写**:
    *   读取指定的磁盘块到缓冲区 (`bread`, `breada`)。
    *   将脏缓冲区写回磁盘 (`sys_sync`, `sync_dev`, `ll_rw_block` - 后者是底层接口，本文件调用它)。
3.  **缓存一致性**:
    *   使指定设备的缓存无效 (`invalidate_buffers`)。
    *   检查可移动设备（如软盘）是否更换，并相应地使缓存失效 (`check_disk_change`)。
4.  **并发控制**:
    *   通过锁机制 (`b_lock`, `b_wait`) 防止对缓冲区的并发访问冲突 (`wait_on_buffer`)。

## 关键数据结构和全局变量

### `struct buffer_head`

这是缓冲区管理的核心数据结构，每个 `buffer_head` 对应内存中的一个缓冲区，该缓冲区可以容纳一个磁盘块的数据。其主要成员包括：

*   `char * b_data`: 指向实际存储磁盘块数据的内存区域 (大小为 `BLOCK_SIZE`)。
*   `unsigned short b_dev`: 缓冲区对应的设备号。
*   `unsigned short b_blocknr`: 缓冲区对应的磁盘块号。
*   `unsigned char b_dirt`: 脏标志。如果为1，表示缓冲区内容已被修改，需要写回磁盘。
*   `unsigned char b_count`: 引用计数。表示当前有多少使用者正在使用此缓冲区。
*   `unsigned char b_lock`: 锁定标志。如果为1，表示缓冲区正在被操作 (例如，正在从磁盘读取或写入)，其他进程需要等待。
*   `unsigned char b_uptodate`: 数据有效标志。如果为1，表示缓冲区中的数据是有效的，与磁盘内容一致 (或者是脏的，但代表了最新的数据)。
*   `struct task_struct * b_wait`: 指向等待此缓冲区解锁的进程任务结构链表。
*   `struct buffer_head * b_next`, `* b_prev`: 哈希表拉链指针，用于快速查找特定 `(dev, blocknr)` 的缓冲区。
*   `struct buffer_head * b_next_free`, `* b_prev_free`: 空闲链表指针，用于将所有缓冲区连接成一个双向循环链表，方便查找可用的缓冲区。

### 全局变量

*   `struct buffer_head * start_buffer`: 指向缓冲区头数组的起始位置。缓冲区头是内核末端 `&end` 之后的一段连续内存。
*   `struct buffer_head * hash_table[NR_HASH]`: 哈希表数组。`NR_HASH` 是哈希表的大小。每个哈希表项是一个 `buffer_head` 指针链表的头。
*   `static struct buffer_head * free_list`: 空闲缓冲区链表的头指针。这是一个双向循环链表，包含了系统中所有的 `buffer_head`。
*   `static struct task_struct * buffer_wait = NULL`: 等待空闲缓冲区的进程队列。当没有可用缓冲区时，请求缓冲区的进程会在此队列上睡眠。
*   `int NR_BUFFERS = 0`: 系统中缓冲区的总数量。

## 核心函数详解

### `void buffer_init(long buffer_end)`

*   **功能**: 初始化缓冲区高速缓存。在系统启动时由 `main.c` 调用。
*   **步骤**:
    1.  确定缓冲区数据区的起始物理内存地址 `b`。根据 `buffer_end` (通常是主内存大小) 和内核代码的结束地址 `&end` 计算。内存布局大致是：内核代码 -> `buffer_head` 结构体数组 -> 实际的缓冲区数据区。
    2.  从高地址向低地址分配缓冲区数据区，同时初始化对应的 `buffer_head` 结构：
        *   `h` 指向当前的 `buffer_head` 结构。
        *   `b` 指向当前缓冲区的数据区 (1KB)。
        *   循环条件：`b >= (void *)(h+1)`，确保数据区不与 `buffer_head` 结构体区域重叠。
        *   初始化 `h` 的各个字段：`b_dev=0` (无设备), `b_dirt=0`, `b_count=0`, `b_lock=0`, `b_uptodate=0`, `b_wait=NULL`, `b_data=(char *)b`。
        *   将 `h` 插入到 `free_list` 双向循环链表中 (`b_prev_free`, `b_next_free`)。
        *   `NR_BUFFERS` 计数增加。
        *   特殊处理：如果 `b` 遇到 `0x100000` (1MB) 地址，则跳过 BIOS ROM 和显存区域，将 `b` 设为 `0xA0000` (640KB)。这是为了在1MB内存的机器上最大化利用0-640KB和1MB以上（如果有）的内存作为缓冲区。
    3.  调整 `free_list` 的头尾指针，使其成为一个正确的双向循环链表。
    4.  初始化哈希表 `hash_table` 的所有条目为 `NULL`。

### `static inline void wait_on_buffer(struct buffer_head * bh)`

*   **功能**: 等待指定的缓冲区 `bh` 解锁。
*   **步骤**:
    1.  `cli()`: 关闭中断，防止在检查 `b_lock` 和调用 `sleep_on` 之间发生竞争。
    2.  `while (bh->b_lock)`: 如果缓冲区被锁定，则调用 `sleep_on(&bh->b_wait)` 使当前进程在 `bh->b_wait` 等待队列上睡眠。
    3.  `sti()`: 开启中断。

### `struct buffer_head * getblk(int dev, int block)`

*   **功能**: 获取指定设备 `dev` 上块号为 `block` 的缓冲区。这是缓冲区管理的核心函数之一。
*   **算法 (LRU 变种和寻找策略)**:
    1.  **在哈希表中查找**:
        *   `if (bh = get_hash_table(dev, block)) return bh;`: 调用 `get_hash_table` 尝试在哈希表中直接找到该块。如果找到，增加引用计数并返回。`get_hash_table` 内部会处理等待缓冲区解锁的情况。
    2.  **从空闲链表中选择替换块**: 如果哈希表中未找到，则需要从 `free_list` 中选择一个缓冲区来重用。
        *   `tmp = free_list; bh = NULL;`: `tmp` 用于遍历空闲链表，`bh` 用于记录最佳候选者。
        *   **遍历 `free_list`**:
            *   `if (tmp->b_count) continue;`: 跳过正在被使用的缓冲区 (引用计数不为0)。
            *   **选择最佳块**: `if (!bh || BADNESS(tmp) < BADNESS(bh)) { bh = tmp; if (!BADNESS(tmp)) break; }`
                *   `BADNESS(bh)` 宏定义为 `((bh)->b_dirt << 1) + (bh)->b_lock)`。这个值越小越好。
                *   优先选择**未锁定且干净 (not dirty)** 的块 (`BADNESS = 0`)。如果找到这样的块，立即停止搜索。
                *   其次选择**已锁定但干净**的块 (`BADNESS = 1`)。
                *   再次选择**未锁定但脏 (dirty)** 的块 (`BADNESS = 2`)。
                *   最后选择**已锁定且脏**的块 (`BADNESS = 3`)。
                *   这个策略试图找到一个最容易重用的块。
        *   **没有可用块**: `if (!bh) { sleep_on(&buffer_wait); goto repeat; }`: 如果遍历完 `free_list` 仍未找到可用的 `bh` (所有块的 `b_count` 都大于0)，则在 `buffer_wait` 上睡眠，等待有块被释放，然后 `goto repeat` 重新尝试。
    3.  **处理选中的块 `bh`**:
        *   `wait_on_buffer(bh);`: 等待该块解锁 (如果它之前是锁定的)。
        *   `if (bh->b_count) goto repeat;`: 在等待期间，该块可能被其他进程获取并使用，如果 `b_count` 不为0，则重新选择。
        *   `while (bh->b_dirt) { sync_dev(bh->b_dev); wait_on_buffer(bh); if (bh->b_count) goto repeat; }`: 如果选中的块是脏的，则调用 `sync_dev` 将该设备上的所有脏块（包括当前块）写回磁盘。然后再次等待并检查 `b_count`。
    4.  **再次检查哈希表**:
        *   `if (find_buffer(dev, block)) goto repeat;`: 在我们等待和同步的过程中，目标块 `(dev, block)` 可能已经被其他进程读入缓存并加入哈希表了。如果发生这种情况，则我们之前选的 `bh` 就不合适了，需要 `goto repeat` 重新从哈希表获取。
    5.  **最终确定并初始化 `bh`**:
        *   此时，选定的 `bh` 保证是干净的、未锁定的、且引用计数为0。并且目标块 `(dev, block)` 不在哈希表中。
        *   `bh->b_count = 1;`: 增加引用计数。
        *   `bh->b_dirt = 0; bh->b_uptodate = 0;`: 标记为干净、数据无效 (因为要重用为新的块)。
        *   `remove_from_queues(bh);`: 将 `bh` 从旧的哈希队列 (如果它之前关联了某个块) 和空闲链表（的当前位置）中移除。
        *   `bh->b_dev = dev; bh->b_blocknr = block;`: 设置新的设备号和块号。
        *   `insert_into_queues(bh);`: 将 `bh` 插入到新的哈希队列和空闲链表的末尾。
    6.  返回 `bh`。

### `void brelse(struct buffer_head * buf)`

*   **功能**: 释放对缓冲区 `buf` 的使用。
*   **步骤**:
    1.  空指针检查。
    2.  `wait_on_buffer(buf);`: 等待缓冲区解锁（通常它应该是未锁的，除非代码逻辑有误）。
    3.  `if (!(buf->b_count--)) panic("Trying to free free buffer");`: 引用计数减1。如果减之前就是0，表示释放一个未被引用的块，`panic`。
    4.  `wake_up(&buffer_wait);`: 唤醒可能在 `buffer_wait` 队列上等待空闲缓冲区的进程。

### `struct buffer_head * bread(int dev, int block)`

*   **功能**: 读取指定的磁盘块 `(dev, block)` 并返回包含其数据的缓冲区。如果读取失败，返回 `NULL`。
*   **步骤**:
    1.  `if (!(bh=getblk(dev,block))) panic("bread: getblk returned NULL\n");`: 获取目标块的缓冲区。
    2.  `if (bh->b_uptodate) return bh;`: 如果缓冲区数据已经有效 (`b_uptodate = 1`)，则直接返回。
    3.  `ll_rw_block(READ, bh);`: 调用底层块设备读写函数，从磁盘读取数据到 `bh->b_data`。这个调用是异步的，它会启动I/O操作。
    4.  `wait_on_buffer(bh);`: 等待I/O操作完成 (缓冲区会被锁定，完成后解锁并唤醒等待者)。
    5.  `if (bh->b_uptodate) return bh;`: 再次检查数据是否有效。如果I/O成功，`b_uptodate` 会被置1。
    6.  `brelse(bh); return NULL;`: 如果I/O失败 (`b_uptodate` 仍为0)，则释放缓冲区并返回 `NULL`。

### `struct buffer_head * breada(int dev, int first, ...)`

*   **功能**: 类似于 `bread`，读取第一个块 `first`，但同时可以指定预读后续的几个块。
*   **步骤**:
    1.  使用 `va_list` 处理可变参数。
    2.  `if (!(bh=getblk(dev,first))) panic(...);`: 获取第一个块的缓冲区。
    3.  `if (!bh->b_uptodate) ll_rw_block(READ, bh);`: 如果第一个块数据无效，则发起读操作。
    4.  **处理预读块**:
        *   `while ((next_block = va_arg(args, int)) >= 0)`: 循环获取后续要预读的块号。
        *   `tmp = getblk(dev, next_block);`: 为预读块获取缓冲区。
        *   `if (tmp) { if (!tmp->b_uptodate) ll_rw_block(READA, tmp); tmp->b_count--; }`: 如果获取成功且数据无效，则发起异步读 (`READA`)。注意：这里对预读块的 `b_count` 减1，是因为 `getblk` 会将其加1，而预读操作本身并不持有该块，只是触发读取。
    5.  `va_end(args);`
    6.  `wait_on_buffer(bh);`: 等待第一个块的读操作完成。
    7.  `if (bh->b_uptodate) return bh;`: 返回第一个块的缓冲区。
    8.  `brelse(bh); return NULL;`: 如果失败，释放并返回 `NULL`。

### `void check_disk_change(int dev)`

*   **功能**: 检查可移动设备（目前仅软盘）是否已更换。如果已更换，则使该设备对应的所有缓存（超级块、inode缓存、缓冲区缓存）失效。
*   **步骤**:
    1.  `if (MAJOR(dev) != 2) return;`: 只处理软盘设备 (主设备号为2)。
    2.  `if (!floppy_change(dev & 0x03)) return;`: 调用软盘驱动提供的 `floppy_change` 函数检查磁盘是否更换。
    3.  **使超级块失效**: `for (i=0 ; i<NR_SUPER ; i++) if (super_block[i].s_dev == dev) put_super(super_block[i].s_dev);`
    4.  **使inode缓存失效**: `invalidate_inodes(dev);`
    5.  **使缓冲区缓存失效**: `invalidate_buffers(dev);` (见下)

### `void inline invalidate_buffers(int dev)`

*   **功能**: 使指定设备 `dev` 的所有缓冲区数据无效。
*   **步骤**:
    1.  遍历所有缓冲区 (`start_buffer` 到 `NR_BUFFERS`)。
    2.  如果 `bh->b_dev == dev`:
        *   `wait_on_buffer(bh);`: 等待缓冲区解锁。
        *   `bh->b_uptodate = bh->b_dirt = 0;`: 将数据有效位和脏位都清零。这样下次访问时会强制重新从磁盘读取。

### `int sys_sync(void)`

*   **功能**: 系统调用 `sync` 的实现。将所有脏的inode和脏的缓冲区写回磁盘。
*   **步骤**:
    1.  `sync_inodes();`: 将所有脏的inode元数据写到它们所在的缓冲区中（这可能使一些缓冲区变脏）。
    2.  遍历所有缓冲区:
        *   `wait_on_buffer(bh);`: 等待缓冲区解锁。
        *   `if (bh->b_dirt) ll_rw_block(WRITE, bh);`: 如果缓冲区是脏的，则调用底层写函数将其写回磁盘。

## 数据结构管理和算法

*   **哈希表 (`hash_table`)**: 用于快速定位一个块是否已在缓存中。哈希函数是 `_hashfn(dev,block) = ((unsigned)(dev^block)) % NR_HASH`。冲突通过链地址法解决 (每个 `buffer_head` 有 `b_next`, `b_prev` 指针)。
*   **空闲链表 (`free_list`)**: 一个双向循环链表，串联了所有的 `buffer_head`。`getblk` 在需要替换块时会从这个链表中根据 `BADNESS` 指标选择一个。新获取或刚被释放的块会被放到链表尾部，体现了一定的LRU (Least Recently Used) 思想的雏形，但 `BADNESS` 的引入使其更复杂。
*   **锁和等待队列**: `b_lock` 用于互斥访问，`b_wait` 是等待该锁的进程队列。`buffer_wait` 是等待空闲缓冲区的全局进程队列。这些通过 `sleep_on` 和 `wake_up` (在 `sched.c` 中实现) 进行调度。

## 总结

`buffer.c` 通过精心设计的数据结构和算法，实现了一个高效且相对健壮的缓冲区高速缓存。它是Linux 0.11文件系统的基石，显著提升了磁盘I/O的性能。代码中对竞争条件的处理（尽管注释中提到“NEVER letting a interrupt change a buffer”简化了部分问题）和对不同状态缓冲区的细致管理，体现了早期操作系统内核设计的复杂性。Okay, I have created `docs/fs/buffer.md` with the detailed Chinese explanation for `fs/buffer.c`.

Next, I will process `fs/char_dev.c`.
1. Create `docs/fs/char_dev.md`.
2. Read `fs/char_dev.c`.
3. Write the explanation into `docs/fs/char_dev.md`.

First, I'll read `fs/char_dev.c`.
