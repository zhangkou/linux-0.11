# pipe.c 文件详解

`pipe.c` 文件是 Linux 0.11 内核中实现管道 (pipe) 机制的核心部分。管道是一种单向的、先进先出 (FIFO) 的通信通道，通常用于在两个或多个相关进程之间传递数据。一个进程写入管道的数据可以被另一个进程读取。在 Linux 0.11 中，管道是基于内存的，使用一个物理页面作为环形缓冲区。

此文件主要包含三个函数：
*   `read_pipe()`: 从管道读取数据。
*   `write_pipe()`: 向管道写入数据。
*   `sys_pipe()`: 实现 `pipe()` 系统调用，用于创建一个新的管道。

## 核心功能

1.  **创建管道 (`sys_pipe`)**:
    *   分配一个内存 inode (`get_pipe_inode()`)，这个 inode 专门用于管道。
    *   为此 inode 分配一个物理页面 (`get_free_page()`) 作为管道的环形缓冲区。该页面的地址存储在 `inode->i_size` 中 (这是一个对 `i_size` 字段的非标准用法，因为管道内容不在磁盘上，所以 `i_size` 不代表文件大小)。
    *   在系统打开文件表 (`file_table`) 中找到两个空闲的 `struct file` 条目。
    *   在当前进程的文件描述符表中找到两个空闲槽位。
    *   将这两个文件描述符分别与两个 `struct file` 条目关联，并将这两个 `struct file` 条目都指向同一个管道 inode。
    *   一个 `struct file` 条目被设置为读模式 (`f_mode = 1`)，另一个被设置为写模式 (`f_mode = 2`)。
    *   将这两个文件描述符（读端和写端）返回给用户程序。
2.  **从管道读取数据 (`read_pipe`)**:
    *   处理管道为空或数据不足的情况，如果需要则使读取进程睡眠。
    *   从管道的环形缓冲区中读取数据到用户提供的缓冲区。
    *   更新管道的尾指针 (`PIPE_TAIL`)。
    *   唤醒可能在等待写入的进程。
3.  **向管道写入数据 (`write_pipe`)**:
    *   处理管道已满或剩余空间不足的情况，如果需要则使写入进程睡眠。
    *   将用户提供的数据写入管道的环形缓冲区。
    *   更新管道的头指针 (`PIPE_HEAD`)。
    *   唤醒可能在等待读取的进程。
    *   处理没有读者的情况 (发送 `SIGPIPE` 信号)。

## 关键数据结构和宏

### `struct m_inode * inode` (用于管道)

*   管道使用一个内存 inode (`m_inode`) 来表示，但它不对应磁盘上的任何文件。
*   `inode->i_pipe = 1`: 标记此 inode 为管道。
*   `inode->i_size`: **不存储文件大小**，而是存储指向管道数据缓冲区 (一个物理页面) 的**线性地址**。
*   `inode->i_count`: 对于管道 inode，`i_count` 通常为2，表示有一个读者和一个写者。当读者或写者关闭其文件描述符时，`i_count` 会减少。
*   `inode->i_wait`: 等待队列，用于在管道满（写者等待）或管道空（读者等待）时使进程睡眠。

### `PAGE_SIZE`

*   一个物理页面的大小 (通常为4096字节)。管道的环形缓冲区大小即为一个页面。

### 管道头尾指针宏 (在 `<linux/fs.h>` 中定义)

*   `PIPE_HEAD(*inode)`: 获取管道的头指针（下一个可写位置的索引）。实际上是 `inode->i_zone[0]`。
*   `PIPE_TAIL(*inode)`: 获取管道的尾指针（下一个可读位置的索引）。实际上是 `inode->i_zone[1]`。
*   `PIPE_SIZE(*inode)`: 计算管道中当前存在的数据字节数。 `(PIPE_HEAD(*inode) - PIPE_TAIL(*inode)) & (PAGE_SIZE - 1)`。
*   `PIPE_EMPTY(*inode)`: 判断管道是否为空。 `PIPE_HEAD(*inode) == PIPE_TAIL(*inode)`。
*   `PIPE_FULL(*inode)`: 判断管道是否已满。 `PIPE_SIZE(*inode) == (PAGE_SIZE - 1)` (保留一个字节以区分空和满)。

这些宏操作 `inode->i_zone[0]` 和 `inode->i_zone[1]` (通常用于存储数据块指针) 来作为管道的头尾指针，这是一种巧妙的字段复用。

## 主要函数详解

### `int read_pipe(struct m_inode * inode, char * buf, int count)`

