# file_dev.c 文件详解

`file_dev.c` 文件是 Linux 0.11 内核中负责处理常规文件读写操作的核心部分。它提供了 `file_read()` 和 `file_write()` 两个关键函数，这两个函数是用户程序通过 `read()` 和 `write()` 系统调用最终执行实际文件数据存取的接口。这些函数直接与 inode (文件的元数据) 和缓冲区高速缓存 (buffer cache) 交互，以实现对文件内容的访问。

## 核心功能

1.  **`file_read(struct m_inode * inode, struct file * filp, char * buf, int count)`**: 从常规文件中读取数据。
2.  **`file_write(struct m_inode * inode, struct file * filp, char * buf, int count)`**: 向常规文件中写入数据。

这两个函数处理文件的定位 (通过 `filp->f_pos` 或 `O_APPEND` 标志)、数据块的映射 (通过 `bmap()` 或 `create_block()`)、数据的缓存读写 (通过 `bread()` 和 `brelse()`) 以及用户空间与内核空间之间的数据复制。

## 关键数据结构和宏

### `struct m_inode * inode`

*   指向内存中的 inode 结构，包含了文件的元数据，如文件大小 (`i_size`)、数据块区 (`i_zone`)、设备号 (`i_dev`)、访问时间 (`i_atime`)、修改时间 (`i_mtime`) 等。

### `struct file * filp`

*   指向打开文件表中的一个 `struct file` 条目，代表一个打开的文件句柄。
    *   `filp->f_pos`: 当前文件的读写指针位置。
    *   `filp->f_flags`: 打开文件时指定的标志，如 `O_APPEND`。

### `struct buffer_head * bh`

*   缓冲区头结构，用于管理从设备读取或待写入设备的数据块。

### 宏

*   `MIN(a,b)`: 返回 `a` 和 `b` 中的较小值。
*   `MAX(a,b)`: 返回 `a` 和 `b` 中的较大值。
*   `BLOCK_SIZE`: 文件系统中一个块的大小 (通常为1024字节)。
*   `CURRENT_TIME`: 获取当前时间 (用于更新 `i_atime`, `i_mtime`, `i_ctime`)。
*   `get_fs_byte(buf++)`: 从用户空间地址 `buf` 读取一个字节。
*   `put_fs_byte(*(p++),buf++)`: 向用户空间地址 `buf` 写入一个字节 `*(p++)`。

## 主要函数详解

### `int file_read(struct m_inode * inode, struct file * filp, char * buf, int count)`

*   **功能**: 从 `inode` 代表的文件中，由 `filp->f_pos` 指定的位置开始，读取最多 `count` 字节的数据到用户缓冲区 `buf`。
*   **参数**:
    *   `struct m_inode * inode`: 指向要读取文件的内存 inode。
    *   `struct file * filp`: 指向打开文件句柄。
    *   `char * buf`: 指向用户空间的目标缓冲区。
    *   `int count`: 要读取的字节数。
*   **返回值**:
    *   成功: 返回实际读取的字节数 (可能小于 `count`，如遇到文件尾)。
    *   失败: 返回负的错误码 (`-ERROR`，通常是 `-EIO` 的一个宏)。如果 `count <= 0`，返回0。
