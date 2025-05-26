# read_write.c 文件详解

`read_write.c` 文件是 Linux 0.11 内核中实现文件读 (`sys_read()`)、写 (`sys_write()`) 和定位 (`sys_lseek()`) 这三个核心系统调用的关键部分。这些系统调用是用户程序与文件系统进行数据交互的基础。

该文件本身不包含对特定文件类型（如常规文件、字符设备、块设备、管道）的完整读写逻辑，而是作为一个**分派中心 (dispatcher)**。它首先进行一些通用的检查（如文件描述符有效性、参数合法性），然后根据文件描述符对应的 inode 类型，调用相应文件类型或设备类型的具体读写函数。

## 核心功能

1.  **`sys_lseek(fd, offset, origin)`**: 修改文件描述符 `fd` 对应的打开文件实例的当前读写位置 (`f_pos`)。
    *   `origin`: 定位方式，可以是 `SEEK_SET` (从文件头开始)、`SEEK_CUR` (从当前位置开始) 或 `SEEK_END` (从文件尾开始)。
    *   不支持对管道或不可定位设备进行 `lseek`。
2.  **`sys_read(fd, buf, count)`**: 从文件描述符 `fd` 读取最多 `count` 字节的数据到用户缓冲区 `buf`。
    *   根据文件类型，分派到不同的读取函数：
        *   管道: `read_pipe()` (在 `fs/pipe.c`)
        *   字符设备: `rw_char()` (在 `fs/char_dev.c`)
        *   块设备: `block_read()` (在 `fs/block_dev.c`)
        *   常规文件或目录: `file_read()` (在 `fs/file_dev.c`)
    *   对于常规文件和目录，会检查读取是否超出文件大小。
3.  **`sys_write(fd, buf, count)`**: 将用户缓冲区 `buf` 中的 `count` 字节数据写入文件描述符 `fd`。
    *   根据文件类型，分派到不同的写入函数：
        *   管道: `write_pipe()` (在 `fs/pipe.c`)
        *   字符设备: `rw_char()` (在 `fs/char_dev.c`)
        *   块设备: `block_write()` (在 `fs/block_dev.c`)
        *   常规文件: `file_write()` (在 `fs/file_dev.c`)
    *   不允许对目录进行写操作 (由 `file_write` 内部逻辑或更早的权限检查处理，但这里没有显式阻止对目录调用 `file_write`)。

## 关键数据结构和宏

### `struct file * file`

*   指向当前进程文件描述符表 (`current->filp[fd]`) 中的一个条目，代表一个打开的文件句柄。
    *   `file->f_inode`: 指向该文件对应的内存 inode。
    *   `file->f_pos`: 当前文件的读写指针。`sys_lseek` 直接修改此值。`sys_read` 和 `sys_write` 会间接通过调用特定类型的读写函数来更新它。
    *   `file->f_mode`: 打开文件时指定的访问模式 (主要用于管道，判断是读端还是写端)。

### `struct m_inode * inode`

*   从 `file->f_inode` 获取，代表文件的元数据。
    *   `inode->i_mode`: 文件类型和权限。`sys_read` 和 `sys_write` 根据此值进行分派。
    *   `inode->i_pipe`: 标记是否为管道。
    *   `inode->i_zone[0]`: 对于设备文件，存储设备号。
    *   `inode->i_size`: 文件大小 (用于常规文件和目录的读操作边界检查)。

### 宏

*   `NR_OPEN`: 进程允许打开的最大文件描述符数。
*   `IS_SEEKABLE(MAJOR(dev))`: (在 `<linux/fs.h>` 中定义) 检查主设备号 `MAJOR(dev)` 对应的设备是否支持 `lseek` 操作。通常字符设备（如tty）和管道不支持。
*   `S_ISCHR(mode)`, `S_ISBLK(mode)`, `S_ISDIR(mode)`, `S_ISREG(mode)`: 用于判断 inode 模式表示的文件类型。
*   `verify_area(buf, count)`: (在 `kernel/system_call.s` 或 `mm/memory.c` 中实现) 检查用户提供的缓冲区 `buf` (长度 `count`) 对于写操作是否有效和可访问。`sys_read` 在这里调用它，确保用户提供的读缓冲区是可写的。

## 主要函数详解

### `int sys_lseek(unsigned int fd, off_t offset, int origin)`

*   **功能**: 定位打开文件的读写指针。
*   **步骤**:
    1.  **有效性检查**:
        *   `fd >= NR_OPEN`: 文件描述符是否越界。
        *   `!(file = current->filp[fd])`: 文件是否已打开。
        *   `!(file->f_inode)`: 文件是否有对应的 inode。
        *   `!IS_SEEKABLE(MAJOR(file->f_inode->i_dev))`: 设备是否支持定位 (例如，硬盘支持，tty不支持)。此宏在0.11中定义为只允许主设备号为3（硬盘）的设备进行seek。
        *   如果任一检查失败，返回 `-EBADF`。
    2.  **管道检查**: `if (file->f_inode->i_pipe) return -ESPIPE;`: 不允许对管道进行 `lseek`。
    3.  **根据 `origin` 计算新位置**:
        *   `case 0` (`SEEK_SET`):
            *   `if (offset < 0) return -EINVAL;`: 偏移不能为负。
            *   `file->f_pos = offset;`
        *   `case 1` (`SEEK_CUR`):
            *   `if (file->f_pos + offset < 0) return -EINVAL;`: 新位置不能为负。
            *   `file->f_pos += offset;`
        *   `case 2` (`SEEK_END`):
            *   `if ((tmp = file->f_inode->i_size + offset) < 0) return -EINVAL;`: 新位置不能为负。
            *   `file->f_pos = tmp;`
        *   `default`: 无效的 `origin`，返回 `-EINVAL`。
    4.  **返回新位置**: `return file->f_pos;`

