# 第05章：内核主函数 main.c

> 源码文件：`init/main.c`
> 运行环境：内核空间（进程0，特权级0）
> 核心任务：初始化所有子系统，创建进程1，系统进入运行态

---

## 5.1 概述

`main()` 是整个 Linux 内核的 C 语言入口点。
从 `head.s` 通过 `ret` 指令跳入，此时：
- CPU 处于保护模式、分页已开启
- 中断已关闭（`cli`）
- 内核正以特权级0（Ring 0）运行

**main() 的两大阶段：**
1. **初始化阶段**：依次初始化内存、中断、设备、调度器等子系统
2. **运行阶段**：切换到用户模式，成为进程0（idle进程），创建进程1（init进程）

---

## 5.2 系统调用宏的妙用

```c
// init/main.c 第7~26行
#define __LIBRARY__     // 告诉 unistd.h 展开系统调用内联函数
#include <unistd.h>

// 定义内联系统调用：无需通过用户空间库，直接在内核中调用
static inline _syscall0(int, fork)   // int fork()
static inline _syscall0(int, pause)  // int pause()
static inline _syscall1(int, setup, void *, BIOS)  // int setup(void *BIOS)
static inline _syscall0(int, sync)   // int sync()
```

**为什么要用内联系统调用？**
注释解释得很清楚：

```c
/*
 * we need this inline - forking from kernel space will result
 * in NO COPY ON WRITE (!!!), until an execve is executed. This
 * is no problem, but for the stack. This is handled by not letting
 * main() use the stack at all after fork(). Thus, no function
 * calls - which means inline code for fork too.
 */
```

翻译：从内核空间 fork 不会触发写时复制（COW），
直到执行 execve 之前，父进程（进程0）和子进程（进程1）共享代码和**栈**！
解决方案：`fork()` 和 `pause()` 使用内联（不用真正的函数调用栈），
这样 main() 在 fork() 之后不使用任何栈操作。

---

## 5.3 硬件信息读取

```c
// init/main.c 第58~60行：从 setup.s 收集的数据中读取系统参数
#define EXT_MEM_K      (*(unsigned short *)0x90002)  // 扩展内存(KB)
#define DRIVE_INFO     (*(struct drive_info *)0x90080) // 硬盘参数
#define ORIG_ROOT_DEV  (*(unsigned short *)0x901FC)  // 根设备号
```

```c
// CMOS 实时时钟读取宏
#define CMOS_READ(addr) ({ \
    outb_p(0x80|addr, 0x70); \  // 向CMOS地址端口写地址(0x80=禁止NMI)
    inb_p(0x71);             \  // 从CMOS数据端口读值
})

#define BCD_TO_BIN(val) ((val)=((val)&15) + ((val)>>4)*10)
// BCD码转二进制: 例如 0x21 → (1) + (2×10) = 21
```

---

## 5.4 time_init() —— 读取系统时间

```c
// init/main.c 第76~96行
static void time_init(void)
{
    struct tm time;

    do {
        time.tm_sec  = CMOS_READ(0);   // 秒  (CMOS寄存器0)
        time.tm_min  = CMOS_READ(2);   // 分  (寄存器2)
        time.tm_hour = CMOS_READ(4);   // 时  (寄存器4)
        time.tm_mday = CMOS_READ(7);   // 日  (寄存器7)
        time.tm_mon  = CMOS_READ(8);   // 月  (寄存器8)
        time.tm_year = CMOS_READ(9);   // 年  (寄存器9, 仅年份后两位)
    } while (time.tm_sec != CMOS_READ(0));
    // 重读一次秒，如果不同则说明发生了秒进位，需要重读所有值

    // BCD码转二进制（CMOS存储的是BCD格式）
    BCD_TO_BIN(time.tm_sec);
    BCD_TO_BIN(time.tm_min);
    BCD_TO_BIN(time.tm_hour);
    BCD_TO_BIN(time.tm_mday);
    BCD_TO_BIN(time.tm_mon);
    BCD_TO_BIN(time.tm_year);
    time.tm_mon--;           // tm_mon 从0开始(0=一月)，CMOS从1开始

    startup_time = kernel_mktime(&time);
    // kernel_mktime 将 struct tm 转换为 Unix 时间戳
    // (从1970-01-01 00:00:00 UTC到现在的秒数)
}
```