*   **功能**: 从 `inode` 代表的管道中读取最多 `count` 字节的数据到用户缓冲区 `buf`。
*   **步骤**:
    1.  **循环读取**: 当 `count > 0` (还需要读取数据) 时：
        a.  **等待数据**: `while (!(size = PIPE_SIZE(*inode)))`
            *   如果管道为空 (`size` 为0)：
                *   `wake_up(&inode->i_wait);`: 唤醒可能在等待写入的进程 (因为现在可能有空间了，尽管这里是读者)。
                *   `if (inode->i_count != 2) return read;`: 检查是否有写者。管道 inode 的 `i_count` 在初始化时为2 (一个读端，一个写端)。如果 `i_count` 不为2 (例如写端已关闭，`i_count` 可能为1)，则表示不会再有数据写入，直接返回已读取的字节数 `read`。
                *   `sleep_on(&inode->i_wait);`: 如果有写者但管道为空，则当前读取进程在 `inode->i_wait` 上睡眠，等待数据被写入。
        b.  **计算可读取字节数 `chars`**:
            *   `chars = PAGE_SIZE - PIPE_TAIL(*inode);`: 计算从尾指针到缓冲区末尾的连续字节数。
            *   `if (chars > count) chars = count;`: 不能超过用户请求的 `count`。
            *   `if (chars > size) chars = size;`: 也不能超过管道中实际有的数据量 `size`。
        c.  **更新计数器**: `count -= chars; read += chars;`
        d.  **记录旧尾指针**: `size = PIPE_TAIL(*inode);` (这里的 `size` 临时存储旧的尾指针，用于复制数据)。
        e.  **更新管道尾指针**: `PIPE_TAIL(*inode) += chars; PIPE_TAIL(*inode) &= (PAGE_SIZE - 1);` (使其在环形缓冲区内回绕)。
        f.  **复制数据到用户空间**:
            *   `while (chars-- > 0) put_fs_byte(((char *)inode->i_size)[size++], buf++);`
            *   从管道缓冲区 (`(char *)inode->i_size`) 的旧尾指针 `size` 处开始，复制 `chars` 个字节到用户缓冲区 `buf`。
    2.  **唤醒写者**: `wake_up(&inode->i_wait);` 读取操作完成后，管道中可能有了更多可用空间，唤醒可能在等待的写者。
    3.  **返回已读字节数**: `return read;`

### `int write_pipe(struct m_inode * inode, char * buf, int count)`

*   **功能**: 将用户缓冲区 `buf` 中的 `count` 字节数据写入 `inode` 代表的管道。
*   **步骤**:
    1.  **循环写入**: 当 `count > 0` (还需要写入数据) 时：
        a.  **等待空间**: `while (!(size = (PAGE_SIZE - 1) - PIPE_SIZE(*inode)))`
            *   `size` 计算管道的剩余可用空间 (总大小 `PAGE_SIZE-1` 减去已用大小 `PIPE_SIZE`)。
            *   如果管道已满 (`size` 为0)：
                *   `wake_up(&inode->i_wait);`: 唤醒可能在等待读取的进程。
                *   `if (inode->i_count != 2) { current->signal |= (1 << (SIGPIPE - 1)); return written ? written : -1; }`: 检查是否有读者。如果 `i_count` 不为2 (例如读端已关闭)，则向当前进程发送 `SIGPIPE` 信号，并返回已写入的字节数 `written` (如果大于0)，否则返回-1。
                *   `sleep_on(&inode->i_wait);`: 如果有读者但管道已满，则当前写入进程在 `inode->i_wait` 上睡眠，等待空间被释放。
        b.  **计算可写入字节数 `chars`**:
            *   `chars = PAGE_SIZE - PIPE_HEAD(*inode);`: 计算从头指针到缓冲区末尾的连续可用空间。
            *   `if (chars > count) chars = count;`: 不能超过用户请求写入的 `count`。
            *   `if (chars > size) chars = size;`: 也不能超过管道实际的剩余可用空间 `size`。
        c.  **更新计数器**: `count -= chars; written += chars;`
        d.  **记录旧头指针**: `size = PIPE_HEAD(*inode);` (这里的 `size` 临时存储旧的头指针，用于复制数据)。
        e.  **更新管道头指针**: `PIPE_HEAD(*inode) += chars; PIPE_HEAD(*inode) &= (PAGE_SIZE - 1);` (使其在环形缓冲区内回绕)。
        f.  **从用户空间复制数据**:
            *   `while (chars-- > 0) ((char *)inode->i_size)[size++] = get_fs_byte(buf++);`
            *   从用户缓冲区 `buf` 读取 `chars` 个字节，写入管道缓冲区 (`(char *)inode->i_size`) 的旧头指针 `size` 处开始的位置。
    2.  **唤醒读者**: `wake_up(&inode->i_wait);` 写入操作完成后，管道中有了新数据，唤醒可能在等待的读者。
    3.  **返回已写字节数**: `return written;`

