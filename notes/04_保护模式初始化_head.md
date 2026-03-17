# 第04章：保护模式初始化 head.s

> 源码文件：`boot/head.s`
> 运行环境：32位保护模式，物理地址 0x00000
> 核心任务：设置段寄存器，建立 IDT/GDT，开启分页，跳入 main()

---

## 4.1 概述

head.s 是 system 内核镜像的最开头，加载到物理地址 **0x00000** 处运行。
它是第一段在保护模式下执行的 32 位代码。

**主要工作：**
1. 重新设置段寄存器（使用正式的保护模式选择子）
2. 设置栈指针
3. 建立完整的 IDT（256个中断描述符）
4. 建立完整的 GDT（含 TSS、LDT 预留空间）
5. 检测数学协处理器 (x87 FPU)
6. 验证 A20 是否真正开启
7. **建立页目录和4张页表**（恒等映射前 16MB 物理内存）
8. **开启分页**（设置 CR3 和 CR0.PG）
9. 跳转到 `main()`

---

## 4.2 head.s 的特殊地位

```
物理地址
0x00000 ┌─────────────────────┐
        │  _pg_dir (页目录)    │ ← head.s 从这里开始!
        │  = startup_32       │   分页开启后，这里被页目录覆盖!
        │  (head.s代码)        │
0x01000 ├─────────────────────┤
        │  pg0 (第一张页表)    │ 4KB
0x02000 ├─────────────────────┤
        │  pg1                 │ 4KB
0x03000 ├─────────────────────┤
        │  pg2                 │ 4KB
0x04000 ├─────────────────────┤
        │  pg3                 │ 4KB
0x05000 ├─────────────────────┤
        │  _tmp_floppy_area    │ 1KB (DMA缓冲区)
        ├─────────────────────┤
        │  内核其余代码/数据   │
        │  (C语言编译的部分)   │
        └─────────────────────┘
```

**关键事实：** head.s 的代码本身会被页目录和页表**覆盖**！setup_idt 和 setup_gdt 函数在 0x1000 之前，分页开启后它们的位置变成了 pg0，但此时这些函数已经执行完毕，不再需要了。

---

## 4.3 逐行代码解析

### 第一步：初始化段寄存器

```asm
; head.s 第17~23行
.text
.globl _idt, _gdt, _pg_dir, _tmp_floppy_area
_pg_dir:                    ; 页目录将放在这里 (地址=0x00000)
startup_32:
    movl $0x10, %eax        ; 0x10 = 选择子 (GDT[2] = 数据段)
                            ; 0b00010000: 索引=2, TI=0(GDT), RPL=0
    mov %ax, %ds            ; DS = 数据段选择子
    mov %ax, %es
    mov %ax, %fs
    mov %ax, %gs
    lss _stack_start, %esp  ; 加载栈指针
                            ; _stack_start 定义在 kernel/sched.c 中
                            ; SS:ESP = 内核栈
```

**_stack_start 的定义（kernel/sched.c）：**
```c
long user_stack[PAGE_SIZE>>2];  // 4096/4 = 1024 个 long = 4KB 栈空间
struct {
    long *a;        // 栈顶指针
    short b;        // 段选择子
} stack_start = {
    &user_stack[PAGE_SIZE>>2],   // 栈顶 = user_stack 数组末尾
    0x10                          // 段选择子 = 数据段 (0x10)
};
```

### 第二步：建立 IDT

```asm
; head.s 第23~24行
    call setup_idt      ; 设置IDT
    call setup_gdt      ; 设置GDT
```

