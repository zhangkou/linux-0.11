# termios.h 文件详解

`include/termios.h` 文件是 Linux 0.11 内核以及遵循 POSIX 标准的系统中用于定义终端 I/O 接口的头文件。它定义了控制异步通信端口（如串口、控制台终端）的各种参数、数据结构和函数。这个接口允许程序以标准化的方式来控制终端的行为，例如设置波特率、字符大小、校验方式、行编辑模式、流控制以及处理特殊控制字符等。

"termios" 这个名字来源于 "Terminal I/O Settings"。它是早期 Unix 系统中 `termio` 接口的POSIX标准化和扩展版本。

## 核心功能

1.  **定义 `ioctl` 命令常量**: 为与终端相关的 `ioctl` 操作定义了一系列命令码 (如 `TCGETS`, `TCSETS`)。
2.  **定义窗口大小结构 (`struct winsize`)**: 用于获取和设置终端窗口的行数和列数。
3.  **定义终端I/O结构**:
    *   `struct termio`: 旧式的、System V 兼容的终端I/O结构。
    *   `struct termios`: POSIX 标准的终端I/O结构，是 `termio` 的超集，提供了更丰富的控制。
4.  **定义 `termios` 结构中各字段的标志位和常量**:
    *   `c_iflag` (输入模式标志)。
    *   `c_oflag` (输出模式标志)。
    *   `c_cflag` (控制模式标志，如波特率、字符大小、校验等)。
    *   `c_lflag` (本地模式标志，如行编辑、信号字符、回显等)。
    *   `c_cc[]` (控制字符数组的索引，如 `VINTR`, `VEOF` 等)。
5.  **定义 modem 控制线常量**: 如 `TIOCM_DTR`, `TIOCM_RTS`。
6.  **定义流控制和刷新队列常量**: 用于 `tcflow()` 和 `tcflush()`。
7.  **定义 `tcsetattr()` 的动作常量**: 如 `TCSANOW`, `TCSADRAIN`。
8.  **声明与 termios 相关的函数原型**: 如 `tcgetattr()`, `tcsetattr()`, `cfgetispeed()`, `cfsetospeed()` 等。

## `ioctl` 命令常量

这些常量以 `0x54` (`'T'`) 开头，作为 `ioctl` 系统调用的 `cmd` 参数，用于执行与终端相关的特定操作。

*   `#define TCGETS 0x5401`: **Get Settings (获取设置)**。获取当前终端的 `struct termios` 参数。
*   `#define TCSETS 0x5402`: **Set Settings (设置设置)**。立即设置终端的 `struct termios` 参数。任何已在输入或输出队列中但尚未传输的数据都将被丢弃。
*   `#define TCSETSW 0x5403`: **Set Settings Wait (等待后设置)**。等待所有已发送到输出队列的数据传输完毕后，再设置终端的 `struct termios` 参数。输入队列也会被刷新。
*   `#define TCSETSF 0x5404`: **Set Settings Flush (刷新后设置)**。刷新输入和输出队列，然后立即设置终端的 `struct termios` 参数。
*   `#define TCGETA 0x5405`: 获取旧式 `struct termio` 参数 (System V 兼容)。
*   `#define TCSETA 0x5406`: 立即设置旧式 `struct termio` 参数。
*   `#define TCSETAW 0x5407`: 等待输出完成后设置旧式 `struct termio` 参数。
*   `#define TCSETAF 0x5408`: 刷新队列后设置旧式 `struct termio` 参数。
*   `#define TCSBRK 0x5409`: **Send Break (发送 Break 信号)**。如果参数为0，发送一个持续一定时间的 Break 信号 (通常用于串口)。
*   `#define TCXONC 0x540A`: **Flow Control On/Off (流控制开关)**。执行 XON/XOFF 流控制。参数可以是 `TCOOFF` (暂停输出), `TCOON` (恢复输出), `TCIOFF` (发送 STOP 字符), `TCION` (发送 START 字符)。
*   `#define TCFLSH 0x540B`: **Flush Queues (刷新队列)**。刷新输入或输出队列。参数可以是 `TCIFLUSH` (刷新输入), `TCOFLUSH` (刷新输出), `TCIOFLUSH` (刷新输入和输出)。
*   `#define TIOCEXCL 0x540C`: 设置终端为独占模式 (不常用)。
*   `#define TIOCNXCL 0x540D`: 取消终端的独占模式。
*   `#define TIOCSCTTY 0x540E`: **Set Controlling Terminal (设置控制终端)**。使当前终端成为调用进程的控制终端 (如果进程是会话领导者且尚无控制终端)。
*   `#define TIOCGPGRP 0x540F`: **Get Process Group (获取前台进程组ID)**。获取与终端关联的前台进程组ID。
*   `#define TIOCSPGRP 0x5410`: **Set Process Group (设置前台进程组ID)**。设置与终端关联的前台进程组ID。
*   `#define TIOCOUTQ 0x5411`: **Output Queue Count (输出队列字符数)**。获取输出队列中等待传输的字符数量。
*   `#define TIOCSTI 0x5412`: **Simulate Terminal Input (模拟终端输入)**。将一个字符插入到终端的输入队列中，如同它是从终端输入的一样。
*   `#define TIOCGWINSZ 0x5413`: **Get Window Size (获取窗口大小)**。获取终端窗口的行数和列数，存储在 `struct winsize` 中。
*   `#define TIOCSWINSZ 0x5414`: **Set Window Size (设置窗口大小)**。设置终端窗口的行数和列数。
*   `#define TIOCMGET 0x5415`: **Modem Get (获取Modem状态位)**。获取串口Modem控制线的状态位 (如 DTR, RTS, CTS, DCD)。
*   `#define TIOCMBIS 0x5416`: **Modem Bit Set (设置Modem状态位)**。设置指定的Modem状态位。
*   `#define TIOCMBIC 0x5417`: **Modem Bit Clear (清除Modem状态位)**。清除指定的Modem状态位。
*   `#define TIOCMSET 0x5418`: **Modem Set (直接设置Modem状态位)**。将Modem状态位设置为指定值。
*   `#define TIOCGSOFTCAR 0x5419`: 获取软件载波标志的状态 (不常用)。
*   `#define TIOCSSOFTCAR 0x541A`: 设置软件载波标志的状态。
*   `#define TIOCINQ 0x541B` (等同于 `FIONREAD`): **Input Queue Count (输入队列字符数)**。获取输入队列中可读的字符数量。

