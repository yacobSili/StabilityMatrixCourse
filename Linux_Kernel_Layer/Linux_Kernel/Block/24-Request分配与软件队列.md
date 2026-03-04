# IO 流程关键结构体详解（二）：Request 分配与软件队列

> 本文是 **IO 流程关键结构体详解系列** 的第二篇，介绍从 bio 到 request 的转换、Internal Tag 分配机制、以及软件队列的作用。

**系列文章**：[查看完整系列](./README.md)  
**上一篇**：[23-IO流程结构体总览与Bio层](./23-IO流程结构体总览与Bio层.md)  
**下一篇**：[25-IO调度器结构体详解](./25-IO调度器结构体详解.md)

---

## 本篇涵盖的流程阶段

```
┌────────────────────────────────────────────────────────────────────────┐
│   阶段2            阶段3                阶段4                            │
│                                                                        │
│   bio合并   ──▶   分配request    ──▶   插入软件队列                     │
│                   +Internal Tag                                        │
│                                                                        │
│  request_queue   blk_mq_tags        blk_mq_ctx                        │
│  (合并检查)      sbitmap_queue       (Per-CPU队列)                     │
│                  request                                               │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 一、阶段2：Bio 提交与合并

### 1.1 bio 合并的意义

在分配 request 之前，Block 层会尝试将新 bio 合并到已有的 request，这样可以：
- **节省 Internal Tag**（合并后共用一个 request）
- **提高设备吞吐量**（连续扇区合并为一个大IO）
- **减少调度开销**（请求数量减少）

```
┌────────────────────────────────────────────────────────────────────┐
│                      Bio 合并示例                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  合并前:                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                        │
│  │ request1 │  │ request2 │  │ request3 │                        │
│  │ bio: A   │  │ bio: B   │  │ bio: C   │                        │
│  │ sector   │  │ sector   │  │ sector   │                        │
│  │ 1000-1007│  │ 1008-1015│  │ 1016-1023│                        │
│  └──────────┘  └──────────┘  └──────────┘                        │
│  需要3个tag                                                        │
│                                                                    │
│  合并后:                                                           │
│  ┌──────────────────────────────────────┐                         │
│  │          request1                    │                         │
│  │  bio: A ──▶ B ──▶ C                  │                         │
│  │  sector: 1000-1023 (连续24个扇区)     │                         │
│  └──────────────────────────────────────┘                         │
│  只需1个tag                                                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 1.2 struct request_queue - 合并相关字段

**源码位置**: `include/linux/blkdev.h`

**在此阶段的作用**: 提供合并限制参数和合并查找机制。

#### 核心字段 ⭐⭐⭐（合并检查）

| 字段名 | 类型 | 作用 | 影响 |
|--------|------|------|------|
| **设备关联** ||||
| `disk` | `struct gendisk *` | 关联的通用磁盘 | 检查bio是否属于同一设备 |
| `elevator` | `struct elevator_queue *` | IO调度器 | 调度器实现合并逻辑 |
| **合并限制** ||||
| `max_segments` | `unsigned short` | **最大段数**<br/>request能容纳的最大bio_vec数 | 典型值: 128<br/>限制合并上限 |
| `max_segment_size` | `unsigned int` | **最大段大小**<br/>单个bio_vec的最大字节数 | 典型值: 65536 (64KB) |
| `max_sectors` | `unsigned int` | **最大扇区数**<br/>单个request的最大扇区数 | UFS: 2048 (1MB)<br/>NVMe: 更大 |
| `max_hw_sectors` | `unsigned int` | 硬件支持的最大扇区数 | 由驱动设置 |
| **查找优化** ||||
| `last_merge` | `struct request *` | 上次合并的request | 快速查找合并候选 |

#### 重要字段 ⭐⭐

| 字段 | 类型 | 作用 |
|------|------|------|
| `queue_flags` | `unsigned long` | 队列标志（QUEUE_FLAG_*） |
| `nr_requests` | `unsigned long` | Internal Tag数量（可调） |
| `tag_set` | `struct blk_mq_tag_set *` | Tag集合配置 |

#### 1.2.1 合并检查代码

```c
// block/blk-merge.c
bool blk_rq_merge_ok(struct request *rq, struct bio *bio)
{
    // 1. 操作类型必须相同
    if (req_op(rq) != bio_op(bio))
        return false;

    // 2. 必须是同一设备
    if (rq->rq_disk != bio->bi_bdev->bd_disk)
        return false;

    // 3. 检查是否超过段数限制
    if (rq->nr_phys_segments + bio_segments(bio) > 
        blk_rq_get_max_segments(rq))
        return false;

    // 4. 检查是否超过扇区数限制
    if (blk_rq_sectors(rq) + bio_sectors(bio) > 
        blk_rq_get_max_sectors(rq))
        return false;

    // 5. 检查扇区是否连续（前向或后向）
    sector_t req_end = blk_rq_pos(rq) + blk_rq_sectors(rq);
    sector_t bio_start = bio->bi_iter.bi_sector;
    
    if (req_end == bio_start)
        return true;  // 后向合并
    if (bio_start + bio_sectors(bio) == blk_rq_pos(rq))
        return true;  // 前向合并

    return false;
}
```

---

## 二、阶段3：Request 分配与 Internal Tag

