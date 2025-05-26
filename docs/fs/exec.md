# exec.c 文件详解

`exec.c` 文件是 Linux 0.11 内核中负责实现 `execve()` 系统调用的核心组件。`execve()` 系统调用的功能是加载并执行一个新的程序，用新的程序替换当前进程的内存映像（包括代码段、数据段、堆栈等）。这是类Unix系统中创建新进程（通常通过 `fork()` 后跟 `execve()`）和运行命令的基础。

该文件实现了对可执行文件格式的解析（主要是 a.out 格式）、参数和环境变量的复制、内存布局的设置以及最终控制权的转移。Linux 0.11 的 `exec.c` 还引入了**需求加载 (demand-loading)** 的特性，即只将可执行文件的头部读入内存，实际的代码和数据页在发生页错误 (page fault) 时才从磁盘加载，这显著提高了程序启动速度和内存利用率。此外，它还支持 `#!` (shebang) 机制，允许执行以特定解释器开头的脚本文件。

## 核心功能

1.  **可执行文件加载**:
    *   解析 `a.out` 可执行文件头部。
    *   验证文件格式和权限。
2.  **参数和环境变量处理**:
    *   从旧进程的用户空间复制参数 (`argv`) 和环境变量 (`envp`) 到内核空间，然后再设置到新进程的用户空间栈顶。
    *   `count()`: 计算参数/环境变量的个数。
    *   `copy_strings()`: 复制字符串数组（参数或环境变量）。
    *   `create_tables()`: 在新进程的用户栈上创建 `argv` 和 `envp` 的指针表，并设置新的栈顶指针。
3.  **内存管理**:
    *   为新程序设置代码段、数据段和BSS段。
    *   释放旧进程占用的页表和内存区域。
    *   更新进程的LDT (局部描述符表) 以反映新的内存布局 (`change_ldt`)。
    *   实现需求加载：只加载文件头，将可执行文件的inode记录在 `current->executable`，页面错误处理程序负责实际加载数据页。
4.  **Shebang (`#!`) 支持**:
    *   如果可执行文件以 `#!interpreter [optional-arg]` 开头，则加载并执行指定的 `interpreter`，并将原文件名和 `optional-arg` 作为参数传递给解释器。
5.  **进程上下文更新**:
    *   更新当前进程的 `euid` (有效用户ID) 和 `egid` (有效组ID)，处理 `set-UID` 和 `set-GID` 位。
    *   重置信号处理函数。
    *   处理 `close_on_exec` 文件描述符标志。
    *   更新进程的 `brk` (堆顶)、`end_code` (代码段结束)、`end_data` (数据段结束)、`start_stack` (栈起始地址)。
6.  **控制权转移**:
    *   修改内核栈上保存的 `eip` (指令指针) 和 `esp` (栈指针)，使 `execve` 系统调用返回时，CPU在用户模式下从新程序的入口点 (`ex.a_entry`) 开始执行，并使用新的用户栈。

## 关键数据结构和宏

### `struct exec` (`<a.out.h>`)

这是 `a.out` 可执行文件头部的结构体，定义了程序的各种属性：

*   `a_magic`: 魔数，标识文件类型 (如 `ZMAGIC` 表示纯代码段可执行文件，支持需求加载)。
*   `a_text`: 代码段大小 (bytes)。
*   `a_data`: 初始化数据段大小 (bytes)。
*   `a_bss`: 未初始化数据段 (BSS) 大小 (bytes)。这部分在加载时不占文件空间，但在内存中需要分配。
*   `a_entry`: 程序执行入口点的虚拟地址。
*   `a_trsize`: 文本重定位信息大小 (通常为0)。
*   `a_drsize`: 数据重定位信息大小 (通常为0)。
*   `a_syms`: 符号表大小 (bytes)。

### 宏

