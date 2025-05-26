# super.c 文件详解

`super.c` 文件是 Linux 0.11 内核中管理文件系统**超级块 (super-block)** 的核心部分。超级块是文件系统的“神经中枢”，它包含了文件系统的总体信息，如文件系统类型、大小、inode数量、数据块数量、位图（inode位图和数据块位图）的位置和大小等。当一个文件系统被挂载时，它的超级块会被读入内存，并存储在内核的超级块表 (`super_block[]`) 中。

此文件负责：
*   读取磁盘上的超级块到内存。
*   管理内存中的超级块表。
*   提供获取和释放内存超级块的函数。
*   实现文件系统的挂载 (`sys_mount()`) 和卸载 (`sys_umount()`) 系统调用。
*   挂载根文件系统 (`mount_root()`)。

## 核心功能

1.  **超级块管理**:
    *   定义全局内存超级块表 `super_block[NR_SUPER]`。
    *   `get_super(dev)`: 根据设备号 `dev` 从 `super_block` 表中查找对应的已加载超级块。
    *   `put_super(dev)`: 释放指定设备 `dev` 的超级块（主要用于可移动设备，标记为未使用并释放位图占用的缓冲区）。
    *   `read_super(dev)`: 从设备 `dev` 的磁盘上读取超级块信息，加载到 `super_block` 表中的一个空闲条目，并读取 inode 位图和数据块位图到缓冲区。
2.  **并发控制**:
    *   `lock_super(sb)`, `free_super(sb)` (实际是 `unlock_super`), `wait_on_super(sb)`: 对超级块进行加锁、解锁和等待解锁，以防止并发访问冲突。
3.  **文件系统挂载与卸载**:
    *   `sys_mount(dev_name, dir_name, rw_flag)`: 将名为 `dev_name` 的设备挂载到目录 `dir_name` 上。
    *   `sys_umount(dev_name)`: 卸载名为 `dev_name` 的设备（实际是挂载点目录）。
4.  **根文件系统挂载**:
    *   `mount_root()`: 在系统启动时调用，用于挂载根文件系统。这是系统能够访问文件的前提。

## 关键数据结构和宏

### `struct super_block` (内存中的超级块结构)

这是在 `<linux/fs.h>` 中定义的结构体，代表一个在内存中活动的超级块。其主要成员包括：

*   `unsigned short s_ninodes`: 文件系统中总的 inode 数量。
*   `unsigned short s_nzones`: 文件系统中总的数据块数量 (逻辑块)。
*   `unsigned short s_imap_blocks`: inode 位图占用的磁盘块数。
*   `unsigned short s_zmap_blocks`: 数据块位图占用的磁盘块数。
*   `unsigned short s_firstdatazone`: 第一个数据块的起始逻辑块号 (通常是1，因为0号块是引导块)。
*   `unsigned short s_log_zone_size`: 每个逻辑块的大小 (以2的幂次方表示，例如0表示1KB，1表示2KB)。Linux 0.11 中通常是0 (1KB)。
*   `off_t s_max_size`: 文件的最大允许大小。
*   `unsigned short s_magic`: 文件系统魔数 (如 `SUPER_MAGIC`，用于标识Minix文件系统类型)。
*   `struct buffer_head * s_imap[I_MAP_SLOTS]`: 指向 inode 位图块缓冲区的指针数组。`I_MAP_SLOTS` (通常为8) 是最大支持的 inode 位图块数。
*   `struct buffer_head * s_zmap[Z_MAP_SLOTS]`: 指向数据块位图块缓冲区的指针数组。`Z_MAP_SLOTS` (通常为8) 是最大支持的数据块位图块数。
*   `unsigned short s_dev`: 该超级块对应的设备号。
*   `struct m_inode * s_isup`: 指向被覆盖文件系统（被挂载设备）的根 inode。
*   `struct m_inode * s_imount`: 指向该文件系统被挂载到的目录的 inode (即挂载点 inode)。
*   `time_t s_time`: 最后更新时间。
*   `struct task_struct * s_wait`: 等待此超级块解锁的进程任务结构链表。
*   `unsigned char s_lock`: 锁定标志。
*   `unsigned char s_rd_only`:只读标志。
*   `unsigned char s_dirt`: 脏标志 (如果超级块本身被修改，如时间戳)。

### `struct d_super_block` (磁盘上的超级块结构)