### `int sys_pipe(unsigned long * fildes)`

*   **功能**: 实现 `pipe()` 系统调用。在用户提供的 `fildes` 数组 (长度为2) 中返回两个文件描述符，`fildes[0]` 为读端，`fildes[1]` 为写端。
*   **步骤**:
    1.  **分配两个 `struct file` 条目**:
        *   遍历系统文件表 `file_table`，找到两个未被使用 (`f_count == 0`) 的条目，存入 `f[0]` 和 `f[1]`，并将其 `f_count` 置1。
        *   如果找不到足够的条目，则回滚已分配的条目并返回-1。
    2.  **分配两个文件描述符**:
        *   遍历当前进程的文件描述符表 `current->filp`，找到两个空闲槽位，将其索引存入 `fd[0]` 和 `fd[1]`。
        *   将 `current->filp[fd[0]]` 指向 `f[0]`，`current->filp[fd[1]]` 指向 `f[1]`。
        *   如果找不到足够的描述符，则回滚已分配的描述符和 `struct file` 条目，并返回-1。
    3.  **获取管道 inode**:
        *   `if (!(inode = get_pipe_inode()))`: 调用 `get_pipe_inode()` (在 `fs/inode.c` 中实现) 获取一个用于管道的 inode。`get_pipe_inode()` 内部会分配一个内存页作为管道缓冲区，并将页面地址存入 `inode->i_size`。
        *   如果获取失败，则回滚已分配的描述符和 `struct file` 条目，并返回-1。
    4.  **初始化 `struct file` 条目**:
        *   `f[0]->f_inode = f[1]->f_inode = inode;`: 两个文件表条目都指向同一个管道 inode。
        *   `f[0]->f_pos = f[1]->f_pos = 0;`: 文件位置指针设为0 (对于管道意义不大)。
        *   `f[0]->f_mode = 1; /* read */`: `f[0]` (对应 `fd[0]`) 设置为读模式。
        *   `f[1]->f_mode = 2; /* write */`: `f[1]` (对应 `fd[1]`) 设置为写模式。
    5.  **返回文件描述符给用户**:
        *   `put_fs_long(fd[0], 0 + fildes);`: 将读端文件描述符 `fd[0]` 写入用户空间 `fildes[0]`。
        *   `put_fs_long(fd[1], 1 + fildes);`: 将写端文件描述符 `fd[1]` 写入用户空间 `fildes[1]`。
    6.  返回0表示成功。

## 算法和机制

*   **环形缓冲区**: 管道的核心数据结构是一个固定大小 (一页) 的环形缓冲区。通过头指针 (`PIPE_HEAD`) 和尾指针 (`PIPE_TAIL`) 来管理数据的写入和读取。当指针到达缓冲区末尾时，会回绕到开头。
*   **生产者-消费者模型**: 管道的读写操作是典型的生产者-消费者问题。
    *   写者 (生产者) 向管道中放入数据，如果管道满则等待。
    *   读者 (消费者) 从管道中取出数据，如果管道空则等待。
*   **睡眠与唤醒**: 使用 `sleep_on()` 和 `wake_up()` 机制，配合 inode 上的等待队列 `inode->i_wait`，来实现进程在特定条件下的等待和被唤醒。
    *   读者在管道空时睡眠，被写者写入数据后唤醒。
    *   写者在管道满时睡眠，被读者读取数据后唤醒。
*   **EOF 和 SIGPIPE 处理**:
    *   当写端关闭后，读者尝试读取空管道时，会读到文件尾 (返回0字节)。
    *   当读端关闭后，写者尝试写入管道时，会收到 `SIGPIPE` 信号，写入失败。这是通过检查管道 inode 的引用计数 `inode->i_count` 来判断对方是否存在。

## 总结

`pipe.c` 通过简洁的代码实现了 Linux 0.11 中重要的进程间通信机制——管道。它利用了内存页作为环形缓冲区，并结合 inode 和文件表结构，将管道的操作融入了文件系统的框架之中。睡眠/唤醒机制确保了读写进程在管道满或空时的同步。虽然这里的管道实现是基于内存的，不支持命名管道 (FIFO)，但它为shell中的命令流水线等功能提供了基础。Okay, I have created `docs/fs/pipe.md` with the detailed Chinese explanation for `fs/pipe.c`.

Next, I will process `fs/read_write.c`.
1. Create `docs/fs/read_write.md`.
2. Read `fs/read_write.c`.
3. Write the explanation into `docs/fs/read_write.md`.

First, I'll read `fs/read_write.c`.