*   `MAX_ARG_PAGES`: 定义了为参数和环境变量分配的最大页数 (32页，即128KB)。
*   `N_MAGIC(ex)`: 从 `ex` (struct exec) 中提取魔数。
*   `N_TXTOFF(ex)`: 计算文本段 (代码段) 在文件中的偏移量。对于 `ZMAGIC` 格式，这通常是 `BLOCK_SIZE` (1024字节)，因为文件头本身占用一个块。

## 主要函数详解

### `static unsigned long * create_tables(char * p, int argc, int envc)`

*   **功能**: 在新进程的用户栈顶（由指针 `p` 指向参数和环境变量字符串的末尾）创建 `argv` 和 `envp` 指针数组，并设置栈的最终布局。
*   **栈布局 (从高地址到低地址)**:
    1.  环境变量字符串 (null结尾)
    2.  参数字符串 (null结尾)
    3.  `envp` 数组 (指针列表，最后一项为NULL)
    4.  `argv` 数组 (指针列表, 最后一项为NULL)
    5.  `envp` 指针 (指向 `envp` 数组)
    6.  `argv` 指针 (指向 `argv` 数组)
    7.  `argc` (参数个数)
*   **步骤**:
    1.  `sp = (unsigned long *) (0xfffffffc & (unsigned long) p);`: 将 `p` (指向参数/环境变量字符串区域的最高地址的下一个字节) 对齐到4字节边界，作为初始栈顶。
    2.  `sp -= envc+1; envp = sp;`: 为 `envp` 数组 (包括末尾的NULL指针) 分配空间，`envp` 指针指向此数组。
    3.  `sp -= argc+1; argv = sp;`: 为 `argv` 数组 (包括末尾的NULL指针) 分配空间，`argv` 指针指向此数组。
    4.  **压栈**:
        *   `put_fs_long((unsigned long)envp, --sp);`: 将 `envp` 数组的地址压栈。
        *   `put_fs_long((unsigned long)argv, --sp);`: 将 `argv` 数组的地址压栈。
        *   `put_fs_long((unsigned long)argc, --sp);`: 将 `argc` 压栈。
    5.  **填充 `argv` 和 `envp` 数组**:
        *   循环 `argc` 次，将参数字符串的起始地址 (`p` 在填充过程中会向低地址移动指向每个字符串的开头) 存入 `argv` 数组。`p` 在每次存入后跳过该字符串。
        *   `put_fs_long(0, argv);`: `argv` 数组以NULL指针结尾。
        *   类似地填充 `envp` 数组。
    6.  返回最终计算出的新栈顶指针 `sp`。

### `static int count(char ** argv)`

*   **功能**: 计算字符串数组 `argv` (如参数列表或环境变量列表) 中的字符串数量。数组以NULL指针结束。
*   **实现**: 遍历 `argv` 数组，直到遇到一个NULL指针，计数器 `i` 即为字符串数量。使用 `get_fs_long` 从用户空间读取指针。

### `static unsigned long copy_strings(int argc, char ** argv, unsigned long *page, unsigned long p, int from_kmem)`

*   **功能**: 将 `argc` 个字符串从 `argv` (可以指向用户空间或内核空间) 复制到新进程用户栈的预留页面区域中 (由 `page` 数组管理，`p` 是当前在此区域的偏移指针，从高地址向低地址填充)。
*   **参数 `from_kmem`**:
    *   `0`: `argv` 本身在用户空间，它指向的字符串也在用户空间 (例如，原始的 `argv` 和 `envp`)。
    *   `1`: `argv` 本身在内核空间 (例如 `filename` 这种单个字符串的地址)，它指向的字符串在用户空间 (实际是内核准备的，但要通过用户段访问)。
    *   `2`: `argv` 本身在内核空间，它指向的字符串也在内核空间 (例如 `i_name`, `i_arg` 这种解释器名称和参数)。