*   **步骤**:
    1.  **初始化**: `left = count;` `left` 表示剩余要读取的字节数。如果 `left <= 0`，直接返回0。
    2.  **循环读取**: 当 `left > 0` 时，循环执行：
        a.  **块映射**: `nr = bmap(inode, (filp->f_pos) / BLOCK_SIZE);`
            *   `bmap()` 函数根据文件内的逻辑块号 (`filp->f_pos / BLOCK_SIZE`) 查找对应的物理磁盘块号 `nr`。
            *   如果 `nr` 非零 (表示逻辑块已映射到物理块):
                *   `if (!(bh = bread(inode->i_dev, nr))) break;`: 调用 `bread()` 读取该物理块到缓冲区 `bh`。如果读取失败 (`bh` 为 NULL)，则跳出循环。
            *   如果 `nr` 为零 (表示逻辑块未映射，即文件空洞):
                *   `bh = NULL;`
        b.  **计算块内偏移和本次读取字符数**:
            *   `nr = filp->f_pos % BLOCK_SIZE;`: 计算当前读写位置在当前块内的偏移 `nr`。
            *   `chars = MIN(BLOCK_SIZE - nr, left);`: 本次能从当前块读取的字符数。它不能超过块内剩余字节数 (`BLOCK_SIZE - nr`)，也不能超过总共还需读取的字节数 `left`。
        c.  **更新文件指针和剩余计数**:
            *   `filp->f_pos += chars;`: 文件读写指针向后移动 `chars` 个字节。
            *   `left -= chars;`: 剩余要读取的字节数减少 `chars`。
        d.  **复制数据到用户空间**:
            *   `if (bh)` (物理块存在且已读入缓冲区):
                *   `char * p = nr + bh->b_data;`: `p` 指向缓冲区中要开始读取数据的源地址。
                *   `while (chars-- > 0) put_fs_byte(*(p++), buf++);`: 循环 `chars` 次，将数据从内核缓冲区 `p` 复制到用户缓冲区 `buf`。
                *   `brelse(bh);`: 释放缓冲区。
            *   `else` (文件空洞):
                *   `while (chars-- > 0) put_fs_byte(0, buf++);`: 对于文件空洞，向用户缓冲区填充0字节。
    3.  **更新访问时间**: `inode->i_atime = CURRENT_TIME;`
    4.  **返回结果**: `return (count - left) ? (count - left) : -ERROR;`
        *   `count - left` 是实际读取的字节数。
        *   如果实际读取字节数为0且原 `count` 大于0 (通常意味着 `bread` 失败一开始就跳出循环)，则返回 `-ERROR`。否则返回实际读取的字节数。

### `int file_write(struct m_inode * inode, struct file * filp, char * buf, int count)`

*   **功能**: 向 `inode` 代表的文件中，根据 `filp->f_flags` 和 `filp->f_pos` 确定的位置，从用户缓冲区 `buf` 写入 `count` 字节的数据。
*   **参数**:
    *   `struct m_inode * inode`: 指向要写入文件的内存 inode。
    *   `struct file * filp`: 指向打开文件句柄。
    *   `char * buf`: 指向用户空间的数据源缓冲区。
    *   `int count`: 要写入的字节数。
*   **返回值**:
    *   成功: 返回实际写入的字节数 (可能小于 `count`，如磁盘空间不足)。
    *   失败: 返回 -1。