**CMOS寄存器地址映射：**
```
0x00 = 秒 (0-59, BCD)
0x02 = 分 (0-59, BCD)
0x04 = 时 (0-23, BCD, 24小时制)
0x06 = 星期 (1-7)
0x07 = 日 (1-31, BCD)
0x08 = 月 (1-12, BCD)
0x09 = 年 (0-99, BCD, 两位数)
```

---

## 5.5 main() 主函数 —— 全面解析

```c
// init/main.c 第104~149行
void main(void)
{
    // ============= 第一阶段：收集参数 =============
    ROOT_DEV  = ORIG_ROOT_DEV;    // 设置根设备（来自setup.s收集的数据）
    drive_info = DRIVE_INFO;      // 复制硬盘参数

    // 计算内存布局
    memory_end = (1<<20) + (EXT_MEM_K<<10);
    // = 1MB + 扩展内存字节数
    // 例如 EXT_MEM_K=7168 (7MB) → memory_end = 8MB

    memory_end &= 0xfffff000;     // 对齐到页边界（4KB）
    if (memory_end > 16*1024*1024)
        memory_end = 16*1024*1024; // 上限16MB

    // 根据总内存大小决定缓冲区大小
    if (memory_end > 12*1024*1024)          // > 12MB
        buffer_memory_end = 4*1024*1024;    // 缓冲区 = 4MB
    else if (memory_end > 6*1024*1024)      // > 6MB
        buffer_memory_end = 2*1024*1024;    // 缓冲区 = 2MB
    else
        buffer_memory_end = 1*1024*1024;    // 缓冲区 = 1MB

    main_memory_start = buffer_memory_end;  // 主内存从缓冲区之后开始

#ifdef RAMDISK
    main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
    // 如果启用了内存盘，再往后推
#endif

    // ============= 第二阶段：初始化各子系统 =============
    mem_init(main_memory_start, memory_end); // 内存管理初始化
    trap_init();      // 安装异常处理程序（除零、缺页等）
    blk_dev_init();   // 块设备请求队列初始化
    chr_dev_init();   // 字符设备初始化（空函数，实际在tty_init中）
    tty_init();       // TTY/控制台/键盘初始化
    time_init();      // 读取CMOS时钟
    sched_init();     // 调度器初始化（设置进程0的TSS/LDT，安装时钟中断）
    buffer_init(buffer_memory_end); // 缓冲区缓存初始化
    hd_init();        // 硬盘驱动初始化
    floppy_init();    // 软盘驱动初始化
    sti();            // 开启中断 ← 到此中断已全部准备好

    // ============= 第三阶段：进入用户模式 =============
    move_to_user_mode();  // 从内核模式(Ring0)切换到用户模式(Ring3)
                          // 此时成为"进程0" (idle进程)

    // ============= 第四阶段：创建进程1 =============
    if (!fork()) {        // fork() 返回 0 说明这是子进程（进程1）
        init();           // 进程1 执行 init()
    }

    // ============= 进程0的永久使命：idle循环 =============
    for(;;) pause();
    // pause() 使进程0睡眠，等待中断或其他进程就绪时被调度
    // 注意：进程0是特殊的，schedule()永远不会让它真正睡眠
    // 它在没有其他进程运行时被选中执行
}
```

---

## 5.6 内存布局的决策逻辑

```c
// 三个关键变量
memory_end        // 物理内存总量
buffer_memory_end // 缓冲区结束地址（= 主内存起始）
main_memory_start // 主内存起始地址（供进程使用）
```

**示意图（8MB内存为例）：**

```
物理地址
0x000000 ┌──────────────────┐
         │ 内核代码/数据     │ (约几百KB)
0x100000 ├──────────────────┤  1MB
         │                  │
         │ 缓冲区            │ 1~4MB
         │ (block cache)    │  ← buffer_memory_end = 2MB (6~12MB内存时)
0x200000 ├──────────────────┤  2MB = main_memory_start
         │                  │
         │ 主内存区          │ 供进程使用
         │ (分页管理)        │
         │                  │
0x800000 └──────────────────┘  8MB = memory_end
```