**setup_idt 详解：**
```asm
; head.s 第78~93行
setup_idt:
    lea ignore_int, %edx    ; EDX = ignore_int 函数地址
                            ; (所有中断暂时指向这个哑处理器)
    movl $0x00080000, %eax  ; EAX 高16位 = 0x0008 (代码段选择子)
    movw %dx, %ax           ; EAX 低16位 = ignore_int 低16位地址
                            ; EAX 现在是中断门描述符的低4字节

    movw $0x8E00, %dx       ; DX = 0x8E00 (中断门属性)
                            ; P=1(存在), DPL=0(内核), 类型=0xE(32位中断门)
                            ; EDX = 中断门描述符的高4字节（含地址高16位）

    lea _idt, %edi          ; EDI = IDT 基地址
    mov $256, %ecx          ; 设置256个门描述符
rp_sidt:
    movl %eax, (%edi)       ; 写入低4字节
    movl %edx, 4(%edi)      ; 写入高4字节
    addl $8, %edi           ; 移到下一个描述符
    dec %ecx
    jne rp_sidt
    lidt idt_descr          ; 加载 IDT (256个门, 共2048字节)
    ret
```

**中断/陷阱门描述符格式（8字节）：**
```
高4字节:
┌──────────┬───┬──────────┐
│ 偏移31:16 │属性│  (保留)   │
└──────────┴───┴──────────┘

低4字节:
┌──────────┬──────────────┐
│ 段选择子  │  偏移15:0    │
└──────────┴──────────────┘

属性字节:
P=1    : 描述符有效
DPL=0  : 特权级0(内核)
Type=E : 32位中断门 (0x1110)
```

**ignore_int 函数（默认中断处理器）：**
```asm
; head.s 第150~170行
ignore_int:
    pushl %eax          ; 保存寄存器
    pushl %ecx
    pushl %edx
    push %ds
    push %es
    push %fs
    movl $0x10, %eax    ; 加载数据段
    mov %ax, %ds
    mov %ax, %es
    mov %ax, %fs
    pushl $int_msg      ; 参数: 字符串地址
    call _printk        ; 打印 "Unknown interrupt"
    popl %eax
    pop %fs
    pop %es
    pop %ds
    popl %edx
    popl %ecx
    popl %eax
    iret                ; 从中断返回
```

### 第三步：检测数学协处理器

```asm
; head.s 第43~65行
    movl %cr0, %eax
    andl $0x80000011, %eax  ; 保留 PG, PE, ET 位
    orl $2, %eax            ; 设置 MP (Math Present) 位
    movl %eax, %cr0
    call check_x87
    jmp after_page_tables

check_x87:
    fninit                  ; 初始化 FPU（无等待）
    fstsw %ax               ; 将FPU状态字存到AX（无等待）
    cmpb $0, %al
    je 1f                   ; 状态字为0 → 有协处理器
    ; 没有协处理器：设置 EM 位（使用软件模拟）
    movl %cr0, %eax
    xorl $6, %eax           ; 清MP位, 设置EM(Emulate Math)位
    movl %eax, %cr0
    ret
.align 2
1:  .byte 0xDB, 0xE4        ; fsetpm 指令（对387无效，仅针对287）
    ret
```

### 第四步：建立分页机制

这是 head.s 最重要的部分——建立页目录和页表，实现物理地址的**恒等映射**。

```asm
; head.s 第135~141行：将 main() 的参数和返回地址压栈
after_page_tables:
    pushl $0        ; 参数3 (envp=NULL)
    pushl $0        ; 参数2 (argv=NULL)
    pushl $0        ; 参数1 (argc=0)
    pushl $L6       ; main() 的"返回地址" (如果main返回则死循环)
    pushl $_main    ; main() 函数地址（作为call的返回地址）
    jmp setup_paging ; 跳到分页设置

L6: jmp L6          ; 如果main()意外返回，死循环
```

