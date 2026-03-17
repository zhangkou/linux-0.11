# 第14章：文件读写与程序执行

> 源码文件：`fs/read_write.c`、`fs/file_dev.c`、`fs/exec.c`
> 核心概念：read/write系统调用分发、a.out格式、请求调页加载程序

---

## 14.1 文件描述符与file结构

```c
// include/linux/fs.h
struct file {
    unsigned short f_mode;   // 访问模式：O_RDONLY/O_WRONLY/O_RDWR
    unsigned short f_flags;  // 打开标志（O_APPEND等）
    unsigned short f_count;  // 引用计数（dup/fork增加）
    struct m_inode *f_inode; // 对应的inode
    off_t f_pos;             // 文件当前位置（读写指针）
};

// 系统全局文件表（所有进程共享）
struct file file_table[NR_FILE];  // NR_FILE=64

// 进程私有的fd→file映射
// task_struct.filp[NR_OPEN] → file_table 中的条目
```

**fd、file、inode三者关系：**
```
进程A                 进程B
filp[3]→ ┐           filp[5]→ ┐
          ├→ file_table[i]      │  (fork后共享同一file，共享f_pos)
          │   f_pos = 100      ←┘
          │   f_inode → m_inode[j]
          │               i_zone[0..8] → 磁盘数据块

filp[4]→ file_table[k]  (独立打开，独立f_pos)
          f_pos = 0
          f_inode → m_inode[j]  (同一inode，不同file)
```

---

## 14.2 sys_read() —— 读取系统调用

```c
// fs/read_write.c 第55~81行
int sys_read(unsigned int fd, char *buf, int count)
{
    struct file *file;
    struct m_inode *inode;

    // 参数检查
    if (fd >= NR_OPEN || count<0 || !(file=current->filp[fd]))
        return -EINVAL;
    if (!count) return 0;
    verify_area(buf, count);  // 验证用户缓冲区可写

    inode = file->f_inode;

    // 根据文件类型分发到不同的读函数
    if (inode->i_pipe)
        return (file->f_mode & 1) ? read_pipe(inode, buf, count) : -EIO;

    if (S_ISCHR(inode->i_mode))  // 字符设备
        return rw_char(READ, inode->i_zone[0], buf, count, &file->f_pos);
        // i_zone[0] 存放的是字符设备号（对字符设备文件的特殊用法）

    if (S_ISBLK(inode->i_mode))  // 块设备
        return block_read(inode->i_zone[0], &file->f_pos, buf, count);

    if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode)) {  // 目录/普通文件
        if (count + file->f_pos > inode->i_size)
            count = inode->i_size - file->f_pos;  // 不超过文件末尾
        if (count <= 0) return 0;
        return file_read(inode, file, buf, count);
    }

    return -EINVAL;
}
```

**sys_write() 类似，分发逻辑相同，调用 file_write() 等。**

---

## 14.3 file_read() / file_write() —— 普通文件读写

```c
// fs/file_dev.c
int file_read(struct m_inode *inode, struct file *filp,
    char *buf, int count)
{
    int left, chars, nr;
    struct buffer_head *bh;

    if ((left=count) <= 0)
        return 0;

    while (left) {
        // 计算当前位置对应的块号和块内偏移
        if (nr = bmap(inode, (filp->f_pos)/BLOCK_SIZE)) {
            if (!(bh=bread(inode->i_dev, nr)))
                break;
        } else
            bh = NULL;

        nr = filp->f_pos % BLOCK_SIZE;    // 块内偏移
        chars = MIN(BLOCK_SIZE-nr, left); // 本次可读字节数
        filp->f_pos += chars;
        left -= chars;

        if (bh) {
            char *p = nr + bh->b_data;   // 指向块内起始位置
            while (chars-- > 0)
                put_fs_byte(*(p++), buf++);  // 逐字节复制到用户空间
            brelse(bh);
        } else {
            while (chars-- > 0)
                put_fs_byte(0, buf++);   // 空洞文件：读出0
        }
    }
    inode->i_atime = CURRENT_TIME;
    return (count - left) ? (count - left) : -ERROR;
}
```

**bmap() —— 文件块号→磁盘块号的映射：**
```c
// fs/inode.c
int bmap(struct m_inode *inode, int block)
{
    if (block < 7)
        return inode->i_zone[block];  // 直接块

    block -= 7;
    if (block < 512) {   // 一级间接块
        // 读入间接块，从中取出真正的块号
        bh = bread(inode->i_dev, inode->i_zone[7]);
        return ((unsigned short *)bh->b_data)[block];
    }

    block -= 512;        // 二级间接块
    bh = bread(inode->i_dev, inode->i_zone[8]);
    i = ((unsigned short *)bh->b_data)[block>>9];
    bh2 = bread(inode->i_dev, i);
    return ((unsigned short *)bh2->b_data)[block & 511];
}
```