## 数据结构详解

### `struct winsize`

```c
struct winsize {
	unsigned short ws_row;		/* 窗口的行数 */
	unsigned short ws_col;		/* 窗口的列数 */
	unsigned short ws_xpixel;	/* 窗口的水平像素数 (通常不被精确使用) */
	unsigned short ws_ypixel;	/* 窗口的垂直像素数 (通常不被精确使用) */
};
```
用于描述终端窗口的大小。当窗口大小改变时（例如用户调整了xterm窗口），内核会收到通知（通常通过 `SIGWINCH` 信号），并更新此结构。程序可以通过 `TIOCGWINSZ` ioctl 获取这些信息。

### `struct termio` (旧式)

```c
#define NCC 8 // termio 中 c_cc 数组的大小
struct termio {
	unsigned short c_iflag;		/* input mode flags (输入模式标志) */
	unsigned short c_oflag;		/* output mode flags (输出模式标志) */
	unsigned short c_cflag;		/* control mode flags (控制模式标志) */
	unsigned short c_lflag;		/* local mode flags (本地模式标志) */
	unsigned char c_line;		/* line discipline (线路规程) */
	unsigned char c_cc[NCC];	/* control characters (控制字符数组，大小为8) */
};
```
这是 System V Unix 中较早的终端I/O接口结构。Linux 0.11 为了兼容性也支持它。

### `struct termios` (POSIX 标准)

```c
#define NCCS 17 // termios 中 c_cc 数组的大小 (Linux 0.11 定义为17，POSIX后续版本可能更多)
struct termios {
	unsigned long c_iflag;		/* input mode flags */
	unsigned long c_oflag;		/* output mode flags */
	unsigned long c_cflag;		/* control mode flags */
	unsigned long c_lflag;		/* local mode flags */
	unsigned char c_line;		/* line discipline */
	unsigned char c_cc[NCCS];	/* control characters */
};
```
这是 POSIX 标准的终端I/O接口结构，是 `termio` 的扩展，提供了更精细的控制。它是现代Unix-like系统中控制终端属性的主要方式。

#### `c_iflag` (Input Flags - 输入模式标志)