**setup_paging 详解：**
```asm
; head.s 第198~218行
setup_paging:
    ; 第1步: 清零5页内存 (1个页目录 + 4个页表)
    movl $1024*5, %ecx  ; 5 × 1024 个 long = 5 × 4KB
    xorl %eax, %eax
    xorl %edi, %edi     ; 从物理地址 0x000 开始清零
    cld
    rep stosl           ; 每次填写一个 long (4字节) 的0

    ; 第2步: 填写页目录（4个条目对应4张页表）
    movl $pg0+7, _pg_dir        ; 页目录[0] = pg0地址 + 7
                                 ; +7 = 属性: P=1(存在) U/S=1(用户可访问) R/W=1(可写)
    movl $pg1+7, _pg_dir+4      ; 页目录[1] = pg1地址 + 7
    movl $pg2+7, _pg_dir+8      ; 页目录[2] = pg2地址 + 7
    movl $pg3+7, _pg_dir+12     ; 页目录[3] = pg3地址 + 7

    ; 第3步: 填写页表（从最后一项到第一项，反向填写）
    movl $pg3+4092, %edi    ; EDI = pg3 最后一个条目的地址
    movl $0xfff007, %eax    ; 最后一个物理页的地址 + 属性
                             ; 0xfff000 = 第4096个页 = 16MB - 4KB
                             ; +7 = P=1, U/S=1, R/W=1
    std                     ; 方向标志=1 (向低地址填写)
1:  stosl                   ; 存 EAX 到 ES:EDI，EDI -= 4
    subl $0x1000, %eax      ; 物理地址减一页(4KB)
    jge 1b                  ; 直到 EAX < 0 为止

    ; 第4步: 加载页目录基地址到 CR3
    xorl %eax, %eax
    movl %eax, %cr3         ; CR3 = 0x00000 (页目录在物理地址0)

    ; 第5步: 开启分页
    movl %cr0, %eax
    orl $0x80000000, %eax   ; 设置 CR0.PG 位 (bit31)
    movl %eax, %cr0         ; ← 分页正式开启！
    ret                     ; 返回到 after_page_tables 的 pushl $_main
                            ; ret 从栈弹出 _main 地址，相当于 call main()
```

---

## 4.4 分页机制详解

### 建立的页表结构

```
页目录 (4KB, 物理地址 0x0000)
├── PDE[0] = 0x1007  → pg0 (物理地址 0x1000) + 属性7
├── PDE[1] = 0x2007  → pg1 (物理地址 0x2000) + 属性7
├── PDE[2] = 0x3007  → pg2 (物理地址 0x3000) + 属性7
├── PDE[3] = 0x4007  → pg3 (物理地址 0x4000) + 属性7
└── PDE[4..1023] = 0 (无效)

页表 pg0 (4KB, 物理地址 0x1000)
├── PTE[0]   = 0x000007  → 物理页 0x00000 (第1页)
├── PTE[1]   = 0x001007  → 物理页 0x01000 (第2页)
├── PTE[2]   = 0x002007  → 物理页 0x03000
├── ...
└── PTE[1023] = 0xFFF007 → 物理页 0xFFF000 (第4096页)
```

### 恒等映射（Identity Mapping）

head.s 建立的是**恒等映射**：线性地址 = 物理地址

```
线性地址 0x00000000 → 物理地址 0x00000000 (内核代码)
线性地址 0x00001000 → 物理地址 0x00001000 (pg0)
...
线性地址 0x00FFFFFF → 物理地址 0x00FFFFFF (16MB上限)
```

**映射范围：** 0x0000_0000 ~ 0x00FF_FFFF (16MB)

**为什么选恒等映射？**
- 内核代码本身就加载在低地址，不需要重映射
- 简单直接，启动时不需要复杂的虚拟地址管理
- 后续进程的地址空间通过各自的页表来区分

### 页目录/页表条目属性

```
低3位属性:
  bit0 = P   (Present): 1=页存在, 0=未映射/已换出
  bit1 = R/W  (Read/Write): 1=可读写, 0=只读
  bit2 = U/S  (User/Supervisor): 1=用户可访问, 0=仅内核

head.s 使用的属性 = 7 = 0b111:
  P=1 (存在)
  R/W=1 (可写)
  U/S=1 (用户可访问)
  → 所有页均可被用户空间访问（后续mm会收紧权限）
```

---

## 4.5 ret 跳转到 main() 的巧妙之处