---

## 14.4 sys_lseek() —— 文件定位

```c
// fs/read_write.c 第25~53行
int sys_lseek(unsigned int fd, off_t offset, int origin)
{
    struct file *file;
    int tmp;

    if (fd >= NR_OPEN || !(file=current->filp[fd]) || !(file->f_inode)
        || !IS_SEEKABLE(MAJOR(file->f_inode->i_dev)))
        return -EBADF;
    if (file->f_inode->i_pipe) return -ESPIPE;  // 管道不可seek

    switch (origin) {
        case 0:   // SEEK_SET：从文件头
            if (offset < 0) return -EINVAL;
            file->f_pos = offset;
            break;
        case 1:   // SEEK_CUR：从当前位置
            if (file->f_pos + offset < 0) return -EINVAL;
            file->f_pos += offset;
            break;
        case 2:   // SEEK_END：从文件尾
            if ((tmp = inode->i_size + offset) < 0) return -EINVAL;
            file->f_pos = tmp;
            break;
    }
    return file->f_pos;
}
```

---

## 14.5 a.out 可执行文件格式

Linux 0.11 只支持 **a.out（Assembler Output）** 格式：

```c
// include/a.out.h
struct exec {
    unsigned long a_magic;    // 魔数: 0x04100301 (ZMAGIC)
    unsigned a_text;          // 代码段大小（字节）
    unsigned a_data;          // 数据段大小（字节）
    unsigned a_bss;           // BSS段大小（未初始化数据）
    unsigned a_syms;          // 符号表大小
    unsigned a_entry;         // 程序入口地址
    unsigned a_trsize;        // 代码段重定位信息大小
    unsigned a_drsize;        // 数据段重定位信息大小
};

// a.out 文件布局:
// [exec头部 32字节] [代码段] [数据段] [BSS段(不占文件空间)] [符号表] ...
// ZMAGIC格式: 代码段从第1页(1024字节)处开始，前1024字节是头部+填充
```

---

## 14.6 do_execve() —— 程序执行（exec系统调用）

这是 exec 系列系统调用的核心实现：

```c
// fs/exec.c（简化版）
int do_execve(unsigned long *eip, long tmp, char *filename,
    char **argv, char **envp)
{
    struct m_inode *inode;
    struct buffer_head *bh;
    struct exec ex;
    // ...

    // ===== 第1步：打开可执行文件 =====
    if (!(inode=namei(filename))) return -ENOENT;
    // 权限检查（必须有执行权限，且不是目录）

    // ===== 第2步：读取文件头（exec结构） =====
    if (!(bh = bread(inode->i_dev, inode->i_zone[0]))) { ... }
    ex = *((struct exec *)bh->b_data);  // 复制32字节头部
    brelse(bh);

    // ===== 第3步：检查是否是#!脚本 =====
    // 如果文件头两字节是"#!"，则解析解释器路径并递归执行

    // ===== 第4步：验证a.out格式 =====
    if (N_MAGIC(ex) != ZMAGIC || ex.a_trsize || ex.a_drsize ||
        ex.a_text+ex.a_data+ex.a_bss > 0x3000000)
        return -ENOEXEC;

    // ===== 第5步：处理参数和环境变量 =====
    // 将 argv[] 和 envp[] 从用户空间复制到临时页
    p = copy_strings(envc, envp, page, p, 0);
    p = copy_strings(argc, argv, page, p, 0);

    // ===== 第6步：释放旧进程的内存空间 =====
    if (current->executable)
        iput(current->executable);
    current->executable = inode;
    for (i=0; i<32; i++)
        current->sigaction[i].sa_handler = NULL;  // 清除信号处理函数
    for (i=3; i<NR_OPEN; i++)
        if (current->close_on_exec & (1<<i))
            sys_close(i);  // 关闭 close-on-exec 的文件
    free_page_tables(get_base(current->ldt[1]), get_limit(0x0f));
    free_page_tables(get_base(current->ldt[2]), get_limit(0x17));
    // 注意：此后如果失败将无法恢复原进程！（exec是不可逆的）

    // ===== 第7步：设置新的地址空间（LDT） =====
    change_ldt(ex.a_text, page);
    // change_ldt 设置代码段和数据段的大小

    // ===== 第8步：建立参数映射 =====
    p = (unsigned long)create_tables((char *)p, argc, envc);

    // ===== 第9步：设置新的执行起点 =====
    // 修改内核栈上保存的用户态EIP和ESP
    eip[0] = ex.a_entry;     // 程序入口地址（通常是_start函数）
    eip[3] = p;               // 新的用户栈指针（含argc/argv/envp）

    current->end_code = ex.a_text;
    current->end_data = ex.a_data + ex.a_text;
    current->brk = current->end_data + ex.a_bss;

    // ===== 第10步：exec使用请求调页 =====
    // 注意：此时没有读入任何代码！
    // 程序代码由缺页异常（do_no_page）按需从磁盘加载
    // current->executable = inode 告诉缺页处理程序从哪里加载

    return 0;
}
```

