# 第07章：进程调度 kernel/sched.c

> 源码文件：`kernel/sched.c`、`include/linux/sched.h`
> 核心概念：task_struct、调度算法、时钟中断、睡眠/唤醒

---

## 7.1 概述

`sched.c` 是 Linus 自称的 "main kernel file"（主内核文件），包含：
- 进程调度器 `schedule()`
- 进程等待/唤醒原语 `sleep_on` / `wake_up`
- 时钟中断处理 `do_timer()`
- 定时器管理 `add_timer()`
- 调度器初始化 `sched_init()`

---

## 7.2 进程状态

```c
// include/linux/sched.h 第19~23行
#define TASK_RUNNING        0  // 就绪/运行中（在运行队列中）
#define TASK_INTERRUPTIBLE  1  // 可中断睡眠（等待资源，可被信号唤醒）
#define TASK_UNINTERRUPTIBLE 2 // 不可中断睡眠（等待资源，不能被信号打断）
#define TASK_ZOMBIE         3  // 僵尸（已退出，等待父进程收集）
#define TASK_STOPPED        4  // 暂停（收到SIGSTOP）
```

**状态转换图：**
```
                    fork()
                      │
                      ▼
              TASK_RUNNING (0)
             ↗            ↘
   时间片用完       ←←    IO/资源可用
             ↘            ↗
   schedule()  TASK_INTERRUPTIBLE (1)
                      ↑
                      │ 信号到达
               TASK_UNINTERRUPTIBLE (2)
                      │
                      │ exit()
               TASK_ZOMBIE (3)
                      │
                      │ wait()
                   释放
```

---

## 7.3 task_union —— 进程描述符与内核栈的精妙共用

```c
// kernel/sched.c 第53~65行
union task_union {
    struct task_struct task;  // 进程描述符（从低地址开始）
    char stack[PAGE_SIZE];    // 内核栈（4096字节）
};

static union task_union init_task = {INIT_TASK,};
// 进程0的task_struct和内核栈共用同一个4KB页！
```

**内存布局：**
```
一个进程的4KB页：
┌─────────────────────┐ ← 低地址（物理页起始）
│  struct task_struct  │ (约700字节)
│  (进程描述符)         │
├─────────────────────┤
│  (空白区)             │
├─────────────────────┤
│  内核栈               │ 向低地址增长
│  ↓↓↓                │
└─────────────────────┘ ← 高地址（物理页末尾）
                          ← 内核栈初始 ESP = (long)p + PAGE_SIZE
```

**好处：**
- 只需一次 `get_free_page()` 即可同时获得进程描述符和内核栈
- 通过 ESP 快速找到当前进程：`current = (task_struct *)(esp & 0xFFFFF000)`

---

## 7.4 schedule() —— 调度算法详解

```c
// kernel/sched.c 第104~142行
void schedule(void)
{
    int i, next, c;
    struct task_struct **p;

    // ====== 第一步：处理 alarm 和信号唤醒 ======
    for (p = &LAST_TASK; p > &FIRST_TASK; --p)
        if (*p) {
            // 检查 alarm 定时器是否超时
            if ((*p)->alarm && (*p)->alarm < jiffies) {
                (*p)->signal |= (1<<(SIGALRM-1));  // 设置SIGALRM信号
                (*p)->alarm = 0;
            }
            // 有未屏蔽信号 && 处于可中断睡眠 → 唤醒
            if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&
                (*p)->state == TASK_INTERRUPTIBLE)
                (*p)->state = TASK_RUNNING;
        }

    // ====== 第二步：选出时间片最多的就绪进程 ======
    while (1) {
        c = -1;
        next = 0;     // 默认选进程0（idle）
        i = NR_TASKS;
        p = &task[NR_TASKS];

        while (--i) {
            if (!*--p) continue;
            // 找所有就绪进程中 counter 最大的
            if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
                c = (*p)->counter, next = i;
        }

        if (c) break;  // 找到了有时间片的就绪进程

        // 所有就绪进程时间片都耗尽（c==0）：重新计算时间片
        for (p = &LAST_TASK; p > &FIRST_TASK; --p)
            if (*p)
                (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
                // 新时间片 = 旧时间片/2 + 优先级
                // 睡眠进程也参与计算，使长期等待的进程获得更高优先级
    }

    switch_to(next);  // 切换到选中的进程
}
```