```asm
after_page_tables:
    pushl $0            ; envp
    pushl $0            ; argv
    pushl $0            ; argc
    pushl $L6           ; "返回地址"
    pushl $_main        ; main() 的地址 ← 当作返回地址

    jmp setup_paging    ; 执行分页设置

; setup_paging 结尾:
    ret                 ; pop EIP = $_main → 跳转到 main()
                        ; 同时 ESP 调整好，参数已在栈上
```

这是一个精巧的设计：
- 用 `ret` 代替 `call` 跳转到 main()
- 把 argc/argv/envp 提前压栈
- main() 按普通C函数调用约定获取参数
- 如果 main() 意外 `ret`，弹出 `$L6`，进入死循环

---

## 4.6 GDT 最终布局

head.s 中的最终 GDT（比 setup.s 中的更完整）：

```asm
; head.s 第234~238行
_gdt:
    .quad 0x0000000000000000    ; GDT[0]: NULL (必须为空)
    .quad 0x00c09a0000000fff    ; GDT[1]: 内核代码段 (16MB, DPL=0)
    .quad 0x00c0920000000fff    ; GDT[2]: 内核数据段 (16MB, DPL=0)
    .quad 0x0000000000000000    ; GDT[3]: 临时，不用
    .fill 252, 8, 0             ; GDT[4..255]: 预留给TSS和LDT
```

**各进程使用 GDT 的方式：**
```
GDT[0]  = NULL
GDT[1]  = 内核代码段 (CS=0x08)
GDT[2]  = 内核数据段 (SS/DS=0x10)
GDT[3]  = 临时
GDT[4]  = 进程0 TSS (TR=0x20)
GDT[5]  = 进程0 LDT (LDTR=0x28)
GDT[6]  = 进程1 TSS (TR=0x30)
GDT[7]  = 进程1 LDT (LDTR=0x38)
...
GDT[4+n*2] = 进程n TSS
GDT[5+n*2] = 进程n LDT
最多支持 (256-4)/2 = 126 个进程... 但 NR_TASKS=64 是限制
```

---

## 4.7 head.s 执行完后的内存状态

```
物理地址
0x00000 ┌───────────────────┐
        │  页目录 (4KB)      │ ← _pg_dir = startup_32 被覆盖!
0x01000 ├───────────────────┤
        │  pg0 (4KB)        │
0x02000 ├───────────────────┤
        │  pg1 (4KB)        │
0x03000 ├───────────────────┤
        │  pg2 (4KB)        │
0x04000 ├───────────────────┤
        │  pg3 (4KB)        │
0x05000 ├───────────────────┤
        │  floppy DMA区     │ (1KB)
0x05400 ├───────────────────┤
        │  内核C代码         │ (main,sched,fork等)
        ├───────────────────┤
        │  BSS段 (未初始化)  │
        ├───────────────────┤
        │  IDT (2KB)        │ 256×8字节
        ├───────────────────┤
        │  GDT (2KB)        │ 256×8字节
        ├───────────────────┤
        │  内核栈            │
~0x7FFF ├───────────────────┤
        │  可用内存           │ (setup.s收集的硬件数据在0x90000)
        └───────────────────┘
```

---

## 4.8 总结

| 操作 | 对应代码位置 | 效果 |
|------|-------------|------|
| 设置段寄存器 | startup_32 | 所有段均使用 GDT 描述符 |
| 设置内核栈 | `lss _stack_start` | 内核有了 4KB 栈空间 |
| 建立 IDT | setup_idt | 256个门均指向 ignore_int |
| 建立 GDT | setup_gdt | 加载最终 GDT |
| 检测 FPU | check_x87 | 决定是否启用数学模拟 |
| 验证 A20 | startup_32 中的循环 | 确认 A20 真正开启 |
| 建立页表 | setup_paging | 恒等映射前16MB |
| 开启分页 | `orl CR0.PG` | 虚拟内存正式生效 |
| 跳入 main | `ret` 弹出 `$_main` | 进入 C 语言内核 |

---

**下一章：[第05章 - 内核主函数 main.c](05_内核主函数_main.md)**

*main() 将完成所有子系统的初始化，创建进程0和进程1，系统进入运行状态*
