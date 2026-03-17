# 第12章：缓冲区管理 fs/buffer.c

> 源码文件：`fs/buffer.c`
> 核心概念：缓冲区缓存、LRU空闲链表、哈希表、bread/bwrite

---

## 12.1 概述

缓冲区缓存（Buffer Cache）是文件系统和块设备之间的高速缓存层：

```
文件系统代码
    │ bread(dev, block)
    ▼
缓冲区缓存 (buffer.c)
    ├── 命中：直接返回内存中的数据
    └── 未命中：从磁盘读入，缓存后返回
    ▼
ll_rw_block() → 块设备驱动
    ▼
磁盘/软盘
```

**优势：**
- 减少磁盘IO次数（内存速度比磁盘快1000倍以上）
- 多次读同一块：只第一次触发磁盘IO
- 写操作可以批量延迟（写脏缓冲区）

---

## 12.2 buffer_head 数据结构

```c
// include/linux/fs.h
struct buffer_head {
    char          *b_data;        // 指向1024字节数据区
    unsigned long  b_blocknr;     // 块号
    unsigned short b_dev;         // 设备号
    unsigned char  b_uptodate;    // 1=数据有效，0=需要从磁盘读
    unsigned char  b_dirt;        // 1=已修改（脏），需要写回磁盘
    unsigned char  b_count;       // 使用引用计数（>0时不可被替换）
    unsigned char  b_lock;        // 1=正在进行IO（锁定中）
    struct task_struct *b_wait;   // 等待该缓冲区解锁的进程
    struct buffer_head *b_prev;   // 哈希链表（同一哈希桶内）
    struct buffer_head *b_next;
    struct buffer_head *b_prev_free; // 空闲链表（LRU双向循环链表）
    struct buffer_head *b_next_free;
};
```

---

## 12.3 缓冲区的内存布局

```c
// fs/buffer.c 第30~34行
extern int end;   // 内核BSS段结束后的第一个地址（链接器定义）
struct buffer_head *start_buffer = (struct buffer_head *) &end;
// buffer_head 数组从内核代码/数据后面开始

struct buffer_head *hash_table[NR_HASH];  // NR_HASH=307 个哈希桶
static struct buffer_head *free_list;
```

**内存布局图（buffer_init后）：**
```
&end (内核BSS结束)
    ↓
┌──────────────────────┐
│ buffer_head[0]       │ (元数据结构，~40字节)
│ buffer_head[1]       │
│ buffer_head[2]       │
│ ...                  │
│ buffer_head[NR_BUFFERS-1] │
├──────────────────────┤
│  (空隙)               │
├──────────────────────┤
│ 数据块[NR_BUFFERS-1]  │ 1024字节，b_data指向这里
│ 数据块[NR_BUFFERS-2]  │ （从高地址向低地址分配）
│ ...                  │
│ 数据块[0]            │
└──────────────────────┘ ← buffer_end (= buffer_memory_end)
```

---

## 12.4 buffer_init() —— 初始化缓冲区

```c
// fs/buffer.c 第348~381行
void buffer_init(long buffer_end)
{
    struct buffer_head *h = start_buffer;  // 从内核末尾开始
    void *b;
    int i;

    // 640KB~1MB是VGA/BIOS区域，不能用作缓冲区数据
    if (buffer_end == 1<<20)   // 如果缓冲区只到1MB
        b = (void *)(640*1024); // 数据从640KB处开始
    else
        b = (void *)buffer_end;  // 从缓冲区上限开始

    // 从高地址向低地址分配数据块，从低地址向高地址分配buffer_head
    // 两者对向增长，直到相遇
    while ((b -= BLOCK_SIZE) >= ((void *)(h+1))) {
        h->b_dev = 0;
        h->b_dirt = 0;
        h->b_count = 0;
        h->b_lock = 0;
        h->b_uptodate = 0;
        h->b_wait = NULL;
        h->b_next = NULL;
        h->b_prev = NULL;
        h->b_data = (char *)b;  // 指向数据区
        h->b_prev_free = h-1;
        h->b_next_free = h+1;
        h++;
        NR_BUFFERS++;           // 统计缓冲区总数
        // 跳过640KB~1MB的洞
        if (b == (void *)0x100000)
            b = (void *)0xA0000;
    }

    // 构建循环链表
    h--;
    free_list = start_buffer;
    free_list->b_prev_free = h;
    h->b_next_free = free_list;

    // 初始化哈希表（所有桶为空）
    for (i=0; i<NR_HASH; i++)
        hash_table[i] = NULL;
}
```