### `int sys_read(unsigned int fd, char * buf, int count)`

*   **功能**: 从文件描述符 `fd` 读取数据。
*   **步骤**:
    1.  **参数有效性检查**:
        *   `fd >= NR_OPEN`: 描述符越界。
        *   `count < 0`: 请求读取字节数为负。
        *   `!(file = current->filp[fd])`: 文件未打开。
        *   任一失败则返回 `-EINVAL`。
    2.  `if (!count) return 0;`: 如果请求读取0字节，直接返回0。
    3.  `verify_area(buf, count);`: 检查用户目标缓冲区 `buf` 对于写入 `count` 字节是否有效。
    4.  `inode = file->f_inode;`: 获取 inode。
    5.  **根据 inode 类型分派**:
        *   `if (inode->i_pipe)`: 如果是管道：
            *   `(file->f_mode & 1)` 检查文件打开模式是否允许读 (模式1代表读)。
            *   如果是读端，调用 `read_pipe(inode, buf, count)`。
            *   否则 (如试图从管道写端读取)，返回 `-EIO`。
        *   `if (S_ISCHR(inode->i_mode))`: 如果是字符设备：
            *   调用 `rw_char(READ, inode->i_zone[0], buf, count, &file->f_pos)`。`inode->i_zone[0]` 存储设备号。
        *   `if (S_ISBLK(inode->i_mode))`: 如果是块设备：
            *   调用 `block_read(inode->i_zone[0], &file->f_pos, buf, count)`。
        *   `if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode))`: 如果是目录或常规文件：
            *   `if (count + file->f_pos > inode->i_size) count = inode->i_size - file->f_pos;`: 调整 `count`，确保读取不超过文件末尾。从当前位置 `file->f_pos` 开始，最多只能读到 `inode->i_size`。
            *   `if (count <= 0) return 0;`: 如果调整后 `count` 为0或负 (例如，`f_pos` 已经在文件尾或之后)，返回0。
            *   调用 `file_read(inode, file, buf, count)`。
    6.  **未知类型**: `printk("(Read)inode->i_mode=%06o\n\r", inode->i_mode); return -EINVAL;` 如果 inode 类型不是以上任何一种，打印错误并返回 `-EINVAL`。

### `int sys_write(unsigned int fd, char * buf, int count)`

*   **功能**: 向文件描述符 `fd` 写入数据。
*   **步骤**:
    1.  **参数有效性检查**: 同 `sys_read`，检查 `fd`, `count`, `file`。
    2.  `if (!count) return 0;`: 如果请求写入0字节，直接返回0。
    3.  `inode = file->f_inode;`: 获取 inode。
    4.  **根据 inode 类型分派**:
        *   `if (inode->i_pipe)`: 如果是管道：
            *   `(file->f_mode & 2)` 检查文件打开模式是否允许写 (模式2代表写)。
            *   如果是写端，调用 `write_pipe(inode, buf, count)`。
            *   否则 (如试图向管道读端写入)，返回 `-EIO`。
        *   `if (S_ISCHR(inode->i_mode))`: 如果是字符设备：
            *   调用 `rw_char(WRITE, inode->i_zone[0], buf, count, &file->f_pos)`。
        *   `if (S_ISBLK(inode->i_mode))`: 如果是块设备：
            *   调用 `block_write(inode->i_zone[0], &file->f_pos, buf, count)`。
        *   `if (S_ISREG(inode->i_mode))`: 如果是常规文件：
            *   调用 `file_write(inode, file, buf, count)`。
    5.  **未知类型或不允许写**: `printk("(Write)inode->i_mode=%06o\n\r", inode->i_mode); return -EINVAL;`
        *   注意：这里没有显式阻止对目录 (`S_ISDIR`) 的写操作分派。但 `file_write` 本身通常只用于常规文件，对目录执行 `file_write` 可能会失败或产生未定义行为，这取决于 `file_write` 内部的逻辑 (例如，它依赖 `create_block`，而目录不应该随意创建数据块)。通常，对目录的修改是通过 `mkdir`, `rmdir`, `link`, `unlink` 等特定系统调用完成的。

## 总结

`read_write.c` 是 Linux 0.11 中实现通用文件 I/O 操作 (读、写、定位) 的核心。它通过检查文件描述符对应的 inode 类型，将具体的 I/O 请求分派给相应类型文件的处理函数（如 `file_read/write` 用于常规文件，`pipe_read/write` 用于管道，`rw_char` 用于字符设备，`block_read/write` 用于块设备）。这种分层和分派的结构使得内核能够以统一的方式处理来自用户空间的 `read()`, `write()`, `lseek()` 系统调用，同时支持多种不同的文件和设备类型。文件中的逻辑清晰地展现了如何从一个文件描述符追溯到其背后的 inode，并根据 inode 的特性来决定后续的操作流程。Okay, I have created `docs/fs/read_write.md` with the detailed Chinese explanation for `fs/read_write.c`.

Next, I will process `fs/stat.c`.
1. Create `docs/fs/stat.md`.
2. Read `fs/stat.c`.
3. Write the explanation into `docs/fs/stat.md`.

First, I'll read `fs/stat.c`.
