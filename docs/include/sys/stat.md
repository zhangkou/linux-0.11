# sys/stat.h 文件详解

`include/sys/stat.h` 文件是 Linux 0.11 内核以及遵循 POSIX 标准的系统中用于定义文件属性（元数据）和相关操作的头文件。它主要定义了 `struct stat` 结构体，该结构体用于存储通过 `stat()`, `fstat()` (以及后来的 `lstat()`) 系统调用获取到的文件详细信息。此外，该文件还定义了一系列宏，用于解析 `struct stat` 中的 `st_mode` 字段（文件模式），以判断文件类型和访问权限。

## 核心功能

1.  **定义 `struct stat` 结构体**: 用于存储文件的详细属性。
2.  **定义文件类型宏**: 用于从 `st_mode` 字段中提取文件类型信息 (如常规文件、目录、字符设备、块设备、FIFO)。
3.  **定义文件权限位宏**: 用于表示和检查文件的用户、组和其他用户的读、写、执行权限，以及特殊权限位 (SUID, SGID, Sticky Bit)。
4.  **定义文件类型检查宏**: 如 `S_ISREG()`, `S_ISDIR()` 等，方便判断文件类型。
5.  **声明与文件状态和模式相关的函数原型**: 如 `stat()`, `fstat()`, `chmod()`, `mkdir()`, `mkfifo()`, `umask()`。

## 数据结构详解

### `struct stat`

这是核心的数据结构，用于保存文件的各种元数据。

```c
#include <sys/types.h> // 包含了 dev_t, ino_t, mode_t, nlink_t, uid_t, gid_t, off_t, time_t 的定义

struct stat {
	dev_t	st_dev;		/* 文件所在设备的设备号 */
	ino_t	st_ino;		/* Inode 号 */
	umode_t	st_mode;	/* 文件类型和权限模式 (unsigned short in 0.11) */
	nlink_t	st_nlink;	/* 硬链接数量 (unsigned short in 0.11) */
	uid_t	st_uid;		/* 文件所有者的用户 ID (unsigned short in 0.11) */
	gid_t	st_gid;		/* 文件所有者的组 ID (unsigned char in 0.11) */
	dev_t	st_rdev;	/* 如果文件是设备文件，则此字段存储实际的设备号 */
	off_t	st_size;	/* 文件大小 (以字节为单位) */
	time_t	st_atime;	/* 最后访问时间 (access time) */
	time_t	st_mtime;	/* 最后修改时间 (modification time) */
	time_t	st_ctime;	/* Inode 状态最后改变时间 (change time) */
};
```
*   `st_dev`: 包含该文件的文件系统所在的设备的标识符。
*   `st_ino`: 文件的 inode 编号，在同一文件系统中是唯一的。`st_dev` 和 `st_ino` 组合起来可以唯一标识一个文件。
*   `st_mode`: 这是一个非常重要的字段，包含了文件的类型和访问权限。其具体位定义在下面会详细说明。Linux 0.11 中 `umode_t` 是 `unsigned short`。
*   `st_nlink`: 指向此 inode 的硬链接的数量。当此值为0时，文件占用的数据块才会被释放。
*   `st_uid`: 文件所有者的用户ID。
*   `st_gid`: 文件所属的组ID。
*   `st_rdev`: 如果此文件是一个设备文件（字符设备或块设备），则 `st_rdev` 包含了该设备的实际主设备号和次设备号。对于其他类型文件，此字段通常无意义。
*   `st_size`: 对于常规文件，表示文件的字节大小。对于目录，大小取决于实现（通常是目录项总大小）。对于符号链接，是链接所指向路径名的长度。
*   `st_atime`: 文件的最后访问时间。当文件被读取时更新。
*   `st_mtime`: 文件的最后修改时间。当文件内容被写入时更新。
*   `st_ctime`: 文件的 inode 最后状态改变时间。当 inode 的元数据（如权限、所有者、链接数、内容等）发生改变时更新。

## 宏定义详解 (`st_mode` 字段相关)

`st_mode` 字段是一个位掩码，其中不同的位代表不同的含义。

### 文件类型宏

这些宏与 `include/const.h` 中定义的 `I_xxx` 常量相对应，但通常以 `S_IFxxx` 的形式出现，并且有一个掩码 `S_IFMT` 用于提取文件类型部分。

*   `#define S_IFMT 00170000`: **文件类型掩码 (File type mask)**。`st_mode & S_IFMT` 可以得到纯粹的文件类型值。
    *   `00170000` (八进制) = `0b000111100000000000` (二进制)。这表明文件类型信息存储在 `st_mode` 的特定几位中。

*   `#define S_IFREG 0100000`: **常规文件 (Regular file)**。
*   `#define S_IFBLK 0060000`: **块特殊文件 (Block special file)**，即块设备。
*   `#define S_IFDIR 0040000`: **目录文件 (Directory)**。
*   `#define S_IFCHR 0020000`: **字符特殊文件 (Character special file)**，即字符设备。
*   `#define S_IFIFO 0010000`: **FIFO 或命名管道 (Named pipe)**。

### 文件类型检查宏