### 2.1 Tag 分配流程概览

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Internal Tag 分配流程                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   bio无法合并                                                           │
│        │                                                               │
│        ▼                                                               │
│   blk_mq_get_request()                                                │
│        │                                                               │
│        ├─▶ 1. blk_mq_get_ctx()          获取当前CPU的ctx              │
│        ├─▶ 2. blk_mq_map_queue()        ctx映射到hctx                 │
│        ├─▶ 3. blk_mq_get_tag()          从sched_tags分配tag           │
│        │      │                                                        │
│        │      ├─▶ __sbitmap_queue_get()   从sbitmap分配               │
│        │      │   │                                                    │
│        │      │   ├─▶ 成功 ──▶ 返回tag (0-61)                         │
│        │      │   │                                                    │
│        │      │   └─▶ 失败 ──▶ io_schedule() 阻塞等待                 │
│        │      │                                                        │
│        │      └─▶ rq = tags->static_rqs[tag]  获取预分配的request     │
│        │                                                               │
│        └─▶ 4. blk_mq_rq_ctx_init()      初始化request字段             │
│               rq->q = q                                               │
│               rq->mq_ctx = ctx                                        │
│               rq->mq_hctx = hctx                                      │
│               rq->internal_tag = tag                                  │
│               rq->tag = BLK_MQ_NO_TAG                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.2 struct blk_mq_tags - Tag 管理结构

**源码位置**: `block/blk-mq-tag.h:8-31`

**在此阶段的作用**: 管理 Internal Tag 池，提供 tag 分配和 request 映射。

#### 2.2.1 完整结构体定义

```c
/*
 * Tag address space map.
 */
struct blk_mq_tags {
    unsigned int nr_tags;              // 总tag数量
    unsigned int nr_reserved_tags;     // 保留tag数量
    
    atomic_t active_queues;            // 活跃队列数（shared模式）
    
    struct sbitmap_queue *bitmap_tags;     // 普通tag位图指针
    struct sbitmap_queue *breserved_tags;  // 保留tag位图指针
    
    struct sbitmap_queue __bitmap_tags;     // 内联普通tag位图
    struct sbitmap_queue __breserved_tags;  // 内联保留tag位图
    
    struct request **rqs;              // request指针数组（动态）
    struct request **static_rqs;       // request指针数组（静态预分配）
    struct list_head page_list;        // 页列表（内存管理）
    
    spinlock_t lock;                   // 保护rqs数组
};
```

#### 2.2.2 核心字段详解 ⭐⭐⭐

| 字段名 | 类型 | 作用 | 典型值 |
|--------|------|------|--------|
| **Tag数量** ||||
| `nr_tags` | `unsigned int` | **总tag数量**<br/>= nr_reserved_tags + 普通tags | UFS设备: 62<br/>(2 * queue_depth) |
| `nr_reserved_tags` | `unsigned int` | **保留tag数量**<br/>给高优先级请求预留 | 通常为 nr_tags/10 |
| **位图管理** ||||
| `bitmap_tags` | `struct sbitmap_queue *` | **普通tag位图指针**<br/>指向 __bitmap_tags 或共享位图 | 实际分配tag的核心 |
| `breserved_tags` | `struct sbitmap_queue *` | **保留tag位图指针** | 高优先级请求使用 |
| **Request映射** ||||
| `static_rqs` | `struct request **` | **静态request数组**<br/>`static_rqs[tag]` = request | tag到request的映射<br/>预分配避免动态分配 |
| `rqs` | `struct request **` | request指针数组<br/>（运行时更新） | 快速查找正在使用的request |

**详细说明**：

**Tag 到 Request 的映射关系**：
```
blk_mq_tags {
    nr_tags = 62
    nr_reserved_tags = 6
    
    static_rqs ───┐
}                 │
                  ▼
    ┌──────────────────────────────────────────────────────┐
    │ [0] [1] [2] [3] [4] [5] │ [6] [7] ... [61]           │
    └──────────────────────────────────────────────────────┘
      │   │   │   │   │   │   │   │          │
      │   │   │   │   │   │   │   │          └─▶ request 61
      │   │   │   │   │   │   │   └────────────▶ request 7
      │   │   │   │   │   │   └────────────────▶ request 6
      │   │   │   │   │   └─────────────────────▶ request 5
      └───┴───┴───┴───┴──────────────────────────▶ request 0-4
          Reserved Tags (6个)        Normal Tags (56个)

// 分配时:
tag = __sbitmap_queue_get(tags->bitmap_tags);  // 假设返回10
rq = tags->static_rqs[tag];                     // 获取预分配的request
rq->internal_tag = tag;                         // 设置tag
```

**shared sbitmap 模式**：
```c
// 单设备多队列共享tag池（现代内核默认模式）
if (blk_mq_is_sbitmap_shared(set->flags)) {
    // 所有hctx共享同一个sbitmap
    tags->bitmap_tags = &queue->sched_bitmap_tags;
    tags->breserved_tags = &queue->sched_breserved_tags;
} else {
    // 每个hctx独立sbitmap
    tags->bitmap_tags = &tags->__bitmap_tags;
    tags->breserved_tags = &tags->__breserved_tags;
}
```

#### 2.2.3 重要字段 ⭐⭐

| 字段 | 类型 | 作用 |
|------|------|------|
| `active_queues` | `atomic_t` | 活跃队列数（shared模式scaling） |
| `__bitmap_tags` | `struct sbitmap_queue` | 内联位图（非shared模式） |
| `__breserved_tags` | `struct sbitmap_queue` | 内联保留位图 |
| `page_list` | `struct list_head` | request内存页列表 |
| `lock` | `spinlock_t` | 保护rqs数组 |

---

### 2.3 struct sbitmap_queue - 可扩展位图队列

**源码位置**: `include/linux/sbitmap.h:105-136`

**在此阶段的作用**: 高效的 tag 分配机制，支持多CPU并发、批量唤醒、避免争用。

#### 2.3.1 完整结构体定义