*   **步骤**:
    1.  根据 `from_kmem` 设置段寄存器 `fs` 来正确访问源数据。
    2.  循环 `argc` 次，每次处理一个字符串：
        a.  获取字符串指针 `tmp`。
        b.  计算字符串长度 `len` (包括末尾的 `\0`)。
        c.  检查预留的128KB空间是否足够 (`p-len < 0`)。
        d.  从后向前 (`--p; --tmp; --len;`) 复制字符串字节：
            *   `offset = p % PAGE_SIZE;`: 计算在当前目标页内的偏移。
            *   如果需要新页 (`--offset < 0`):
                *   从 `page[p/PAGE_SIZE]` 获取页，如果尚未分配 (`!pag`)，则 `get_free_page()` 分配一个新页并存入 `page` 数组。
            *   `*(pag + offset) = get_fs_byte(tmp);`: 从源 `tmp` 读取一字节，写入目标页 `pag` 的 `offset` 位置。
    3.  恢复 `fs` 段寄存器。
    4.  返回更新后的偏移指针 `p`。

### `static unsigned long change_ldt(unsigned long text_size, unsigned long * page)`

*   **功能**: 修改当前进程的LDT (局部描述符表)，以适应新程序代码段和数据/栈段的内存布局。同时，将之前为参数和环境变量复制而分配的页面 `page[]` 映射到新数据段的顶端。
*   **步骤**:
    1.  **计算代码段和数据段的限长和基址**:
        *   `code_limit = text_size + PAGE_SIZE - 1; code_limit &= 0xFFFFF000;`: 代码段限长向上舍入到页边界。
        *   `data_limit = 0x4000000;`: 数据段/栈段的总限长设为64MB (Linux 0.11中用户空间的最大大小)。
        *   `code_base = get_base(current->ldt[1]);`: 获取LDT中代码段的原基址 (通常是 `task_struct` 中进程号乘以64MB)。
        *   `data_base = code_base;`: 数据段和代码段使用相同的基址。
    2.  **设置LDT**:
        *   `set_base(current->ldt[1], code_base); set_limit(current->ldt[1], code_limit);`: 设置LDT中代码段描述符 (索引1) 的基址和限长。
        *   `set_base(current->ldt[2], data_base); set_limit(current->ldt[2], data_limit);`: 设置LDT中数据/栈段描述符 (索引2) 的基址和限长。
    3.  **更新 `fs` 段寄存器**: `__asm__("pushl $0x17\n\tpop %%fs"::);`。`0x17` 是新的数据段选择子 (索引2, TI=1, RPL=3)。这确保后续的 `put_page` 使用新的数据段。
    4.  **映射参数/环境变量页面**:
        *   `data_base += data_limit;`: `data_base` 现在指向数据段的末尾 (即用户空间最高地址 `基址+64MB`)。
        *   循环 `MAX_ARG_PAGES` 次，从后向前 (`i--`):
            *   `data_base -= PAGE_SIZE;`: `data_base` 向下移动一个页。
            *   `if (page[i]) put_page(page[i], data_base);`: 如果 `page[i]` (之前 `copy_strings` 分配的物理页) 存在，则将其映射到当前 `data_base` 指向的线性地址。`put_page` 会建立页表映射。
    5.  返回数据段的限长 `data_limit` (64MB)。

### `int do_execve(unsigned long * eip, long tmp, char * filename, char ** argv, char ** envp)`

*   **功能**: `execve` 系统调用的核心实现函数。
*   **参数**:
    *   `unsigned long * eip`: 指向内核栈上保存的用户态寄存器 `eip` 的指针。`eip[0]` 是用户态 `eip`，`eip[1]` 是用户态 `cs`。此函数会修改 `eip[0]` (新程序的入口点) 和 `eip[3]` (新程序的栈顶 `esp`)。
    *   `long tmp`: 未使用。
    *   `char * filename`: 要执行的程序文件名。
    *   `char ** argv`: 参数列表。
    *   `char ** envp`: 环境变量列表。
