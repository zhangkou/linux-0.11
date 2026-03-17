# 第03章：硬件初始化 setup.s

> 源码文件：`boot/setup.s`
> 运行环境：实模式，物理地址 0x90200
> 核心任务：收集硬件信息，从实模式切换到保护模式

---

## 3.1 概述

setup.s 是实模式下最后运行的代码，它是实模式与保护模式之间的桥梁。

**主要工作：**
1. 通过 BIOS 中断收集硬件参数（内存大小、磁盘参数、视频模式等）
2. 将收集的参数存储在 0x90000 处（临时保存区）
3. 将内核 system 从 0x10000 **移动**到 0x00000（低地址）
4. 建立初始 GDT（全局描述符表）
5. **开启 A20 地址线**（允许访问 > 1MB 的内存）
6. 重新编程 8259A 中断控制器（避免中断冲突）
7. 设置 CR0 寄存器的 PE 位，正式进入保护模式
8. 跳转到 head.s（0x00000）执行

---

## 3.2 硬件信息收集区布局

setup.s 把所有收集到的数据保存在 0x90000 开始的区域（原来 bootsect 存放的地方）：

```
0x90000  +──────────────────────+
         │  光标位置 (2字节)     │ 偏移 0x00 - DX (DH=行, DL=列)
         │                      │
0x90002  │  扩展内存大小 (2字节) │ 偏移 0x02 - KB为单位的扩展内存
         │                      │
0x90004  │  显示页号 (2字节)     │ 偏移 0x04 - BH=页号
0x90006  │  视频模式 (2字节)     │ 偏移 0x06 - AL=视频模式, AH=列宽
         │                      │
0x90008  │  EGA/VGA信息 (6字节) │ 偏移 0x08-0x0D
         │                      │
0x90080  │  硬盘0参数 (16字节)  │ 偏移 0x80 - 从BIOS向量表复制
0x90090  │  硬盘1参数 (16字节)  │ 偏移 0x90
         │                      │
0x901FC  │  ROOT_DEV (2字节)    │ 偏移 0x1FC - bootsect遗留的根设备
0x901FE  │  boot_flag (2字节)   │ 偏移 0x1FE - 0xAA55
         +──────────────────────+
```

**main.c 中如何访问这些数据：**
```c
// init/main.c 第58~60行
#define EXT_MEM_K    (*(unsigned short *)0x90002)  // 扩展内存KB数
#define DRIVE_INFO   (*(struct drive_info *)0x90080) // 硬盘参数
#define ORIG_ROOT_DEV (*(unsigned short *)0x901FC)  // 根设备号
```

---

## 3.3 逐段代码解析

### 第一步：保存光标位置

```asm
; setup.s 第36~41行
start:
    mov ax, #INITSEG    ; AX = 0x9000
    mov ds, ax          ; DS = 0x9000 (数据目的地)
    mov ah, #0x03       ; BIOS int 0x10 AH=3: 读光标位置
    xor bh, bh          ; BH = 0 (第0显示页)
    int 0x10            ; 返回: DH=行号, DL=列号
    mov [0], dx         ; 保存到 DS:0 = 0x90000
```

### 第二步：获取扩展内存大小

```asm
; setup.s 第43~47行
    mov ah, #0x88       ; BIOS int 0x15 AH=0x88: 获取扩展内存大小
    int 0x15            ; 返回: AX = 1MB以上的内存KB数
                        ; 例如: 7168 = 7MB扩展内存 → 总计8MB
    mov [2], ax         ; 保存到 DS:2 = 0x90002
```

**注意：** 这里返回的是 **1MB 以上**的内存，不包含低端1MB。
总内存 = 1MB + AX × 1024 字节。

### 第三步：获取显示卡参数

```asm
; setup.s 第49~63行
    mov ah, #0x0f       ; BIOS int 0x10 AH=0x0F: 获取当前视频模式
    int 0x10
    mov [4], bx         ; BH=显示页, 保存到 0x90004
    mov [6], ax         ; AL=视频模式, AH=列宽, 保存到 0x90006

    mov ah, #0x12       ; BIOS int 0x10 AH=0x12: EGA/VGA子功能
    mov bl, #0x10       ; BL=0x10: 返回EGA配置信息
    int 0x10
    mov [8], ax         ; 保存到 0x90008
    mov [10], bx        ; BH=颜色/单色标志, BL=已安装内存(0=64K,1=128K...)
    mov [12], cx        ; CH=特征位, CL=开关设置
```