控制输入的处理方式。八进制值。
*   `IGNBRK (0000001)`: 忽略 BREAK 条件。
*   `BRKINT (0000002)`: 如果设置了 `IGNBRK` 则此位无效。如果未设置 `IGNBRK` 但设置了 `BRKINT`，则 BREAK 会使输入输出队列被刷新，并向前台进程组发送 `SIGINT` 信号。
*   `IGNPAR (0000004)`: 忽略输入字符中的奇偶校验错误。
*   `PARMRK (0000010)`: 标记奇偶校验错误。如果设置，发生奇偶校验或帧错误的字节会以 `\377 \0 X` (X是数据) 或 `\377 Y X` (Y是错误类型)的形式传递给输入队列。
*   `INPCK (0000020)`: 允许输入奇偶校验。
*   `ISTRIP (0000040)`: 将接收到的字符的第8位（最高位）剥离，使其成为7位字符。
*   `INLCR (0000100)`: 将输入中的换行符 (`NL`, `\n`) 转换为空格符 (`CR`, `\r`)。
*   `IGNCR (0000200)`: 忽略输入中的回车符 (`CR`)。
*   `ICRNL (0000400)`: 将输入中的回车符 (`CR`) 转换为换行符 (`NL`)。如果设置了 `IGNCR` 则此位无效。
*   `IUCLC (0001000)`: (非POSIX) 将输入的大写字母映射为小写字母。
*   `IXON (0002000)`: 允许输出软件流控制 (XON/XOFF)。接收到 STOP 字符 (`VSTOP`) 时暂停输出，接收到 START 字符 (`VSTART`) 时恢复输出。
*   `IXANY (0004000)`: (非POSIX) 允许任何字符重新启动输出（而不仅仅是 START 字符）。
*   `IXOFF (0010000)`: 允许输入软件流控制。当输入队列接近满时，系统发送 STOP 字符；当有足够空间时，发送 START 字符。
*   `IMAXBEL (0020000)`: (非POSIX) 当输入队列满时，响铃 (BEL, `\007`) 而不是丢弃字符。

#### `c_oflag` (Output Flags - 输出模式标志)

控制输出的处理方式。八进制值。
*   `OPOST (0000001)`: 执行实现定义的输出后处理。
*   `OLCUC (0000002)`: (非POSIX) 将输出的小写字母映射为大写字母。
*   `ONLCR (0000004)`: (XSI) 将输出中的换行符 (`NL`) 转换为空格回车符 (`CR-NL`)。
*   `OCRNL (0000010)`: 将输出中的回车符 (`CR`) 转换为换行符 (`NL`)。
*   `ONOCR (0000020)`: 在第0列不输出回车符。
*   `ONLRET (0000040)`: 换行符执行回车功能。
*   `OFILL (0000100)`: 发送填充字符以实现延迟，而不是使用定时延迟。
*   `OFDEL (0000200)`: 填充字符是 DEL (`\177`)。如果未设置，则是 NUL (`\0`)。
*   `NLDLY (0000400)`: 换行延迟掩码。值: `NL0` (无延迟), `NL1`。
*   `CRDLY (0003000)`: 回车延迟掩码。值: `CR0`, `CR1`, `CR2`, `CR3`。
*   `TABDLY (0014000)`: 水平制表符延迟掩码。值: `TAB0`, `TAB1`, `TAB2`, `TAB3` (或 `XTABS` 表示将制表符扩展为空格)。
*   `BSDLY (0020000)`: 退格延迟掩码。值: `BS0`, `BS1`。
*   `VTDLY (0040000)`: 垂直制表符延迟掩码。值: `VT0`, `VT1`。
*   `FFDLY (0040000)`: 换页延迟掩码。值: `FF0`, `FF1`。(注意: VTDLY 和 FFDLY 在0.11中使用了相同的值，可能是笔误或共享)

#### `c_cflag` (Control Flags - 控制模式标志)

控制硬件特性，如波特率、字符大小、校验等。八进制值。
*   `CBAUD (0000017)`: 波特率掩码 (低5位)。
    *   `B0` - `B38400`: 定义了各种标准波特率常量。`B0` 表示挂断线路。`EXTA` 和 `EXTB` 是 `B19200` 和 `B38400` 的别名。
*   `CSIZE (0000060)`: 字符大小掩码。
    *   `CS5`, `CS6`, `CS7`, `CS8`: 分别表示5、6、7、8位字符。
*   `CSTOPB (0000100)`: 设置两个停止位，而不是一个。
*   `CREAD (0000200)`: 允许接收字符。
*   `PARENB (0000400)`: 允许奇偶校验生成和检测。
*   `CPARODD (0001000)`: 使用奇校验，否则使用偶校验 (如果`PARENB`设置)。
*   `HUPCL (0002000)`: **Hang Up on Last Close (最后关闭时挂断)**。当最后一个持有该端口的进程关闭它时，降低Modem控制线 (通常是 DTR)，导致Modem挂断。
*   `CLOCAL (0004000)`: **Ignore Modem Control Lines (忽略Modem控制线)**。表示线路是本地直连，不使用Modem信号。
*   `CIBAUD (03600000)`: (非POSIX) 输入波特率掩码 (如果与输出波特率不同)。在0.11中注释为 `(not used)`。
*   `CRTSCTS (020000000000)`: (非POSIX) 允许 RTS/CTS (硬件) 流控制。