---

## 14.7 请求调页执行（Demand Loading）

这是 Linux 0.11 的一个重要优化，Linus 在注释中骄傲地说：
```
"Demand-loading implemented 01.12.91 - no need to read anything but
the header into memory... less than 2 hours work to get demand-loading
completely implemented."
```

**执行过程：**
```
execve("/bin/sh", argv, envp)
    │
    ▼ do_execve():
    │ 只读文件头（32字节）
    │ 设置 current->executable = sh的inode
    │ 设置新的EIP = ex.a_entry
    │ 不加载任何代码！
    │
    ▼ iret 返回用户态（EIP=程序入口）
    │
    │ CPU尝试执行第一条指令
    │ → 缺页异常！（页表中该地址无效）
    ▼
do_no_page():
    │ address 在 start_code ~ end_data 范围内
    │ 计算对应的文件块号：block = 1 + (address-start_code)/BLOCK_SIZE
    │ bread_page() 从磁盘读入4个1KB块（填满4KB页）
    │ put_page() 建立映射
    ▼
程序从被打断的指令继续执行（透明！）
    │
    │ 需要更多代码/数据时，继续触发缺页，继续按需加载
    ▼
程序正常运行
```

---

## 14.8 create_tables() —— 构建用户栈

```c
// fs/exec.c 第46~70行
static unsigned long *create_tables(char *p, int argc, int envc)
{
    unsigned long *argv, *envp;
    unsigned long *sp;

    sp = (unsigned long *)(0xfffffffc & (unsigned long)p);  // 4字节对齐

    // 在栈上分配指针数组空间
    sp -= envc+1;   // envp数组（含NULL结尾）
    envp = sp;
    sp -= argc+1;   // argv数组（含NULL结尾）
    argv = sp;

    // 压入main()的参数（从右到左）
    put_fs_long((unsigned long)envp, --sp); // envp指针
    put_fs_long((unsigned long)argv, --sp); // argv指针
    put_fs_long((unsigned long)argc, --sp); // argc整数

    // 填充argv指针数组
    while (argc-- > 0) {
        put_fs_long((unsigned long)p, argv++);  // argv[i] = 字符串地址
        while (get_fs_byte(p++))                 // 跳过字符串
            ;
    }
    put_fs_long(0, argv);  // argv[argc] = NULL

    // 填充envp指针数组
    while (envc-- > 0) {
        put_fs_long((unsigned long)p, envp++);
        while (get_fs_byte(p++))
            ;
    }
    put_fs_long(0, envp);   // envp[envc] = NULL

    return sp;  // 返回新的栈顶（指向argc）
}
```

**栈布局（main函数被调用前）：**
```
高地址
┌──────────────────┐
│  环境变量字符串   │ "HOME=/\0"...
│  参数字符串       │ "-/bin/sh\0"
├──────────────────┤
│  envp[0] = &env0 │
│  envp[1] = NULL  │
├──────────────────┤
│  argv[0] = &arg0 │
│  argv[1] = NULL  │
├──────────────────┤ ← SP+12 (main的第3个参数=envp)
│  envp             │ ← SP+8  (main的第2个参数=argv)
│  argv             │ ← SP+4  (main的第1个参数=argc)
│  argc = 1        │ ← SP (main的参数，按C调用约定)
└──────────────────┘
低地址
```

---

## 14.9 总结

| 系统调用 | 路径 | 说明 |
|---------|------|------|
| read(fd, buf, n) | `sys_read()` → `file_read()` | 根据inode类型分发 |
| write(fd, buf, n) | `sys_write()` → `file_write()` | 同上 |
| lseek(fd, off, w) | `sys_lseek()` | 修改 f_pos |
| execve(path, argv, envp) | `do_execve()` | 请求调页，只读头部 |

**execve的请求调页设计精髓：**
```
旧方式：将整个程序加载到内存才能执行（浪费内存，启动慢）
新方式：只读文件头，程序代码按需从磁盘加载
好处：
  1. 启动极快（不需要加载整个程序）
  2. 多个进程运行同一程序可共享代码页（try_to_share）
  3. 程序不需要预先知道自己要多少内存
```

---

**下一章：[第15章 - 块设备驱动](15_块设备驱动.md)**
