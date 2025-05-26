# open.c 文件详解

`open.c` 文件是 Linux 0.11 内核中实现文件打开 (`sys_open()`, `sys_creat()`)、关闭 (`sys_close()`) 以及其他一些与文件和文件系统属性相关的系统调用（如 `sys_utime()`, `sys_access()`, `sys_chdir()`, `sys_chroot()`, `sys_chmod()`, `sys_chown()`）的核心部分。这些系统调用是用户程序与文件系统进行交互的基础操作。

`sys_open()` 是最核心的函数之一，它负责将路径名转换为一个文件描述符，该描述符随后可用于 `read()`, `write()`, `lseek()` 等操作。`sys_close()` 则释放文件描述符及其占用的资源。

## 核心功能

1.  **文件打开与创建**:
    *   `sys_open(filename, flag, mode)`: 打开或创建指定路径 `filename` 的文件。
        *   `flag`: 打开标志 (如 `O_RDONLY`, `O_WRONLY`, `O_RDWR`, `O_CREAT`, `O_APPEND`, `O_TRUNC`, `O_EXCL`)。
        *   `mode`: 如果创建文件，则指定文件的权限模式 (会受 `umask` 影响)。
        *   核心逻辑依赖于 `fs/namei.c` 中的 `open_namei()` 函数来解析路径并获取/创建 inode。
        *   成功时，在当前进程的文件描述符表中找到一个空闲槽位，并从系统文件表 (`file_table`) 中分配一个 `struct file` 条目，初始化后将两者关联。
    *   `sys_creat(pathname, mode)`: 创建一个新文件（或截断一个已存在的文件）。这等价于以 `O_CREAT | O_TRUNC` 标志调用 `sys_open()`。
2.  **文件关闭**:
    *   `sys_close(fd)`: 关闭文件描述符 `fd`。
        *   清除进程文件描述符表中的对应条目。
        *   减少系统文件表 (`file_table`) 中对应 `struct file` 条目的引用计数。
        *   如果 `struct file` 条目的引用计数降为0，则调用 `iput()` 释放对应的 inode。
3.  **文件属性修改**:
    *   `sys_utime(filename, times)`: 修改文件的访问时间 (`actime`) 和修改时间 (`modtime`)。如果 `times` 为 `NULL`，则设置为当前时间。
    *   `sys_chmod(filename, mode)`: 修改文件的权限模式。
    *   `sys_chown(filename, uid, gid)`: 修改文件的所有者用户ID (`uid`) 和组ID (`gid`) (仅限超级用户)。
4.  **访问权限检查**:
    *   `sys_access(filename, mode)`: 检查当前进程对指定文件是否拥有某种访问权限 (`mode`，如 `R_OK`, `W_OK`, `X_OK`)。
5.  **目录操作**:
    *   `sys_chdir(filename)`: 改变当前进程的工作目录 (`current->pwd`)。
    *   `sys_chroot(filename)`: 改变当前进程的根目录 (`current->root`) (仅限超级用户)。
6.  **其他 (未实现)**:
    *   `sys_ustat(dev, ubuf)`: 获取文件系统状态 (返回 `-ENOSYS`，表示未实现)。

## 关键数据结构和宏

### `struct file` (`file_table[]`)

*   系统范围的打开文件表，每个条目代表一个打开的文件实例。详见 `docs/fs/file_table.md`。
    *   `f_mode`: 文件 inode 的模式 (类型和权限)。
    *   `f_flags`: 打开文件时指定的标志。
    *   `f_count`: 此 `struct file` 条目的引用计数。
    *   `f_inode`: 指向此文件对应的内存 inode。
    *   `f_pos`: 当前文件的读写偏移。

### `current->filp[fd]`

*   当前进程的文件描述符表，是一个 `struct file` 指针数组。`fd` 是文件描述符。

### `struct m_inode`

*   内存中的 inode 结构，存储文件的元数据。详见 `docs/fs/inode.md`。

### `fcntl.h` 和 `sys/stat.h` 中的宏

*   文件打开标志 (`O_RDONLY`, `O_CREAT`, `O_APPEND` 等)。
*   文件模式/权限位 (`S_IRUSR`, `S_IWGRP` 等)。
*   文件类型宏 (`S_ISCHR`, `S_ISBLK`, `S_ISDIR` 等)。

## 主要函数详解

### `int sys_open(const char * filename, int flag, int mode)`

