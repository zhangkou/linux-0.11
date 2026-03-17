# 第13章：文件路径解析 fs/namei.c

> 源码文件：`fs/namei.c`（最大的文件系统文件，17KB）
> 核心概念：路径解析、目录遍历、文件创建/删除、挂载点处理

---

## 13.1 概述

namei.c 是文件系统中最复杂的模块，负责将路径字符串（如 `/usr/bin/sh`）解析为 inode。

**主要功能：**
- 路径名解析（`namei()`、`dir_namei()`）
- 文件/目录创建（`sys_mknod()`、`sys_mkdir()`）
- 文件/目录删除（`sys_unlink()`、`sys_rmdir()`）
- 硬链接（`sys_link()`）
- 挂载点穿越处理

---

## 13.2 路径解析的核心：get_dir()

```c
// fs/namei.c（简化版逻辑）
// 将路径名解析为最终目录的inode

static struct m_inode *get_dir(const char *pathname)
{
    char c;
    const char *thisname;
    struct m_inode *inode;
    struct buffer_head *bh;
    int namelen, inr, idev;
    struct dir_entry *de;

    // 确定起始目录：
    // 绝对路径（'/'开头）从根目录开始
    // 相对路径从当前工作目录开始
    if (!current->root || !current->root->i_count)
        panic("No root inode");
    if (!(c=get_fs_byte(pathname))) {
        current->root->i_count++;
        return current->root;
    }
    if (c == '/') {
        inode = current->root;  // 绝对路径：从根目录
        pathname++;
    } else
        inode = current->pwd;   // 相对路径：从当前目录
    inode->i_count++;

    // 逐级解析路径分量
    while (1) {
        thisname = pathname;
        // 找下一个 '/' 或字符串结尾
        for (namelen=0; (c=get_fs_byte(pathname++)) && c!='/'; namelen++)
            ;
        if (!c) return inode;  // 已到达路径末尾，返回当前目录inode

        // 在当前目录中查找该分量
        if (!(bh = find_entry(&inode, thisname, namelen, &de))) {
            iput(inode);
            return NULL;
        }
        inr = de->inode;     // 找到的子目录的inode号
        idev = inode->i_dev;
        brelse(bh);
        iput(inode);

        // 读取子目录的inode
        if (!(inode = iget(idev, inr))) return NULL;

        // 挂载点穿越：如果该目录是另一个文件系统的挂载点
        // 则切换到被挂载文件系统的根目录
        if (inode->i_mount) {
            // ... 挂载点处理
        }
    }
}
```

---

## 13.3 find_entry() —— 在目录中查找文件名

```c
// fs/namei.c 第91~172行（核心目录查找）
static struct buffer_head *find_entry(struct m_inode **dir,
    const char *name, int namelen, struct dir_entry **res_dir)
{
    int entries;
    int block, i;
    struct buffer_head *bh;
    struct dir_entry *de;
    struct super_block *sb;

    // 处理特殊的 ".." 情况（可能穿越伪根目录或挂载点）
    if (namelen==2 && get_fs_byte(name)=='.' && get_fs_byte(name+1)=='.') {
        if ((*dir) == current->root)
            namelen = 1;  // 根目录的 ".." 就是自己 "."
        else if ((*dir)->i_num == ROOT_INO) {
            // 在文件系统根目录的 ".." = 穿越挂载点
            sb = get_super((*dir)->i_dev);
            if (sb->s_imount) {
                iput(*dir);
                (*dir) = sb->s_imount;
                (*dir)->i_count++;
            }
        }
    }

    // 计算目录条目总数
    entries = (*dir)->i_size / (sizeof(struct dir_entry));
    *res_dir = NULL;

    // 遍历目录的所有数据块
    block = 0;
    while (block < 7) {  // 只检查直接块（简化处理）
        if (!(*dir)->i_zone[block]) { block++; continue; }
        if (!(bh = bread((*dir)->i_dev, (*dir)->i_zone[block])))
            { block++; continue; }

        // 在块中逐个检查目录项
        i = 0;
        de = (struct dir_entry *)bh->b_data;
        while (i < entries && i < BLOCK_SIZE/sizeof(struct dir_entry)) {
            if (i*sizeof(struct dir_entry) >= bh->b_size) {
                brelse(bh);
                bh = NULL;
                break;
            }
            if (match(namelen, name, de)) {
                *res_dir = de;
                return bh;  // 找到了！返回含该目录项的缓冲区
            }
            de++;
            i++;
        }
        brelse(bh);
        block++;
    }
    return NULL;  // 未找到
}
```

---

## 13.4 match() —— 文件名比较

