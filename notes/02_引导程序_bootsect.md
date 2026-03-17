# 第02章：引导程序 bootsect.s

> 源码文件：`boot/bootsect.s`
> 运行环境：实模式，物理地址 0x7C00（后搬到 0x90000）
> 代码大小：512字节（一个扇区）

---

## 2.1 概述

**bootsect.s** 是整个操作系统的第一段代码，由 BIOS 负责加载。

BIOS 在上电自检后，会：
1. 从启动设备（软盘/硬盘）读取**第一个扇区**（512字节）
2. 检查最后两字节是否为 `0xAA55`（引导标志）
3. 将该512字节复制到内存 **0x7C00** 处
4. 跳转执行

bootsect.s 的使命：
1. 把自己**搬到 0x90000**（腾出空间）
2. 加载 **setup.s** 到 0x90200
3. 加载 **system（内核）** 到 0x10000
4. 跳转到 setup.s

---

## 2.2 内存布局（bootsect 执行期间）

```
物理地址
0x00000 ┌──────────────────────┐
        │  BIOS中断向量表       │  (1KB)
0x00400 ├──────────────────────┤
        │  BIOS数据区           │
0x07C00 ├──────────────────────┤
        │  bootsect.s (512B)   │ ← BIOS加载到这里
0x07E00 ├──────────────────────┤
        │  (空闲)               │
0x10000 ├──────────────────────┤
        │  system (内核)        │ ← bootsect加载到这里
        │  最大 0x30000 = 192KB │
0x90000 ├──────────────────────┤
        │  bootsect 搬到这里    │ (512B, 覆盖旧内容)
0x90200 ├──────────────────────┤
        │  setup.s 加载到这里   │ (4扇区 = 2048B)
0x90A00 ├──────────────────────┤
        │  (剩余空间)           │
0x9FF00 │  临时栈               │
0xA0000 ├──────────────────────┤
        │  显存/BIOS ROM        │
0xFFFFF └──────────────────────┘
```

---

## 2.3 关键常量定义

```asm
; boot/bootsect.s 第6行
SYSSIZE  = 0x3000   ; 内核最大大小: 0x30000字节 = 192KB (单位:16字节的click)

SETUPLEN = 4        ; setup.s 占用扇区数
BOOTSEG  = 0x07c0   ; bootsect 初始段地址 (0x07c0 × 16 = 0x7C00)
INITSEG  = 0x9000   ; bootsect 搬到 0x9000 段 (0x9000 × 16 = 0x90000)
SETUPSEG = 0x9020   ; setup.s 加载到 0x9020 段 (0x9020 × 16 = 0x90200)
SYSSEG   = 0x1000   ; system 加载到 0x1000 段 (0x1000 × 16 = 0x10000)
ENDSEG   = SYSSEG + SYSSIZE  ; = 0x4000 (加载结束位置)

ROOT_DEV = 0x306    ; 根设备: 第二块硬盘第一分区
                    ; 高字节 0x03 = 硬盘, 低字节 0x06 = 第二块盘第一分区
```

---

## 2.4 逐行代码解析

### 第一步：自我搬移（0x7C00 → 0x90000）

```asm
; boot/bootsect.s 第46~56行
entry start
start:
    mov ax, #BOOTSEG    ; AX = 0x07C0 (当前所在段)
    mov ds, ax          ; DS = 0x07C0 (源段)
    mov ax, #INITSEG    ; AX = 0x9000 (目标段)
    mov es, ax          ; ES = 0x9000 (目标段)
    mov cx, #256        ; 复制 256 个字 = 512字节 = 1扇区
    sub si, si          ; SI = 0 (源偏移)
    sub di, di          ; DI = 0 (目标偏移)
    rep                 ; 重复 CX 次：
    movw                ; 复制一个字 DS:SI → ES:DI，并 SI+=2, DI+=2
    jmpi go, INITSEG    ; 远跳：CS=0x9000, IP=go偏移量
                        ; 即跳到新位置 0x90000+go 继续执行
```