*   **功能**: 打开或创建文件。
*   **步骤**:
    1.  **应用umask**: `mode &= 0777 & ~current->umask;` (如果创建文件，从请求的 `mode` 中去除 `umask` 禁止的权限位)。
    2.  **查找空闲文件描述符**:
        *   `for(fd=0 ; fd<NR_OPEN ; fd++) if (!current->filp[fd]) break;`
        *   如果 `fd >= NR_OPEN` (没有空闲描述符)，返回 `-EINVAL` (POSIX 应返回 `-EMFILE`)。
    3.  **清除 `close_on_exec` 标志**: `current->close_on_exec &= ~(1 << fd);` 新打开的文件默认在 `exec`时不关闭。
    4.  **查找空闲系统文件表条目**:
        *   `f = 0 + file_table; for (i=0 ; i<NR_FILE ; i++,f++) if (!f->f_count) break;`
        *   如果 `i >= NR_FILE` (系统文件表已满)，返回 `-EINVAL` (POSIX 应返回 `-ENFILE`)。
    5.  **关联描述符和文件表条目**: `(current->filp[fd] = f)->f_count++;` 将进程的 `fd` 指向找到的 `struct file` 条目 `f`，并增加 `f` 的引用计数。
    6.  **调用 `open_namei()`**: `if ((i = open_namei(filename, flag, mode, &inode)) < 0)`
        *   `open_namei()` (在 `fs/namei.c`) 负责路径解析、权限检查、文件创建（如果 `O_CREAT`）、文件截断（如果 `O_TRUNC`）等，并返回目标文件的 inode。
        *   如果 `open_namei` 失败 (`i < 0`)，则回滚：将 `current->filp[fd]` 设回 `NULL`，`f->f_count` 减回0，并返回错误码 `i`。
    7.  **特殊处理TTY设备**:
        *   如果打开的是字符设备 (`S_ISCHR(inode->i_mode)`)：
            *   主设备号为4 (如 `/dev/tty1`)：如果当前进程是会话领导者 (`current->leader`) 且没有控制终端 (`current->tty < 0`)，则将此tty设为当前进程的控制终端，并设置tty的进程组。
            *   主设备号为5 (`/dev/tty`)：如果当前进程没有控制终端 (`current->tty < 0`)，则打开失败，返回 `-EPERM`。
    8.  **特殊处理块设备**: `if (S_ISBLK(inode->i_mode)) check_disk_change(inode->i_zone[0]);` 检查可移动块设备（如软盘）是否已更换。
    9.  **初始化 `struct file` 条目 `f`**:
        *   `f->f_mode = inode->i_mode;` (从 inode 复制模式)
        *   `f->f_flags = flag;` (打开时指定的标志)
        *   `f->f_count = 1;` (此时只有当前 `fd` 引用它)
        *   `f->f_inode = inode;` (指向获取到的 inode)
        *   `f->f_pos = 0;` (初始读写位置为0)
    10. **返回文件描述符**: `return (fd);`

### `int sys_creat(const char * pathname, int mode)`

*   **功能**: 创建文件（如果已存在则截断）。
*   **实现**: 简单调用 `sys_open(pathname, O_CREAT | O_TRUNC, mode);`。`O_WRONLY` 标志会在 `open_namei` 中根据 `O_TRUNC` 暗含添加。

### `int sys_close(unsigned int fd)`

*   **功能**: 关闭文件描述符 `fd`。
*   **步骤**:
    1.  **有效性检查**: `if (fd >= NR_OPEN) return -EINVAL;`
    2.  **清除 `close_on_exec` 标志**: `current->close_on_exec &= ~(1 << fd);` (虽然通常此标志在关闭时意义不大，但代码如此)。
    3.  **获取文件表条目**: `if (!(filp = current->filp[fd])) return -EINVAL;` 如果 `fd` 未打开，返回错误。
    4.  **从进程描述符表中移除**: `current->filp[fd] = NULL;`
    5.  **引用计数检查**: `if (filp->f_count == 0) panic("Close: file count is 0");` (不应发生)。
    6.  **减少引用计数**: `if (--filp->f_count) return (0);` 如果减少后引用计数仍大于0 (表示还有其他描述符指向此 `struct file` 条目，例如通过 `dup` 或 `fork`)，则直接返回0。
    7.  **释放 inode**: `iput(filp->f_inode);` 当 `filp->f_count` 减到0时，调用 `iput()` 来减少对应 inode 的引用计数。如果 inode 的引用计数也因此降为0，并且其链接数为0，则 `iput()` 内部会负责截断文件（释放数据块）和释放 inode本身（在位图中标记为空闲）。
    8.  返回0表示成功。

### `int sys_utime(char * filename, struct utimbuf * times)`