```c
// fs/namei.c 第63~78行
static int match(int len, const char *name, struct dir_entry *de)
{
    register int same __asm__("ax");

    if (!de || !de->inode || len > NAME_LEN) return 0;
    if (len < NAME_LEN && de->name[len]) return 0;  // 名字长度不匹配

    __asm__(
        "cld\n\t"
        "fs ; repe ; cmpsb\n\t"  // 用FS段比较（name在用户空间）
        "setz %%al"               // ZF=1（全匹配）则AL=1
        : "=a" (same)
        : "0" (0), "S" ((long)name), "D" ((long)de->name), "c" (len)
        : "cx","di","si"
    );
    return same;  // 1=匹配, 0=不匹配
}
```

---

## 13.5 namei() —— 完整路径→inode

```c
// fs/namei.c
struct m_inode *namei(const char *pathname)
{
    const char *basename;
    int inr, namelen;
    struct m_inode *dir;
    struct buffer_head *bh;
    struct dir_entry *de;

    // 1. 解析到最后一个'/'，获取父目录inode和文件名
    if (!(dir = dir_namei(pathname, &namelen, &basename)))
        return NULL;

    // 如果是根路径'/'，直接返回根目录
    if (!namelen) return dir;

    // 2. 在父目录中查找文件名
    bh = find_entry(&dir, basename, namelen, &de);
    if (!bh) {
        iput(dir);
        return NULL;
    }

    inr = de->inode;
    brelse(bh);
    iput(dir);

    // 3. 读取文件的inode
    dir = iget(current->root->i_dev, inr);
    return dir;
}
```

---

## 13.6 open_namei() —— open()系统调用的路径解析

```c
// fs/namei.c
int open_namei(const char *pathname, int flag, int mode,
    struct m_inode **res_inode)
{
    const char *basename;
    int inr, dev, namelen;
    struct m_inode *dir, *inode;
    struct buffer_head *bh;
    struct dir_entry *de;

    // 处理O_TRUNC等标志...

    if (!(dir = dir_namei(pathname, &namelen, &basename)))
        return -ENOENT;

    if (!namelen) {           // 打开目录本身
        if (!(flag & (O_ACCMODE|O_CREAT|O_TRUNC))) {
            *res_inode = dir;
            return 0;
        }
        iput(dir);
        return -EISDIR;
    }

    bh = find_entry(&dir, basename, namelen, &de);

    if (!bh) {
        // 文件不存在
        if (!(flag & O_CREAT)) {
            iput(dir);
            return -ENOENT;
        }
        // O_CREAT：创建新文件
        if (!permission(dir, MAY_WRITE)) {
            iput(dir);
            return -EACCES;
        }
        inode = new_inode(dir->i_dev);
        if (!inode) { iput(dir); return -ENOSPC; }
        inode->i_uid = current->euid;
        inode->i_mode = mode & ~current->umask & 0777;
        inode->i_dirt = 1;

        // 在父目录中添加目录项
        bh = add_entry(dir, basename, namelen, &de);
        if (!bh) {
            inode->i_nlinks--;
            iput(inode);
            iput(dir);
            return -ENOSPC;
        }
        de->inode = inode->i_num;
        bh->b_dirt = 1;
        brelse(bh);
        iput(dir);
        *res_inode = inode;
        return 0;
    }

    // 文件已存在
    inr = de->inode;
    dev = dir->i_dev;
    brelse(bh);
    iput(dir);

    if (flag & O_EXCL) return -EEXIST;  // O_EXCL: 必须新建

    if (!(inode = iget(dev, inr))) return -EACCES;

    // 检查权限...
    // 处理O_TRUNC...

    *res_inode = inode;
    return 0;
}
```

---

## 13.7 sys_mkdir() —— 创建目录

```c
// fs/namei.c
int sys_mkdir(const char *pathname, int mode)
{
    const char *basename;
    int namelen;
    struct m_inode *dir, *inode;
    struct buffer_head *bh, *dir_block;
    struct dir_entry *de;

    if (!(dir = dir_namei(pathname, &namelen, &basename)))
        return -ENOENT;
    if (!namelen) { iput(dir); return -ENOENT; }
    if (!permission(dir, MAY_WRITE)) { iput(dir); return -EPERM; }

    bh = find_entry(&dir, basename, namelen, &de);
    if (bh) {  // 已存在
        brelse(bh); iput(dir); return -EEXIST;
    }

    // 1. 分配新目录的inode
    if (!(inode = new_inode(dir->i_dev))) { iput(dir); return -ENOSPC; }
    inode->i_size = 32;     // 初始含2个目录项（"."和".."）
    inode->i_dirt = 1;
    inode->i_mode = I_FDIR | (mode & 0777 & ~current->umask);
    inode->i_nlinks = 2;    // "."自身和父目录中的目录项

    // 2. 分配目录的第一个数据块
    if (!(inode->i_zone[0] = new_block(inode->i_dev))) {
        inode->i_nlinks = 0; iput(inode); iput(dir); return -ENOSPC;
    }
    inode->i_dirt = 1;
    if (!(dir_block = bread(inode->i_dev, inode->i_zone[0]))) {
        // 错误处理...
    }

    // 3. 写入 "." 和 ".." 目录项
    de = (struct dir_entry *)dir_block->b_data;
    de->inode = inode->i_num;
    strcpy(de->name, ".");
    de++;
    de->inode = dir->i_num;
    strcpy(de->name, "..");
    inode->i_nlinks = 2;
    dir_block->b_dirt = 1;
    brelse(dir_block);
    inode->i_dirt = 1;

    // 4. 在父目录中添加新目录的目录项
    bh = add_entry(dir, basename, namelen, &de);
    de->inode = inode->i_num;
    bh->b_dirt = 1;
    dir->i_nlinks++;
    dir->i_dirt = 1;
    iput(dir);
    iput(inode);
    brelse(bh);
    return 0;
}
```