**关键点：** `jmpi go, INITSEG` 是远跳指令（jmp far），同时修改 CS 和 IP。执行后 CPU 在 0x90000 处的副本中继续运行。

```asm
; 第57~62行：设置段寄存器
go: mov ax, cs      ; CS 已经是 0x9000（刚才的远跳设置的）
    mov ds, ax      ; DS = 0x9000
    mov es, ax      ; ES = 0x9000
    mov ss, ax      ; SS = 0x9000 (栈段)
    mov sp, #0xFF00 ; SP = 0xFF00，栈顶在 0x9FF00
                    ; 注: 0xFF00 >> 512字节，远离代码区
```

### 第二步：加载 setup.s

```asm
; 第67~77行
load_setup:
    mov dx, #0x0000     ; DH=0(磁头0), DL=0(驱动器0，即软盘A)
    mov cx, #0x0002     ; CH=0(磁道0), CL=2(从第2扇区开始)
                        ; 注: 第1扇区是bootsect，从第2扇区读setup
    mov bx, #0x0200     ; ES:BX = 0x9000:0x0200 = 0x90200 (加载地址)
    mov ax, #0x0200+SETUPLEN  ; AH=2(BIOS读扇区功能), AL=4(读4扇区)
    int 0x13            ; 调用BIOS磁盘中断
    jnc ok_load_setup   ; CF=0 表示成功，跳转
    mov dx, #0x0000
    mov ax, #0x0000     ; AH=0 是 BIOS 磁盘复位功能
    int 0x13            ; 复位磁盘，重试
    j load_setup        ; 无限重试（出错就死循环重试）
```

**BIOS int 0x13, AH=2 参数说明：**
```
AH = 2         (读扇区功能)
AL = 扇区数     (要读多少扇区)
CH = 磁道号     (柱面低8位)
CL = 扇区号     (位1-6) + 磁道高2位(位7-8)
DH = 磁头号
DL = 驱动器号   (0=软盘A, 0x80=第一硬盘)
ES:BX = 缓冲区地址
返回: CF=0成功, CF=1失败(AH=错误码)
```

### 第三步：获取磁盘参数

```asm
; 第81~90行
ok_load_setup:
    mov dl, #0x00
    mov ax, #0x0800     ; AH=8: 获取驱动器参数
    int 0x13
    mov ch, #0x00       ; 清除磁道高2位
    seg cs              ; 下一条指令用 CS 作为段寄存器（覆盖默认的DS）
    mov sectors, cx     ; 保存每磁道扇区数 (CL 低6位)
    mov ax, #INITSEG
    mov es, ax          ; 恢复 ES = 0x9000（int 0x13 可能修改了ES）
```

### 第四步：显示启动信息

```asm
; 第92~102行（显示 "Loading system ..."）
    mov ah, #0x03       ; BIOS int 0x10 AH=3: 读光标位置
    xor bh, bh          ; BH=0 (第0页)
    int 0x10            ; 返回: DH=行, DL=列

    mov cx, #24         ; 字符串长度 24 字节
    mov bx, #0x0007     ; BH=0(页0), BL=7(浅灰色属性)
    mov bp, #msg1       ; ES:BP 指向字符串（ES已是 0x9000）
    mov ax, #0x1301     ; AH=0x13 写字符串, AL=1(光标跟随移动)
    int 0x10            ; 显示字符串
```

字符串内容：
```asm
; 第244~247行
msg1:
    .byte 13,10         ; \r\n (回车换行)
    .ascii "Loading system ..."
    .byte 13,10,13,10   ; \r\n\r\n
```

### 第五步：加载内核（system）

```asm
; 第107~110行
    mov ax, #SYSSEG     ; AX = 0x1000
    mov es, ax          ; ES = 0x1000 (system加载到0x10000)
    call read_it        ; 调用读入例程（把整个内核读入）
    call kill_motor     ; 关闭软盘马达
```

**read_it 函数分析：**

