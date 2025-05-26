# stat.c 文件详解

`stat.c` 文件是 Linux 0.11 内核中实现获取文件状态信息功能的核心部分。它主要包含 `sys_stat()` 和 `sys_fstat()` 这两个系统调用的实现。这两个系统调用允许用户程序查询一个文件或一个已打开文件描述符的详细元数据，如文件类型、权限、大小、所有者、时间戳等。这些信息被填充到用户提供的 `struct stat` 结构中。

## 核心功能

1.  **`cp_stat(struct m_inode * inode, struct stat * statbuf)`**: 这是一个内部辅助函数，负责将内存 inode (`m_inode`) 中的元数据复制到用户空间提供的 `struct stat` 缓冲区 (`statbuf`)。
2.  **`sys_stat(char * filename, struct stat * statbuf)`**: 实现 `stat()` 系统调用。根据提供的路径名 `filename` 查找对应的文件，并将其状态信息填充到 `statbuf`。
3.  **`sys_fstat(unsigned int fd, struct stat * statbuf)`**: 实现 `fstat()` 系统调用。根据提供的已打开文件描述符 `fd`，获取其对应的文件状态信息，并填充到 `statbuf`。

## 关键数据结构和宏

### `struct stat` (在 `<sys/stat.h>` 中定义)

这是一个标准的用户空间结构体，用于接收文件的状态信息。其主要成员包括：

*   `st_dev`: 文件所在的设备号。
*   `st_ino`: Inode 号。
*   `st_mode`: 文件类型和权限模式。
*   `st_nlink`: 硬链接数量。
*   `st_uid`: 文件所有者的用户 ID。
*   `st_gid`: 文件所有者的组 ID。
*   `st_rdev`: 如果文件是设备文件，则此字段存储实际的设备号 (主设备号和次设备号的组合)。对于其他文件类型，此字段通常为0。
*   `st_size`: 文件大小 (以字节为单位)。
*   `st_atime`: 最后访问时间 (access time)。
*   `st_mtime`: 最后修改时间 (modification time)。
*   `st_ctime`: Inode 状态最后改变时间 (change time)。

### `struct m_inode` (内存中的 inode 结构)

*   这是内核内部表示文件元数据的数据结构。`cp_stat` 函数从这个结构中读取信息。详见 `docs/fs/inode.md`。

### 宏

*   `verify_area(statbuf, sizeof(*statbuf))`: (在 `kernel/system_call.s` 或 `mm/memory.c` 中实现) 检查用户提供的 `statbuf` 缓冲区对于写入操作是否有效和可访问。
*   `put_fs_byte(source_byte, dest_addr_in_userspace)`: 将内核空间的一字节 `source_byte` 写入用户空间的地址 `dest_addr_in_userspace`。

## 主要函数详解

### `static void cp_stat(struct m_inode * inode, struct stat * statbuf)`

*   **功能**: 将内核 `m_inode` 结构中的信息复制到用户空间的 `struct stat` 缓冲区。
*   **步骤**:
    1.  **验证用户缓冲区**: `verify_area(statbuf, sizeof(*statbuf));` 确保用户提供的 `statbuf` 指针是有效的，并且其指向的内存区域对于写入 `struct stat` 大小的数据是可访问的。
    2.  **创建临时 `struct stat` 结构**: `struct stat tmp;` 在内核栈上创建一个临时的 `stat` 结构。
    3.  **填充临时结构**: 将 `inode` 中的各个字段对应地填充到 `tmp` 结构中：
        *   `tmp.st_dev = inode->i_dev;`
        *   `tmp.st_ino = inode->i_num;`
        *   `tmp.st_mode = inode->i_mode;`
        *   `tmp.st_nlink = inode->i_nlinks;`
        *   `tmp.st_uid = inode->i_uid;`
        *   `tmp.st_gid = inode->i_gid;`
        *   `tmp.st_rdev = inode->i_zone[0];` (对于设备文件，`i_zone[0]` 存储其实际设备号；对于其他文件，此值可能无意义或为0)。
        *   `tmp.st_size = inode->i_size;`
        *   `tmp.st_atime = inode->i_atime;`
        *   `tmp.st_mtime = inode->i_mtime;`
        *   `tmp.st_ctime = inode->i_ctime;`
    4.  **逐字节复制到用户空间**:
        *   `for (i = 0; i < sizeof(tmp); i++) put_fs_byte(((char *)&tmp)[i], &((char *)statbuf)[i]);`
        *   由于 `statbuf` 指向用户空间，不能直接进行结构体赋值。这里通过循环，逐个字节地将内核栈上的 `tmp` 结构的内容使用 `put_fs_byte` 复制到用户空间的 `statbuf`。

### `int sys_stat(char * filename, struct stat * statbuf)`

*   **功能**: 获取指定路径名 `filename` 的文件状态。
*   **步骤**:
    1.  **路径名解析**: `if (!(inode = namei(filename))) return -ENOENT;`
        *   调用 `namei()` (在 `fs/namei.c` 中实现) 将路径名 `filename` 转换为对应的内存 inode。
        *   如果 `namei()` 返回 `NULL` (例如，文件不存在或路径无效)，则返回 `-ENOENT` (无此文件或目录)。
    2.  **复制状态信息**: `cp_stat(inode, statbuf);` 调用 `cp_stat` 将获取到的 `inode` 信息填充到用户提供的 `statbuf` 中。
    3.  **释放 inode**: `iput(inode);` 释放对 `inode` 的引用 (因为 `namei()` 会增加 inode 的引用计数)。
    4.  **返回成功**: `return 0;`

### `int sys_fstat(unsigned int fd, struct stat * statbuf)`

*   **功能**: 获取已打开文件描述符 `fd` 的文件状态。
*   **步骤**:
    1.  **文件描述符有效性检查**:
        *   `if (fd >= NR_OPEN || !(f = current->filp[fd]) || !(inode = f->f_inode)) return -EBADF;`
        *   检查 `fd` 是否越界。
        *   检查 `current->filp[fd]` (指向 `struct file` 的指针) 是否为空，即 `fd` 是否已打开。
        *   检查 `f->f_inode` (指向 `struct m_inode` 的指针) 是否为空，即打开的文件是否有对应的 inode。
        *   如果任一检查失败，返回 `-EBADF` (坏文件描述符)。
    2.  **复制状态信息**: `cp_stat(inode, statbuf);` 调用 `cp_stat` 将获取到的 `inode` (来自 `f->f_inode`) 信息填充到用户提供的 `statbuf` 中。
    3.  **返回成功**: `return 0;` (注意：这里不需要 `iput(inode)`，因为 `sys_fstat` 只是查询已打开文件的状态，并没有增加 inode 的引用计数。对 inode 的引用由打开文件 `struct file` 持有。)

## 总结

`stat.c` 文件通过 `cp_stat` 辅助函数和两个系统调用接口 `sys_stat` 与 `sys_fstat`，为用户程序提供了查询文件系统对象元数据的标准方法。`sys_stat` 通过路径名定位文件，而 `sys_fstat` 则通过已打开的文件描述符操作。这两个调用都依赖于内核内部的 inode 结构来获取信息，并通过安全的逐字节复制方式将这些信息传递到用户空间。这些功能对于许多标准的命令行工具 (如 `ls -l`) 和应用程序检查文件属性至关重要。Okay, I have created `docs/fs/stat.md` with the detailed Chinese explanation for `fs/stat.c`.

Next, I will process `fs/super.c`.
1. Create `docs/fs/super.md`.
2. Read `fs/super.c`.
3. Write the explanation into `docs/fs/super.md`.

First, I'll read `fs/super.c`.