#### `c_lflag` (Local Flags - 本地模式标志)

控制终端的本地行为，如行编辑、信号字符处理、回显等。八进制值。
*   `ISIG (0000001)`: 当接收到 `INTR`, `QUIT`, `SUSP`, `DSUSP` (后者0.11未定义) 特殊字符时，产生相应的信号。
*   `ICANON (0000002)`: **Canonical Mode (规范模式)**。启用规范输入处理（行编辑）。输入按行缓冲，直到接收到 NL, EOF, EOL 字符才对用户程序可用。在非规范模式下，输入不按行缓冲，读取操作根据 `VMIN` 和 `VTIME` 的设置行为。
*   `XCASE (0000004)`: (非POSIX, 与 `IUCLC` 和 `OLCUC` 一起使用) 如果 `ICANON` 也设置了，则进行大小写转换。
*   `ECHO (0000010)`: **回显输入字符**。将用户输入的字符立即回显到终端。
*   `ECHOE (0000020)`: 如果 `ICANON` 也设置了，则 `VERASE` 字符会擦除屏幕上最后一个字符（例如，通过发送 "backspace-space-backspace"）。
*   `ECHOK (0000040)`: 如果 `ICANON` 也设置了，则 `VKILL` 字符会擦除当前整行。
*   `ECHONL (0000100)`: 如果 `ICANON` 也设置了，则即使 `ECHO` 未设置，也回显 NL 字符。
*   `NOFLSH (0000200)`: 禁止在产生 `SIGINT`, `SIGQUIT`, `SIGSUSP` 信号时刷新输入输出队列。
*   `TOSTOP (0000400)`: 如果设置，则当后台进程尝试向其控制终端写入时，向该进程的进程组发送 `SIGTTOU` 信号。
*   `ECHOCTL (0001000)`: (非POSIX) 如果 `ECHO` 也设置了，则控制字符（ASCII码小于0x20）除了 `TAB`, `NL`, `START`, `STOP` 外，会被回显为 `^X` (X是字符ASCII码加0x40)。
*   `ECHOPRT (0002000)`: (非POSIX) 如果 `ECHO` 和 `ICANON` 都设置了，则当 `VERASE` 字符删除字符时，被删除的字符会以某种方式显示出来（例如，在`\`和`/`之间打印）。
*   `ECHOKE (0004000)`: (非POSIX) 如果 `ICANON` 也设置了，则 `VKILL` 字符会更有效地擦除行（例如，不仅是光标移动，还清除屏幕内容）。
*   `FLUSHO (0010000)`: (非POSIX) 输出被刷新。通过键入DISCARD字符可以切换此状态。
*   `PENDIN (0040000)`: (非POSIX) 当下一个字符被读取时，重新打印输入队列中所有未读的字符 (用于 `VLNEXT` 后的输入)。
*   `IEXTEN (0100000)`: 允许实现定义的输入处理。

#### `c_cc[NCCS]` (Control Characters - 控制字符数组)

数组的索引由 `Vxxx` 常量定义：
*   `VINTR (0)`: Interrupt 字符。
*   `VQUIT (1)`: Quit 字符。
*   `VERASE (2)`: Erase 字符 (退格)。
*   `VKILL (3)`: Kill (行删除) 字符。
*   `VEOF (4)`: End-of-file 字符。
*   `VTIME (5)`: 非规范模式下的读取超时定时器 (单位：0.1秒)。
*   `VMIN (6)`: 非规范模式下的读取操作返回前的最小字符数。
*   `VSWTC (7)`: (System V, 未在POSIX中定义) Switch character (用于shell层切换)。
*   `VSTART (8)`: Start (XON) 字符。
*   `VSTOP (9)`: Stop (XOFF) 字符。
*   `VSUSP (10)`: Suspend 字符 (Ctrl-Z)。
*   `VEOL (11)`: End-of-line (附加的行结束符)。
*   `VREPRINT (12)`: Reprint (重绘当前行) 字符 (Ctrl-R)。
*   `VDISCARD (13)`: Discard (丢弃输出) 字符 (Ctrl-O)。
*   `VWERASE (14)`: Word erase (删除前一个单词) 字符 (Ctrl-W)。
*   `VLNEXT (15)`: Literal next (输入下一个字符的字面值) 字符 (Ctrl-V)。
*   `VEOL2 (16)`: 另一个附加的行结束符。

### Modem 控制线常量 (`TIOCM_xxx`)

用于 `TIOCMGET`, `TIOCMBIS`, `TIOCMBIC`, `TIOCMSET` ioctl命令。
*   `TIOCM_LE (0x001)`: Line Enable (未使用)。
*   `TIOCM_DTR (0x002)`: Data Terminal Ready。
*   `TIOCM_RTS (0x004)`: Request To Send。
*   `TIOCM_ST (0x008)`: Secondary Transmit。
*   `TIOCM_SR (0x010)`: Secondary Receive。
*   `TIOCM_CTS (0x020)`: Clear To Send。
*   `TIOCM_CAR (0x040)`: Carrier Detect (也叫 `TIOCM_CD`)。
*   `TIOCM_RNG (0x080)`: Ring Indicator (也叫 `TIOCM_RI`)。
*   `TIOCM_DSR (0x100)`: Data Set Ready。

### `tcflow()` 和 `TCXONC` 使用的常量

*   `TCOOFF (0)`: 暂停输出 (等同于发送 STOP 字符)。
*   `TCOON (1)`: 恢复输出 (等同于发送 START 字符)。
*   `TCIOFF (2)`: 发送一个 STOP 字符。
*   `TCION (3)`: 发送一个 START 字符。

### `tcflush()` 和 `TCFLSH` 使用的常量

*   `TCIFLUSH (0)`: 刷新输入队列中已接收但尚未被程序读取的数据。
*   `TCOFLUSH (1)`: 刷新输出队列中已写入但尚未传输的数据。
*   `TCIOFLUSH (2)`: 同时刷新输入和输出队列。

### `tcsetattr()` 的 `optional_actions` 参数常量

指定何时应用对终端属性的更改。
*   `TCSANOW (0)`: 更改立即生效。
*   `TCSADRAIN (1)`: 更改在所有已写入输出队列的数据都已传输完毕后生效。常用于需要确保输出不被新设置干扰的情况（如更改波特率）。
*   `TCSAFLUSH (2)`: 更改在所有已写入输出队列的数据都已传输完毕后生效，并且在更改生效前会丢弃所有已接收但尚未被程序读取的输入数据。

## 函数原型声明

*   `typedef int speed_t;`: 定义 `speed_t` 类型（通常是 `int` 或 `unsigned int`），用于表示波特率。
*   `speed_t cfgetispeed(struct termios *termios_p);`: 获取 `termios_p` 结构中指定的输入波特率。
*   `speed_t cfgetospeed(struct termios *termios_p);`: 获取 `termios_p` 结构中指定的输出波特率。
*   `int cfsetispeed(struct termios *termios_p, speed_t speed);`: 在 `termios_p` 结构中设置输入波特率为 `speed`。
*   `int cfsetospeed(struct termios *termios_p, speed_t speed);`: 在 `termios_p` 结构中设置输出波特率为 `speed`。
*   `int tcdrain(int fildes);`: 等待与文件描述符 `fildes` 关联的终端的所有已写入输出队列的数据传输完毕。
*   `int tcflow(int fildes, int action);`: 对与文件描述符 `fildes` 关联的终端执行流控制操作 `action` (`TCOOFF`, `TCOON`, `TCIOFF`, `TCION`)。
*   `int tcflush(int fildes, int queue_selector);`: 刷新与文件描述符 `fildes` 关联的终端的指定队列 `queue_selector` (`TCIFLUSH`, `TCOFLUSH`, `TCIOFLUSH`)。
*   `int tcgetattr(int fildes, struct termios *termios_p);`: 获取与文件描述符 `fildes` 关联的终端的当前 `termios` 属性，并存储在 `termios_p` 指向的结构中。
*   `int tcsendbreak(int fildes, int duration);`: 在与文件描述符 `fildes` 关联的异步串行数据线上发送一个持续 `duration`（实现定义的单位，通常为0表示0.25-0.5秒）的 Break 信号。
*   `int tcsetattr(int fildes, int optional_actions, struct termios *termios_p);`: 根据 `optional_actions` 指定的方式，将与文件描述符 `fildes` 关联的终端的属性设置为 `termios_p` 指向的结构中的值。

## 总结

`include/termios.h` 是 POSIX 标准终端控制接口的核心。它通过 `struct termios` 结构体和一系列相关的标志、常量及函数，为应用程序提供了一种强大而灵活的方式来配置和控制终端的行为。这包括输入输出的数据处理方式、行编辑规则、特殊字符的定义、硬件流控制、软件流控制、波特率设置等等。理解 `termios` 对于编写任何需要与终端进行精细交互的程序（如文本编辑器、shell、通信程序）都至关重要。Linux 0.11 的这个头文件基本遵循了 POSIX 的定义，为TTY子系统的实现奠定了基础。