```c
/**
 * struct sbitmap_queue - Scalable bitmap with the added ability to wait on free
 * bits.
 *
 * A sbitmap_queue uses multiple wait queues and rolling wakeups to
 * avoid contention on the wait queue spinlock.
 */
struct sbitmap_queue {
    struct sbitmap sb;              // 核心位图
    
    unsigned int wake_batch;        // 批量唤醒数
    atomic_t wake_index;            // 下一个唤醒的等待队列索引
    
    struct sbq_wait_state *ws;      // 等待队列数组
    atomic_t ws_active;             // 活跃等待队列数
    
    unsigned int min_shallow_depth; // 最小浅深度
};

// 等待队列状态
struct sbq_wait_state {
    atomic_t wait_cnt;              // 唤醒前需要释放的tag数
    wait_queue_head_t wait;         // 等待队列
} ____cacheline_aligned_in_smp;

// 核心位图
struct sbitmap {
    unsigned int depth;             // 总位数
    unsigned int shift;             // word大小的log2值
    unsigned int map_nr;            // word数量
    bool round_robin;               // 是否轮询分配
    struct sbitmap_word *map;       // word数组
};
```

#### 2.3.2 核心字段详解 ⭐⭐⭐

| 字段名 | 类型 | 作用 | 示例 |
|--------|------|------|------|
| **位图核心** ||||
| `sb` | `struct sbitmap` | **核心位图结构**<br/>• `depth`: 总tag数(62)<br/>• `shift`: word大小log2(6)<br/>• `map`: 位图数组 | depth=62, shift=6<br/>需要 ceil(62/64)=1个word |
| **等待队列** ||||
| `ws` | `struct sbq_wait_state *` | **等待队列数组**<br/>多个等待队列避免争用 | 8个等待队列<br/>(SBQ_WAIT_QUEUES=8) |
| `ws_active` | `atomic_t` | 活跃等待队列计数 | 有进程等待时>0 |
| **批量唤醒** ||||
| `wake_batch` | `unsigned int` | **批量唤醒阈值**<br/>释放这么多tag后唤醒一次 | nr_tags >> sbq->sb.shift<br/>例如: 62 >> 6 = 0 → 设为1 |
| `wake_index` | `atomic_t` | 下一个唤醒的ws索引 | 0-7轮询 |

**详细说明**：

**位图布局**（以nr_tags=62为例）：
```
sbitmap {
    depth = 62
    shift = 6      (word大小 = 2^6 = 64位)
    map_nr = 1     (需要1个word存储62位)
    
    map[0]: |0|1|1|0|1|1|1|0|...|1|1| (64位)
            └┬┘                       └┬┘
             已分配                    未使用(62-63位)
             
    map ────▶ sbitmap_word {
                  word: 0xabcd...ef01     (位图)
                  depth: 62
                  cleared: 20             (已清除未唤醒数)
              }
}
```

**多等待队列避免争用**：
```
sbitmap_queue {
    ws[8] ───┐
}            │
             ▼
    ┌─────────────────────────────────────────────────────────┐
    │ ws[0] │ ws[1] │ ws[2] │ ... │ ws[7] │                   │
    └─────────────────────────────────────────────────────────┘
       │       │       │             │
       │       │       │             └─▶ wait_queue (CPU群D)
       │       │       └───────────────▶ wait_queue (CPU群C)
       │       └───────────────────────▶ wait_queue (CPU群B)
       └───────────────────────────────▶ wait_queue (CPU群A)

优点: 减少等待队列spinlock争用
```

**批量唤醒机制**：
```c
// 释放tag时（blk_mq_put_tag）
static inline void __sbitmap_queue_clear(...) {
    sbitmap_clear_bit(&bt->sb, bitnr);
    
    // 累积已释放tag数
    atomic_inc(&ws->wait_cnt);
    
    // 达到wake_batch才唤醒
    if (atomic_read(&ws->wait_cnt) >= bt->wake_batch) {
        atomic_set(&ws->wait_cnt, 0);
        wake_up_nr(&ws->wait, bt->wake_batch);  // 批量唤醒
    }
}
```

#### 2.3.3 分配和等待代码

```c
// block/blk-mq-tag.c
unsigned int blk_mq_get_tag(struct blk_mq_alloc_data *data)
{
    struct blk_mq_tags *tags = blk_mq_tags_from_data(data);
    struct sbitmap_queue *bt;
    struct sbq_wait_state *ws;
    DEFINE_SBQ_WAIT(wait);
    unsigned int tag_offset;
    int tag;

    // 1. 确定使用普通tag还是保留tag
    bt = tags->bitmap_tags;
    tag_offset = tags->nr_reserved_tags;

    // 2. 尝试获取tag
retry:
    tag = __sbitmap_queue_get(bt);
    if (tag != BLK_MQ_NO_TAG)
        goto found_tag;

    // 3. 没有可用tag，准备等待
    if (data->flags & BLK_MQ_REQ_NOWAIT)
        return BLK_MQ_NO_TAG;  // 非阻塞模式直接返回

    // 4. 加入等待队列
    ws = bt_wait_ptr(bt, data->hctx);
    do {
        sbitmap_prepare_to_wait(bt, ws, &wait, TASK_UNINTERRUPTIBLE);
        
        // 再次尝试
        tag = __sbitmap_queue_get(bt);
        if (tag != BLK_MQ_NO_TAG)
            break;

        // 阻塞等待
        io_schedule();  // 关键：进程进入D状态
        
        // 被唤醒后重试
        tag = __sbitmap_queue_get(bt);
    } while (tag == BLK_MQ_NO_TAG);

    sbitmap_finish_wait(bt, ws, &wait);

found_tag:
    return tag + tag_offset;
}
```

---

### 2.4 struct request - IO 请求（初始化阶段）