### 第四步：获取硬盘参数

```asm
; setup.s 第65~87行（获取 hd0 参数）
    mov ax, #0x0000
    mov ds, ax          ; DS = 0x0000 (BIOS中断向量表所在)
    lds si, [4*0x41]    ; 从中断向量表加载 INT 0x41 的向量
                        ; INT 0x41 指向第一块硬盘参数表
                        ; DS:SI = 硬盘参数表物理地址
    mov ax, #INITSEG
    mov es, ax          ; ES = 0x9000
    mov di, #0x0080     ; ES:DI = 0x9000:0x0080 = 0x90080
    mov cx, #0x10       ; 复制16字节
    rep
    movsb               ; 将硬盘参数表从 DS:SI 复制到 ES:DI
```

**硬盘参数表结构（16字节）：**
```
偏移0-1:  柱面数
偏移2:    磁头数
偏移3-4:  写预补偿起始柱面
偏移5:    每磁道扇区数（最大扇区数）
偏移6:    控制字节
偏移7-9:  停靠柱面
偏移10-11: 落笔柱面
偏移12:   扇区数（磁头每转扇区数）
```

```asm
; 获取 hd1 参数（类似）
    lds si, [4*0x46]    ; INT 0x46 指向第二块硬盘参数表
    mov di, #0x0090     ; 保存到 0x90090
    mov cx, #0x10
    rep
    movsb

; 检查是否真的有第二块硬盘
    mov ax, #0x01500    ; BIOS int 0x13 AH=0x15: 获取磁盘类型
    mov dl, #0x81       ; DL=0x81 表示第二块硬盘
    int 0x13
    jc no_disk1         ; CF=1 表示错误（无此盘）
    cmp ah, #3          ; AH=3 表示是硬盘
    je is_disk1
no_disk1:
    ; 如果没有第二块硬盘，将 0x90090 的参数清零
    mov di, #0x0090
    mov cx, #0x10
    mov ax, #0x00
    rep
    stosb               ; 用0填充16字节
is_disk1:
```

### 第五步：移动 system 到 0x00000

```asm
; setup.s 第113~126行
    cli                 ; 关中断！此后不允许中断
    mov ax, #0x0000
    cld                 ; 方向标志清零（向高地址复制）
do_move:
    mov es, ax          ; ES = 目标段（从0x0000开始）
    add ax, #0x1000     ; AX += 0x1000
    cmp ax, #0x9000     ; 是否到达0x9000段（即0x90000）？
    jz end_move         ; 是则结束
    mov ds, ax          ; DS = 源段（从0x1000开始，即物理0x10000）
    sub di, di          ; DI = 0
    sub si, si          ; SI = 0
    mov cx, #0x8000     ; 复制 0x8000 个字 = 64KB
    rep
    movsw               ; 复制 DS:SI → ES:DI
    jmp do_move         ; 继续下一段（每次移动64KB）
```

**为什么要移动？**
- setup.s 加载时，system 在 0x10000（64KB处）
- 但内核期望运行在 0x00000（物理地址0）
- head.s 的第一条指令就在 0x00000

**移动过程示意：**
```
移动前：0x10000~0x8FFFF = 内核（最大 0x80000 = 512KB）
移动后：0x00000~0x7FFFF = 内核
```

### 第六步：加载 GDT 和 IDT

```asm
; setup.s 第130~135行
end_move:
    mov ax, #SETUPSEG   ; DS = 0x9020（setup自己的段）
    mov ds, ax
    lidt idt_48         ; 加载IDT描述符（空IDT，limit=0）
    lgdt gdt_48         ; 加载GDT描述符
```

**setup.s 中定义的 GDT：**
```asm
; setup.s 第205~216行
gdt:
    .word 0,0,0,0       ; GDT[0]: 空描述符（规定必须为0）

    ; GDT[1]: 内核代码段
    .word 0x07FF        ; limit低16位 = 0x7FF → limit = 0x7FF (段界限)
                        ; 粒度G=1(4KB), 所以 size = (0x7FF+1)*4KB = 8MB
    .word 0x0000        ; base低16位 = 0
    .word 0x9A00        ; P=1(存在), DPL=0(内核), S=1, Type=A(代码可读执行)
    .word 0x00C0        ; G=1(4KB粒度), D=1(32位), base高字节=0

    ; GDT[2]: 内核数据段
    .word 0x07FF        ; 同上，8MB
    .word 0x0000        ; base = 0
    .word 0x9200        ; P=1, DPL=0, S=1, Type=2(数据可读写)
    .word 0x00C0        ; G=1, D=1
```

