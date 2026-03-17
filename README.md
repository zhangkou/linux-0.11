# Linux 0.11 内核源码

Linus Torvalds 于 1991 年发布的早期 Linux 内核，约 10,000 行 C + 汇编代码，完整实现了一个可运行的 Unix-like 操作系统内核。

---

## 目录结构

```
linux-0.11/
├── boot/               启动代码（汇编）
│   ├── bootsect.s      引导扇区（512字节，加载内核）
│   ├── setup.s         硬件初始化（收集硬件信息，进入保护模式）
│   └── head.s          32位初始化（建立页表，跳转到 main()）
│
├── init/               内核入口
│   └── main.c          kernel main()，各子系统初始化，创建进程0/1
│
├── kernel/             核心子系统
│   ├── sched.c         进程调度器（schedule, sleep_on, wake_up）
│   ├── fork.c          进程创建（sys_fork, copy_process）
│   ├── exit.c          进程退出（do_exit, sys_waitpid）
│   ├── signal.c        信号处理（do_signal, sys_sigaction）
│   ├── system_call.s   系统调用入口（int 0x80）
│   ├── traps.c         异常/中断处理（除零、缺页等）
│   ├── blk_drv/        块设备驱动
│   │   ├── ll_rw_blk.c 通用块层（请求队列，电梯算法）
│   │   ├── hd.c        硬盘驱动（IDE/ATA，中断驱动IO）
│   │   ├── floppy.c    软盘驱动（DMA传输）
│   │   └── ramdisk.c   内存盘
│   └── chr_drv/        字符设备驱动
│       ├── tty_io.c    TTY 层（行规程，三队列）
│       ├── console.c   VGA 文本控制台（ANSI转义，快速滚屏）
│       ├── keyboard.S  键盘驱动（扫描码转换）
│       ├── serial.c    串口驱动（UART，RS-232）
│       └── rs_io.s     串口中断处理（汇编）
│
├── mm/                 内存管理
│   ├── memory.c        分页管理（mem_map，写时复制，请求调页）
│   └── page.s          缺页异常底层处理
│
├── fs/                 文件系统（Minix FS）
│   ├── buffer.c        缓冲区缓存（bread/bwrite，LRU替换）
│   ├── inode.c         inode 管理
│   ├── super.c         超级块管理
│   ├── namei.c         路径解析（字符串 → inode）
│   ├── bitmap.c        位图分配（inode/数据块）
│   ├── read_write.c    read/write/lseek 系统调用
│   ├── exec.c          execve（请求调页执行，a.out格式）
│   ├── open.c          open/close/creat 系统调用
│   ├── pipe.c          管道
│   └── ...
│
├── include/            头文件
│   ├── linux/          内核内部头文件（sched.h, fs.h, tty.h...）
│   ├── asm/            体系结构相关（io.h, system.h...）
│   └── sys/            POSIX 系统头文件
│
├── lib/                C 运行时库（内核用）
├── tools/              构建工具（build 程序，生成 Image）
├── Makefile            构建脚本
└── notes/              源码学习笔记（见下方）
```

---

## 核心数据结构

| 结构 | 文件 | 作用 |
|------|------|------|
| `task_struct` | `include/linux/sched.h` | 进程描述符（状态、内存、文件、信号等） |
| `tss_struct` | `include/linux/sched.h` | 任务状态段（硬件任务切换） |
| `m_inode` | `include/linux/fs.h` | 内存 inode（文件元数据） |
| `buffer_head` | `include/linux/fs.h` | 缓冲区块（双链表+哈希表） |
| `tty_struct` | `include/linux/tty.h` | TTY 设备（三队列+行规程） |
| `request` | `kernel/blk_drv/blk.h` | 块IO请求（电梯排序队列） |

---

## 关键常量

| 常量 | 值 | 含义 |
|------|----|------|
| `NR_TASKS` | 64 | 最大进程数 |
| `PAGE_SIZE` | 4096 | 页大小（4KB） |
| `BLOCK_SIZE` | 1024 | 文件系统块大小（1KB） |
| `NR_BUFFERS` | 动态 | 缓冲区块数（取决于内存大小） |
| `NR_INODE` | 32 | 内存 inode 缓存数量 |
| `NR_FILE` | 64 | 全局文件表大小 |
| `NR_REQUEST` | 32 | 块IO请求池大小 |
| `HZ` | 100 | 时钟频率（每秒100次调度） |

---

## 启动流程

```
上电 → BIOS → bootsect.s → setup.s → head.s → main()
         │          │           │         │
         │    0x7C00→0x90000  收集硬件  建立IDT/GDT
         │    加载setup+system  开启A20   建立页表（16MB）
         │    切换到0x90200    8259A重映  开启分页
         │                    CR0.PE=1   ret → main()
         │                    进入保护模式
         │
         ▼
    main() 初始化：
        mem_init()      内存管理
        trap_init()     异常/中断
        blk_dev_init()  块设备
        chr_dev_init()  字符设备
        tty_init()      TTY
        sched_init()    调度器
        buffer_init()   缓冲区缓存
        hd_init()       硬盘驱动
        sti()           开启中断
        move_to_user_mode() → 进程0（空闲进程）
            └─ fork() → 进程1（init进程）
                    └─ execve("/bin/sh") → 用户shell
```

---

## 源码学习笔记

详细的章节学习笔记位于 [`notes/`](notes/) 目录：

| 章节 | 笔记文件 | 主题 |
|------|----------|------|
| 01 | [01_概述与准备.md](notes/01_概述与准备.md) | 项目结构、x86基础 |
| 02 | [02_引导程序_bootsect.md](notes/02_引导程序_bootsect.md) | BIOS启动、引导扇区 |
| 03 | [03_硬件初始化_setup.md](notes/03_硬件初始化_setup.md) | 硬件探测、保护模式切换 |
| 04 | [04_保护模式初始化_head.md](notes/04_保护模式初始化_head.md) | IDT/GDT、分页初始化 |
| 05 | [05_内核主函数_main.md](notes/05_内核主函数_main.md) | 子系统初始化、进程0/1 |
| 06 | [06_内存管理_mm.md](notes/06_内存管理_mm.md) | 分页、写时复制、请求调页 |
| 07 | [07_进程调度_sched.md](notes/07_进程调度_sched.md) | 调度器、sleep_on/wake_up |
| 08 | [08_进程创建与退出.md](notes/08_进程创建与退出.md) | fork、exit、wait |
| 09 | [09_系统调用机制.md](notes/09_系统调用机制.md) | int 0x80、系统调用表 |
| 10 | [10_信号处理.md](notes/10_信号处理.md) | 信号投递、处理函数 |
| 11 | [11_文件系统_概览.md](notes/11_文件系统_概览.md) | Minix FS、superblock、inode |
| 12 | [12_缓冲区管理_buffer.md](notes/12_缓冲区管理_buffer.md) | 缓冲区缓存、LRU替换 |
| 13 | [13_文件路径解析_namei.md](notes/13_文件路径解析_namei.md) | 路径解析、目录操作 |
| 14 | [14_文件读写与exec.md](notes/14_文件读写与exec.md) | read/write、execve请求调页 |
| 15 | [15_块设备驱动.md](notes/15_块设备驱动.md) | 请求队列、电梯算法、IDE驱动 |
| 16 | [16_字符设备驱动.md](notes/16_字符设备驱动.md) | TTY、VGA控制台、键盘、串口 |

完整目录及学习路线见 [notes/00_学习目录.md](notes/00_学习目录.md)。