**源码位置**: `include/linux/blkdev.h:120-240`

**在此阶段的作用**: 刚分配出来，将 bio 的信息复制到 request，准备进入调度器。

#### 2.4.1 完整结构体定义

```c
struct request {
    // ====== 队列关联 ======
    struct request_queue *q;          // 所属请求队列
    struct blk_mq_ctx *mq_ctx;        // 软件队列（Per-CPU）
    struct blk_mq_hw_ctx *mq_hctx;    // 硬件队列
    
    // ====== 操作标识 ======
    unsigned int cmd_flags;           // 命令标志（从bio->bi_opf复制）
    req_flags_t rq_flags;             // 请求标志（RQF_*）
    
    // ====== Tag标识 ======
    int tag;                          // Driver Tag（初始为-1）
    int internal_tag;                 // Internal Tag（已分配）
    
    // ====== 数据描述 ======
    unsigned int __data_len;          // 总数据长度（字节）
    sector_t __sector;                // 起始扇区号
    
    // ====== Bio链表 ======
    struct bio *bio;                  // bio链表头
    struct bio *biotail;              // bio链表尾
    
    // ====== 队列链表 ======
    struct list_head queuelist;       // 队列链表节点
    
    // ====== 合并相关（union） ======
    union {
        struct hlist_node hash;       // 合并哈希表节点
        struct llist_node ipi_list;   // IPI完成列表
    };
    
    // ====== 调度器相关（union） ======
    union {
        struct rb_node rb_node;       // 红黑树节点（调度器用）
        struct bio_vec special_vec;
        void *completion_data;
    };
    
    // ====== 调度器私有数据（union） ======
    union {
        struct {
            struct io_cq *icq;        // IO上下文队列
            void *priv[2];            // 调度器私有数据
        } elv;
        struct {
            unsigned int seq;
            struct list_head list;
            rq_end_io_fn *saved_end_io;
        } flush;
    };
    
    // ====== 设备相关 ======
    struct gendisk *rq_disk;          // 目标磁盘
    struct block_device *part;        // 目标分区
    
    // ====== 时间戳 ======
    u64 alloc_time_ns;                // 分配时间
    u64 start_time_ns;                // 派发时间
    u64 io_start_time_ns;             // IO开始时间
    
    // ====== 物理段 ======
    unsigned short nr_phys_segments;  // 物理段数量（DMA用）
    unsigned short nr_integrity_segments; // 完整性段数量
    
    // ====== IO优先级 ======
    unsigned short write_hint;        // 写入提示
    unsigned short ioprio;            // IO优先级
    
    // ====== 状态管理 ======
    enum mq_rq_state state;           // 请求状态（IDLE/IN_FLIGHT/COMPLETE）
    refcount_t ref;                   // 引用计数
    
    // ====== 超时管理 ======
    unsigned int timeout;             // 超时时间（jiffies）
    unsigned long deadline;           // 超时deadline
    
    // ====== 调度器/完成（union） ======
    union {
        struct __call_single_data csd; // CPU间调用
        u64 fifo_time;                 // 调度器FIFO时间戳
    };
    
    // ====== 完成回调 ======
    rq_end_io_fn *end_io;             // 完成回调函数
    void *end_io_data;                // 完成回调数据
    
    // ====== 其他 ======
    unsigned short stats_sectors;      // 统计用扇区数
    unsigned short wbt_flags;          // write-back throttle标志
    struct bio_crypt_ctx *crypt_ctx;   // 加密上下文
    struct blk_ksm_keyslot *crypt_keyslot; // 密钥槽
};
```

#### 2.4.2 核心字段详解 ⭐⭐⭐（分配阶段填充）

| 字段名 | 类型 | 初始值 | 作用 |
|--------|------|--------|------|
| **队列关联** ||||
| `q` | `struct request_queue *` | 当前队列 | 访问队列配置、elevator、tag_set |
| `mq_ctx` | `struct blk_mq_ctx *` | 当前CPU的ctx | 后续插入到 `ctx->rq_lists` |
| `mq_hctx` | `struct blk_mq_hw_ctx *` | 映射的hctx | 派发目标、获取Driver Tag |
| **Tag标识** ||||
| `internal_tag` | `int` | 刚分配的tag<br/>(0-61) | **调度器使用**<br/>request查找: `tags->rqs[tag]` |
| `tag` | `int` | BLK_MQ_NO_TAG<br/>(-1) | **驱动使用**（稍后分配）<br/>硬件队列标签 |
| **从Bio复制** ||||
| `cmd_flags` | `unsigned int` | `bio->bi_opf` | 操作类型+标志<br/>判断读/写、同步/异步 |
| `__sector` | `sector_t` | `bio->bi_iter.bi_sector` | 起始LBA（512字节为单位） |
| `__data_len` | `unsigned int` | `bio->bi_iter.bi_size` | 总字节数 |
| `ioprio` | `unsigned short` | `bio->bi_ioprio` | IO优先级（调度器使用） |
| **Bio链表** ||||
| `bio` | `struct bio *` | 第一个bio | 链表头，驱动遍历获取数据 |
| `biotail` | `struct bio *` | 最后一个bio | 合并时链接新bio |

**关键理解**：