*   **功能**: 修改文件的访问和修改时间。
*   **步骤**:
    1.  `if (!(inode = namei(filename))) return -ENOENT;`: 获取文件 inode。
    2.  `if (times)`: 如果用户提供了 `utimbuf` 结构：
        *   `actime = get_fs_long((unsigned long *) &times->actime);` 从用户空间读取访问时间。
        *   `modtime = get_fs_long((unsigned long *) &times->modtime);` 从用户空间读取修改时间。
    3.  `else actime = modtime = CURRENT_TIME;`: 如果 `times` 为 `NULL`，则都设为当前时间。
    4.  `inode->i_atime = actime; inode->i_mtime = modtime;`: 更新 inode 中的时间戳。
    5.  `inode->i_dirt = 1;`: 标记 inode 为脏，以便写回磁盘。
    6.  `iput(inode);`: 释放 inode。
    7.  返回0。

### `int sys_access(const char * filename, int mode)`

*   **功能**: 检查对文件的访问权限。`mode` 是 `R_OK`, `W_OK`, `X_OK` 的组合。
*   **步骤**:
    1.  `mode &= 0007;`: 只取 `mode` 的低3位 (对应 rwx)。
    2.  `if (!(inode = namei(filename))) return -EACCES;`: 获取文件 inode。如果文件不存在，也返回 `-EACCES` (POSIX 标准是 `-ENOENT`)。
    3.  `i_mode = res = inode->i_mode & 0777;`: 获取 inode 的权限位。
    4.  `iput(inode);`: 释放 inode。
    5.  **权限比较 (基于实际 uid/gid)**:
        *   `if (current->uid == inode->i_uid) res >>= 6;`: 如果是文件所有者，取用户权限位。
        *   `else if (current->gid == inode->i_gid) res >>= 3;`: 如果是文件所属组，取组权限位。（注意：这里原文是 `res >>= 6;` 应该是一个笔误，应为 `res >>= 3;` 来获取组权限。但实际0.11代码就是这样写的，这意味着如果不是所有者，则无论是同组还是其他用户，都只看其他用户权限位，这与通常的Unix权限模型不符）。
        *   `if ((res & 0007 & mode) == mode) return 0;`: 如果相应的权限位满足请求的 `mode`，返回0 (允许访问)。
    6.  **超级用户特殊处理**:
        *   `if ((!current->uid) && (!(mode & 1) || (i_mode & 0111))) return 0;`
            *   如果当前用户是 root (`!current->uid`)：
                *   并且请求的不是执行权限 (`!(mode & 1)`)，则允许。
                *   或者请求的是执行权限，且文件本身至少对某类用户有执行权限 (`i_mode & 0111`)，则允许。(Root用户执行文件仍需文件本身有可执行位)。
    7.  返回 `-EACCES` (拒绝访问)。

### `sys_chdir`, `sys_chroot`, `sys_chmod`, `sys_chown`

这些函数的实现相对直接：

*   **`sys_chdir(filename)`**: `namei()` 获取目标目录 inode，检查是否为目录，然后更新 `current->pwd`。
*   **`sys_chroot(filename)`**: 类似 `chdir`，但更新 `current->root` (需要超级用户权限，由 `namei` 内部或其调用的 `permission` 间接处理，但这里没有显式 `suser()` 检查，依赖于对根目录的写操作权限）。
*   **`sys_chmod(filename, mode)`**: `namei()` 获取 inode，检查权限 (当前用户必须是文件所有者或超级用户)，然后更新 `inode->i_mode` 并标记为脏。
*   **`sys_chown(filename, uid, gid)`**: `namei()` 获取 inode，检查是否为超级用户，然后更新 `inode->i_uid`, `inode->i_gid` 并标记为脏。

## 总结

`open.c` 文件是 Linux 0.11 文件系统操作的枢纽。它不仅处理了文件的打开和关闭这两个最基本的操作，还包括了对文件元数据（时间戳、权限、所有者）的修改和访问权限的检查。这些功能共同构成了用户与文件系统交互的核心接口层。通过与 `namei.c` (路径解析和 inode 操作) 和 `file_table.c` (系统打开文件表管理) 的紧密协作，`open.c` 使得用户程序能够以一致和抽象的方式来使用文件系统资源。对TTY设备和可移动块设备的特殊处理也体现了早期内核设计的具体考虑。Okay, I have created `docs/fs/open.md` with the detailed Chinese explanation for `fs/open.c`.

Next, I will process `fs/pipe.c`.
1. Create `docs/fs/pipe.md`.
2. Read `fs/pipe.c`.
3. Write the explanation into `docs/fs/pipe.md`.

First, I'll read `fs/pipe.c`.