**GDT 描述符格式（8字节）：**
```
字节7  字节6  字节5  字节4  字节3  字节2  字节1  字节0
┌─────┬──────┬─────┬─────┬─────┬──────────────┬──────────────┐
│Base │G|D|0 │P|DPL │Limit│Base │    Base      │    Limit     │
│31:24│|X|A │S|Type│19:16│23:16│    15:0      │    15:0      │
└─────┴──────┴─────┴─────┴─────┴──────────────┴──────────────┘

G  = 粒度 (0=字节, 1=4KB)
D  = 操作数宽度 (0=16位, 1=32位)
P  = 段存在位
DPL = 描述符特权级 (0=最高内核, 3=用户)
S  = 描述符类型 (0=系统段, 1=代码/数据段)
```

### 第七步：开启 A20 地址线

```asm
; setup.s 第138~144行
    call empty_8042     ; 等待键盘控制器就绪
    mov al, #0xD1       ; 命令：写输出端口
    out #0x64, al       ; 发送到键盘控制器命令端口
    call empty_8042     ; 等待命令被接收
    mov al, #0xDF       ; 数据：A20使能位(bit1=1=A20打开)
    out #0x60, al       ; 发送到键盘控制器数据端口
    call empty_8042     ; 等待完成
```

**为什么需要开启 A20？**

```
历史原因：IBM PC/AT 的 8086 只有 20 根地址线（A0~A19），
访问超过 1MB 地址时，地址会自动"回绕"（如访问 0x100000
实际访问 0x000000）。

80286 及以后有更多地址线，但为兼容性，A20 线默认被
键盘控制器芯片（8042）封闭。

不开启A20：地址第20位永远为0 → 无法访问 > 1MB
开启A20：所有地址线正常工作 → 可访问全部内存
```

**empty_8042 辅助函数：**
```asm
; setup.s 第198~203行
empty_8042:
    .word 0x00eb, 0x00eb  ; 两条短跳转 (jmp $+2) = 延时
    in al, #0x64          ; 读键盘控制器状态端口
    test al, #2           ; 测试 bit1 (输入缓冲区满标志)
    jnz empty_8042        ; 若满则等待（控制器忙）
    ret
```

### 第八步：重新编程 8259A 中断控制器

```asm
; setup.s 第154~179行
; 8259A-1 (主): 硬件中断重映射到 0x20~0x27
; 8259A-2 (从): 硬件中断重映射到 0x28~0x2F

    mov al, #0x11       ; ICW1: 边沿触发，级联，需要ICW4
    out #0x20, al       ; → 主8259命令端口
    .word 0x00eb, 0x00eb  ; IO延时
    out #0xA0, al       ; → 从8259命令端口
    .word 0x00eb, 0x00eb

    mov al, #0x20       ; ICW2: 主8259中断起始向量 = 0x20
    out #0x21, al       ; (IRQ0对应INT 0x20, IRQ7对应INT 0x27)
    .word 0x00eb, 0x00eb
    mov al, #0x28       ; ICW2: 从8259中断起始向量 = 0x28
    out #0xA1, al       ; (IRQ8对应INT 0x28, IRQ15对应INT 0x2F)
    .word 0x00eb, 0x00eb

    mov al, #0x04       ; ICW3: 主片，从片连接在 IRQ2
    out #0x21, al
    .word 0x00eb, 0x00eb
    mov al, #0x02       ; ICW3: 从片，连接到主片 IRQ2
    out #0xA1, al
    .word 0x00eb, 0x00eb

    mov al, #0x01       ; ICW4: 8086模式
    out #0x21, al
    .word 0x00eb, 0x00eb
    out #0xA1, al
    .word 0x00eb, 0x00eb

    mov al, #0xFF       ; OCW1: 屏蔽所有中断
    out #0x21, al
    .word 0x00eb, 0x00eb
    out #0xA1, al
```

**为什么需要重新编程 8259A？**