---

## 12.5 哈希表与LRU链表

缓冲区同时维护两种数据结构：

**哈希表（快速查找）：**
```c
#define _hashfn(dev,block) (((unsigned)(dev^block)) % NR_HASH)
#define hash(dev,block) hash_table[_hashfn(dev,block)]

// 根据(设备号, 块号)快速找到缓冲区
// NR_HASH = 307（质数，减少哈希冲突）
```

**LRU空闲链表（缓冲区替换）：**
```
free_list → [最近最久未使用] ↔ ... ↔ [最近使用的] ↔ free_list

当需要替换时，从 free_list->b_prev（最久未使用）开始查找
新分配/使用的缓冲区插入 free_list->b_prev（链表尾部）
```

---

## 12.6 getblk() —— 获取缓冲区（核心函数）

```c
// fs/buffer.c 第206~251行
#define BADNESS(bh) (((bh)->b_dirt<<1) + (bh)->b_lock)
// BADNESS: 脏块代价=2（需要写回），锁定块代价=1（需要等待）

struct buffer_head *getblk(int dev, int block)
{
    struct buffer_head *tmp, *bh;

repeat:
    // 1. 先在哈希表中查找（命中率通常很高）
    if (bh = get_hash_table(dev, block))
        return bh;  // 缓存命中！

    // 2. 在空闲链表中找最合适的缓冲区替换
    tmp = free_list;
    do {
        if (tmp->b_count) continue;  // 正在使用，跳过
        if (!bh || BADNESS(tmp) < BADNESS(bh)) {
            bh = tmp;
            if (!BADNESS(tmp)) break;  // 最理想（不脏、不锁定）
        }
    } while ((tmp = tmp->b_next_free) != free_list);

    // 没有可用缓冲区：等待有人释放
    if (!bh) {
        sleep_on(&buffer_wait);
        goto repeat;
    }

    // 等待该缓冲区解锁（可能正在IO）
    wait_on_buffer(bh);
    if (bh->b_count) goto repeat;  // 睡眠期间被抢走了，重试

    // 如果是脏块，先写回磁盘
    while (bh->b_dirt) {
        sync_dev(bh->b_dev);
        wait_on_buffer(bh);
        if (bh->b_count) goto repeat;
    }

    // 睡眠期间有人可能已经将这个块读入缓存了，重新检查
    if (find_buffer(dev, block))
        goto repeat;

    // 确定获得这个缓冲区，重新分配给新的(dev, block)
    bh->b_count = 1;
    bh->b_dirt = 0;
    bh->b_uptodate = 0;
    remove_from_queues(bh);  // 从旧位置移除
    bh->b_dev = dev;
    bh->b_blocknr = block;
    insert_into_queues(bh);  // 插入新位置（哈希表+LRU链表尾部）
    return bh;
}
```

---

## 12.7 bread() —— 读取数据块（最常用）

```c
// fs/buffer.c 第267~281行
struct buffer_head *bread(int dev, int block)
{
    struct buffer_head *bh;

    if (!(bh = getblk(dev, block)))
        panic("bread: getblk returned NULL\n");

    // 如果数据已有效（缓存命中），直接返回
    if (bh->b_uptodate)
        return bh;

    // 缓存未命中：发起磁盘读请求
    ll_rw_block(READ, bh);

    // 等待IO完成（此时进程睡眠）
    wait_on_buffer(bh);

    if (bh->b_uptodate)  // 读成功
        return bh;

    // 读失败（坏块等）
    brelse(bh);
    return NULL;
}
```