这也是在 `<linux/fs.h>` 中定义的结构体，与 `struct super_block` 的前一部分成员完全对应，代表超级块在磁盘上存储的格式。`read_super` 就是将磁盘上的 `d_super_block` 读入内存 `super_block` 结构的前面部分。

### `super_block[NR_SUPER]`

*   全局内存超级块表，是一个 `super_block` 类型的数组，大小为 `NR_SUPER` (通常为8)。系统所有已挂载文件系统的超级块都存储在这里。

### 宏

*   `NR_SUPER`: 系统内存中超级块表的大小。
*   `ROOT_DEV`: 根设备的设备号 (在 `init/main.c` 中初始化)。
*   `SUPER_MAGIC`: Minix 文件系统的魔数。
*   `I_MAP_SLOTS`, `Z_MAP_SLOTS`: 位图槽数量。
*   `set_bit(bitnr, addr)`: (在此文件中定义，但与 `fs/bitmap.c` 中的版本略有不同) 测试并设置位图中某一位，返回原值。这里使用 `bt` (bit test) 和 `setb` (set byte if below/carry clear) 指令。

## 主要函数详解

### `struct super_block * get_super(int dev)`

*   **功能**: 根据设备号 `dev` 从 `super_block` 表中查找并返回对应的已加载超级块。
*   **步骤**:
    1.  如果 `dev` 为0，返回 `NULL`。
    2.  遍历 `super_block` 数组：
        *   如果 `s->s_dev == dev` (找到匹配的设备号)：
            *   `wait_on_super(s);`: 等待该超级块解锁 (防止在检查期间状态改变)。
            *   `if (s->s_dev == dev) return s;`: 再次确认设备号未变，然后返回该超级块。
            *   如果设备号变了 (说明在等待期间此超级块被重用于其他设备)，则从头重新搜索。
    3.  如果未找到，返回 `NULL`。

### `void put_super(int dev)`

*   **功能**: 释放指定设备 `dev` 的超级块。这通常用于可移动设备被卸载或更换后，清理其在内存中的超级块结构。
*   **步骤**:
    1.  `if (dev == ROOT_DEV) ... return;`: 不能释放根文件系统的超级块。
    2.  `if (!(sb = get_super(dev))) return;`: 获取该设备的超级块。
    3.  `if (sb->s_imount) ... return;`: 如果该文件系统仍被挂载在某个目录 (`s_imount` 非空)，则不能释放 (打印警告)。
    4.  `lock_super(sb);`: 锁定超级块。
    5.  `sb->s_dev = 0;`: 将设备号清零，标记此超级块条目为空闲。
    6.  **释放位图缓冲区**: 遍历 `sb->s_imap` 和 `sb->s_zmap` 数组，调用 `brelse()` 释放它们占用的缓冲区。
    7.  `free_super(sb);` (即 `unlock_super`): 解锁超级块。

### `static struct super_block * read_super(int dev)`

*   **功能**: 从设备 `dev` 读取超级块信息，加载到内存，并读取位图。
*   **步骤**:
    1.  `check_disk_change(dev);`: 检查可移动设备是否已更换。
    2.  `if (s = get_super(dev)) return s;`: 如果超级块已在内存中，则直接返回。
    3.  **查找空闲 `super_block` 条目**: 遍历 `super_block` 数组，找到一个 `s->s_dev == 0` 的空闲条目 `s`。如果找不到，返回 `NULL`。
    4.  **初始化选中的 `s`**:
        *   `s->s_dev = dev;` (临时占用)
        *   清空 `s_isup`, `s_imount`, `s_time`, `s_rd_only`, `s_dirt`。
    5.  `lock_super(s);`
    6.  **读取磁盘上的超级块**:
        *   `if (!(bh = bread(dev, 1))) ... return NULL;`: 读取设备 `dev` 的第1块 (超级块通常位于此，0号块是引导块) 到缓冲区 `bh`。
        *   `*((struct d_super_block *) s) = *((struct d_super_block *) bh->b_data);`: 将缓冲区中的 `d_super_block` 数据复制到内存 `s` 结构的前面部分。
        *   `brelse(bh);`
    7.  **魔数校验**: `if (s->s_magic != SUPER_MAGIC) ... return NULL;`: 如果魔数不匹配，表示不是有效的Minix文件系统。
    8.  **初始化位图指针**: 将 `s->s_imap` 和 `s->s_zmap` 中的所有指针设为 `NULL`。
    9.  **读取 inode 位图和数据块位图**:
        *   `block = 2;` (位图从第2块开始)。
        *   循环读取 `s->s_imap_blocks` 个 inode 位图块到 `s->s_imap[i]`。
        *   循环读取 `s->s_zmap_blocks` 个数据块位图块到 `s->s_zmap[i]`。
        *   如果在读取过程中任何 `bread` 失败，或者最终 `block` 号与预期不符，则认为失败，释放已读取的位图缓冲区，并返回 `NULL`。
    10. **标记 inode 0 和 block 0 为已使用**:
        *   `s->s_imap[0]->b_data[0] |= 1;`: inode 0 不使用。
        *   `s->s_zmap[0]->b_data[0] |= 1;`: 数据块 0 (引导块) 不用于文件数据。
    11. `free_super(s);` 解锁超级块。
    12. 返回加载好的超级块 `s`。