```
BIOS 设置（有冲突）：         Linux 需要：
IRQ0 (定时器)  → INT 0x08    IRQ0 → INT 0x20
IRQ1 (键盘)    → INT 0x09    IRQ1 → INT 0x21
...                           ...

问题：INT 0x08~0x0F 是 Intel CPU 的内部异常！
  INT 0x08 = 双重故障 (Double Fault)
  INT 0x09 = 协处理器段越界

解决：将硬件中断重映射到 INT 0x20~0x2F (安全范围)
```

**重映射后的中断分配：**
```
INT 0x00~0x1F: CPU异常（除零、缺页、双重故障等）
INT 0x20: 定时器 (IRQ0, 100Hz)
INT 0x21: 键盘 (IRQ1)
INT 0x22: 从8259级联 (IRQ2)
INT 0x23: 串口2 (IRQ3)
INT 0x24: 串口1 (IRQ4)
INT 0x25: 并口2 (IRQ5)
INT 0x26: 软盘 (IRQ6)
INT 0x27: 并口1 (IRQ7)
INT 0x28: 实时时钟 (IRQ8)
INT 0x2E: 硬盘 (IRQ14)
INT 0x80: 系统调用 (Linux专用)
```

### 第九步：进入保护模式

```asm
; setup.s 第191~193行
    mov ax, #0x0001     ; AX = 1
    lmsw ax             ; 加载机器状态字：CR0 的低16位 = AX
                        ; 设置 CR0.PE = 1 (Protection Enable)
                        ; 这一行让CPU从实模式进入保护模式！
    jmpi 0, 8          ; 远跳: CS = 8 (GDT[1] 内核代码段选择子)
                        ;        IP = 0 (偏移0)
                        ; 物理地址 = GDT[1].base + 0 = 0x00000
                        ; 即跳转到 head.s 的第一条指令
```

**`lmsw ax` 与 `mov cr0, eax` 的关系：**
- `lmsw` (Load Machine Status Word) 是旧指令，只修改CR0低16位
- 等价于设置 `CR0 |= 0x0001`（只设PE位）
- 进入保护模式后 CPU 立即重新解释段寄存器为选择子

**`jmpi 0, 8` 的含义：**
```
CS = 8 = 0b00001000
         └─────┘└┘
         索引=1  TI=0(GDT)  RPL=0(内核)

→ 使用 GDT[1] (内核代码段)
→ 基址=0x00000，加上偏移0
→ 物理地址 = 0x00000 → head.s
```

---

## 3.4 完整执行流程图

```
setup.s start (0x90200)
    │
    ├── 读光标位置 → [0x90000]
    ├── 读扩展内存 → [0x90002]
    ├── 读视频参数 → [0x90004-0x0D]
    ├── 读硬盘0参数 → [0x90080]
    ├── 读硬盘1参数 → [0x90090]
    │
    ├── CLI (关中断)
    │
    ├── 移动 system: 0x10000~0x8FFFF → 0x00000~0x7FFFF
    │
    ├── LIDT (空IDT, 此时不处理中断)
    ├── LGDT (建立临时GDT: NULL + 代码段8MB + 数据段8MB)
    │
    ├── 开启 A20 地址线 (通过8042键盘控制器)
    │
    ├── 重编程 8259A (IRQ→INT 0x20~0x2F)
    ├── 屏蔽所有IRQ
    │
    ├── CR0.PE = 1  ← 进入保护模式！
    │
    └── jmpi 0, 8  → head.s (0x00000)
```

---

## 3.5 关键技术总结

### A20地址线的意义
```
开启前: 0x100000 访问实际上访问 0x000000（回绕）
开启后: 0x100000 正确访问 1MB以上的内存
```

### 保护模式切换的不可逆性
一旦设置 CR0.PE=1，CPU 就进入保护模式。
此时所有段寄存器都被解释为选择子，必须立即执行远跳来刷新CS。

### 为什么要在进保护模式前移动内核
因为移动操作使用实模式的 BIOS 兼容内存模型。
保护模式下内存布局不同，移动会更复杂。
进入保护模式后，0x10000 处的原始副本就不再使用了。

---

**下一章：[第04章 - 保护模式初始化 head.s](04_保护模式初始化_head.md)**

*head.s 将建立完整的分页机制，让内核进入真正的虚拟地址空间*