---

## 13.8 sys_unlink() —— 删除文件

```c
// fs/namei.c
int sys_unlink(const char *name)
{
    const char *basename;
    int namelen;
    struct m_inode *dir, *inode;
    struct buffer_head *bh;
    struct dir_entry *de;

    if (!(dir = dir_namei(name, &namelen, &basename)))
        return -ENOENT;
    if (!namelen) { iput(dir); return -ENOENT; }
    if (!permission(dir, MAY_WRITE)) { iput(dir); return -EPERM; }

    bh = find_entry(&dir, basename, namelen, &de);
    if (!bh) { iput(dir); return -ENOENT; }

    if (!(inode = iget(dir->i_dev, de->inode))) {
        iput(dir); brelse(bh); return -ENOENT;
    }

    if ((dir->i_mode & S_ISVTX) && !suser() &&
        current->euid != inode->i_uid && current->euid != dir->i_uid) {
        iput(dir); iput(inode); brelse(bh); return -EPERM;
    }

    if (S_ISDIR(inode->i_mode)) {
        iput(inode); iput(dir); brelse(bh); return -EPERM;
    }

    if (!inode->i_nlinks) {
        printk("Deleting nonexistent file (%04x:%d), %d\n",
            inode->i_dev, inode->i_num, inode->i_nlinks);
        inode->i_nlinks = 1;
    }

    // ★关键：清除目录项中的inode号
    de->inode = 0;
    bh->b_dirt = 1;
    brelse(bh);

    // 减少硬链接数
    inode->i_nlinks--;
    inode->i_dirt = 1;
    inode->i_ctime = CURRENT_TIME;

    iput(inode);
    iput(dir);
    return 0;
    // 注意：当 i_nlinks 减为0时，iput()中会调用 truncate() 真正删除数据
}
```

---

## 13.9 路径解析中的特殊处理

**挂载点穿越：**
```
假设 /dev/hdb1 挂载在 /mnt

访问 /mnt/file 时：
1. 解析 "/" → 根目录inode
2. 解析 "mnt" → /mnt目录的inode（i_mount=1，标记为挂载点）
3. 检测到挂载点：通过 s_imount 找到被挂载FS的根目录inode
4. 继续在新FS中解析 "file"

访问 /mnt/.. 时（返回父目录）：
1. 在/mnt/下的根目录（ROOT_INO）遇到 ".."
2. 通过 get_super() 找到超级块的 s_imount
3. 返回 /mnt 所在的父目录，而不是被挂载FS的内部上级
```

**安全限制（chroot）：**
```c
// find_entry() 中
if ((*dir) == current->root)
    namelen = 1;  // "/.." 变成 "/."
// 进程无法通过 ".." 逃出自己的根目录（chroot jail）
```

---

## 13.10 总结

| 函数 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `namei(path)` | 路径字符串 | inode* | 将路径解析为inode |
| `dir_namei(path)` | 路径 | 父目录inode | 解析到最后一级目录 |
| `find_entry(dir, name)` | 目录inode, 文件名 | buffer_head | 在目录中查找文件 |
| `add_entry(dir, name)` | 目录inode, 文件名 | buffer_head | 在目录中添加条目 |
| `open_namei(path, flags)` | 路径, 打开标志 | inode* | open()系统调用使用 |
| `sys_mkdir(path)` | 路径 | 错误码 | 创建目录 |
| `sys_unlink(path)` | 路径 | 错误码 | 删除文件（减链接数） |
| `sys_link(old, new)` | 两个路径 | 错误码 | 创建硬链接 |

---

**下一章：[第14章 - 文件读写与程序执行](14_文件读写与exec.md)**