```asm
; 第151~196行：复杂的磁盘读入循环
read_it:
    ; 这个函数按磁道读入，尽量一次多读以提高速度
    ; 处理64KB边界问题（DMA不能跨64KB边界）

rp_read:
    mov ax, es
    cmp ax, #ENDSEG     ; 是否已加载完毕？(es >= 0x4000 即完成)
    jb ok1_read
    ret                 ; 完成，返回
ok1_read:
    ; 计算当前磁道剩余扇区数
    ; 处理64KB边界：如果当前位置+剩余扇区会跨越64KB，则只读到边界
    call read_track     ; 实际读取
    ; 更新 head/track/sread，推进到下一轨道或磁头
    ...
```

**kill_motor 函数：**
```asm
; 第233~239行：关闭软盘马达
kill_motor:
    push dx
    mov dx, #0x3f2      ; 软盘控制器数字输出寄存器端口
    mov al, #0          ; 所有位清零 = 关闭所有马达
    outb                ; 输出到端口
    pop dx
    ret
```

### 第六步：确定根设备，跳转 setup

```asm
; 第117~139行：检查根设备
    seg cs
    mov ax, root_dev    ; 读取预定义的根设备号
    cmp ax, #0          ; 是否为0 (未定义)？
    jne root_defined    ; 非0则已定义，直接用
    seg cs
    mov bx, sectors     ; 获取每道扇区数
    mov ax, #0x0208     ; /dev/ps0 - 1.2Mb 软盘
    cmp bx, #15         ; 15扇区/道 = 1.2MB
    je root_defined
    mov ax, #0x021c     ; /dev/PS0 - 1.44Mb 软盘
    cmp bx, #18         ; 18扇区/道 = 1.44MB
    je root_defined
undef_root:
    jmp undef_root      ; 不认识的设备，死循环
root_defined:
    seg cs
    mov root_dev, ax    ; 保存确定的根设备号

    jmpi 0, SETUPSEG    ; 远跳到 setup.s (CS=0x9020, IP=0)
```

---

## 2.5 引导扇区末尾标志

```asm
; 第249~253行
.org 508           ; 从偏移508处开始（512-4=508）
root_dev:
    .word ROOT_DEV ; 偏移508-509: 根设备号 (0x306)
boot_flag:
    .word 0xAA55   ; 偏移510-511: BIOS启动标志
                   ; BIOS正是靠这两字节判断是否为引导扇区！
```

---

## 2.6 设备号编码规则

Linux 0.11 设备号编码：
```
设备号 = 主设备号 × 256 + 次设备号

主设备号：
  1 = 内存设备 (ramdisk)
  2 = 软盘
  3 = 硬盘

软盘次设备号：
  0 = /dev/fd0 (自动识别格式的A盘)
  4 = /dev/fd0d360  (360KB)
  8 = /dev/at0     (1.2MB)
  28 = /dev/PS0    (1.2MB)
  28+4 = /dev/PS0
  0x1c = 28        (1.44MB A盘)

硬盘次设备号：
  0 = 整个第一硬盘
  1-4 = 第一硬盘1-4分区
  5 = 整个第二硬盘
  6-9 = 第二硬盘1-4分区

所以 ROOT_DEV=0x306：
  主=3(硬盘), 次=6(第二块硬盘第一分区)
  即 /dev/hdb1
```

---

## 2.7 总结

| 步骤 | 操作 | 地址 |
|------|------|------|
| 1 | BIOS 加载 bootsect | 0x7C00 |
| 2 | bootsect 自我搬移 | → 0x90000 |
| 3 | 加载 setup.s | → 0x90200 |
| 4 | 加载 system (内核) | → 0x10000 |
| 5 | 跳转执行 setup.s | 0x90200 |

**bootsect 的精妙之处：**
- 仅 512 字节，却完成了全部引导工作
- 自我搬移腾出低地址空间给内核
- 使用 BIOS 中断，不依赖任何驱动程序
- 容错设计：磁盘读取失败会无限重试

---

**下一章：[第03章 - 硬件初始化 setup.s](03_硬件初始化_setup.md)**

*setup.s 将收集所有硬件信息，完成从实模式到保护模式的历史性跨越*
