Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 零拷贝 Ring Buffer 日志数据流
> **文档定位**：A-ULP（统一日志与打印系统）零拷贝 Ring Buffer 日志数据流的唯一权威设计\
> **文档版本**：v1.0.1-fix2\
> **最后更新**： 2026-11-23\
> **上级文档**：[Airymax Unify Design 总纲](../10-architecture/10-unify-design.md) §5\
> **设计依据**：综合修正方案 §4.2.2（A-ULP 设计）+ §6.2.1 C-07（DMA 一致性内存修正）

---

## SSoT 声明

> **单一权威源声明**：本文件是 **Airymax 日志 Ring Buffer 数据流** 的唯一权威源。内存方案（alloc_pages + mmap）、无锁 Ring Buffer 设计（reserve/commit 两阶段 + 三索引 head/committed_head/tail SPSC 所有权分离，P0-D1/D2 修复）、Fastpath 路径、eventfd 通知、Logger Daemon 分离、Panic 回退路径均以本文件为唯一权威定义。
>
> 技术选型声明：日志内存采用 **`alloc_pages(GFP_KERNEL)` + mmap**（**不使用 DMA 一致性内存**，x86_64 默认缓存一致）。本数据流遵循 [10-unify-design.md](../10-architecture/10-unify-design.md) 的 A-ULP 模块设计与 [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 的 128B 记录格式。整体技术选型对齐 Unify Design：sched_tac（SCHED_DEADLINE/SCHED_FIFO/EEVDF + seL4 MCS 映射，不使用 sched_ext）+ 纯 C LSM（不使用 BPF LSM）+ IORING_OP_URING_CMD + registered buffer + mmap（不使用 page flipping）。上列 io_uring 相关选型面向 IPC 数据流；**本数据流的唤醒通道是 eventfd**（§4），不经 io_uring。[SC] 共享契约头文件的物理宿主为 `kernel/include/uapi/linux/airymax/`。

---

## 文档信息卡

- **目标读者**：agent-linux 内核开发者、Logger Daemon 开发者、性能工程师
- **前置知识**：理解 [10-unify-design.md](../10-architecture/10-unify-design.md) A-ULP 模块、[09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) 128B 记录格式
- **预计阅读时间**：30 分钟
- **核心概念**：alloc_pages + mmap、无锁 Ring Buffer、reserve/commit 两阶段、Fastpath、eventfd、Logger Daemon、Panic 回退
- **复杂度标识**：高级

---

## §1 内存方案：alloc_pages(GFP_KERNEL) + mmap

### 1.1 为什么不用 DMA 一致性内存

综合修正方案 §6.2.1 C-07 指出，14 个文件误用 DMA 一致性内存（`dma_alloc_coherent`）用于日志/IPC 共享内存。这是错误的，原因有二：

1. **场景不匹配**：DMA 一致性内存的设计目标是为 DMA 设备（网卡、磁盘控制器）提供与 CPU 缓存一致的内存映射。日志场景的生产者是 CPU（内核态），消费者也是 CPU（用户态 Logger Daemon），**没有 DMA 设备参与**，使用 DMA 一致性内存是语义错配。
2. **性能损耗**：在 x86_64 架构下，CPU 默认缓存一致（cache-coherent），`dma_alloc_coherent` 会额外设置 `PAGE_KERNEL_NOCACHE` 或类似属性，禁用 cache 反而**降低** CPU 访问性能。

### 1.2 正确方案：alloc_pages + mmap（P0-D8 修复：分页写保护）

A-ULP 采用 `alloc_pages(GFP_KERNEL)` 分配连续物理页 + `remap_pfn_range` 映射到用户态。

> **⚠️ 实现状态标注（v1.0.1-fix2）**：P0-D8 分页写保护（header / tail / data 分页、按页设置只读）为**设计态，未落地**。当前实现（`kernel/log/airy_log_ring.c`）为**单一 `kaddr` 连续区**：`struct airy_log_ring_header`（40B，含 `head`/`committed_head`/`tail`/`capacity`/`record_size`/`frozen`）置于区首，`record[0]` 紧随其后；设备按整个区一次性映射，消费者侧以 `PROT_READ` 映射。**设备不强制写保护**：有写权限的进程自行以可写方式映射仍能篡改，因此只读映射是纪律而非隔离。以下 P0-D8 设计保留为规划稿，落地时以物理文件为准。

**内存布局**（P0-D8 早期设计稿，tail 页已不适用）：

```
Page 0:   struct airy_log_ring_ro_header  → PROT_READ（用户态只读）
Page 1:   struct airy_log_ring_rw_tail    → 已废弃：tail 由生产者独占写
Page 2+:  record[0] .. record[N-1]        → PROT_READ（用户态只读）
```

> v1.0.1-fix2 起 `tail` 由生产者覆盖写时前移（§2.2），消费者不再持有任何共享写权限。若后续落地 P0-D8，只需把 header 与 data 分页设为只读，不存在"可写 tail 页"。

```c
/* kernel/log/airy_log_ring.c —— 内存分配（P0-D8 设计稿，未实现） */

/* mmap 偏移定义（vm_pgoff 单位为页） */
#define AIRY_LOG_MMAP_OFF_RO_HEADER  0  /* Page 0: read-only header */
#define AIRY_LOG_MMAP_OFF_RW_TAIL    1  /* Page 1: writable tail */
#define AIRY_LOG_MMAP_OFF_RO_DATA    2  /* Page 2+: read-only data */

/* 只读 header（用户态不可写，防止篡改 head/committed_head/capacity/frozen） */
struct airy_log_ring_ro_header {
    __u64   head;           /* 生产者 reserve 位置 */
    __u64   committed_head; /* 生产者 commit 位置（对齐实现字段名） */
    __u32   capacity;       /* Ring 容量（记录数） */
    __u32   record_size;    /* 单条记录大小（128B） */
    __u32   frozen;         /* 冻结标志（Micro-Supervisor 设置） */
    __u32   pad;
    /* 填充至 PAGE_SIZE 由 alloc_pages 保证（__GFP_ZERO） */
};

/* 可写 tail（用户态 Logger Daemon 写消费位置） */
struct airy_log_ring_rw_tail {
    __u64   tail;           /* 消费者 read 位置（Logger Daemon 写） */
    /* 填充至 PAGE_SIZE */
};

struct airy_log_ring {
    struct page *pages;         /* 底层物理页（连续：header + tail + data） */
    void        *kaddr;         /* 内核态虚拟地址 */
    size_t       size;          /* Ring 总大小（页对齐） */
    unsigned int order;         /* alloc_pages order */
    /* 便捷访问指针 */
    struct airy_log_ring_ro_header *ro_hdr;
    struct airy_log_ring_rw_tail   *rw_tail;
    void                            *data_base;
};

static int airy_log_ring_alloc(struct airy_log_ring *ring, size_t data_size)
{
    /* 总大小 = 1 页 ro_header + 1 页 rw_tail + data_size（页对齐） */
    size_t total = PAGE_SIZE + PAGE_SIZE + PAGE_ALIGN(data_size);
    ring->order = get_order(total);
    ring->size  = PAGE_SIZE << ring->order;
    ring->pages = alloc_pages(GFP_KERNEL | __GFP_ZERO, ring->order);
    if (!ring->pages)
        return -ENOMEM;
    ring->kaddr   = page_address(ring->pages);
    ring->ro_hdr  = (struct airy_log_ring_ro_header *)ring->kaddr;
    ring->rw_tail = (struct airy_log_ring_rw_tail *)(ring->kaddr + PAGE_SIZE);
    ring->data_base = ring->kaddr + 2 * PAGE_SIZE;
    return 0;
}

/* mmap 到用户态 Logger Daemon（P0-D8 设计稿，未实现：按 offset 区分保护属性） */
static int airy_log_ring_mmap(struct file *f, struct vm_area_struct *vma)
{
    struct airy_log_ring *ring = f->private_data;
    unsigned long pgoff = vma->vm_pgoff;
    unsigned long size  = vma->vm_end - vma->vm_start;
    pgprot_t prot;

    vm_flags_set(vma, VM_SHARED | VM_DONTEXPAND | VM_DONTDUMP);

    switch (pgoff) {
    case AIRY_LOG_MMAP_OFF_RO_HEADER:
        /* P0-D8 设计：header 页只读，禁止用户态篡改 head/committed_head/frozen */
        prot = pgprot_modify(vma->vm_page_prot, PROT_READ, 0);
        vma->vm_flags &= ~VM_WRITE;
        return remap_pfn_range(vma, vma->vm_start,
                               page_to_pfn(ring->pages),
                               PAGE_SIZE, prot);
    case AIRY_LOG_MMAP_OFF_RW_TAIL:
        /* 已废弃：tail 改由生产者覆盖写时独占前移，见 §2.2 */
        prot = vma->vm_page_prot;
        return remap_pfn_range(vma, vma->vm_start,
                               page_to_pfn(ring->pages) + 1,
                               PAGE_SIZE, prot);
    case AIRY_LOG_MMAP_OFF_RO_DATA:
        /* P0-D8 设计：data 页只读，禁止用户态篡改日志记录 */
        prot = pgprot_modify(vma->vm_page_prot, PROT_READ, 0);
        vma->vm_flags &= ~VM_WRITE;
        return remap_pfn_range(vma, vma->vm_start,
                               page_to_pfn(ring->pages) + 2,
                               size, prot);
    default:
        return -EINVAL;
    }
}
```

> **⚠️ P0-D8 修复说明**（v1.0.1-fix，**设计态，未实现**）：原设计将 header + tail + data 放在同一连续内存区域，用户态以 `PROT_READ|PROT_WRITE` 映射整片区域，导致用户态可篡改 header 元数据（`magic`/`head`/`committed_head`/`frozen` 等）和日志记录数据。设计稿将三者分离到不同页：header 页（Page 0）和 data 页（Page 2+）以 `PROT_READ` 只读映射到用户态，仅 tail 页（Page 1）以 `PROT_READ|PROT_WRITE` 映射。**当前实现未落地此设计**——实际为单一 `kaddr` 连续区（`header` + `record[]`，见 `airy_log_ring_init()`），写保护未实现。

### 1.3 缓存一致性保证

x86_64 架构下，CPU 缓存默认一致（cache-coherent），内核生产者写入 `kaddr` 后，用户态消费者通过 mmap 映射的虚拟地址读取，能看到最新数据。可见性通过以下机制保证：

- **x86 内存模型**：x86 是 TSO（Total Store Order），store 后的 load 在其他 CPU 立即可见（经过 store buffer 刷新）
- **原子索引**：生产者 commit 时使用 `smp_store_release()`，消费者读取索引使用 `smp_load_acquire()`，确保 128B 记录写入在索引更新之前完成
- **无需 cache flush**：因为缓存一致，无需调用 `flush_cache_range()` 等昂贵操作

在 arm64 与 riscv64 上，日志内存同样是正常可缓存映射，可见性由 `smp_store_release()` / `smp_load_acquire()` 配对保证，不需要显式 `flush_dcache_page()`；只有真正非一致的映射（如非一致 DMA 设备）才需要显式维护，本路径不涉及。

---

## §2 无锁 Ring Buffer 设计：reserve/commit 两阶段写入、原子索引

### 2.1 两阶段写入模型

A-ULP 的 Ring Buffer 采用 **reserve/commit 两阶段写入**模型，避免生产者持锁期间被中断/抢占导致死锁：

```
生产者（内核态）                  消费者（Logger Daemon）
┌──────────────────────┐        ┌──────────────────────┐
│ 1. reserve(slot)     │        │ 1. 读取 committed_head│
│    → 获得 slot 偏移  │        │    （acquire）        │
│ 2. memcpy 128B 到 slot│        │ 2. 从私有序号消费     │
│ 3. commit(slot)      │        │    [cursor, committed_head)│
│    → 发布槽位序号     │        │ 3. 推进私有游标       │
│    → 推进 committed_head│      │    （不写共享头）     │
└──────────────────────┘        │ 4. 按 tail 校正并计丢失│
                                └──────────────────────┘
```

### 2.2 原子索引（P0-D1 修复：三索引 SPSC 所有权分离）

Ring Buffer 使用三个原子索引（v1.0.1-fix：原两索引模型存在消费者读到未提交数据的根本性缺陷）。索引字段名对齐实现（`kernel/log/airy_log_ring.c` `struct airy_log_ring_header`）：`committed_head`（原文档旧名 `commit`，已统一）：

- `head`：生产者预留位置（仅生产者写，消费者读）——预留即自增，不表示内容已完整
- `committed_head`：连续已发布位置（仅生产者写，消费者读）——标记连续可消费前缀，消费者只读 `[私有游标, committed_head)`
- `tail`：最早仍有效的槽位（仅生产者写，消费者读）——环满时生产者前移丢弃最旧记录；消费者只读它校正游标、统计自身落后造成的丢失

> **⚠️ P0-D1 修复说明**（v1.0.1-fix）：原设计仅有 `head`/`tail` 两索引，消费者读取 `[tail, head)` 区间。但 `head` 在 reserve 时即自增，此时 128B 数据尚未写入完成，消费者可能读到半成品数据。引入独立 `committed_head` 后，提交按槽位发布序号推进连续前缀，消费者只读已发布记录。

> **⚠️ P0-D2 落地说明**（v1.0.1-fix2）：设计稿曾计划"满则返回 NULL 丢弃"，但丢弃会让生产者侧无从补偿。当前实现为**覆盖写**：reserve 发现 `head - tail >= capacity` 时先把 `tail` 前移到 `head - capacity + 1`，再发布新槽位。`tail` 仍只有生产者写，SPSC 所有权不变量成立；被覆盖的段由消费者按 `tail` 校正私有游标并上报丢失计数（tracepoint `airy_ring_buffer_write` 的 overrun 位）。生产者写失败（`frozen`）才返回错误。

```c
struct airy_log_ring_header {
    __u64   head;           /* 生产者预留位置（仅生产者写） */
    __u64   committed_head; /* 连续已发布位置（仅生产者写）—— P0-D1 修复 */
    __u64   tail;           /* 最早有效槽位（仅生产者写） */
    __u32   capacity;       /* Ring 容量（记录数，2 的幂） */
    __u32   record_size;    /* 单条记录大小（128B） */
    __u32   frozen;         /* 冻结标志（Micro-Supervisor 设置） */
    __u32   pad;
};
```

### 2.3 reserve 实现

```c
/* kernel/log/airy_log_ring.c —— reserve */
struct airy_log_record *airy_log_reserve(struct airy_log_ring *ring, __u64 *pos,
                                         bool *overrun)
{
    struct airy_log_ring_header *hdr = ring->kaddr;
    bool dropped = false;
    __u64 head, tail, want;
    __u32 idx;

    /* unlikely：99%+ 不冻结；与 freeze 的 release 写配对 */
    if (unlikely(READ_ONCE(hdr->frozen)))
        return NULL;   /* 调用方回退到 printk */

    do {
        head = smp_load_acquire(&hdr->head);
        tail = smp_load_acquire(&hdr->tail);

        dropped = head - tail >= hdr->capacity;
        if (dropped) {
            want = head - hdr->capacity + 1;
            if (want > tail)
                cmpxchg(&hdr->tail, tail, want);   /* 生产者前移，丢弃最旧 */
        }
        idx = head % hdr->capacity;
    } while (cmpxchg(&hdr->head, head, head + 1) != head);

    if (overrun)
        *overrun = dropped;
    if (pos)
        *pos = head;
    return (struct airy_log_record *)(ring->kaddr + sizeof(*hdr) +
                                      idx * hdr->record_size);
}
```

### 2.4 commit 实现

发布序号写在记录的 `reserved[8]` 首 8 字节（`pos + 1`，0 表示未发布）——预置为零的槽位因此不会被误判为已发布：

```c
/* kernel/log/airy_log_ring.c —— commit */
void airy_log_commit(struct airy_log_ring *ring, __u64 pos)
{
    struct airy_log_ring_header *hdr = ring->kaddr;
    struct airy_log_record *rec;
    __u64 done;

    rec = (struct airy_log_record *)(ring->kaddr + sizeof(*hdr) +
                                     (pos % hdr->capacity) * hdr->record_size);
    /* 1. release 发布本槽位：记录体先可见，序号后可见 */
    airy_log_slot_pub(rec, pos + 1);

    /* 2. 推进连续已发布前缀；多生产者并发时 cmpxchg 重试 */
    for (;;) {
        done = smp_load_acquire(&hdr->committed_head);
        rec = (struct airy_log_record *)(ring->kaddr + sizeof(*hdr) +
                                         (done % hdr->capacity) * hdr->record_size);
        if (airy_log_slot_seq(rec) != done + 1)
            break;
        if (cmpxchg(&hdr->committed_head, done, done + 1) != done)
            continue;
    }

    /* 3. eventfd 通知 Logger Daemon（非阻塞） */
    if (ring->efd)
        eventfd_signal(ring->efd);
}
```


---

## §3 Fastpath 路径：仅 memcpy 128B raw binary（~20ns 裸 memcpy）

### 3.1 Fastpath 三步

A-ULP 的 Fastpath 严格三步，禁止任何格式化操作：

```c
/* kernel/log/airy_log_ring.c —— Fastpath */
int airy_log_write(struct airy_log_ring *ring, __u16 level, __u16 facility,
                   const void *payload, __u32 payload_len)
{
    struct airy_log_record *rec;
    struct timespec64 ts;
    bool overrun;
    __u64 pos;

    /* 1. reserve 128B slot（~10ns） */
    rec = airy_log_reserve(ring, &pos, &overrun);
    if (unlikely(!rec))
        return -AIRY_ELOG_FULL;   /* 仅冻结时走到这里 */

    /* 2. 填充字段 + memcpy raw binary（~20ns） */
    rec->magic       = AIRY_LOG_MAGIC;
    rec->level       = level;
    rec->facility    = facility;
    ktime_get_real_ts64(&ts);
    rec->timestamp_ns = timespec64_to_ns(&ts);
    rec->caller_id   = (__u32)current->pid;
    rec->payload_len = min_t(__u32, payload_len,
                             (__u32)sizeof(rec->payload));
    memcpy(rec->payload, payload, rec->payload_len);

    /* 3. commit 发布序号 + eventfd（~10ns） */
    airy_log_commit(ring, pos);
#if IS_ENABLED(CONFIG_AIRY_AGENT_TRACING)
    trace_airy_ring_buffer_write(level, rec->payload_len, overrun);
#endif
    return 0;
}
```

### 3.2 Fastpath 性能拆解

| 步骤 | 操作 | 耗时 |
|------|------|------|
| 1. reserve | `cmpxchg` 原子自增 head | ~10ns |
| 2. memcpy | 128B `memcpy`（raw binary，无格式化） | ~20ns |
| 3. commit | `smp_store_release` + `eventfd_signal` | ~10ns |
| **总计** | | **~40-100ns** |

Fastpath 不进行任何 `sprintf`/`snprintf`/字符串拼接，payload 是原始二进制，由 Logger Daemon 异步解析。

### 3.3 分支预测优化

Fastpath 使用 `likely` / `unlikely` 标注关键分支，确保 99%+ 的正常路径不触发分支预测失败：

- `unlikely(READ_ONCE(hdr->frozen))`：冻结检查，99%+ 不冻结
- `unlikely(!rec)`：reserve 失败检查，99%+ 成功（失败只来自冻结）
- `if (overrun)`：覆盖丢弃，仅在环满时置位，经 tracepoint 上报而不进主路径

---

## §4 eventfd 通知：Logger Daemon 消费通知

### 4.1 通知机制

生产者 commit 后通过 `eventfd_signal()` 通知 Logger Daemon。`eventfd_signal` 是非阻塞的，仅累加计数：

```c
/* kernel/log/airy_log_dev.c —— AIRY_LOG_IOC_EFD */
static long airy_log_ioctl_efd(struct airy_log_ring *ring, unsigned long arg)
{
    s32 efd;
    int ctx;

    if (copy_from_user(&efd, (void __user *)arg, sizeof(efd)))
        return -EFAULT;
    ctx = eventfd_ctx_fdget(efd);
    if (IS_ERR(ctx))
        return PTR_ERR(ctx);
    airy_log_ring_register_eventfd(ring, ctx);
    return 0;
}
```

Logger Daemon 自行创建 `eventfd(EFD_CLOEXEC | EFD_NONBLOCK)`，经该 ioctl 注册给内核，再把 eventfd 纳入守护框架的 epoll 集合；计数 > 0 时被唤醒并批量消费。

### 4.2 批量消费

Logger Daemon 不写任何共享状态：私有 `cursor` 逐槽前进，槽位发布序号不等于 `cursor + 1` 即视为已被覆盖，计入 `lost` 并跳过：

```c
/* services/daemons/logger_d/logd_ring.c —— 批量消费 */
int logd_ring_drain(struct logd_ring *ring, logd_emit emit, void *ctx,
                    unsigned int *nr)
{
    hdr  = ring->map;
    base = (const unsigned char *)ring->map + ring->hdr_size;
    head = __atomic_load_n(&hdr->committed_head, __ATOMIC_ACQUIRE);
    tail = __atomic_load_n(&hdr->tail, __ATOMIC_ACQUIRE);

    if (ring->cursor < tail) {          /* 被新记录覆盖的那段 */
        ring->lost += tail - ring->cursor;
        ring->cursor = tail;
    }
    if (ring->cursor > head)
        ring->cursor = head;

    while (ring->cursor < head) {
        const struct airy_log_record *rec = (const void *)
            (base + (ring->cursor % ring->capacity) * ring->rec_size);

        if (slot_seq(rec) != ring->cursor + 1) {
            ring->lost++;
            ring->cursor++;
            continue;                   /* 半新半旧的槽位绝不落盘 */
        }
        ret = emit(ctx, rec);
        if (ret)
            return ret;
        ring->cursor++;
        n++;
        if (n >= LOGD_DRAIN_MAX)
            break;
    }

    /* 未取空时保留 eventfd 计数，让下一轮 epoll 立即再次唤醒 */
    if (ring->cursor >= head) {
        uint64_t v;

        while (read(ring->efd, &v, sizeof(v)) == sizeof(v))
            ;
    }
    if (nr)
        *nr = n;
    return 0;
}
```

### 4.3 无上下文切换

`eventfd_signal` 只在有等待者时唤醒进程；没有消费者挂起时仅累加计数，Fastpath 全程无格式化、无阻塞。真正的上下文切换只发生在 Daemon 被唤醒消费的那一刻，且一次唤醒批量取走上限 `LOGD_DRAIN_MAX`（4096）条，而不是每条日志一次。

---

## §5 Logger Daemon 分离：mmap 读取 + sprintf 格式化 + 过滤 + 落盘

### 5.1 分离原则

A-ULP 严格遵循"内核 Fastpath 禁止格式化"原则。所有耗时的操作（格式化、过滤、落盘）全部下沉到用户态 Logger Daemon：

```
内核 Fastpath（~50-100ns）           用户态 Logger Daemon（异步，ms 级）
┌──────────────────────────┐        ┌──────────────────────────┐
│ 1. reserve 128B slot     │        │ 1. mmap 读取 Ring Buffer │
│ 2. memcpy 128B raw       │ ──eventfd──▶ 2. sprintf 格式化     │
│ 3. commit 原子索引       │        │ 3. 过滤（level/facility） │
│ 4. eventfd_signal（非阻塞）│        │ 4. 落盘 + 轮转 + 压缩    │
└──────────────────────────┘        └──────────────────────────┘
```

### 5.2 Logger Daemon 职责

| 职责 | 实现位置 | 说明 |
|------|---------|------|
| mmap 读取 | `services/daemons/logger_d/logd_ring.c` | `_GEOM` 取几何、只读 mmap、私有游标推进 |
| sprintf 格式化 | `services/daemons/logger_d/logd_file.c` | `logd_fmt()` 把 128B 记录转为单行文本 |
| 过滤 | `services/daemons/logger_d/logger_d.c` | 按 `level` 过滤（`logger.conf` 的 `level` 键） |
| 落盘 | `services/daemons/logger_d/logd_file.c` | 追加 `/var/log/agentrt/airy.log`，按量/按时轮转，按 `flush_ms` 周期 `fdatasync` |
| 轮转保留 | `services/daemons/logger_d/logd_file.c` | 改名保留序号窗口，最旧文件先删除 |
| 配置装载 | `services/daemons/logger_d/logd_cfg.c` | 严格 INI，只认 `[logger]`，未知键整次拒绝 |
| 生命周期 | `services/daemons/logger_d/logger_d.c` | 守护框架注册、eventfd 源、信号重载 |
| 压缩 | — | 首版未实现：不以配置键伪装支持 |

### 5.3 格式化示例

`logd_fmt()` 输出的每一行是一条记录：

```
2026-11-23T08:00:00.000000000Z INFO  kern[1234] hello
```

前缀为 UTC 的 RFC3339 纳秒时间戳、五档级别（`DEBUG`/`INFO`/`WARN`/`ERROR`/`FATAL`）、来源名（内核为 `kern`，其余为 `fac-NNNN`）与 pid；正文按可打印字符原样输出，其余字节转义为 `\xNN`，保证落盘文件始终是纯文本、可直接被 grep/awk 处理。行尾的 `\n`/`\r` 先剥除，避免一条记录变成多行。

配置为 `/etc/agentrt/logger.conf`（严格 INI，键见 `services/daemons/logger_d/etc/logger.conf`）：`device`、`path`、`level`、`flush_ms`、`max_bytes`、`max_age_s`、`keep`。级别名与 `log_types.h` 的枚举一一对应，非法值和未知键在启动或重载时整次拒绝。

### 5.4 Logger Daemon 崩溃恢复

Logger Daemon 是用户态进程，可能崩溃。A-ULP 的设计确保 Logger Daemon 崩溃不丢日志：

1. **Ring Buffer 不丢**：Ring Buffer 在内核态，Logger Daemon 崩溃不影响生产者写入
2. **满策略**：覆盖写——Ring 满时生产者前移 `tail` 丢弃最旧记录（详见 §2.2），生产者始终是 `tail` 的唯一写者
3. **systemd 自动重启**：unit 为 `Type=notify` + 看门狗，崩溃后自动重启
4. **重启后继续消费**：游标不持久化，重启后从 0 起：先对齐到 `tail`（把已被覆盖的段计入 `lost`），再从 `tail` 读到 `committed_head`。仍在环中的记录会被重新落盘，已被覆盖的记录无法找回，但仍占用环的窗口。

---

## §6 Panic 回退：printk_safe 原生路径

### 6.1 为什么需要 Panic 回退

当系统进入 Panic 状态时，Ring Buffer 可能已损坏（内存损坏导致 `airy_log_ring_header` 元数据不一致），或 Logger Daemon 已死亡（无法消费 Ring Buffer）。此时若仍尝试写入 Ring Buffer，可能导致：

- **递归 Panic**：Ring Buffer 写入触发再次 Panic，无限循环
- **死锁**：`cmpxchg` 在 NMI 上下文不安全
- **数据丢失**：Ring Buffer 损坏后写入的数据无法被消费

### 6.2 回退现状

0.0.5 落地的回退是**放弃入环**，而不是实现一条专用 panic 写路径：

```c
/* kernel/log/airy_log_ring.c —— 冻结即拒写 */
if (unlikely(READ_ONCE(hdr->frozen)))
    return NULL;   /* airy_log_write() 返回 -AIRY_ELOG_FULL */
```

- 回放控制台（`airy_log_console.c`）在 `write()` 里把 printk 文本写入环；写入失败即返回，不再格式化、不再重试
- printk 自身的串口/终端控制台链条不受影响，Panic 输出照常可见
- `_FREEZE` ioctl 由 Micro-Supervisor 调用（CAP_SYS_ADMIN），置位后所有入环写入立即失败

> **规划态**：独立 panic notifier 与 `printk_safe_flush_on_panic()` 挂钩尚未落地；在此之前，Panic 场景的可得证据是内核原生控制台输出，而不是环内容。

### 6.3 printk_safe 特性

Linux 6.6 的 `printk_safe` 提供 NMI-safe 的 per-CPU 临时缓冲区：

- **无锁**：per-CPU 缓冲区，无需锁，NMI 上下文安全
- **无依赖**：不依赖调度器、内存分配、RCU
- **延迟刷新**：Panic 时通过 `printk_safe_flush_on_panic()` 刷入主 log buffer
- **大小有限**：每 CPU 8192 字节

### 6.4 回退的触发条件

| 触发条件 | 处理 |
|---------|------|
| `frozen=true`（`_FREEZE` ioctl） | 入环写入返回 `-AIRY_ELOG_FULL`，回放控制台放弃本条 |
| Panic / NMI | 无专用钩子；环不再接收，panic 输出走 printk 原生控制台 |
| Logger Daemon 死亡 | 环继续写入并按覆盖策略丢弃最旧，重启后从 `tail` 继续 |
| 消费者落后于 `tail` | 丢失段计入 `lost`，不影响内核写入 |

---

## §7 性能对比：传统 printk ~5-10μs → Airymax 全路径 ~50-100ns

> 本章数字均为设计目标（基于 reserve/commit 路径的指令级估算），正式基准待 0.0.5 实机验证后回填。

### 7.1 性能对比表

| 场景 | 传统 printk | Airymax 全路径 | 提升 |
|------|------------|--------------|------|
| 单条日志写入 | ~5-10μs | ~50-100ns | 100-500 倍 |
| 格式化位置 | 内核态（`vsnprintf`） | 用户态（Logger Daemon） | — |
| 锁竞争 | `logbuf_lock`（全局锁） | 无锁（`cmpxchg` 原子索引） | — |
| 上下文切换 | 可能触发控制台刷新 | 无（`eventfd_signal` 非阻塞） | — |
| Panic 安全性 | 依赖 `printk_safe` | 正常用 Ring；冻结或 Panic 时放弃入环，输出走 printk 原生控制台 |

### 7.2 性能拆解对比

| 步骤 | 传统 printk | Airymax |
|------|------------|---------|
| 格式化 | `vsnprintf` ~2-5μs | 无（payload raw binary） |
| 锁获取 | `logbuf_lock` ~500ns-1μs | 无（原子索引 ~10ns） |
| 数据拷贝 | `memcpy` 到 logbuf ~100ns | `memcpy` 128B ~20ns |
| 通知 | 控制台刷新（可能阻塞） | `eventfd_signal` ~10ns |
| **总计** | ~5-10μs | ~50-100ns |

### 7.3 吞吐量对比

下表为 8 核 x86_64 目标平台上的设计目标值，尚未在目标硬件上完成正式基准，不作为验收依据：

| 方案 | 单核吞吐 | 8 核总吞吐 |
|------|---------|----------|
| 传统 printk | ~10 万条/秒 | ~30 万条/秒（锁竞争） |
| Airymax Ring Buffer | ~1000 万条/秒 | ~8000 万条/秒（无锁） |

### 7.4 延迟 SLO

A-ULP 的延迟 SLO（受 [170-performance/05-agent-latency-slo.md](../170-performance/05-agent-latency-slo.md) 约束）：

| 指标 | SLO | 设计值 |
|------|-----|------|
| Fastpath p99 延迟 | ≤200ns | ~80-120ns |
| Fastpath p999 延迟 | ≤500ns | ~150-200ns |
| Logger Daemon 消费延迟 | ≤10ms | ~1-5ms |

---

## §8 相关文档

- [10-unify-design.md](../10-architecture/10-unify-design.md) §5 —— A-ULP 模块总纲
- [09-sc-log-types-contract.md](../30-interfaces/09-sc-log-types-contract.md) —— 128B 记录格式契约
- [11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md) §3 —— Panic 回退与 printk_safe
- [07-ipc-fastpath.md](../30-interfaces/07-ipc-fastpath.md) —— IPC fastpath（与日志 fastpath 设计同源）
- [170-performance/02-memory-performance.md](../170-performance/02-memory-performance.md) —— 内存性能 SLO
- 综合修正方案 §4.2.2 / §6.2.1 C-07 —— 设计依据

---

## §9 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-07-17 | 初始版本：零拷贝 Ring Buffer 日志数据流；alloc_pages + mmap 内存方案（不使用 DMA 一致性内存）；无锁 reserve/commit 两阶段写入；Fastpath ~50-100ns；eventfd 通知；Logger Daemon 分离；Panic 回退 printk_safe；性能对比 100-500 倍提升 |
| v1.0.1 | 2026-07-21 | 版本号统一：按 IRON-7 铁律，所有文档版本号统一为 v1.0.1（禁止 v1.0/v1.1/v1.1.1/v1.2/v2.0 中间过渡版本） |
| v1.0.1-fix | 2026-07-26 | P0-D1/D2 修复：引入三索引模型（head/committed_head/tail SPSC 所有权分离），消费者读取 `[tail, committed_head)` 而非 `[tail, head)` 避免读到未提交数据；commit 实现改为写入独立 committed_head 索引 + per-slot magic 扫描推进；内核 mmap 添加 P0-D3/P0-D8 修复说明；v1.0.1 复核：P0-D8 分页写保护标注为设计态（实现为单一 kaddr 连续区），满策略与实现统一为覆盖写，索引字段名统一为 committed_head |
| v1.0.1-fix2 | 2026-11-23 | 对齐已落地实现：用户态 ABI 固化为 `AIRY_LOG_RING_ABI 1`（header 40B + `_EFD`/`_GEOM`/`_FREEZE` ioctl），发布序号写入记录 reserved 前 8 字节，tail 由生产者覆盖写时独占前移、消费者只维护私有游标并按 tail 校正丢失，`_EFD` 取代 io_uring 注册缓冲；补入实际 `reserve`/`commit`/`write`、`logd_ring_drain` 与 `logd_fmt` 代码，落盘路径改为 `/var/log/agentrt/airy.log`、首版不压缩；§1.3 明确 arm64/riscv64 由 release/acquire 配对保证可见性、无需显式 flush；panic 钩子标注为规划态，性能数字标注为设计值 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved. | 零拷贝 Ring Buffer 日志数据流 | v1.0.1-fix2 | 2026-11-23