**调度策略分析：**
```
counter = 剩余时间片（每次时钟中断减1）
priority = 初始时间片（默认15）

新进程创建时：counter = priority = 15（约0.15秒）

时间片耗尽后重算：
  counter_new = counter_old/2 + priority
  - 一直运行的进程：counter趋向于 0/2 + 15 = 15
  - 长期睡眠的进程：counter可以积累到 0+15=15, 然后7+15=22, 即counter较高
  - IO密集型进程天然比CPU密集型进程有更多时间片积累
```

---

## 7.5 switch_to() —— 任务切换宏

```c
// include/linux/sched.h 第171~184行
#define switch_to(n) {                              \
    struct {long a,b;} __tmp;                       \
    __asm__(                                        \
        "cmpl %%ecx, _current\n\t"  /* 是当前进程？ */ \
        "je 1f\n\t"                 /* 是则不切换 */ \
        "movw %%dx, %1\n\t"         /* TSS选择子写入__tmp.b */ \
        "xchgl %%ecx, _current\n\t" /* 原子交换：更新current指针 */ \
        "ljmp %0\n\t"               /* 远跳到TSS描述符：触发硬件任务切换！ */ \
        "cmpl %%ecx, _last_task_used_math\n\t"      \
        "jne 1f\n\t"                                \
        "clts\n"           /* 清除TS位（数学协处理器切换） */ \
        "1:"                                        \
        :: "m" (*&__tmp.a), "m" (*&__tmp.b),        \
           "d" (_TSS(n)), "c" ((long) task[n])      \
    );                                              \
}
```

**`ljmp %0` 的魔法：**
```
ljmp 到一个 TSS 描述符选择子时，CPU自动执行硬件任务切换：
1. 保存当前所有寄存器到当前进程的 TSS
2. 从目标进程的 TSS 恢复所有寄存器
3. 更新 TR（Task Register）指向新的TSS
4. 切换地址空间（CR3 = 新进程的页目录基址）

这是 x86 硬件级的上下文切换，只需一条指令！
```

---

## 7.6 sleep_on() / wake_up() —— 等待队列

Linux 0.11 使用**栈式等待队列**（wait queue on stack）：

```c
// kernel/sched.c 第151~165行
void sleep_on(struct task_struct **p)
{
    struct task_struct *tmp;

    if (!p) return;
    if (current == &(init_task.task))
        panic("task[0] trying to sleep"); // 进程0不能睡眠

    tmp = *p;       // 保存等待队列头（上一个等待者）
    *p = current;   // 将自己插入队列头
    current->state = TASK_UNINTERRUPTIBLE;
    schedule();     // 让出CPU

    // 被唤醒后继续执行：
    if (tmp)
        tmp->state = 0;  // 同时唤醒队列中的前一个等待者
}
```

**等待队列的隐式链表（用栈实现）：**
```
假设3个进程都在等同一资源 *p：

调用顺序：进程A → 进程B → 进程C

内存状态（栈上的 tmp 变量）：
  进程C的栈: tmp = 进程B  → *p = 进程C
  进程B的栈: tmp = 进程A  → *p = 进程B（被进程C覆盖为进程C前执行此行）
  进程A的栈: tmp = NULL   → *p = 进程A

等待链: *p → 进程C → 进程B → 进程A → NULL

wake_up(p):
  将 *p（进程C）置为RUNNING
  → 进程C被调度，schedule()返回
  → 进程C执行 if(tmp) tmp->state=0 → 唤醒进程B
  → 进程B被调度，执行 if(tmp) tmp->state=0 → 唤醒进程A
  即：LIFO（后进先出）唤醒链
```

```c
void wake_up(struct task_struct **p)
{
    if (p && *p) {
        (**p).state = 0;  // 设为TASK_RUNNING
        *p = NULL;        // 清除队列头
    }
}
```

---

## 7.7 do_timer() —— 时钟中断处理

每10毫秒（100Hz）调用一次：

```c
// kernel/sched.c 第305~336行
void do_timer(long cpl)  // cpl = 当前特权级 (0=内核, 3=用户)
{
    // 1. 处理蜂鸣器（扬声器定时）
    if (beepcount)
        if (!--beepcount)
            sysbeepstop();

    // 2. 记账：统计进程用户态/内核态时间
    if (cpl)
        current->utime++;  // 用户态时间
    else
        current->stime++;  // 内核态（系统）时间

    // 3. 软件定时器：检查是否有到期的 timer_list
    if (next_timer) {
        next_timer->jiffies--;
        while (next_timer && next_timer->jiffies <= 0) {
            void (*fn)(void);
            fn = next_timer->fn;
            next_timer->fn = NULL;
            next_timer = next_timer->next;
            (fn)();  // 调用定时器回调函数
        }
    }

    // 4. 软盘马达定时器
    if (current_DOR & 0xf0)
        do_floppy_timer();

    // 5. 递减时间片，检查是否需要重新调度
    if ((--current->counter) > 0) return;  // 时间片未耗尽，继续运行
    current->counter = 0;
    if (!cpl) return;   // 在内核态不抢占（非可抢占内核）
    schedule();         // 时间片耗尽 + 用户态 → 重新调度
}
```

