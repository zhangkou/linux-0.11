# namei.c 文件详解

`namei.c` (name to inode) 文件是 Linux 0.11 内核中处理路径名解析和相关文件操作的核心组件。它的主要功能是将用户提供的路径名（如 `/usr/bin/ls` 或 `../file.txt`）转换为对应的内存 inode ( `m_inode` ) 结构，并在此基础上实现如打开文件、创建文件/目录、删除文件/目录、创建链接等系统调用。

`namei` (name-to-inode translation) 是类Unix系统中文件操作的基础，因为它将人类可读的路径名与文件系统内部管理的 inode 连接起来。

## 核心功能

1.  **路径名解析**:
    *   `get_dir(pathname)`: 从路径名中分离出目录部分，并返回该目录的 inode。它处理绝对路径和相对路径，以及 `.` 和 `..` 的解析。
    *   `dir_namei(pathname, &namelen, &name)`: 解析路径名，返回最后一级路径组成部分（文件名或目录名）所在的目录的 inode，以及该组成部分的名称和长度。
    *   `namei(pathname)`: 解析完整路径名，返回路径名最终指向的文件的 inode。
2.  **目录项操作**:
    *   `find_entry(dir, name, namelen, res_dir)`: 在指定目录 `dir` 中查找名为 `name` 的目录项 `res_dir`。
    *   `add_entry(dir, name, namelen, res_dir)`: 在指定目录 `dir` 中添加一个名为 `name` 的新目录项 `res_dir`。
    *   `match(len, name, de)`: 比较给定名称 `name` 和目录项 `de` 中的名称是否匹配。
3.  **权限检查**:
    *   `permission(inode, mask)`: 检查当前进程对指定 inode 是否拥有 `mask` (读/写/执行) 所指示的权限。
4.  **文件和目录操作 (系统调用实现)**:
    *   `open_namei(pathname, flag, mode, res_inode)`: 实现 `sys_open()` 的核心逻辑，包括路径解析、文件创建 (如果 `O_CREAT` 置位)、权限检查和返回目标 inode。
    *   `sys_mknod(filename, mode, dev)`: 创建一个特殊文件 (块设备、字符设备) 或FIFO文件。
    *   `sys_mkdir(pathname, mode)`: 创建一个新目录。
    *   `sys_rmdir(name)`: 删除一个空目录。
    *   `sys_unlink(name)`: 删除一个文件 (硬链接)。
    *   `sys_link(oldname, newname)`: 创建一个到 `oldname` 的硬链接 `newname`。
5.  **辅助功能**:
    *   `empty_dir(inode)`: 检查目录是否为空 (只包含 `.` 和 `..`)。

## 关键数据结构和宏

### `struct m_inode *`

*   指向内存 inode 结构的指针。`namei.c` 中的许多函数都围绕着获取、操作和释放这些 inode 展开。

### `struct dir_entry`

*   在 `<linux/fs.h>` 中定义，代表磁盘上目录文件中的一个条目。
    *   `unsigned short inode`: 该目录项对应的 inode 编号。如果为0，表示该目录项空闲。
    *   `char name[NAME_LEN]`: 文件名或目录名 (最大长度 `NAME_LEN`，通常为14)。

### `struct buffer_head * bh`

*   目录文件本身也是以数据块的形式存储在磁盘上的，通过缓冲区高速缓存进行读写。`bh` 指向包含目录项的缓冲区。

### 宏

*   `NAME_LEN`: 文件名的最大长度。
*   `ACC_MODE(x)`: 从打开标志 `x` (如 `O_RDONLY`, `O_WRONLY`, `O_RDWR`) 中提取访问模式掩码 (MAY_READ, MAY_WRITE)。
*   `MAY_EXEC`, `MAY_WRITE`, `MAY_READ`: 权限掩码，分别对应执行、写、读权限。
*   `NO_TRUNCATE`: (可选宏) 如果定义，则超过 `NAME_LEN` 的文件名会导致错误；否则会被截断。Linux 0.11 默认是截断。
*   `DIR_ENTRIES_PER_BLOCK`: 每个磁盘块能容纳的 `dir_entry` 数量。

## 主要函数详解

### `static int permission(struct m_inode * inode, int mask)`