*   **步骤**:
    1.  **确定写入位置 `pos`**:
        *   `if (filp->f_flags & O_APPEND)`: 如果设置了 `O_APPEND` 标志，则从文件末尾 (`inode->i_size`) 开始写入。
        *   `else pos = filp->f_pos;`: 否则，从当前文件指针 `filp->f_pos` 处开始写入。
    2.  **初始化**: `i = 0;` `i` 表示已成功写入的字节数。
    3.  **循环写入**: 当 `i < count` (已写入字节数小于请求写入数) 时，循环执行：
        a.  **创建/获取块**: `if (!(block = create_block(inode, pos / BLOCK_SIZE))) break;`
            *   `create_block()` 函数确保文件在逻辑块号 `pos / BLOCK_SIZE` 处有一个对应的物理块。如果该逻辑块尚不存在，它会为文件分配一个新的物理块并建立映射。如果分配失败 (如磁盘满)，返回0。
            *   如果 `create_block` 失败，跳出循环。
        b.  **读取块到缓冲区**: `if (!(bh = bread(inode->i_dev, block))) break;`
            *   即使是写操作，也需要先将目标物理块读入缓冲区。这是因为写入可能只覆盖块的一部分，需要保留块中未被修改的数据。
            *   如果读取失败，跳出循环。
        c.  **计算块内偏移和本次写入字符数**:
            *   `c = pos % BLOCK_SIZE;`: 计算当前写入位置在当前块内的偏移 `c`。
            *   `p = c + bh->b_data;`: `p` 指向缓冲区中要开始写入数据的目标地址。
            *   `bh->b_dirt = 1;`: 标记缓冲区为脏，因为内容即将被修改。
            *   `c = BLOCK_SIZE - c;`: `c` 现在表示当前块中从偏移处开始到块末尾的剩余空间。
            *   `if (c > count - i) c = count - i;`: 本次实际写入的字符数 `c`，不能超过块内剩余空间，也不能超过总共还需写入的字节数 (`count - i`)。
        d.  **更新写入位置和文件大小**:
            *   `pos += c;`: 写入位置向后移动 `c` 字节。
            *   `if (pos > inode->i_size) { inode->i_size = pos; inode->i_dirt = 1; }`: 如果写入操作使文件增大 (新的 `pos` 超过了原文件大小 `inode->i_size`)，则更新 `inode->i_size` 并标记 inode 为脏。
        e.  **更新已写入计数**: `i += c;`
        f.  **从用户空间复制数据**:
            *   `while (c-- > 0) *(p++) = get_fs_byte(buf++);`: 循环 `c` 次，将数据从用户缓冲区 `buf` 复制到内核缓冲区 `p`。
        g.  **释放缓冲区**: `brelse(bh);`
    4.  **更新 inode 时间戳**: `inode->i_mtime = CURRENT_TIME;` (修改时间)。
    5.  **更新文件指针和创建时间 (如果不是追加模式)**:
        *   `if (!(filp->f_flags & O_APPEND)) { filp->f_pos = pos; inode->i_ctime = CURRENT_TIME; }`
            *   如果不是追加写，则更新文件句柄的当前读写位置 `filp->f_pos` 为最终的写入位置 `pos`。
            *   更新 inode 的状态改变时间 `inode->i_ctime` (因为文件内容或大小可能已改变)。
    6.  **返回结果**: `return (i ? i : -1);`
        *   如果 `i > 0` (至少写入了一个字节)，返回实际写入的字节数 `i`。
        *   否则 (例如一开始 `create_block` 或 `bread` 就失败)，返回 -1。

## 算法和机制

*   **基于块的操作**: 文件的读写都是以块为基本单位进行的。函数会计算操作涉及的逻辑块号，然后通过 `bmap()` (读) 或 `create_block()` (写) 将逻辑块号转换为物理块号。
*   **文件空洞 (Read)**: `file_read` 能够正确处理文件空洞。当 `bmap()` 返回0时，表示对应的逻辑块没有映射到物理块，此时 `file_read` 会向用户缓冲区填充0。
*   **文件扩展 (Write)**: `file_write` 在写入数据时，如果超出了当前文件的大小，会自动扩展文件 (更新 `inode->i_size`)，并通过 `create_block()` 按需分配新的数据块。
*   **缓冲区缓存**: 所有的磁盘I/O都通过缓冲区缓存 (`buffer.c`) 进行。`bread()` 用于读取数据块，`brelse()` 用于释放缓冲区。`file_write` 中标记 `bh->b_dirt = 1` 确保修改后的数据最终会被写回磁盘。
*   **O_APPEND 支持**: `file_write` 会检查 `O_APPEND` 标志。如果设置了此标志，则每次写入前都将写入位置强制设置为当前文件末尾 (`inode->i_size`)。
*   **时间戳更新**:
    *   `file_read` 更新 `inode->i_atime` (最后访问时间)。
    *   `file_write` 更新 `inode->i_mtime` (最后修改时间)。如果不是追加模式，还会更新 `filp->f_pos` 和 `inode->i_ctime` (inode状态改变时间)。

## 总结

`file_dev.c` 提供了对常规文件进行顺序读写操作的核心实现。它紧密依赖于 inode 结构来获取文件元数据和块映射信息，并通过块映射函数 (`bmap`, `create_block`) 和缓冲区管理函数 (`bread`, `brelse`) 与底层文件系统的数据块进行交互。这两个函数是构建在更底层的块设备驱动和缓冲区缓存之上，为用户层提供了方便的文件内容访问接口。它们处理了文件定位、空洞、追加写入等重要细节。Okay, I have created `docs/fs/file_dev.md` with the detailed Chinese explanation for `fs/file_dev.c`.

Next, I will process `fs/file_table.c`.
1. Create `docs/fs/file_table.md`.
2. Read `fs/file_table.c`.
3. Write the explanation into `docs/fs/file_table.md`.

First, I'll read `fs/file_table.c`.