### `int sys_mount(char * dev_name, char * dir_name, int rw_flag)`

*   **功能**: 将设备 `dev_name` 挂载到目录 `dir_name`。
*   **步骤**:
    1.  `if (!(dev_i = namei(dev_name))) return -ENOENT;`: 获取设备文件的 inode `dev_i`。
    2.  `dev = dev_i->i_zone[0];`: 从设备文件的 inode 中获取设备号。
    3.  `if (!S_ISBLK(dev_i->i_mode)) ... return -EPERM;`: 检查设备文件是否为块设备。
    4.  `iput(dev_i);`
    5.  `if (!(dir_i = namei(dir_name))) return -ENOENT;`: 获取挂载点目录的 inode `dir_i`。
    6.  **挂载点检查**:
        *   `if (dir_i->i_count != 1 || dir_i->i_num == ROOT_INO) ... return -EBUSY;`: 挂载点目录不能是根目录，并且其引用计数应为1 (表示没有其他进程正在使用它或其子目录)。
        *   `if (!S_ISDIR(dir_i->i_mode)) ... return -EPERM;`: 挂载点必须是目录。
    7.  `if (!(sb = read_super(dev))) ... return -EBUSY;`: 读取要挂载设备的超级块。如果失败 (如设备无效或文件系统损坏)，返回 `-EBUSY`。
    8.  **重复挂载检查**:
        *   `if (sb->s_imount) ... return -EBUSY;`: 如果该设备已经被挂载到其他地方 (`sb->s_imount` 非空)。
        *   `if (dir_i->i_mount) ... return -EPERM;`: 如果该目录已经作为其他设备的挂载点 (`dir_i->i_mount` 为1)。
    9.  **执行挂载**:
        *   `sb->s_imount = dir_i;`: 将被挂载设备的超级块的 `s_imount` 指向挂载点目录 inode `dir_i`。
        *   `dir_i->i_mount = 1;`: 标记挂载点目录 inode `dir_i` 为已挂载。
        *   `dir_i->i_dirt = 1;`: 标记挂载点 inode 为脏 (因为 `i_mount` 状态改变了)。**注意**: 这里没有 `iput(dir_i)`，对 `dir_i` 的引用将在 `umount` 时释放。
    10. 返回0。 `rw_flag` (读写标志) 在Linux 0.11中未被 `sys_mount` 直接使用来设置超级块的只读状态，但可以在 `read_super` 后手动设置 `sb->s_rd_only`。

### `int sys_umount(char * dev_name)`