*   **功能**: 检查当前进程 (`current`) 对 `inode` 是否拥有 `mask` 指定的权限。
*   **步骤**:
    1.  **特殊情况**: 如果 inode 的链接数 `i_nlinks` 为0 (表示文件已被删除，但 inode 尚存)，则不允许任何访问。
    2.  **所有者权限**: 如果当前进程的有效用户ID `current->euid` 与 inode 的用户ID `inode->i_uid` 相同，则使用文件模式 `mode` 的用户权限位 (右移6位)。
    3.  **组权限**: 否则，如果当前进程的有效组ID `current->egid` 与 inode 的组ID `inode->i_gid` 相同，则使用文件模式 `mode` 的组权限位 (右移3位)。
    4.  **其他用户权限**: 否则，使用文件模式 `mode` 的其他用户权限位。
    5.  **权限比较**: `((mode & mask & 0007) == mask)`: 将获取到的权限位 (取低3位，即rwx) 与请求的 `mask` 进行按位与操作。如果结果等于 `mask`，表示拥有所有请求的权限。
    6.  **超级用户**: `|| suser()`: 如果是超级用户，则拥有所有权限。
    7.  返回1表示有权限，0表示无权限。

### `static int match(int len, const char * name, struct dir_entry * de)`

*   **功能**: 比较用户提供的文件名 `name` (长度为 `len`) 与目录项 `de->name` 是否匹配。返回1表示匹配，0表示不匹配。
*   **步骤**:
    1.  **有效性检查**: `!de || !de->inode || len > NAME_LEN`，如果目录项无效、inode号为0 (空目录项) 或请求比较的长度超过 `NAME_LEN`，则认为不匹配。
    2.  **长度精确匹配检查**: `if (len < NAME_LEN && de->name[len]) return 0;`: 如果请求的 `len` 小于 `NAME_LEN`，但目录项中 `de->name[len]` 不是 `\0`，说明目录项中的文件名比 `len` 长，不匹配。
    3.  **内联汇编比较**: 使用 `fs ; repe ; cmpsb` 指令串比较。
        *   `fs`: 设置 `cmpsb` 的源操作数 `DS:SI` 指向用户空间 (通过 `fs` 段寄存器)。
        *   `repe ; cmpsb`: 重复比较 `DS:[SI]` 和 `ES:[DI]` 指向的字节，直到不相等或 `CX` 为0。
        *   `setz %%al`: 如果比较结果完全相同 (ZF标志置位)，则 `al` 置1，否则置0。
        *   `esi` 指向 `name`，`edi` 指向 `de->name`，`ecx` 设为 `len`。

### `static struct buffer_head * find_entry(struct m_inode ** dir, const char * name, int namelen, struct dir_entry ** res_dir)`

*   **功能**: 在目录 `*dir` 中查找名为 `name` (长度 `namelen`) 的目录项，并通过 `*res_dir` 返回该目录项的指针，同时返回包含该目录项的缓冲区头。
*   **步骤**:
    1.  **处理文件名长度**: 根据 `NO_TRUNCATE` 宏决定是截断还是拒绝过长的文件名。
    2.  `entries = (*dir)->i_size / (sizeof (struct dir_entry));`: 计算目录中的总条目数。
    3.  **处理 `..` (父目录)**:
        *   `if ((*dir) == current->root)`: 如果当前目录是根目录，查找 `..` 实际上是查找 `.` (将 `namelen` 改为1)。
        *   `else if ((*dir)->i_num == ROOT_INO)`: 如果当前目录是某个文件系统的根目录 (inode号为 `ROOT_INO`)，查找 `..` 需要跨越挂载点。
            *   `sb = get_super((*dir)->i_dev);`: 获取当前文件系统的超级块。
            *   `if (sb->s_imount)`: 如果该超级块记录了它被挂载到的 inode (`s_imount`，即挂载点的 inode)。
            *   `iput(*dir); (*dir) = sb->s_imount; (*dir)->i_count++;`: 释放当前目录 inode，并将 `*dir` 指向挂载点 inode (即父文件系统的目录)。
    4.  **遍历目录项**:
        *   从目录的第一个数据块 `(*dir)->i_zone[0]` 开始读取。
        *   循环 `i` 从0到 `entries`：
            *   如果当前缓冲区 `bh` 已处理完毕 (`(char *)de >= BLOCK_SIZE + bh->b_data`)，则释放当前缓冲区，通过 `bmap()` 找到下一个包含目录项的逻辑块，并用 `bread()` 读取它。
            *   `if (match(namelen, name, de))`: 如果找到匹配的目录项：
                *   `*res_dir = de; return bh;` 设置结果并返回。
        *   `de++; i++;` 指向下一个目录项。
    5.  如果未找到，释放最后一个缓冲区 (如果持有)，返回 `NULL`。

### `static struct m_inode * get_dir(const char * pathname)`