```c
// 初始化代码（简化版）
static struct request *blk_mq_rq_ctx_init(struct blk_mq_alloc_data *data,
                                          unsigned int tag)
{
    struct blk_mq_tags *tags = blk_mq_tags_from_data(data);
    struct request *rq = tags->static_rqs[tag];

    // 1. 队列关联
    rq->q = data->q;
    rq->mq_ctx = data->ctx;
    rq->mq_hctx = data->hctx;

    // 2. Tag标识
    if (data->q->elevator) {
        rq->tag = BLK_MQ_NO_TAG;        // ✗ Driver Tag未分配
        rq->internal_tag = tag;         // ✓ Internal Tag已分配
    } else {
        rq->tag = tag;                  // 无调度器直接分配Driver Tag
        rq->internal_tag = BLK_MQ_NO_TAG;
    }

    // 3. 操作标识
    rq->cmd_flags = data->cmd_flags;
    rq->rq_flags = 0;

    // 4. 状态初始化
    rq->state = MQ_RQ_IDLE;
    refcount_set(&rq->ref, 1);

    return rq;
}

// Bio转换为request
static void blk_mq_bio_to_request(struct request *rq, struct bio *bio,
                                   unsigned int nr_segs)
{
    rq->__sector = bio->bi_iter.bi_sector;
    rq->__data_len = bio->bi_iter.bi_size;
    rq->bio = rq->biotail = bio;
    rq->ioprio = bio->bi_ioprio;
    rq->write_hint = bio->bi_write_hint;
    rq->nr_phys_segments = nr_segs;
    
    // 更新统计
    rq->stats_sectors = blk_rq_sectors(rq);
}
```

#### 2.4.3 重要字段 ⭐⭐

| 字段 | 类型 | 用途 | 更新时机 |
|------|------|------|---------|
| **队列链表** ||||
| `queuelist` | `struct list_head` | 队列链表节点 | 插入ctx/调度器/dispatch时 |
| `rb_node` | `struct rb_node` | 红黑树节点 | 调度器按扇区排序时 |
| `hash` | `struct hlist_node` | 合并哈希表 | 支持后向合并时 |
| **状态管理** ||||
| `rq_flags` | `req_flags_t` | 请求标志 | RQF_STARTED/RQF_QUEUED等 |
| `state` | `enum mq_rq_state` | 状态 | IDLE → IN_FLIGHT → COMPLETE |
| `ref` | `refcount_t` | 引用计数 | 防止过早释放 |
| **时间戳** ||||
| `alloc_time_ns` | `u64` | 分配时间 | 延迟统计（分配到派发） |
| `start_time_ns` | `u64` | 派发时间 | 延迟统计（派发到完成） |

#### 2.4.4 次要字段 ⭐（分组说明）

| 字段组 | 字段 | 说明 |
|--------|------|------|
| **调度器私有** | `elv.icq`, `elv.priv[2]`, `fifo_time` | 调度器使用 |
| **物理映射** | `nr_phys_segments`, `nr_integrity_segments` | DMA映射 |
| **设备信息** | `rq_disk`, `part` | 目标磁盘和分区 |
| **超时管理** | `timeout`, `deadline` | 超时检测 |
| **特殊功能** | `crypt_ctx`, `wbt_flags`, `write_hint` | 加密、throttle、提示 |
| **完成回调** | `end_io`, `end_io_data` | 驱动完成回调 |
| **Flush专用** | `flush.seq`, `flush.list` | flush请求 |

---

## 三、阶段4：插入软件队列

### 3.1 struct blk_mq_ctx - 软件队列（Per-CPU）

**源码位置**: `block/blk-mq.h:16-40`

**在此阶段的作用**: 暂存本 CPU 提交的请求，是 CPU 和硬件队列之间的缓冲。

#### 3.1.1 完整结构体定义

```c
/**
 * struct blk_mq_ctx - State for a software queue facing the submitting CPUs
 */
struct blk_mq_ctx {
    struct {
        spinlock_t      lock;                       // 保护rq_lists
        struct list_head rq_lists[HCTX_MAX_TYPES];  // 请求链表数组
    } ____cacheline_aligned_in_smp;

    unsigned int        cpu;                        // 所属CPU
    unsigned short      index_hw[HCTX_MAX_TYPES];   // 在hctx中的索引
    struct blk_mq_hw_ctx *hctxs[HCTX_MAX_TYPES];    // 映射到的硬件队列

    /* incremented at dispatch time */
    unsigned long       rq_dispatched[2];           // 已派发计数（读/写）
    unsigned long       rq_merged;                  // 已合并计数

    /* incremented at completion time */
    unsigned long       rq_completed[2];            // 已完成计数

    struct request_queue *queue;                    // 所属请求队列
    struct blk_mq_ctxs  *ctxs;                      // 所属ctx集合
    struct kobject      kobj;                       // sysfs对象
} ____cacheline_aligned_in_smp;
```

#### 3.1.2 核心字段详解 ⭐⭐⭐

| 字段名 | 类型 | 作用 | 使用场景 |
|--------|------|------|---------|
| **请求存储** ||||
| `lock` | `spinlock_t` | **保护rq_lists** | 插入/取出请求时加锁 |
| `rq_lists[HCTX_MAX_TYPES]` | `struct list_head` | **请求链表数组**<br/>• [0]: DEFAULT<br/>• [1]: READ<br/>• [2]: POLL | request通过queuelist挂载 |
| **映射关系** ||||
| `cpu` | `unsigned int` | **所属CPU编号** | 确定Per-CPU队列 |
| `index_hw[type]` | `unsigned short` | **在hctx中的索引**<br/>即 `hctx->ctxs[index_hw]` = 本ctx | 用于设置/清除<br/>`hctx->ctx_map`位图 |
| `hctxs[type]` | `struct blk_mq_hw_ctx *` | **映射到的硬件队列** | 后续派发的目标 |
| `queue` | `struct request_queue *` | 所属请求队列 | 访问队列配置 |

**详细说明**：