**bread_page()** — 同时读取4个连续块（提升性能）：
```c
void bread_page(unsigned long address, int dev, int b[4])
{
    struct buffer_head *bh[4];
    int i;

    // 第一步：同时发起4个读请求（不等待）
    for (i=0; i<4; i++)
        if (b[i]) {
            if (bh[i] = getblk(dev, b[i]))
                if (!bh[i]->b_uptodate)
                    ll_rw_block(READ, bh[i]);  // 异步发起IO
        } else
            bh[i] = NULL;

    // 第二步：依次等待并复制到内存页（此时4个IO可能已经并行完成）
    for (i=0; i<4; i++, address += BLOCK_SIZE)
        if (bh[i]) {
            wait_on_buffer(bh[i]);
            if (bh[i]->b_uptodate)
                COPYBLK((unsigned long)bh[i]->b_data, address);
            brelse(bh[i]);
        }
}
```

---

## 12.8 brelse() —— 释放缓冲区引用

```c
// fs/buffer.c 第253~261行
void brelse(struct buffer_head *buf)
{
    if (!buf) return;
    wait_on_buffer(buf);      // 如果正在IO，等待完成
    if (!(buf->b_count--))
        panic("Trying to free free buffer");
    wake_up(&buffer_wait);    // 唤醒等待缓冲区的进程
    // 注意：brelse不会真正释放内存，只减少引用计数
    // 缓冲区数据继续留在内存，供下次命中使用
}
```

---

## 12.9 wait_on_buffer() —— 等待IO完成

```c
// fs/buffer.c 第36~42行
static inline void wait_on_buffer(struct buffer_head *bh)
{
    cli();                        // 关中断（原子操作）
    while (bh->b_lock)
        sleep_on(&bh->b_wait);   // 等待解锁（IO完成）
    sti();
}
```

**b_lock 的生命周期：**
```
getblk() 返回缓冲区（b_lock=0, b_uptodate可能为0）
    │
    │ ll_rw_block(READ, bh):
    │   lock_buffer(bh)   → b_lock=1
    │   提交IO请求给驱动
    │
    │ 磁盘驱动完成IO中断：
    │   b_uptodate=1
    │   unlock_buffer(bh) → b_lock=0, wake_up(b_wait)
    │
    ▼
wait_on_buffer() 返回，数据可用
```

---

## 12.10 缓冲区的并发控制

Linux 0.11 的缓冲区设计原则（注释中明确说明）：
```
中断例程永远不会修改 buffer_head 结构（只能修改数据内容）
中断例程可以：设置 b_uptodate=1, 解锁 b_lock
中断例程不能：修改 b_count, b_dirt, 哈希链表指针等

这样避免了复杂的锁机制，只需在少数地方用 cli/sti 保护
```

---

## 12.11 总结

| 函数 | 作用 |
|------|------|
| `buffer_init()` | 初始化缓冲区池（buffer_head数组+数据块） |
| `getblk(dev, block)` | 获取或分配缓冲区（可能触发LRU替换+脏块写回） |
| `bread(dev, block)` | 读取数据块（命中则直接返回，否则从磁盘读） |
| `bread_page()` | 一次读4个块（请求调页优化） |
| `brelse(bh)` | 释放缓冲区引用（不删除数据，供再次命中） |
| `ll_rw_block(op, bh)` | 提交异步IO请求给块设备驱动 |
| `sys_sync()` | 将所有脏缓冲区写回磁盘 |
| `wait_on_buffer(bh)` | 等待IO完成（睡眠直到b_lock=0） |

---

**下一章：[第13章 - 文件路径解析 namei.c](13_文件路径解析_namei.md)**