**缓冲区大小选择的依据：**
- 缓冲区越大，文件系统性能越好（更多数据可以缓存）
- 但太大会压缩进程可用内存
- Linux 0.11 使用简单的阈值策略

---

## 5.7 move_to_user_mode() —— 降权切换

这是一个关键的内核特权操作，定义在 `include/asm/system.h`：

```c
// include/asm/system.h (内联汇编)
#define move_to_user_mode() \
__asm__ ("movl %%esp,%%eax\n\t"  \
    "pushl $0x17\n\t"    \  // SS = 0x17 = 用户数据段选择子
    "pushl %%eax\n\t"    \  // ESP（栈指针，保持不变）
    "pushfl\n\t"         \  // EFLAGS
    "pushl $0x0f\n\t"    \  // CS = 0x0f = 用户代码段选择子
    "pushl $1f\n\t"      \  // EIP (返回地址 = 标号1处)
    "iret\n"             \  // 模拟中断返回，降权到 Ring3
    "1:\tmovl $0x17,%%eax\n\t" \  // 标号1: 设置数据段
    "movw %%ax,%%ds\n\t" \
    "movw %%ax,%%es\n\t" \
    "movw %%ax,%%fs\n\t" \
    "movw %%ax,%%gs"        \
    :::"ax")
```

**段选择子含义：**
```
0x0f = 0b00001111: 索引=1, TI=1(LDT), RPL=3(用户)  → LDT[1] = 用户代码段
0x17 = 0b00010111: 索引=2, TI=1(LDT), RPL=3(用户)  → LDT[2] = 用户数据段
```

**iret 降权原理：**
```
iret 从栈上弹出 EIP、CS、EFLAGS、ESP、SS
当 CS.RPL > 当前特权级时，CPU 执行特权级降低，
同时从栈上额外弹出 ESP 和 SS（切换到用户栈）

构造的假栈帧:
┌─────────┐ ← 高地址
│  SS=0x17 │  用户数据段
│  ESP     │  当前栈指针
│  EFLAGS  │
│  CS=0x0f │  用户代码段（RPL=3）
│  EIP=1f  │  返回到标号1处
└─────────┘ ← 低地址（ESP）

iret 执行后：CPU 从 Ring0 降级到 Ring3
```

---

## 5.8 fork() —— 创建进程1

```c
// init/main.c 第138~148行
move_to_user_mode();    // 此后以用户模式运行（进程0）
if (!fork()) {          // fork() 创建进程1
    init();             // 进程1执行init()
}
// 进程0继续执行到这里
for(;;) pause();
```

**fork() 返回值：**
- 父进程（进程0）：返回进程1的PID（>0，条件为假，不进入if）
- 子进程（进程1）：返回0（条件为真，进入if，执行init()）

**为什么用 `!fork()` 而不是 `fork() == 0`？**
Linus 注意到 fork() 使用内联实现（避免栈操作），`!fork()` 是最简单的写法。

---

## 5.9 init() —— 进程1的工作