**软件队列与硬件队列的映射**：
```
8个CPU，1个硬件队列的情况（UFS设备典型配置）

blk_mq_ctx (Per-CPU)                    blk_mq_hw_ctx
┌──────────┐                            ┌────────────────────┐
│  ctx 0   │ index_hw[0]=0              │     hctx 0         │
│  CPU 0   │ ────────────────────┐      │                    │
└──────────┘                     │      │  ctxs[] ────┐      │
┌──────────┐                     │      │             │      │
│  ctx 1   │ index_hw[0]=1       ├─────▶│  [0] [1] .. [7]    │
│  CPU 1   │ ────────────────────┤      │   │   │      │     │
└──────────┘                     │      │   │   │      │     │
┌──────────┐                     │      │  ctx_map:          │
│  ctx 2   │ index_hw[0]=2       │      │  [0][1][2][3]      │
│  CPU 2   │ ────────────────────┤      │   [4][5][6][7]     │
└──────────┘                     │      │   ↑ 标记哪些       │
     ...                         │      │     ctx有请求       │
┌──────────┐                     │      │                    │
│  ctx 7   │ index_hw[0]=7       │      │                    │
│  CPU 7   │ ────────────────────┘      └────────────────────┘
└──────────┘

插入流程:
1. CPU 2提交IO → 使用ctx 2
2. request插入: ctx2->rq_lists[DEFAULT]
3. 标记位图: hctx->ctx_map 的第2位设为1
4. 派发时: hctx遍历ctx_map, 取出所有有请求的ctx
```

**插入代码**：
```c
// block/blk-mq.c
void __blk_mq_insert_request(struct blk_mq_hw_ctx *hctx, 
                              struct request *rq, bool at_head)
{
    struct blk_mq_ctx *ctx = rq->mq_ctx;
    enum hctx_type type = hctx->type;

    lockdep_assert_held(&ctx->lock);

    trace_block_rq_insert(rq);

    // 插入链表
    if (at_head)
        list_add(&rq->queuelist, &ctx->rq_lists[type]);
    else
        list_add_tail(&rq->queuelist, &ctx->rq_lists[type]);
    
    // 标记hctx的ctx_map（通知hctx此ctx有请求）
    blk_mq_hctx_mark_pending(hctx, ctx);
}

// 标记pending
static void blk_mq_hctx_mark_pending(struct blk_mq_hw_ctx *hctx,
                                     struct blk_mq_ctx *ctx)
{
    const int bit = ctx->index_hw[hctx->type];

    if (!sbitmap_test_bit(&hctx->ctx_map, bit))
        sbitmap_set_bit(&hctx->ctx_map, bit);  // 设置位图
}
```

#### 3.1.3 重要字段 ⭐⭐（统计）

| 字段 | 类型 | 作用 |
|------|------|------|
| `rq_dispatched[2]` | `unsigned long` | 已派发计数（读/写） |
| `rq_merged` | `unsigned long` | 已合并计数 |
| `rq_completed[2]` | `unsigned long` | 已完成计数 |

#### 3.1.4 次要字段 ⭐

| 字段 | 说明 |
|------|------|
| `ctxs` | 所属ctx集合（管理结构） |
| `kobj` | sysfs对象（`/sys/block/sda/mq/0/cpu0/`） |

---

### 3.2 软件队列的实际作用

#### 3.2.1 为什么需要软件队列？

```
┌────────────────────────────────────────────────────────────────────┐
│                    软件队列的必要性                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  问题: 多CPU并发提交IO到一个硬件队列会导致锁争用                      │
│                                                                    │
│  方案: Per-CPU软件队列作为缓冲                                       │
│                                                                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │ CPU 0   │  │ CPU 1   │  │ CPU 2   │  │ CPU 3   │             │
│  │ 提交IO  │  │ 提交IO  │  │ 提交IO  │  │ 提交IO  │             │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘             │
│       │            │            │            │                   │
│       ▼            ▼            ▼            ▼                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │  ctx 0  │  │  ctx 1  │  │  ctx 2  │  │  ctx 3  │  无争用!    │
│  │ rq_lists│  │ rq_lists│  │ rq_lists│  │ rq_lists│             │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘             │
│       └────────┬────────┴────┬────────┴────┘                     │
│                │              │                                   │
│                ▼              ▼                                   │
│         ┌───────────────────────────┐                             │
│         │      Hardware Queue       │                             │
│         │  批量收集ctx中的请求       │  减少锁争用                  │
│         └───────────────────────────┘                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 从软件队列收集请求

```c
// block/blk-mq.c - 硬件队列收集请求
void blk_mq_flush_busy_ctxs(struct blk_mq_hw_ctx *hctx, struct list_head *list)
{
    struct flush_busy_ctx_data data = {
        .hctx = hctx,
        .list = list,
    };

    // 遍历ctx_map中标记为有请求的ctx
    sbitmap_for_each_set(&hctx->ctx_map, flush_busy_ctx, &data);
}

