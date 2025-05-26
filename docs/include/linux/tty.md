# tty.h 文件详解

`include/linux/tty.h` 文件是 Linux 0.11 内核中与 TTY (Teletypewriter，即终端)设备管理和 I/O 操作相关的核心头文件。它定义了 TTY 设备的核心数据结构，如 `struct tty_queue` (TTY 缓冲区队列) 和 `struct tty_struct` (TTY设备本身的状态和配置)，以及用于操作这些结构和处理TTY字符的宏和函数原型。

TTY 子系统是用户与系统交互的主要接口，负责处理来自物理终端（如串口连接的终端、控制台键盘）的输入，并将输出显示到这些终端。

## 核心功能

1.  **定义 TTY 缓冲区队列结构 (`struct tty_queue`)**: 用于存储输入（如键盘输入）和输出（待显示到屏幕）的字符数据。
2.  **定义 TTY 设备结构 (`struct tty_struct`)**: 包含了TTY设备的配置信息（如 `termios` 设置）、进程组ID、流控制状态、读写队列以及实际的底层写函数指针。
3.  **定义 TTY 队列操作宏**: 提供了一系列宏来方便地操作 `tty_queue`，如判断队列空/满、获取/放入字符等。
4.  **定义特殊控制字符宏**: 提供了访问 `termios` 结构中定义的特殊控制字符（如中断、退出、删除等）的宏。
5.  **声明 TTY 相关的全局变量和函数原型**: 如 `tty_table` (TTY设备表)以及TTY初始化、读写、行规范处理等函数。

## 数据结构详解

### `struct tty_queue`

*   **功能**: 实现了一个字符环形缓冲区，用于 TTY 的输入和输出。
*   **成员**:
    *   `unsigned long data`: 在 Linux 0.11 的 `tty_io.c` 中，此字段用于存储等待此队列的进程的任务号（或特定条件下是等待队列的 `task_struct **` 指针，但主要用于 `sleep_if_empty` 和 `sleep_if_full` 的唤醒逻辑）。它不直接存储字符数据。
    *   `unsigned long head`: 缓冲区的头指针（写指针）。新字符写入 `buf[head]`，然后 `head` 增加。
    *   `unsigned long tail`: 缓冲区的尾指针（读指针）。从 `buf[tail]` 读取字符，然后 `tail` 增加。
    *   `struct task_struct * proc_list`: 指向等待此队列（例如，等待输入数据或等待输出空间）的进程链表的头。
    *   `char buf[TTY_BUF_SIZE]`: 实际存储字符数据的环形缓冲区，大小为 `TTY_BUF_SIZE` (1024字节)。

### `struct tty_struct`

*   **功能**: 代表一个 TTY 设备（如一个虚拟控制台或一个串口终端）的所有状态和配置信息。
*   **成员**:
    *   `struct termios termios`: 存储该 TTY 设备的终端I/O设置（波特率、字符大小、校验、流控制、本地标志、控制字符等）。`termios` 结构在 `<termios.h>` 中定义。
    *   `int pgrp`: 与此 TTY 关联的进程组ID。用于将信号（如 `SIGINT`, `SIGTSTP`）发送到前台进程组。
    *   `int stopped`: 流控制状态标志。如果非零，表示输出流被停止 (例如，用户按下了 `Ctrl-S`)。
    *   `void (*write)(struct tty_struct * tty)`: 一个函数指针，指向实际将字符写入底层硬件的函数（例如，对于控制台是 `con_write`，对于串口是 `rs_write`）。
    *   `struct tty_queue read_q`: 原始输入队列 (raw queue)。从键盘或串口接收到的原始字符首先存入这里。
    *   `struct tty_queue write_q`: 输出队列。待发送到显示设备或串口的字符存入这里，由 `(*write)` 函数处理。
    *   `struct tty_queue secondary`: "cooked" 或规范模式处理队列 (也称 canon_q 或 cooked_q)。当 TTY 处于规范模式 (canonical mode) 时，`read_q` 中的字符经过行编辑处理（如处理退格、行删除、EOF等）后，整行数据才被放入此队列，供用户进程读取。

## 宏定义详解

### TTY 队列操作宏

这些宏用于方便地操作 `tty_queue` 环形缓冲区：

*   `#define TTY_BUF_SIZE 1024`: 定义了TTY队列缓冲区的大小。
*   `#define INC(a) ((a) = ((a)+1) & (TTY_BUF_SIZE-1))`: 原子地增加头/尾指针 `a`，并处理环形回绕 (通过与 `TTY_BUF_SIZE-1` 进行按位与)。
*   `#define DEC(a) ((a) = ((a)-1) & (TTY_BUF_SIZE-1))`: 原子地减少头/尾指针 `a`，并处理回绕。
*   `#define EMPTY(a) ((a).head == (a).tail)`: 判断队列 `a` 是否为空。
*   `#define LEFT(a) (((a).tail-(a).head-1)&(TTY_BUF_SIZE-1))`: 计算队列 `a` 中剩余的空闲空间字节数。减1是为了区分队列满和队列空的状态。
*   `#define LAST(a) ((a).buf[(TTY_BUF_SIZE-1)&((a).head-1)])`: 获取队列 `a` 中最后一个被写入的字符（即 `head` 指针前一个位置的字符）。
*   `#define FULL(a) (!LEFT(a))`: 判断队列 `a` 是否已满 (即剩余空间为0)。
*   `#define CHARS(a) (((a).head-(a).tail)&(TTY_BUF_SIZE-1))`: 计算队列 `a` 中当前存储的字符数。
*   `#define GETCH(queue,c) (void)({c=(queue).buf[(queue).tail];INC((queue).tail);})`: 从队列 `queue` 中取出一个字符到变量 `c`，并移动尾指针。
*   `#define PUTCH(c,queue) (void)({(queue).buf[(queue).head]=(c);INC((queue).head);})`: 将字符 `c` 放入队列 `queue`，并移动头指针。

