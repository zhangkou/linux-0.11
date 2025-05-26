# signal.h 文件详解

`include/signal.h` 文件是 Linux 0.11 内核以及遵循 POSIX 标准的系统中用于定义信号处理相关常量、数据结构和函数原型的标准头文件。信号是一种异步通知机制，允许内核或其他进程通知一个进程发生了某个事件。

此头文件定义了各种信号的编号、信号处理动作的选项、信号集操作以及用于发送和处理信号的函数原型。

## 核心功能

1.  **定义信号编号**: 列出系统中所有支持的信号及其对应的整数编号 (如 `SIGHUP`, `SIGINT`, `SIGKILL` 等)。
2.  **定义信号处理相关的类型**:
    *   `sig_atomic_t`: 原子操作类型，用于在信号处理函数中安全访问的变量。
    *   `sigset_t`: 信号集类型，用于表示一组信号。
3.  **定义 `struct sigaction` 结构**: 用于 `sigaction()` 系统调用，以更灵活的方式指定信号处理方式。
4.  **定义信号处理动作常量**:
    *   `SIG_DFL`: 默认信号处理动作。
    *   `SIG_IGN`: 忽略信号。
5.  **定义 `sigaction` 标志**: 如 `SA_NOCLDSTOP`, `SA_NOMASK`, `SA_ONESHOT` (尽管注释提到 `sigactions` 在0.11中未完全实现，但头文件为了POSIX兼容性定义了这些)。
6.  **定义信号集操作函数中使用的常量**: `SIG_BLOCK`, `SIG_UNBLOCK`, `SIG_SETMASK`。
7.  **声明信号处理相关的函数原型**: 如 `signal()`, `kill()`, `sigaction()`, `sigprocmask()` 等。

## 宏和类型定义详解

### `typedef int sig_atomic_t;`

*   **功能**: 定义 `sig_atomic_t` 为 `int` 类型。这个类型用于声明在信号处理函数中可以被原子访问的变量。这意味着对这种类型变量的读写操作应该是不可中断的，以避免在信号处理期间发生竞争条件。在简单系统中，`int` 通常被认为是原子可访问的。

### `typedef unsigned int sigset_t; /* 32 bits */`

*   **功能**: 定义 `sigset_t` 为 `unsigned int` 类型，用于表示一个信号集。
*   **解释**: 在 Linux 0.11 中，信号集是一个32位的整数，每一位对应一个信号（从信号1到信号31，信号0未使用）。如果某一位被置1，表示对应的信号在该集合中。由于最多有 `_NSIG` (32) 个信号，一个32位无符号整数足以表示所有信号的状态。

### 信号编号 (`SIG...`)

这些宏定义了各个信号的唯一正整数编号。

*   `#define _NSIG 32`: 定义了系统中信号的总数（包括信号0，但信号0不用）。实际有效信号是1到31。
*   `#define NSIG _NSIG`: `NSIG` 是 `_NSIG` 的别名。