// 回调函数 - 收集单个ctx的请求
static bool flush_busy_ctx(struct sbitmap *sb, unsigned int bitnr, void *data)
{
    struct flush_busy_ctx_data *flush_data = data;
    struct blk_mq_hw_ctx *hctx = flush_data->hctx;
    struct blk_mq_ctx *ctx = hctx->ctxs[bitnr];  // 通过索引获取ctx
    enum hctx_type type = hctx->type;

    spin_lock(&ctx->lock);
    // 将ctx的请求链表移动到临时list
    list_splice_tail_init(&ctx->rq_lists[type], flush_data->list);
    // 清除ctx_map标记
    sbitmap_clear_bit(sb, bitnr);
    spin_unlock(&ctx->lock);
    
    return true;
}
```

---

## 四、完整流程代码路径

### 4.1 从 bio 到 request 的完整代码

```c
// block/blk-mq.c
blk_status_t blk_mq_submit_bio(struct bio *bio)
{
    struct request_queue *q = bdev_get_queue(bio->bi_bdev);
    struct request *rq;

    // 1. 尝试bio合并到已有request
    if (blk_attempt_bio_merge(q, bio, ...)) {
        // 合并成功，无需分配新request
        return BLK_STS_OK;
    }

    // 2. 分配request + Internal Tag
    rq = blk_mq_get_request(q, bio, ...);
    if (!rq) {
        // Tag不足，bio会被阻塞或重试
        return BLK_STS_RESOURCE;
    }

    // 3. Bio转换为request
    blk_mq_bio_to_request(rq, bio, nr_segs);

    // 4. 插入软件队列（或直接插入调度器）
    blk_mq_sched_insert_request(rq, false, true, true);
    
    return BLK_STS_OK;
}
```

### 4.2 请求插入流程

```c
// block/blk-mq-sched.c
void blk_mq_sched_insert_request(struct request *rq, bool at_head,
                                 bool run_queue, bool async)
{
    struct request_queue *q = rq->q;
    struct elevator_queue *e = q->elevator;
    struct blk_mq_ctx *ctx = rq->mq_ctx;
    struct blk_mq_hw_ctx *hctx = rq->mq_hctx;

    if (e) {
        // 有调度器：插入调度器队列
        LIST_HEAD(list);
        list_add(&rq->queuelist, &list);
        e->type->ops.insert_requests(hctx, &list, at_head);
    } else {
        // 无调度器：插入软件队列
        spin_lock(&ctx->lock);
        __blk_mq_insert_request(hctx, rq, at_head);
        spin_unlock(&ctx->lock);
    }

    // 触发派发
    if (run_queue)
        blk_mq_run_hw_queue(hctx, async);
}
```

---

## 五、关键数据流转

### 5.1 Request 在各阶段的字段变化

| 阶段 | internal_tag | tag | state | queuelist | 位置 |
|------|-------------|-----|-------|-----------|------|
| 分配 | 10 (已分配) | -1 | IDLE | 空 | 刚分配 |
| 插入ctx | 10 | -1 | IDLE | 挂在ctx->rq_lists | 软件队列 |
| 收集 | 10 | -1 | IDLE | 临时list | 准备派发 |
| 插入调度器 | 10 | -1 | IDLE | 调度器队列 | 等待选择 |
| 派发 | 10 | 5 (已分配) | IN_FLIGHT | hctx->dispatch | 等待Driver Tag |
| 驱动处理 | 10 | 5 | IN_FLIGHT | 驱动队列 | 硬件执行 |
| 完成 | -1 (释放) | -1 (释放) | COMPLETE | 空 | 完成回收 |

### 5.2 Tag 的两次等待

```
┌────────────────────────────────────────────────────────────────────┐
│                     Tag 分配的两个阶段                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  第一次等待: Internal Tag (阶段3)                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ blk_mq_get_tag(sched_tags)                                │     │
│  │   ├─▶ 成功: tag=10, 继续                                  │     │
│  │   └─▶ 失败: io_schedule() 阻塞                            │     │
│  │             进程进入D状态                                   │     │
│  │             等待其他IO完成释放tag                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                    │
│  第二次等待: Driver Tag (阶段6，下篇详解)                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │ blk_mq_get_driver_tag(tags)                               │     │
│  │   ├─▶ 成功: tag=5, 派发到驱动                             │     │
│  │   └─▶ 失败: 不阻塞，放入hctx->dispatch                     │     │
│  │             等待下次重试                                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 六、实例分析

### 6.1 8核UFS设备的Tag配置

```bash
# 实际设备输出
$ cat /sys/block/sda/queue/nr_requests
62

$ cat /sys/block/sda/device/queue_depth
31

$ ls /sys/block/sda/mq/
0    # 单个硬件队列

$ cat /sys/block/sda/mq/0/cpu_list
0-7  # 8个CPU都映射到hctx 0
```

**结构体配置**：
```
blk_mq_tags (sched_tags)
├─ nr_tags = 62              Internal Tag池
├─ nr_reserved_tags = 6      保留6个给高优先级
├─ bitmap_tags ──▶ sbitmap_queue {
│                     depth = 62
│                     ws[8]    8个等待队列
│                  }
└─ static_rqs[62]            预分配62个request

blk_mq_tags (tags)
├─ nr_tags = 31              Driver Tag池
├─ nr_reserved_tags = 3
├─ bitmap_tags ──▶ sbitmap_queue {
│                     depth = 31
│                  }
└─ static_rqs[31]            预分配31个request

blk_mq_hw_ctx (hctx 0)
├─ sched_tags ──▶ 上面的sched_tags
├─ tags ──▶ 上面的tags
├─ nr_ctx = 8                8个软件队列
└─ ctxs[] = {ctx0, ctx1, ..., ctx7}

blk_mq_ctx (ctx 0-7, 每个CPU一个)
├─ cpu = 0/1/.../7
├─ index_hw[0] = 0/1/.../7   在hctx->ctxs中的索引
├─ hctxs[0] ──▶ hctx 0
└─ rq_lists[DEFAULT]         请求链表
```

### 6.2 并发场景分析

**场景：4个CPU同时提交IO**