*   **功能**: 卸载文件系统。`dev_name` 通常是挂载点目录的路径。
*   **步骤**:
    1.  `if (!(inode = namei(dev_name))) return -ENOENT;`: 获取 `dev_name` 对应的 inode (这应该是挂载点目录的 inode)。
    2.  `dev = inode->i_zone[0];`: **错误**: 这里试图从挂载点目录的 `i_zone[0]` 获取设备号，这通常是不对的。正确的设备号应该从超级块中获取，或者说，`dev_name` 应该直接是设备文件名。Linux 0.11 的 `umount` 实现是基于设备名，而不是挂载点。如果 `dev_name` 是挂载点，`inode->i_zone[0]` 不会是设备的设备号。假设这里的 `dev_name` 是设备文件名。
    3.  `if (!S_ISBLK(inode->i_mode)) ... return -ENOTBLK;`: 检查是否为块设备文件。
    4.  `iput(inode);`
    5.  `if (dev == ROOT_DEV) return -EBUSY;`: 不能卸载根文件系统。
    6.  `if (!(sb = get_super(dev)) || !(sb->s_imount)) return -ENOENT;`: 获取该设备的超级块，并检查它是否真的被挂载 (`sb->s_imount` 应该指向挂载点 inode)。
    7.  `if (!sb->s_imount->i_mount) printk(...);`: 检查挂载点 inode 的 `i_mount` 标志是否正确设置。
    8.  **检查是否有文件仍在使用**: `for (inode=inode_table+0 ... ) if (inode->i_dev==dev && inode->i_count) return -EBUSY;`: 遍历内存 inode 表，如果发现任何属于该设备的 inode 仍被使用 (`i_count > 0`)，则返回 `-EBUSY`。
    9.  **执行卸载**:
        *   `sb->s_imount->i_mount = 0;`: 清除挂载点 inode 的 `i_mount` 标志。
        *   `iput(sb->s_imount);`: 释放对挂载点 inode 的引用 (这是 `sys_mount` 时未释放的那个)。
        *   `sb->s_imount = NULL;`
        *   `iput(sb->s_isup); sb->s_isup = NULL;`: 如果这个文件系统覆盖了另一个文件系统 (通常不会在简单挂载中发生，`s_isup` 通常是根文件系统的根 inode，或者是更复杂挂载场景的产物)，则释放它。
        *   `put_super(dev);`: 释放该设备的超级块。
        *   `sync_dev(dev);`: 将设备上所有脏的缓冲区写回磁盘。
    10. 返回0。

### `void mount_root(void)`

*   **功能**: 在系统启动时挂载根文件系统。
*   **步骤**:
    1.  检查 `d_inode` 大小是否正确。
    2.  初始化系统文件表 `file_table`。
    3.  如果根设备是软盘 (主设备号为2)，提示用户插入软盘并等待按键。
    4.  初始化内存超级块表 `super_block` (清空设备号、锁和等待队列)。
    5.  `if (!(p = read_super(ROOT_DEV))) panic(...);`: 读取根设备的超级块。
    6.  `if (!(mi = iget(ROOT_DEV, ROOT_INO))) panic(...);`: 获取根设备的根 inode。
    7.  `mi->i_count += 3;`: 根 inode 的引用计数特殊增加 (逻辑上被当前进程的 `pwd` 和 `root`，以及超级块的 `s_isup` 和 `s_imount` 使用)。
    8.  `p->s_isup = p->s_imount = mi;`: 设置超级块的 `s_isup` (被覆盖的根) 和 `s_imount` (挂载点) 都指向根 inode 本身。
    9.  `current->pwd = mi; current->root = mi;`: 设置当前进程 (init进程) 的当前工作目录和根目录为根 inode。
    10. 遍历数据块位图和 inode 位图，统计并打印空闲块和空闲 inode 的数量。这里使用 `set_bit` 检查每一位，如果该位原先是0 (空闲)，`set_bit` 会返回0，然后 `free` 计数增加。这实际上是一个检查并顺便“使用”（在位图上标记为使用，尽管这里只是为了统计）所有块和inode的过程，这看起来不太对，`set_bit`不应在这里被用来修改位图状态。正确的做法应该是检查位图中的0的数量。Linux 0.11这里的统计方法可能有误或有特殊含义（例如，假设位图初始可能不干净）。

## 总结

`super.c` 是文件系统管理的核心，它通过超级块的加载、管理和卸载，实现了文件系统在内核中的注册和注销。挂载 (`sys_mount`) 和卸载 (`sys_umount`) 功能使得Linux可以支持多个文件系统，并将它们组织成一个统一的目录树。`mount_root` 函数是系统启动的关键步骤，它使得内核能够访问根文件系统上的程序和数据，从而继续系统的初始化过程。超级块中包含的位图信息是文件和数据块分配的基础。并发控制锁确保了对超级块操作的原子性。Okay, I have created `docs/fs/super.md` with the detailed Chinese explanation for `fs/super.c`.

Next, I will process `fs/truncate.c`.
1. Create `docs/fs/truncate.md`.
2. Read `fs/truncate.c`.
3. Write the explanation into `docs/fs/truncate.md`.

First, I'll read `fs/truncate.c`.