```c
// init/main.c 第168~209行
void init(void)
{
    int pid, i;

    // 1. 挂载文件系统
    setup((void *) &drive_info);
    // setup() 系统调用：读取分区表，挂载根文件系统
    // drive_info 包含从 0x90080 读取的硬盘参数

    // 2. 打开控制台（stdin/stdout/stderr）
    (void) open("/dev/tty0", O_RDWR, 0);  // fd=0 (stdin)
    (void) dup(0);                          // fd=1 (stdout)
    (void) dup(0);                          // fd=2 (stderr)

    // 3. 打印内存信息
    printf("%d buffers = %d bytes buffer space\n\r",
        NR_BUFFERS, NR_BUFFERS*BLOCK_SIZE);
    printf("Free mem: %d bytes\n\r",
        memory_end - main_memory_start);

    // 4. 执行 /etc/rc 初始化脚本
    if (!(pid=fork())) {            // 进程2
        close(0);
        if (open("/etc/rc", O_RDONLY, 0))  // 打开rc脚本作为stdin
            _exit(1);               // 如果没有rc文件，退出
        execve("/bin/sh", argv_rc, envp_rc); // 执行shell运行rc脚本
        _exit(2);                   // execve失败则退出
    }
    if (pid > 0)
        while (pid != wait(&i))     // 等待rc脚本执行完毕
            ;

    // 5. 无限循环：不断重启登录shell
    while (1) {
        if ((pid=fork()) < 0) {     // 进程3,4,5...
            printf("Fork failed in init\r\n");
            continue;
        }
        if (!pid) {                  // 子进程
            close(0); close(1); close(2);
            setsid();                // 创建新会话（脱离终端）
            (void) open("/dev/tty0", O_RDWR, 0);  // 重新打开终端
            (void) dup(0);
            (void) dup(0);
            _exit(execve("/bin/sh", argv, envp)); // 执行登录shell
        }
        while (1)                    // 父进程等待shell退出
            if (pid == wait(&i))
                break;
        printf("\n\rchild %d died with code %04x\n\r", pid, i);
        sync();                      // 同步文件系统
        // 循环继续，重新fork一个shell
    }
    _exit(0);
}
```

**进程创建树：**
```
进程0 (idle/task0)  ← main() + for(;;) pause()
    │
    └── fork() → 进程1 (init)
                    │
                    ├── fork() → 进程2 (执行/etc/rc)
                    │
                    └── fork() → 进程3 (登录shell /bin/sh)
                                    │
                                    └── (用户交互)
```

---

## 5.10 初始化顺序的依赖关系

```
mem_init()          ← 必须最先调用，其他模块需要分配内存
    │
    ▼
trap_init()         ← 安装异常处理，防止后续初始化触发异常时崩溃
    │
    ▼
blk_dev_init()      ← 块设备请求队列（需要内存）
chr_dev_init()      ← 字符设备（目前为空）
tty_init()          ← 终端初始化（需要chr_dev结构）
    │
    ▼
time_init()         ← 读CMOS（独立，无依赖）
    │
    ▼
sched_init()        ← 调度器（需要GDT，安装时钟中断IRQ0）
    │
    ▼
buffer_init()       ← 缓冲区（需要内存，需要知道buffer_memory_end）
    │
    ▼
hd_init()           ← 硬盘（需要中断系统准备好）
floppy_init()       ← 软盘（需要中断系统准备好）
    │
    ▼
sti()               ← 开中断（所有中断处理程序都已装好）
    │
    ▼
move_to_user_mode() ← 降权（最后一步，进入进程0）
```

---

## 5.11 关键全局变量

```c
// init/main.c 中的全局变量
static long memory_end = 0;         // 物理内存总量（字节）
static long buffer_memory_end = 0;  // 缓冲区结束地址
static long main_memory_start = 0;  // 进程可用内存起始地址

struct drive_info { char dummy[32]; } drive_info; // 硬盘参数备份

// 来自 kernel/sched.c (全局，所有文件可访问)
extern long volatile jiffies;  // 系统启动后的时钟滴答计数
extern long startup_time;      // 系统启动的Unix时间戳
```

---

## 5.12 总结

| 函数 | 调用位置 | 作用 |
|------|---------|------|
| `mem_init()` | `mm/memory.c` | 初始化物理页分配器 |
| `trap_init()` | `kernel/traps.c` | 安装 CPU 异常处理 |
| `sched_init()` | `kernel/sched.c` | 设置进程0，安装时钟中断 |
| `buffer_init()` | `fs/buffer.c` | 初始化块缓存哈希表 |
| `hd_init()` | `kernel/blk_drv/hd.c` | 注册硬盘中断 |
| `tty_init()` | `kernel/chr_drv/tty_io.c` | 初始化控制台 |
| `move_to_user_mode()` | `asm/system.h` | 降权到 Ring3（进程0诞生） |
| `fork()` | `kernel/fork.c` | 创建进程1（init进程） |
| `init()` | 本文件 | 挂载FS，启动shell |

---

**下一章：[第06章 - 内存管理](06_内存管理_mm.md)**

*深入 mm/memory.c，理解 Linux 0.11 的分页、请求调页和写时复制机制*