```
时刻T0: 4个CPU同时提交

CPU 0                  CPU 1                  CPU 2                  CPU 3
  │                      │                      │                      │
  ├─ blk_mq_submit_bio   ├─ blk_mq_submit_bio   ├─ blk_mq_submit_bio   ├─ blk_mq_submit_bio
  │                      │                      │                      │
  ├─ get_tag(tag=0)      ├─ get_tag(tag=1)      ├─ get_tag(tag=2)      ├─ get_tag(tag=3)
  │  from sched_tags     │  from sched_tags     │  from sched_tags     │  from sched_tags
  │                      │                      │                      │
  ├─ rq0=static_rqs[0]   ├─ rq1=static_rqs[1]   ├─ rq2=static_rqs[2]   ├─ rq3=static_rqs[3]
  │                      │                      │                      │
  ├─ rq0->mq_ctx=ctx0    ├─ rq1->mq_ctx=ctx1    ├─ rq2->mq_ctx=ctx2    ├─ rq3->mq_ctx=ctx3
  │                      │                      │                      │
  ├─ lock(&ctx0->lock)   ├─ lock(&ctx1->lock)   ├─ lock(&ctx2->lock)   ├─ lock(&ctx3->lock)
  │  (无争用!)           │  (无争用!)           │  (无争用!)           │  (无争用!)
  │                      │                      │                      │
  ├─ insert to          ├─ insert to          ├─ insert to          ├─ insert to
  │  ctx0->rq_lists     │  ctx1->rq_lists     │  ctx2->rq_lists     │  ctx3->rq_lists
  │                      │                      │                      │
  ├─ set ctx_map[0]      ├─ set ctx_map[1]      ├─ set ctx_map[2]      ├─ set ctx_map[3]
  │                      │                      │                      │
  └─ unlock              └─ unlock              └─ unlock              └─ unlock


时刻T1: 硬件队列派发（批量收集）

hctx 0
  │
  ├─ blk_mq_run_hw_queue()
  │
  ├─ blk_mq_flush_busy_ctxs()  收集所有有请求的ctx
  │    ├─ 遍历 ctx_map [0,1,2,3] 位都是1
  │    ├─ 从 ctx0 取出 rq0
  │    ├─ 从 ctx1 取出 rq1
  │    ├─ 从 ctx2 取出 rq2
  │    └─ 从 ctx3 取出 rq3
  │
  └─ 批量派发 [rq0, rq1, rq2, rq3]

优势: 
1. CPU提交阶段无锁争用（每个CPU自己的ctx）
2. 硬件队列批量收集，减少派发开销
```

---

## 七、总结与下篇预告

### 7.1 本文要点

1. **阶段2：bio合并**
   - `request_queue` 提供合并限制参数
   - 合并可节省 Internal Tag

2. **阶段3：Request分配与Internal Tag**
   - `blk_mq_tags` 管理 tag 池（nr_tags=62）
   - `sbitmap_queue` 高效分配机制（多等待队列、批量唤醒）
   - `request` 初始化（从bio复制信息，获取internal_tag）

3. **阶段4：插入软件队列**
   - `blk_mq_ctx` 是 Per-CPU 队列（避免锁争用）
   - 通过 `index_hw` 和 `ctx_map` 实现高效映射

### 7.2 关键对比

| 概念 | Internal Tag | 软件队列 |
|------|-------------|---------|
| **数量** | nr_requests (62) | CPU数量 (8) |
| **作用** | 限制并发请求数 | 减少锁争用 |
| **分配** | sbitmap_queue | 静态分配 |
| **等待** | io_schedule() 阻塞 | 无等待 |

### 7.3 下篇预告

**[25-IO调度器结构体详解](./25-IO调度器结构体详解.md)** 将介绍：

- **阶段5：调度器处理**
  - `elevator_queue` - 调度器队列
  - `deadline_data` - mq-deadline 全局数据
  - `dd_per_prio` - 每个优先级的数据

**关键问题**：
- 调度器如何根据 IO 优先级排序？
- 红黑树和 FIFO 如何配合？
- 防饥饿机制如何实现？

---

## 附录：快速参考

### A. 核心结构体速查

| 结构体 | 核心字段 | 作用 |
|--------|---------|------|
| `blk_mq_tags` | `nr_tags`, `bitmap_tags`, `static_rqs` | Tag池管理 |
| `sbitmap_queue` | `sb`, `ws`, `wake_batch` | 位图分配 |
| `request` | `q`, `mq_ctx`, `mq_hctx`, `internal_tag`, `bio` | IO请求 |
| `blk_mq_ctx` | `rq_lists`, `index_hw`, `hctxs` | 软件队列 |

### B. 常用访问函数

```c
// Tag相关
blk_mq_tags_from_data(data)     // 获取tags（sched_tags或tags）
blk_mq_tag_is_reserved(tags, tag)  // 是否为保留tag
blk_mq_get_tag(data)            // 分配Internal Tag
blk_mq_put_tag(tags, ctx, tag)  // 释放tag

// Request相关
req_op(rq)                      // 获取操作类型
blk_rq_pos(rq)                  // 起始扇区 (rq->__sector)
blk_rq_sectors(rq)              // 扇区数 (__data_len >> 9)
blk_rq_bytes(rq)                // 字节数 (__data_len)
```

### C. 调试命令

```bash
# 查看tag使用情况
cat /sys/kernel/debug/block/sda/hctx0/sched_tags
# 输出:
# nr_tags=62
# nr_reserved_tags=6
# active_queues=1

# 查看位图状态
cat /sys/kernel/debug/block/sda/hctx0/sched_tags_bitmap

# 查看软件队列统计
cat /sys/block/sda/mq/0/cpu0/dispatched
cat /sys/block/sda/mq/0/cpu0/merged
cat /sys/block/sda/mq/0/cpu0/completed
```

---

**上一篇**: [23-IO流程结构体总览与Bio层](./23-IO流程结构体总览与Bio层.md)  
**下一篇**: [25-IO调度器结构体详解](./25-IO调度器结构体详解.md)  
**返回**: [系列文章目录](./README.md)