主要的信号定义包括：
*   `#define SIGHUP 1`: **Hangup (挂断)**。通常在控制终端关闭或父进程退出时发送给前台进程组。
*   `#define SIGINT 2`: **Interrupt (中断)**。通常由用户按下 `Ctrl-C` 产生，用于中断当前运行的程序。
*   `#define SIGQUIT 3`: **Quit (退出)**。通常由用户按下 `Ctrl-\` 产生，除了终止程序外，通常还会产生核心转储 (core dump)。
*   `#define SIGILL 4`: **Illegal Instruction (非法指令)**。CPU执行了一条无法识别或特权不足的指令。
*   `#define SIGTRAP 5`: **Trace trap (跟踪陷阱)**。通常用于调试器，如断点或单步执行。
*   `#define SIGABRT 6`: **Abort (异常中止)**。由 `abort()` 函数产生，表示程序自我检测到严重错误。
*   `#define SIGIOT 6`: **IOT instruction (IOT指令)**。`SIGABRT` 的同义词，源于PDP-11的IOT指令。
*   `#define SIGUNUSED 7`: (未使用)。
*   `#define SIGFPE 8`: **Floating-Point Exception (浮点异常)**。如除以零、浮点溢出等。
*   `#define SIGKILL 9`: **Kill (杀死)**。这是一个无法被捕获、阻塞或忽略的信号。它会立即终止目标进程。
*   `#define SIGUSR1 10`: **User-defined signal 1 (用户定义信号1)**。可由用户程序自定义用途。
*   `#define SIGSEGV 11`: **Segmentation Violation (段错误)**。进程试图访问其无权访问或不存在的内存区域。
*   `#define SIGUSR2 12`: **User-defined signal 2 (用户定义信号2)**。
*   `#define SIGPIPE 13`: **Broken pipe (管道破裂)**。当进程试图向一个没有读者（或读端已关闭）的管道或FIFO写入数据时产生。
*   `#define SIGALRM 14`: **Alarm clock (报警时钟)**。由 `alarm()` 系统调用设置的定时器超时后产生。
*   `#define SIGTERM 15`: **Termination (终止)**。请求进程正常终止。这是一个可以被捕获和处理的通用终止信号。
*   `#define SIGSTKFLT 16`: **Stack fault on coprocessor (协处理器栈故障)**。数学协处理器栈发生错误 (Linux 0.11 中可能不常用或未完全支持)。
*   `#define SIGCHLD 17`: **Child process stopped or terminated (子进程停止或终止)**。当子进程终止、停止或在停止后继续运行时，会发送给父进程。
*   `#define SIGCONT 18`: **Continue executing, if stopped (继续执行，如果已停止)**。使一个已停止的进程恢复执行。
*   `#define SIGSTOP 19`: **Stop executing (停止执行)**。与 `SIGKILL` 类似，这是一个无法被捕获、阻塞或忽略的信号。它会暂停进程的执行。
*   `#define SIGTSTP 20`: **Terminal stop signal (终端停止信号)**。通常由用户按下 `Ctrl-Z` 产生，请求进程暂停。可以被捕获和处理。
*   `#define SIGTTIN 21`: **Background process attempting read (后台进程尝试读终端)**。当一个后台进程组中的进程尝试从其控制终端读取数据时产生。
*   `#define SIGTTOU 22`: **Background process attempting write (后台进程尝试写终端)**。当一个后台进程组中的进程尝试向其控制终端写入数据时产生 (如果终端设置为 `TOSTOP` 模式)。

### `sigaction` 相关的标志 (`sa_flags`)

这些标志用于修改 `sigaction()` 系统调用设置的信号处理行为。注释中提到Linux 0.11并未完全实现 `sigaction`，但为了POSIX兼容性定义了这些。

*   `#define SA_NOCLDSTOP 1`: 如果 `sig` 是 `SIGCHLD`，则当子进程停止 (stop) 或在停止后继续 (continue) 时，不产生 `SIGCHLD` 信号给父进程。只有当子进程终止 (terminate) 时才产生。
*   `#define SA_NOMASK 0x40000000`: (在POSIX中通常是 `SA_NODEFER`) 当信号处理函数执行期间，不自动阻塞当前信号。即允许信号处理函数嵌套响应同类型的信号。
*   `#define SA_ONESHOT 0x80000000`: (在POSIX中通常是 `SA_RESETHAND`) 当信号处理函数被调用一次后，信号的处理方式恢复到默认值 (`SIG_DFL`)。

### `sigprocmask()` 的 `how` 参数常量

这些常量用于 `sigprocmask()` 系统调用，指定如何修改进程的信号屏蔽集。

*   `#define SIG_BLOCK 0`: 将参数 `set` 中指定的信号集添加到当前进程的信号屏蔽集中。
*   `#define SIG_UNBLOCK 1`: 从当前进程的信号屏蔽集中移除参数 `set` 中指定的信号。
*   `#define SIG_SETMASK 2`: 将当前进程的信号屏蔽集直接设置为参数 `set` 中指定的信号集。

### 信号处理方式常量 (`sa_handler` 或 `signal()` 的返回值)

*   `#define SIG_DFL ((void (*)(int))0)`: **Default (默认处理)**。表示对信号执行系统默认的操作（例如，对于 `SIGTERM` 是终止进程，对于 `SIGSEGV` 是终止并dump核心）。
*   `#define SIG_IGN ((void (*)(int))1)`: **Ignore (忽略信号)**。表示忽略该信号，不做任何处理（除了 `SIGKILL` 和 `SIGSTOP` 不能被忽略）。

### `struct sigaction` 结构体