*   **功能**: 解析路径名 `pathname`，返回路径中倒数第二级（即最深一级目录）的 inode。例如，对于 `/usr/bin/ls`，返回 `/usr/bin` 的 inode。
*   **步骤**:
    1.  **确定起始 inode**:
        *   如果路径以 `/` 开头，起始 inode 是当前进程的根目录 `current->root`。
        *   否则，起始 inode 是当前进程的当前工作目录 `current->pwd`。
        *   空路径名返回 `NULL`。
    2.  增加起始 inode 的引用计数。
    3.  **循环解析路径的每一部分**:
        *   `thisname = pathname;`: 记录当前路径部分的开始。
        *   检查当前 `inode` 是否是目录 (`S_ISDIR`) 以及是否有执行权限 (`permission(inode, MAY_EXEC)`)。如果没有，释放 `inode` 并返回 `NULL`。
        *   `for(namelen=0; (c=get_fs_byte(pathname++)) && (c!='/'); namelen++);`: 读取当前路径部分，直到遇到 `/` 或字符串末尾，计算其长度 `namelen`。
        *   `if (!c) return inode;`: 如果已到达路径末尾 (没有后续的 `/`)，则当前 `inode` 就是目标目录，返回它。
        *   `if (!(bh = find_entry(&inode, thisname, namelen, &de)))`: 在当前 `inode` 目录中查找名为 `thisname` 的下一级目录项。
        *   获取下一级目录的 inode 号 `inr = de->inode` 和设备号 `idev = inode->i_dev`。
        *   `brelse(bh); iput(inode);`: 释放当前目录项所在的缓冲区和当前目录 inode。
        *   `if (!(inode = iget(idev, inr))) return NULL;`: 获取下一级目录的 inode。如果失败，返回 `NULL`。
        *   继续循环处理下一级路径。

### `struct m_inode * namei(const char * pathname)`

*   **功能**: 解析完整路径名 `pathname`，返回其最终指向的文件的 inode。
*   **步骤**:
    1.  `if (!(dir = dir_namei(pathname, &namelen, &basename))) return NULL;`: 调用 `dir_namei` 获取路径的最后一级文件名 `basename` (长度 `namelen`) 及其所在的目录 `dir`。
    2.  `if (!namelen) return dir;`: 如果 `namelen` 为0，表示路径以 `/` 结尾 (如 `/usr/`)，此时 `dir` 就是目标 inode，直接返回。
    3.  `bh = find_entry(&dir, basename, namelen, &de);`: 在目录 `dir` 中查找名为 `basename` 的文件。
    4.  如果未找到，`iput(dir); return NULL;`。
    5.  获取文件的 inode 号 `inr = de->inode` 和设备号 `dev = dir->i_dev`。
    6.  `brelse(bh); iput(dir);`: 释放目录项缓冲区和目录 inode。
    7.  `dir = iget(dev, inr);`: 获取目标文件的 inode。
    8.  `if (dir) { dir->i_atime = CURRENT_TIME; dir->i_dirt = 1; }`: 如果获取成功，更新其访问时间并标记为脏。
    9.  返回目标文件的 inode `dir`。

### `int open_namei(const char * pathname, int flag, int mode, struct m_inode ** res_inode)`

*   **功能**: `sys_open()` 系统调用的核心实现。它解析路径，处理文件创建、截断等标志，并返回目标文件的 inode。
*   **步骤**:
    1.  **处理标志和模式**:
        *   如果 `O_TRUNC` (截断) 标志被设置但没有指定写模式 (`O_ACCMODE` 为0)，则自动添加 `O_WRONLY`。
        *   `mode &= 0777 & ~current->umask; mode |= I_REGULAR;`: 应用 `umask` 到权限位 `mode`，并设置文件类型为常规文件 (`I_REGULAR`)。
    2.  `if (!(dir = dir_namei(pathname, &namelen, &basename))) return -ENOENT;`: 获取最后一级文件名 `basename` 及其所在目录 `dir`。
    3.  **处理路径以 `/` 结尾的情况**: `if (!namelen)`
        *   如果不是创建、截断等特殊操作，且目标是目录，则允许打开目录本身。
        *   否则，返回 `-EISDIR` (是目录)。
    4.  `bh = find_entry(&dir, basename, namelen, &de);`: 在目录 `dir` 中查找文件。
    5.  **文件不存在**: `if (!bh)`
        *   如果未设置 `O_CREAT` 标志，返回 `-ENOENT`。
        *   检查是否有在 `dir` 中创建文件的权限 (`permission(dir, MAY_WRITE)`)。
        *   `inode = new_inode(dir->i_dev);`: 分配一个新的 inode。
        *   设置新 inode 的用户ID和模式。
        *   `bh = add_entry(dir, basename, namelen, &de);`: 在目录 `dir` 中添加新的目录项。
        *   `de->inode = inode->i_num;`: 将新 inode 号写入目录项。
        *   `*res_inode = inode; return 0;`
    6.  **文件已存在**:
        *   获取 inode 号 `inr` 和设备号 `dev`。
        *   `if (flag & O_EXCL) return -EEXIST;`: 如果设置了 `O_EXCL` (与 `O_CREAT` 一同使用，要求文件必须不存在)，则返回 `-EEXIST`。
        *   `if (!(inode = iget(dev, inr))) return -EACCES;`: 获取文件 inode。
        *   **权限检查**:
            *   如果目标是目录且请求了写访问 (`flag & O_ACCMODE`)，返回 `-EPERM` (因为不能以写模式打开目录)。
            *   `!permission(inode, ACC_MODE(flag))`: 检查对文件是否有请求的访问权限。
        *   更新访问时间 `inode->i_atime`。
        *   `if (flag & O_TRUNC) truncate(inode);`: 如果设置了 `O_TRUNC`，则截断文件。
        *   `*res_inode = inode; return 0;`