这些宏用于方便地检查 `st_mode` 值表示的文件类型。

*   `#define S_ISREG(m) (((m) & S_IFMT) == S_IFREG)`: 判断是否为常规文件。
*   `#define S_ISDIR(m) (((m) & S_IFMT) == S_IFDIR)`: 判断是否为目录。
*   `#define S_ISCHR(m) (((m) & S_IFMT) == S_IFCHR)`: 判断是否为字符设备。
*   `#define S_ISBLK(m) (((m) & S_IFMT) == S_IFBLK)`: 判断是否为块设备。
*   `#define S_ISFIFO(m) (((m) & S_IFMT) == S_IFIFO)`: 判断是否为FIFO。

### 特殊权限位宏

这些位也包含在 `st_mode` 中，位于文件类型和标准权限位之间。

*   `#define S_ISUID 0004000`: **Set-User-ID (SUID) 位**。如果一个可执行文件的此位置位，当它被执行时，进程的有效用户ID将变为文件所有者的用户ID。
*   `#define S_ISGID 0002000`: **Set-Group-ID (SGID) 位**。
    *   对于可执行文件：执行时，进程的有效组ID将变为文件所有者的组ID。
    *   对于目录：在该目录下新创建的文件或子目录将继承该目录的组ID。
*   `#define S_ISVTX 0001000`: **Sticky Bit (粘滞位)**。
    *   对于目录：如果设置了粘滞位，则只有目录的所有者、文件的所有者或超级用户才能删除或重命名该目录下的文件。常用于 `/tmp` 目录。
    *   对于可执行文件 (历史用法)：在早期Unix中，可执行文件的粘滞位表示程序执行结束后其代码段仍保留在交换区，以加快下次加载。现代Linux对此有不同处理。

### 标准文件权限位宏

这9个位定义了文件所有者 (User/Owner)、文件所属组 (Group) 和其他用户 (Others) 的读 (Read)、写 (Write)、执行 (Execute) 权限。

*   **所有者权限 (User)**:
    *   `#define S_IRWXU 00700`: 用户读、写、执行权限掩码 (`rwx`)。
    *   `#define S_IRUSR 00400`: 用户读权限 (`r--`)。
    *   `#define S_IWUSR 00200`: 用户写权限 (`-w-`)。
    *   `#define S_IXUSR 00100`: 用户执行权限 (`--x`)。

*   **组权限 (Group)**:
    *   `#define S_IRWXG 00070`: 组读、写、执行权限掩码。
    *   `#define S_IRGRP 00040`: 组读权限。
    *   `#define S_IWGRP 00020`: 组写权限。
    *   `#define S_IXGRP 00010`: 组执行权限。

*   **其他用户权限 (Others)**:
    *   `#define S_IRWXO 00007`: 其他用户读、写、执行权限掩码。
    *   `#define S_IROTH 00004`: 其他用户读权限。
    *   `#define S_IWOTH 00002`: 其他用户写权限。
    *   `#define S_IXOTH 00001`: 其他用户执行权限。

## 函数原型声明

*   `extern int chmod(const char *_path, mode_t mode);`
    *   **功能**: 修改指定路径 `_path` 的文件的权限模式为 `mode`。
*   `extern int fstat(int fildes, struct stat *stat_buf);`
    *   **功能**: 获取已打开文件描述符 `fildes` 指向的文件的状态信息，并存储在 `stat_buf` 指向的 `struct stat` 结构中。
*   `extern int mkdir(const char *_path, mode_t mode);`
    *   **功能**: 创建一个名为 `_path` 的新目录，其权限模式为 `mode` (会受 `umask` 影响)。
*   `extern int mkfifo(const char *_path, mode_t mode);`
    *   **功能**: 创建一个名为 `_path` 的新FIFO（命名管道），其权限模式为 `mode` (会受 `umask` 影响)。Linux 0.11 内核虽然定义了 `S_IFIFO`，但其文件系统（Minix v1）本身不支持磁盘上的FIFO，此调用可能返回错误或未实现。
*   `extern int stat(const char *filename, struct stat *stat_buf);`
    *   **功能**: 获取指定路径名 `filename` 的文件状态信息，并存储在 `stat_buf` 指向的 `struct stat` 结构中。如果 `filename` 是一个符号链接，`stat()` 会返回链接指向的文件的状态。
*   `extern mode_t umask(mode_t mask);`
    *   **功能**: 设置当前进程的文件模式创建屏蔽码 (file mode creation mask) 为 `mask`，并返回之前的屏蔽码。当创建新文件或目录时，请求的权限模式会先与 `umask` 的反码进行按位与操作 (`mode & ~umask`)。

## 总结

`include/sys/stat.h` 是文件系统操作中不可或缺的一部分，它提供了标准的数据结构 (`struct stat`) 来描述文件的各种属性，并通过一系列宏使得从 `st_mode` 字段中解析文件类型和权限变得简单直观。这些定义和声明的函数是用户程序查询和修改文件元数据、以及进行基于文件类型和权限的逻辑判断的基础。它是实现 POSIX兼容文件系统接口的关键。