*   **功能**: 用于 `sigaction()` 系统调用，以详细指定信号的处理方式。比旧的 `signal()` 接口提供了更多的控制。
*   **成员**:
    *   `void (*sa_handler)(int)`: 指向信号处理函数的指针。也可以是 `SIG_DFL` 或 `SIG_IGN`。参数 `int` 是信号编号。
    *   `sigset_t sa_mask`: 一个信号集，指定在执行 `sa_handler` 期间需要额外阻塞的信号。当前信号总会被阻塞（除非设置了 `SA_NOMASK`）。
    *   `int sa_flags`: 一组标志 (如 `SA_NOCLDSTOP`, `SA_ONESHOT`)，用于修改信号处理的行为。
    *   `void (*sa_restorer)(void)`: (未使用或由内核内部使用) 指向一个“恢复器”函数。在某些架构上，信号处理函数返回后，需要一个特殊的跳板函数来恢复上下文，这个字段可能用于此目的。在Linux x86中，这通常由内核在信号栈帧中设置，用户不需要关心。

## 函数原型声明

*   `void (*signal(int _sig, void (*_func)(int)))(int);`
    *   **功能**: 设置信号 `_sig` 的处理函数为 `_func`。这是旧式的、功能较弱的信号设置接口。
    *   **返回值**: 返回该信号之前的处理函数指针。
*   `int raise(int sig);`
    *   **功能**: 向当前进程发送信号 `sig`。等价于 `kill(getpid(), sig)`。
*   `int kill(pid_t pid, int sig);`
    *   **功能**: 向进程ID为 `pid` 的进程（或一组进程）发送信号 `sig`。
*   `int sigaddset(sigset_t *mask, int signo);`
    *   **功能**: 将信号 `signo` 添加到信号集 `*mask` 中。
*   `int sigdelset(sigset_t *mask, int signo);`
    *   **功能**: 从信号集 `*mask` 中移除信号 `signo`。
*   `int sigemptyset(sigset_t *mask);`
    *   **功能**: 初始化信号集 `*mask`，使其不包含任何信号 (所有位清零)。
*   `int sigfillset(sigset_t *mask);`
    *   **功能**: 初始化信号集 `*mask`，使其包含所有已定义的信号 (所有位设为1，除了对那些不能被阻塞的信号可能无效)。
*   `int sigismember(sigset_t *mask, int signo);`
    *   **功能**: 检查信号 `signo` 是否是信号集 `*mask` 的成员。
    *   **返回值**: 1表示是成员，0表示不是，-1表示错误 (如无效信号)。
*   `int sigpending(sigset_t *set);`
    *   **功能**: 获取当前进程中已发送但被阻塞而未递达的信号集，并将其存储在 `*set` 中。
*   `int sigprocmask(int how, sigset_t *set, sigset_t *oldset);`
    *   **功能**: 根据 `how` 参数（`SIG_BLOCK`, `SIG_UNBLOCK`, `SIG_SETMASK`）修改当前进程的信号屏蔽集。如果 `oldset` 非空，则将修改前的信号屏蔽集存储在 `*oldset` 中。
*   `int sigsuspend(sigset_t *sigmask);`
    *   **功能**: 临时将进程的信号屏蔽集替换为 `*sigmask`，然后使进程挂起，直到一个未被 `*sigmask` 阻塞的信号被捕获并处理完毕。函数返回后，原信号屏蔽集会被恢复。这是原子地等待特定信号的方法。
*   `int sigaction(int sig, struct sigaction *act, struct sigaction *oldact);`
    *   **功能**: 检查或修改信号 `sig` 的处理动作。
    *   如果 `act` 非空，则将信号 `sig` 的处理方式设置为 `*act` 中指定的内容。
    *   如果 `oldact` 非空，则将信号 `sig` 修改前的处理方式存储在 `*oldact` 中。

## 总结

`include/signal.h` 为 Linux 0.11 内核和用户程序提供了处理信号所需的所有基本定义和接口。它定义了各种信号的名称和编号，用于表示和操作信号集的 `sigset_t` 类型，以及用于精确控制信号处理行为的 `struct sigaction` 结构。函数原型则涵盖了从设置信号处理函数 (`signal`, `sigaction`)、发送信号 (`kill`, `raise`) 到管理信号屏蔽 (`sigprocmask`, `sigsuspend`) 和查询待处理信号 (`sigpending`) 的全套操作。尽管注释表明 `sigaction` 在0.11中可能未完全实现，但头文件的设计已经为POSIX兼容的信号处理机制奠定了基础。这些功能对于实现健壮的、能够响应异步事件的程序至关重要。