## 系统调用实现 (简述)

*   **`sys_mknod`**:
    1.  检查是否为超级用户。
    2.  解析路径获取父目录 `dir` 和新节点名 `basename`。
    3.  检查写权限和文件名是否已存在。
    4.  `inode = new_inode(dir->i_dev);` 分配新 inode。
    5.  设置 inode 模式。如果是块设备或字符设备，`inode->i_zone[0]` 存储设备号 `dev`。
    6.  `add_entry()` 在父目录中添加条目。
    7.  释放 inode 和目录。
*   **`sys_mkdir`**:
    1.  类似 `sys_mknod` 的权限和路径检查。
    2.  分配新 inode (`inode`) 和一个数据块 (`inode->i_zone[0]`) 给新目录。
    3.  在该数据块中创建 `.` (指向自身 `inode->i_num`) 和 `..` (指向父目录 `dir->i_num`) 两个目录项。
    4.  设置新目录 inode 的链接数 `i_nlinks = 2`。
    5.  设置 inode 模式为目录。
    6.  `add_entry()` 在父目录中添加新目录的条目。
    7.  增加父目录 `dir` 的链接数 `i_nlinks++` (因为 `..` 指向它)。
*   **`sys_rmdir`**:
    1.  权限和路径检查。获取要删除的目录 `inode` 及其父目录 `dir`。
    2.  多项检查：不能删除 `.`，目录必须为空 (`empty_dir()`)，链接数应为2。
    3.  在父目录 `dir` 中将该目录的条目 `de->inode = 0` (标记为空闲)。
    4.  将目录 `inode` 的链接数 `i_nlinks = 0`。
    5.  减少父目录 `dir` 的链接数 `i_nlinks--`。
    6.  `iput(dir); iput(inode);` (iput 会处理实际的 inode 和块释放)。
*   **`sys_unlink`**:
    1.  权限和路径检查。获取要删除的文件 `inode` 及其父目录 `dir`。
    2.  不能 `unlink` 目录。
    3.  在父目录 `dir` 中将该文件的条目 `de->inode = 0`。
    4.  减少文件 `inode` 的链接数 `i_nlinks--`。
    5.  `iput(inode); iput(dir);` (iput 在 `i_nlinks` 为0时会截断并释放文件)。
*   **`sys_link`**:
    1.  `oldinode = namei(oldname);` 获取源文件的 inode。不能是目录。
    2.  `dir = dir_namei(newname, &namelen, &basename);` 获取新链接名 `basename` 及其所在目录 `dir`。
    3.  检查：新旧文件是否在同一设备 (`dir->i_dev != oldinode->i_dev`)，是否有在 `dir` 中的写权限，新链接名是否已存在。
    4.  `add_entry()` 在 `dir` 中添加新目录项 `de`，使其指向 `oldinode->i_num`。
    5.  增加 `oldinode->i_nlinks++`。
    6.  `iput(oldinode); iput(dir);`。

## 总结

`namei.c` 是 Linux 0.11 文件系统的导航系统和操作执行者。它通过一套精心设计的函数，将用户可见的、层次化的路径名与文件系统内部基于 inode 的扁平结构联系起来。它不仅负责查找文件，还整合了权限检查、文件创建、删除、链接等多种核心文件操作的逻辑。对 `.`、`..`、挂载点和符号链接（尽管0.11中符号链接支持不完整）的正确处理是其复杂性的体现。这个文件是理解文件系统如何工作的关键，展示了从路径到具体数据访问的完整流程。Okay, I have created `docs/fs/namei.md` with the detailed Chinese explanation for `fs/namei.c`.

Next, I will process `fs/open.c`.
1. Create `docs/fs/open.md`.
2. Read `fs/open.c`.
3. Write the explanation into `docs/fs/open.md`.

First, I'll read `fs/open.c`.