*   **步骤**:
    1.  **权限检查**: `if ((0xffff & eip[1]) != 0x000f) panic(...)`: 确保是从用户模式 (CS选择子 RPL=3, 即 `0x000f`) 调用。
    2.  **初始化**: 清空 `page` 数组 (用于存储参数/环境变量的物理页地址)。`p` 指向参数页区域的顶端。
    3.  **获取可执行文件inode**: `if (!(inode = namei(filename))) return -ENOENT;`
    4.  **计算参数/环境变量数量**: `argc = count(argv); envc = count(envp);`
    5.  **`restart_interp:` 标签**: 用于处理 `#!` 解释器时跳转回来。
    6.  **文件类型和权限检查**:
        *   必须是常规文件 (`S_ISREG`)。
        *   计算有效用户ID `e_uid` 和有效组ID `e_gid` (考虑S_ISUID, S_ISGID位)。
        *   检查执行权限 (根据文件所有者、组和其他用户的权限位)。如果没有执行权限且不是超级用户，返回 `-ENOEXEC` (这里应该是 `-EACCES`，但0.11代码是 `-ENOEXEC`)。
    7.  **读取文件头**: `if (!(bh = bread(inode->i_dev, inode->i_zone[0]))) ...`: 读取文件的第一个数据块 (通常包含 `a.out` 头部或 `#!` 行)。
    8.  **处理 `#!` (shebang)**:
        *   `if ((bh->b_data[0] == '#') && (bh->b_data[1] == '!') && (!sh_bang))`: 如果文件以 `#!` 开头且尚未处理过shebang (`sh_bang` 标志)。
        *   解析解释器名称 `interp` 和可选参数 `i_arg`。
        *   `if (sh_bang++ == 0)`: 首次处理shebang时，复制原始的环境变量和参数 (除了 `argv[0]`) 到 `page` 区域。
        *   **重新构建参数列表**: 将脚本文件名、解释器可选参数 (如果有)、解释器名称本身 (作为 `argv[0]`) 的顺序（反向）复制到 `page` 区域。
        *   `argc` 相应增加。
        *   获取解释器文件的inode: `if (!(inode = namei(interp))) ...`
        *   `goto restart_interp;`: 用解释器的inode重新开始执行流程 (权限检查、读头等)。
    9.  **处理 `a.out` 格式**:
        *   `ex = *((struct exec *) bh->b_data); brelse(bh);`: 从缓冲区获取 `exec` 头部结构。
        *   **格式校验**:
            *   `N_MAGIC(ex) != ZMAGIC`: 必须是 `ZMAGIC` (支持需求加载的纯代码段格式)。
            *   `ex.a_trsize || ex.a_drsize`: 不支持重定位信息。
            *   `ex.a_text + ex.a_data + ex.a_bss > 0x3000000`: 总大小不能超过用户空间的一半 (约48MB，因为Linux 0.11用户空间总共64MB，这里限制可能是考虑内核和其他开销)。
            *   `inode->i_size < ex.a_text + ex.a_data + ex.a_syms + N_TXTOFF(ex)`: 文件实际大小必须足够包含头部、代码、数据和符号表。
            *   `N_TXTOFF(ex) != BLOCK_SIZE`: 代码段在文件中的偏移必须是 `BLOCK_SIZE`。
            *   任何不匹配则返回 `-ENOEXEC`。
    10. **复制参数和环境变量 (如果不是shebang二次进入)**:
        *   `if (!sh_bang) { p = copy_strings(envc,envp,page,p,0); p = copy_strings(argc,argv,page,p,0); ... }`
    11. **“无法返回”点 (Point of No Return)**: 从这里开始，旧进程的内存映像将被销毁。
        *   `if (current->executable) iput(current->executable); current->executable = inode;`: 释放旧的可执行文件inode的引用，并让当前进程指向新的可执行文件inode (用于需求加载)。
        *   **重置信号处理**: `for (i=0 ; i<32 ; i++) current->sigaction[i].sa_handler = NULL;`
        *   **处理 `close_on_exec`**: `for (i=0 ; i<NR_OPEN ; i++) if ((current->close_on_exec>>i)&1) sys_close(i);` 关闭标记了 `FD_CLOEXEC` 的文件描述符。
        *   `free_page_tables(...)`: 释放旧进程代码段和数据段占用的页表。
        *   处理FPU状态: `if (last_task_used_math == current) last_task_used_math = NULL; current->used_math = 0;`
    12. **设置新LDT并映射参数页面**:
        *   `p += change_ldt(ex.a_text, page) - MAX_ARG_PAGES*PAGE_SIZE;`: 调用 `change_ldt` 设置新的LDT，并将参数页面映射到新数据段的顶端。`p` 调整为指向参数/环境变量字符串区域的起始处 (在新的用户虚拟地址空间)。
    13. **创建参数表和设置新栈顶**:
        *   `p = (unsigned long) create_tables((char *)p, argc, envc);`: 在新的用户栈上创建 `argv`, `envp` 指针表，`p` 更新为新的栈顶指针 (`esp`)。
    14. **更新进程内存信息**:
        *   `current->brk = ex.a_bss + (current->end_data = ex.a_data + (current->end_code = ex.a_text));`: 设置代码段结束 (`end_code`)、数据段结束 (`end_data`)、BSS段结束 (`brk`，即初始堆顶)。这些都是相对于进程虚拟地址空间基址的偏移。
        *   `current->start_stack = p & 0xfffff000;`: 记录栈的起始页框。
    15. **设置EUID/EGID**: `current->euid = e_uid; current->egid = e_gid;`
    16. **BSS段清零 (需求加载的一部分)**:
        *   `i = ex.a_text + ex.a_data; while (i & 0xfff) put_fs_byte(0, (char *)(i++));`
            *   这行代码的目的是确保从数据段结束到下一个页边界的少量空间被清零。真正的BSS段（可能跨越多个页面）会在发生页错误时，由 `mm/memory.c` 中的 `do_no_page()` 通过 `get_empty_page()` 获取清零页来分配。
    17. **设置返回到用户态的EIP和ESP**:
        *   `eip[0] = ex.a_entry;`: 新程序的入口点。
        *   `eip[3] = p;`: 新程序的栈顶指针。
    18. **成功返回**: `return 0;` (虽然这个0值不会直接返回给用户，因为栈和指令指针都变了)。
    19. **错误处理 (`exec_error2`, `exec_error1`)**:
        *   `iput(inode);`: 释放对可执行文件inode的引用。
        *   `for (i=0 ; i<MAX_ARG_PAGES ; i++) free_page(page[i]);`: 释放为参数/环境变量分配的物理页。
        *   返回错误码 `retval`。

## 总结

`exec.c` 通过一系列复杂但有序的步骤，实现了 `execve()` 系统调用，完成了从一个正在运行的程序到另一个新程序的彻底转变。它涉及了文件系统操作（查找inode，读取文件头）、内存管理（LDT更改，页表释放，参数页面映射，需求加载设置）、进程属性修改（EUID/EGID，信号，文件描述符）以及用户空间的重新构建（栈，参数，环境变量）。`#!` 脚本解释机制和需求加载是其显著特性，体现了Linux早期在兼容性和效率方面的努力。这个文件是理解Linux进程如何执行新程序的关键。Okay, I have created `docs/fs/exec.md` with the detailed Chinese explanation for `fs/exec.c`.

Next, I will process `fs/fcntl.c`.
1. Create `docs/fs/fcntl.md`.
2. Read `fs/fcntl.c`.
3. Write the explanation into `docs/fs/fcntl.md`.

First, I'll read `fs/fcntl.c`.