**重要特性：Linux 0.11 是不可抢占内核（Non-preemptible Kernel）**
- 内核代码执行期间（cpl=0）不会被抢占
- 只有从内核态返回用户态时才检查是否需要调度
- 这大大简化了内核同步问题

---

## 7.8 sched_init() —— 调度器初始化

```c
// kernel/sched.c 第385~412行
void sched_init(void)
{
    int i;
    struct desc_struct *p;

    // 将进程0的 TSS 和 LDT 安装到 GDT
    set_tss_desc(gdt+FIRST_TSS_ENTRY, &(init_task.task.tss));
    set_ldt_desc(gdt+FIRST_LDT_ENTRY, &(init_task.task.ldt));

    // 清除其他进程槽的 GDT 条目
    p = gdt+2+FIRST_TSS_ENTRY;
    for (i=1; i<NR_TASKS; i++) {
        task[i] = NULL;
        p->a = p->b = 0;   // 清除 TSS 描述符
        p++;
        p->a = p->b = 0;   // 清除 LDT 描述符
        p++;
    }

    // 清除 NT 位（嵌套任务标志），避免 iret 引起意外任务切换
    __asm__("pushfl ; andl $0xffffbfff,(%esp) ; popfl");

    ltr(0);   // 加载进程0的 TSS 到 TR 寄存器
    lldt(0);  // 加载进程0的 LDT 到 LDTR 寄存器

    // 配置8253定时器（PIT, Programmable Interval Timer）
    outb_p(0x36, 0x43);         // 通道0, 模式3(方波), 二进制, LSB+MSB
    outb_p(LATCH & 0xff, 0x40); // 计数值低字节
    outb(LATCH >> 8, 0x40);     // 计数值高字节
    // LATCH = 1193180 / 100 = 11932 → 每11932个时钟周期产生一次中断 = 100Hz

    // 安装时钟中断处理程序
    set_intr_gate(0x20, &timer_interrupt);  // IRQ0 → INT 0x20
    outb(inb_p(0x21) & ~0x01, 0x21);       // 解除IRQ0屏蔽

    // 安装系统调用门
    set_system_gate(0x80, &system_call);   // INT 0x80 → system_call
    // set_system_gate: DPL=3，允许用户态调用
}
```

---

## 7.9 软件定时器 add_timer()

```c
// kernel/sched.c 第272~303行
// timer_list 是一个有序链表，按到期时间排列（差值存储）
static struct timer_list {
    long jiffies;        // 相对上一个定时器的延迟
    void (*fn)(void);    // 回调函数
    struct timer_list *next;
} timer_list[TIME_REQUESTS], *next_timer = NULL;

void add_timer(long jiffies, void (*fn)(void))
{
    struct timer_list *p;

    if (!fn) return;
    cli();
    if (jiffies <= 0)
        (fn)();      // 立即执行
    else {
        // 找空闲槽
        for (p = timer_list; p < timer_list + TIME_REQUESTS; p++)
            if (!p->fn) break;
        p->fn = fn;
        p->jiffies = jiffies;
        p->next = next_timer;
        next_timer = p;
        // 按到期时间插入有序链表（差值方式存储）
        while (p->next && p->next->jiffies < p->jiffies) {
            p->jiffies -= p->next->jiffies; // 转换为差值
            // 交换 fn 和 jiffies（原地排序）
            ...
        }
    }
    sti();
}
```

---

## 7.10 总结

| 机制 | 实现 | 特点 |
|------|------|------|
| 调度策略 | 优先级 + 时间片轮转 | IO密集型进程有更高优先级 |
| 任务切换 | x86硬件TSS切换（ljmp） | 一条指令完成上下文保存/恢复 |
| 等待队列 | 栈式隐式链表 | LIFO唤醒，无需显式链表内存 |
| 时钟中断 | 8253 PIT，100Hz | 每10ms触发，`jiffies++` |
| 抢占性 | 不可抢占内核 | 内核态不抢占，简化同步 |

---

**下一章：[第08章 - 进程创建与退出](08_进程创建与退出.md)**