### 特殊控制字符宏

这些宏用于从 `tty->termios.c_cc[]` 数组中获取各种特殊控制字符的值。`c_cc` 数组是 `struct termios` 的一部分，存储了如中断、退出、删除、行结束等控制字符。

*   `#define INTR_CHAR(tty) ((tty)->termios.c_cc[VINTR])`: 中断字符 (通常是 `Ctrl-C`)。
*   `#define QUIT_CHAR(tty) ((tty)->termios.c_cc[VQUIT])`: 退出字符 (通常是 `Ctrl-\`)。
*   `#define ERASE_CHAR(tty) ((tty)->termios.c_cc[VERASE])`: 删除字符 (通常是退格键)。
*   `#define KILL_CHAR(tty) ((tty)->termios.c_cc[VKILL])`: 行删除字符 (通常是 `Ctrl-U`)。
*   `#define EOF_CHAR(tty) ((tty)->termios.c_cc[VEOF])`: 文件结束字符 (通常是 `Ctrl-D`)。
*   `#define START_CHAR(tty) ((tty)->termios.c_cc[VSTART])`: 流控制开始字符 (通常是 `Ctrl-Q`, XON)。
*   `#define STOP_CHAR(tty) ((tty)->termios.c_cc[VSTOP])`: 流控制停止字符 (通常是 `Ctrl-S`, XOFF)。
*   `#define SUSPEND_CHAR(tty) ((tty)->termios.c_cc[VSUSP])`: 挂起作业字符 (通常是 `Ctrl-Z`)。

### `INIT_C_CC` 宏

*   `#define INIT_C_CC "\003\034\177\025\004\0\1\0\021\023\032\0\022\017\027\026\0"`
*   **功能**: 为 `termios.c_cc` 数组提供一组默认的控制字符值。这是一个字符串字面量，每个字符的八进制值对应一个控制字符的ASCII码。
    *   `\003 (ETX)`: VINTR (Ctrl-C)
    *   `\034 (FS)`: VQUIT (Ctrl-\)
    *   `\177 (DEL)`: VERASE
    *   `\025 (NAK)`: VKILL (Ctrl-U)
    *   `\004 (EOT)`: VEOF (Ctrl-D)
    *   `\0`: VTIME (定时器超时值，0表示禁用)
    *   `\1`: VMIN (读取时最少字符数，1表示至少读一个)
    *   `\0`: (旧 SVR4 sxtc，未使用)
    *   `\021 (DC1)`: VSTART (Ctrl-Q)
    *   `\023 (DC3)`: VSTOP (Ctrl-S)
    *   `\032 (SUB)`: VSUSP (Ctrl-Z)
    *   `\0`: VEOL (附加的行结束符，未使用)
    *   `\022 (DC2)`: VREPRINT (Ctrl-R, 重绘行)
    *   `\017 (SI)`: VDISCARD (Ctrl-O, BSD特性，丢弃输出)
    *   `\027 (ETB)`: VWERASE (Ctrl-W, 删除前一个单词)
    *   `\026 (SYN)`: VLNEXT (Ctrl-V, 输入下一个字符的字面值)
    *   `\0`: VEOL2 (另一个附加的行结束符，未使用)

## 外部变量和函数声明

*   `extern struct tty_struct tty_table[];`: 声明全局 TTY 设备表。数组的索引通常对应 TTY 的次设备号。例如，`tty_table[0]` 可能代表控制台，`tty_table[1]` 代表第一个串口等。
*   `void rs_init(void);`: 串口 (RS-232) 初始化函数。
*   `void con_init(void);`: 控制台初始化函数。
*   `void tty_init(void);`: TTY 子系统总初始化函数，通常会调用 `rs_init` 和 `con_init`。
*   `int tty_read(unsigned c, char * buf, int n);`: 从 TTY `c` 的规范队列 (`secondary`) 读取 `n` 字节到 `buf`。
*   `int tty_write(unsigned c, char * buf, int n);`: 将 `buf` 中的 `n` 字节数据写入 TTY `c` 的输出队列 (`write_q`)。
*   `void rs_write(struct tty_struct * tty);`: 串口的底层写函数，将 `tty->write_q` 中的数据发送到串口硬件。
*   `void con_write(struct tty_struct * tty);`: 控制台的底层写函数，将 `tty->write_q` 中的数据输出到屏幕。
*   `void copy_to_cooked(struct tty_struct * tty);`: 将原始输入队列 `tty->read_q` 中的数据进行行编辑（“cooked”模式处理），并将处理后的结果（通常是一整行）放入规范队列 `tty->secondary`。

## 总结

`include/linux/tty.h` 是 Linux 0.11 TTY 子系统的核心定义文件。它通过 `struct tty_queue` 和 `struct tty_struct` 详细描述了 TTY 设备的内部状态和数据流管理方式。各种宏提供了对 TTY 队列和控制字符的便捷访问。函数声明则勾勒出 TTY 初始化的流程以及数据在不同队列和硬件层之间流动的路径。这个文件是理解 Linux 如何处理用户键盘输入、程序输出到屏幕以及串口通信的关键。它体现了 Unix 系统中 TTY 设备的经典设计思想，包括原始模式、规范（cooked）模式处理以及流控制等概念。
