# IO 流程关键结构体详解（四）：硬件队列与 Tag 派发

> 本文是 **IO 流程关键结构体详解系列** 的第四篇，重点介绍硬件队列（`blk_mq_hw_ctx`）的所有字段，以及 Driver Tag 的分配机制。

**系列文章**：[查看完整系列](./README.md)  
**上一篇**：[25-IO调度器结构体详解](./25-IO调度器结构体详解.md)  
**下一篇**：[27-驱动处理与IO完成](./27-驱动处理与IO完成.md)

---

## 本篇涵盖的流程阶段

```
┌────────────────────────────────────────────────────────────────────────┐
│   阶段6                              阶段7                              │
│                                                                        │
│   触发派发与硬件队列           ──▶    Driver Tag 分配                   │
│                                                                        │
│   blk_mq_hw_ctx                     blk_mq_get_driver_tag()           │
│   (收集请求、分配Driver Tag)         request状态更新                    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 一、阶段6：触发派发与硬件队列

### 1.1 blk_mq_hw_ctx 的核心作用

```
┌────────────────────────────────────────────────────────────────────┐
│              blk_mq_hw_ctx - 派发的核心                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  作用1: 连接软件队列和驱动                                          │
│  ┌─────────┐  ┌─────────┐                                         │
│  │  ctx 0  │  │  ctx 1  │  ... (软件队列)                         │
│  └────┬────┘  └────┬────┘                                         │
│       └────┬────────┘                                              │
│            ▼                                                       │
│     ┌─────────────────┐                                            │
│     │  blk_mq_hw_ctx  │                                            │
│     │  • 收集请求      │                                            │
│     │  • 分配tag      │                                            │
│     │  • 调用调度器    │                                            │
│     └────────┬────────┘                                            │
│              ▼                                                     │
│         Storage Driver                                             │
│                                                                    │
│  作用2: 管理两种Tag池                                              │
│     sched_tags (Internal Tag, 62个)                                │
│     tags (Driver Tag, 31个)                                        │
│                                                                    │
│  作用3: 提供dispatch队列（Driver Tag不足时暂存）                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 二、struct blk_mq_hw_ctx - 硬件队列

**源码位置**: `include/linux/blk-mq.h:17-180`

**在此阶段的作用**: 派发的核心，连接调度器、软件队列和驱动。

### 2.1 完整结构体定义

```c
/**
 * struct blk_mq_hw_ctx - State for a hardware queue facing the hardware
 * block device
 */
struct blk_mq_hw_ctx {
    // ====== 派发核心 ======
    struct {
        spinlock_t      lock;           // 保护dispatch列表
        struct list_head dispatch;      // 待派发队列
        unsigned long   state;          // 队列状态（BLK_MQ_S_*）
    } ____cacheline_aligned_in_smp;

    struct delayed_work run_work;       // 延迟派发工作
    cpumask_var_t   cpumask;            // 可运行的CPU掩码
    int             next_cpu;           // 下一个CPU（round-robin）
    int             next_cpu_batch;     // CPU批次计数

    unsigned long   flags;              // 队列标志（BLK_MQ_F_*）

    // ====== 调度器接口 ======
    void            *sched_data;        // 调度器私有数据
    struct request_queue *queue;        // 所属请求队列
    struct blk_flush_queue *fq;         // flush队列

    void            *driver_data;       // 驱动私有数据

    // ====== 软件队列映射 ======
    struct sbitmap  ctx_map;            // 软件队列位图
    struct blk_mq_ctx *dispatch_from;   // 上次派发的ctx
    unsigned int    dispatch_busy;      // 繁忙度

    unsigned short  type;               // 队列类型
    unsigned short  nr_ctx;             // 软件队列数量
    struct blk_mq_ctx **ctxs;           // 软件队列指针数组

    // ====== Driver Tag 等待 ======
    spinlock_t      dispatch_wait_lock; // 等待队列锁
    wait_queue_entry_t dispatch_wait;   // 等待队列入口
    atomic_t        wait_index;         // 等待队列索引

    // ====== Tag 管理（核心） ======
    struct blk_mq_tags *tags;           // Driver Tag池
    struct blk_mq_tags *sched_tags;     // Internal Tag池

    // ====== 统计信息 ======
    unsigned long   queued;             // 已入队请求数
    unsigned long   run;                // 派发次数
    unsigned long   dispatched[BLK_MQ_MAX_DISPATCH_ORDER]; // 按批次统计

    // ====== NUMA与编号 ======
    unsigned int    numa_node;          // NUMA节点
    unsigned int    queue_num;          // 硬件队列编号
    atomic_t        nr_active;          // 活跃请求数

    // ====== CPU热插拔 ======
    struct hlist_node cpuhp_online;
    struct hlist_node cpuhp_dead;

    // ====== sysfs ======
    struct kobject  kobj;

    // ====== Polling ======
    unsigned long   poll_considered;
    unsigned long   poll_invoked;
    unsigned long   poll_success;

    // ====== 调试 ======
    struct dentry   *debugfs_dir;
    struct dentry   *sched_debugfs_dir;

    // ====== 其他 ======
    struct list_head hctx_list;
    struct srcu_struct srcu[];          // Sleepable RCU
};
```

### 2.2 核心字段详解 ⭐⭐⭐

#### 2.2.1 派发核心字段

| 字段名 | 类型 | 作用 | 使用场景 |
|--------|------|------|---------|
| `lock` | `spinlock_t` | **保护dispatch列表** | 插入/取出dispatch请求时加锁 |
| `dispatch` | `struct list_head` | **待派发队列**<br/>Driver Tag分配失败时暂存 | 下次派发时优先处理<br/>保证公平性 |
| `state` | `unsigned long` | **队列状态标志**<br/>• BLK_MQ_S_STOPPED: 停止<br/>• BLK_MQ_S_SCHED_RESTART: 需要重启 | 控制派发行为 |
| `run_work` | `struct delayed_work` | 延迟派发工作 | `blk_mq_delay_run_hw_queues()` |

**dispatch队列详解**：
```
为什么需要dispatch队列？

场景: Driver Tag不足

┌─────────────────────────────────────────────────────────────┐
│ 调度器选择了5个请求，但Driver Tag只剩3个                      │
│                                                             │
│  e->ops.dispatch_request() 返回:                            │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                       │
│  │ r1 │ │ r2 │ │ r3 │ │ r4 │ │ r5 │                       │
│  └────┘ └────┘ └────┘ └────┘ └────┘                       │
│     │      │      │      │      │                          │
│     ▼      ▼      ▼      ▼      ▼                          │
│  blk_mq_get_driver_tag():                                  │
│   ✓tag   ✓tag   ✓tag   ✗失败  ✗失败                        │
│     │      │      │      │      │                          │
│     │      │      │      └──┬───┘                          │
│     │      │      │         │                              │
│     ▼      ▼      ▼         ▼                              │
│  派发到驱动       放入 hctx->dispatch                        │
│                                                             │
│  下次派发时:                                                 │
│  1. 优先处理 dispatch 队列                                   │
│  2. Driver Tag释放后可以派发                                 │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2.2 Tag 管理字段（最核心）

| 字段名 | 类型 | 作用 | 对比 |
|--------|------|------|------|
| **Tag池** ||||
| `sched_tags` | `struct blk_mq_tags *` | **Internal Tag池**<br/>• 数量: nr_requests (62)<br/>• 分配时机: request分配时<br/>• 存储位置: `rq->internal_tag` | 调度器使用<br/>决定并发上限 |
| `tags` | `struct blk_mq_tags *` | **Driver Tag池**<br/>• 数量: queue_depth (31)<br/>• 分配时机: 派发时<br/>• 存储位置: `rq->tag` | 驱动使用<br/>硬件队列标签 |
| **等待机制** ||||
| `dispatch_wait` | `wait_queue_entry_t` | Driver Tag等待队列 | tag不足时请求加入此队列 |
| `dispatch_wait_lock` | `spinlock_t` | 保护等待队列 | - |
| `wait_index` | `atomic_t` | 等待队列索引 | 轮询选择等待队列 |

**关键理解**：
```
blk_mq_hw_ctx {
    sched_tags ───┐  Internal Tag池
                  │  
                  ▼  
    blk_mq_tags {
        nr_tags = 62
        bitmap_tags ──▶ sbitmap_queue (62位)
        static_rqs[62] ──▶ 预分配的request
    }
    
    tags ────────┐  Driver Tag池
                 │
                 ▼
    blk_mq_tags {
        nr_tags = 31
        bitmap_tags ──▶ sbitmap_queue (31位)
        static_rqs[31] ──▶ 预分配的request
    }
}

关键区别:
1. Internal Tag先分配，Driver Tag后分配
2. Internal Tag不足会阻塞（io_schedule）
3. Driver Tag不足不阻塞，放入dispatch队列
```

#### 2.2.3 软件队列映射字段

| 字段名 | 类型 | 作用 |
|--------|------|------|
| `ctx_map` | `struct sbitmap` | **软件队列位图**<br/>标记哪些ctx有请求 | 派发时遍历有请求的ctx<br/>避免遍历空ctx |
| `ctxs` | `struct blk_mq_ctx **` | **软件队列指针数组**<br/>`ctxs[i]` = 第i个ctx | 通过index_hw快速访问 |
| `nr_ctx` | `unsigned short` | 软件队列数量 | 通常等于CPU数 |
| `dispatch_from` | `struct blk_mq_ctx *` | 上次派发的ctx | 无调度器时使用<br/>round-robin |

**ctx_map 位图机制**：
```
8个CPU，1个硬件队列

hctx->ctxs[]    hctx->ctx_map (sbitmap)
┌─────────┐     ┌───┬───┬───┬───┬───┬───┬───┬───┐
│ ctxs[0] │ ──▶ │ 1 │ 0 │ 1 │ 1 │ 0 │ 0 │ 1 │ 0 │
│  = ctx0 │     └───┴───┴───┴───┴───┴───┴───┴───┘
├─────────┤       ↑       ↑   ↑           ↑
│ ctxs[1] │       │       │   │           │
│  = ctx1 │       │       │   │           └─ CPU6有请求
├─────────┤       │       │   └─ CPU3有请求
│ ctxs[2] │       │       └─ CPU2有请求
│  = ctx2 │       └─ CPU0有请求
├─────────┤
│   ...   │     派发时遍历:
├─────────┤     sbitmap_for_each_set(&ctx_map, flush_busy_ctx, ...)
│ ctxs[7] │     只处理位为1的ctx (0,2,3,6)
│  = ctx7 │
└─────────┘
```

#### 2.2.4 调度器接口字段

| 字段名 | 类型 | 作用 |
|--------|------|------|
| `sched_data` | `void *` | **调度器私有数据**<br/>对mq-deadline: 指向`deadline_data` | 访问调度器状态 |
| `queue` | `struct request_queue *` | 所属请求队列 | 访问队列配置、elevator |

#### 2.2.5 统计字段

| 字段名 | 类型 | 作用 |
|--------|------|------|
| `queued` | `unsigned long` | 已入队请求数（累计） |
| `run` | `unsigned long` | 派发次数（累计） |
| `dispatched[7]` | `unsigned long` | **按批次大小统计**<br/>dispatched[0]=单个<br/>dispatched[1]=2个<br/>dispatched[2]=3-4个<br/>... | 性能分析<br/>批处理效果 |

### 2.3 重要字段 ⭐⭐

| 字段 | 类型 | 作用 |
|------|------|------|
| **运行控制** |||
| `flags` | `unsigned long` | 队列标志（BLK_MQ_F_*） |
| `type` | `unsigned short` | 队列类型（DEFAULT/READ/POLL） |
| `cpumask` | `cpumask_var_t` | 可运行的CPU掩码 |
| `dispatch_busy` | `unsigned int` | 繁忙度（EWMA算法） |
| **设备信息** |||
| `numa_node` | `unsigned int` | NUMA节点 |
| `queue_num` | `unsigned int` | 硬件队列编号 |
| `nr_active` | `atomic_t` | 活跃请求数（shared tag） |
| **其他** |||
| `fq` | `struct blk_flush_queue *` | flush队列 |
| `driver_data` | `void *` | 驱动私有数据 |

### 2.4 次要字段 ⭐

| 字段组 | 字段 | 说明 |
|--------|------|------|
| **CPU调度** | `next_cpu`, `next_cpu_batch` | round-robin CPU选择 |
| **Polling** | `poll_considered`, `poll_invoked`, `poll_success` | polling模式统计 |
| **调试** | `debugfs_dir`, `sched_debugfs_dir`, `kobj` | debugfs和sysfs |
| **CPU热插拔** | `cpuhp_online`, `cpuhp_dead` | CPU上下线处理 |
| **其他** | `hctx_list`, `srcu` | 队列管理、sleepable RCU |

---

## 三、派发流程详解

### 3.1 blk_mq_run_hw_queue() - 触发派发

**源码位置**: `block/blk-mq.c`

```c
void blk_mq_run_hw_queue(struct blk_mq_hw_ctx *hctx, bool async)
{
    // 1. 检查队列状态
    if (blk_mq_hctx_stopped(hctx))
        return;

    // 2. 异步派发或同步派发
    if (async) {
        // 异步：调度work
        if (!blk_mq_hctx_has_pending(hctx))
            return;
        kblockd_schedule_work(&hctx->run_work);
    } else {
        // 同步：直接派发
        __blk_mq_run_hw_queue(hctx);
    }
}
```

### 3.2 __blk_mq_sched_dispatch_requests() - 派发请求

```c
void __blk_mq_sched_dispatch_requests(struct blk_mq_hw_ctx *hctx)
{
    struct request_queue *q = hctx->queue;
    struct elevator_queue *e = q->elevator;

    // 1. 优先处理 dispatch 队列
    if (!list_empty_careful(&hctx->dispatch)) {
        if (blk_mq_dispatch_rq_list(hctx, &hctx->dispatch, ...))
            return;  // 派发成功
    }

    // 2. 从调度器获取请求并派发
    if (e) {
        do {
            // 从调度器获取请求
            rq = e->type->ops.dispatch_request(hctx);
            if (!rq)
                break;

            // 分配 Driver Tag
            if (!blk_mq_get_driver_tag(rq)) {
                // 分配失败，放入dispatch队列
                spin_lock(&hctx->lock);
                list_add(&rq->queuelist, &hctx->dispatch);
                spin_unlock(&hctx->lock);
                break;
            }

            // 加入派发列表
            list_add_tail(&rq->queuelist, &rq_list);
        } while (count < max_dispatch);

        // 批量派发
        blk_mq_dispatch_rq_list(hctx, &rq_list, count);
    } else {
        // 无调度器：直接从软件队列取
        blk_mq_flush_busy_ctxs(hctx, &rq_list);
        blk_mq_dispatch_rq_list(hctx, &rq_list, count);
    }
}
```

### 3.3 完整派发流程图

```
┌─────────────────────────────────────────────────────────────────┐
│              blk_mq_run_hw_queue() 派发流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 检查 hctx->state                                            │
│     └─ 已停止? ──▶ 返回                                         │
│                                                                 │
│  2. __blk_mq_sched_dispatch_requests()                         │
│     │                                                           │
│     ├─ A. 优先处理 hctx->dispatch                              │
│     │    └─ blk_mq_dispatch_rq_list(&hctx->dispatch)          │
│     │       (这些请求已有Driver Tag)                           │
│     │                                                           │
│     ├─ B. 从调度器获取新请求                                    │
│     │    loop:                                                 │
│     │      rq = e->ops.dispatch_request(hctx)                 │
│     │       ├─ dd_dispatch_prio_aged_requests() (检查老化)     │
│     │       └─ __dd_dispatch_request() (按优先级选择)          │
│     │      │                                                   │
│     │      ├─ blk_mq_get_driver_tag(rq)                       │
│     │      │   ├─ 成功 ──▶ 加入rq_list                        │
│     │      │   └─ 失败 ──▶ 放入hctx->dispatch, break          │
│     │      │                                                   │
│     │      └─ 继续循环（最多max_dispatch个）                   │
│     │                                                           │
│     └─ C. 批量派发到驱动                                        │
│          blk_mq_dispatch_rq_list(hctx, &rq_list, count)       │
│           └─ hctx->queue->mq_ops->queue_rq()                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、阶段7：Driver Tag 分配

### 4.1 为什么需要 Driver Tag？

```
┌────────────────────────────────────────────────────────────────┐
│                 两种Tag的区别与必要性                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Internal Tag (sched_tags)                                     │
│  ┌──────────────────────────────────────────────────────┐     │
│  │ 作用: 限制内核中的并发请求数                          │     │
│  │ 数量: nr_requests = 62 (可调)                         │     │
│  │ 管理: Block层/调度器                                  │     │
│  │ 等待: 阻塞 (io_schedule)                              │     │
│  │                                                       │     │
│  │ 请求可以在调度器队列中排队等待，不占用硬件资源         │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  Driver Tag (tags)                                             │
│  ┌──────────────────────────────────────────────────────┐     │
│  │ 作用: 匹配硬件队列深度                                 │     │
│  │ 数量: queue_depth = 31 (硬件决定)                     │     │
│  │ 管理: 驱动层                                           │     │
│  │ 等待: 不阻塞，放入dispatch队列                         │     │
│  │                                                       │     │
│  │ 请求获得Driver Tag后才能提交到硬件                     │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  为什么需要两层?                                                │
│  • Internal Tag: 防止内存耗尽（过多request占用内存）           │
│  • Driver Tag: 匹配硬件能力（硬件队列深度有限）                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 Driver Tag 分配代码

**源码位置**: `block/blk-mq.c:1075-1100`

```c
static bool __blk_mq_get_driver_tag(struct request *rq)
{
    struct blk_mq_hw_ctx *hctx = rq->mq_hctx;
    struct sbitmap_queue *bt = hctx->tags->bitmap_tags;
    unsigned int tag_offset = hctx->tags->nr_reserved_tags;
    int tag;

    // 1. 检查是否需要从保留tag分配
    if (blk_mq_tag_is_reserved(hctx->sched_tags, rq->internal_tag)) {
        // Internal Tag是保留的，Driver Tag也用保留的
        bt = hctx->tags->breserved_tags;
        tag_offset = 0;
    } else {
        // 检查队列是否太busy
        if (!hctx_may_queue(hctx, bt))
            return false;
    }

    // 2. 从sbitmap分配tag
    tag = __sbitmap_queue_get(bt);
    if (tag == BLK_MQ_NO_TAG)
        return false;  // 分配失败

    // 3. 设置Driver Tag
    rq->tag = tag + tag_offset;
    return true;
}

// 外层包装 - 支持等待
bool blk_mq_get_driver_tag(struct request *rq)
{
    struct blk_mq_hw_ctx *hctx = rq->mq_hctx;

    if (rq->tag != BLK_MQ_NO_TAG)
        return true;  // 已有tag

    // 尝试分配
    if (__blk_mq_get_driver_tag(rq))
        return true;

    // 分配失败，可能加入等待队列（取决于配置）
    // 注意：这里不会阻塞进程，只是标记hctx需要重试
    return false;
}
```

### 4.3 Driver Tag 分配失败的处理

```
┌─────────────────────────────────────────────────────────────────┐
│          Driver Tag 分配失败的处理流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  调度器返回request                                               │
│        │                                                         │
│        ▼                                                         │
│  blk_mq_get_driver_tag(rq)                                      │
│        │                                                         │
│        ├─ 成功 ──▶ tag=5, 加入派发列表, 继续获取下一个           │
│        │                                                         │
│        └─ 失败 ──▶ 以下处理:                                     │
│              │                                                   │
│              ├─ 1. request 放入 hctx->dispatch                  │
│              │      spin_lock(&hctx->lock);                     │
│              │      list_add(&rq->queuelist, &hctx->dispatch);  │
│              │      spin_unlock(&hctx->lock);                   │
│              │                                                   │
│              ├─ 2. 标记需要重启                                  │
│              │      blk_mq_sched_mark_restart_hctx(hctx);       │
│              │      set_bit(BLK_MQ_S_SCHED_RESTART, &hctx->state);│
│              │                                                   │
│              └─ 3. 停止本次派发                                  │
│                   return; (不再从调度器获取新请求)                │
│                                                                 │
│  下次派发时:                                                     │
│  • 优先处理 hctx->dispatch                                      │
│  • 这些请求已经从调度器选出，只是缺Driver Tag                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、Request 字段在派发阶段的更新

### 5.1 字段状态转换

| 字段 | 分配阶段 | 插入调度器后 | 派发阶段 | 说明 |
|------|---------|-------------|---------|------|
| `internal_tag` | 10 | 10 | 10 | 保持不变 |
| `tag` | -1 | -1 | **5** | ✓ 派发时分配 |
| `state` | IDLE | IDLE | **IN_FLIGHT** | ✓ 状态更新 |
| `rq_flags` | 0 | RQF_QUEUED | **RQF_STARTED** | ✓ 标记已开始 |
| `start_time_ns` | 0 | 0 | **ktime_get_ns()** | ✓ 派发时间戳 |
| `io_start_time_ns` | 0 | 0 | **ktime_get_ns()** | ✓ IO开始时间 |
| `queuelist` | 挂在ctx | 挂在调度器 | 挂在派发列表 | 链表变化 |

### 5.2 关键代码

```c
// block/blk-mq.c - 派发到驱动前的准备
static blk_status_t __blk_mq_try_issue_directly(struct blk_mq_hw_ctx *hctx,
                                                 struct request *rq,
                                                 bool bypass_insert)
{
    struct request_queue *q = rq->q;
    blk_status_t ret;

    // 1. 分配 Driver Tag
    if (!blk_mq_get_driver_tag(rq))
        return BLK_STS_RESOURCE;

    // 2. 更新状态
    rq->rq_flags |= RQF_STARTED;
    rq->state = MQ_RQ_IN_FLIGHT;
    
    // 3. 记录时间戳
    if (blk_stat_enabled(q))
        rq->io_start_time_ns = ktime_get_ns();

    // 4. 调用驱动的 queue_rq
    ret = q->mq_ops->queue_rq(hctx, &bd);

    if (ret == BLK_STS_OK) {
        // 派发成功
        __blk_mq_inc_active_requests(hctx);
    } else {
        // 派发失败，释放Driver Tag
        blk_mq_put_driver_tag(rq);
        rq->rq_flags &= ~RQF_STARTED;
        rq->state = MQ_RQ_IDLE;
    }

    return ret;
}
```

---

## 六、实际设备配置分析

### 6.1 8核UFS设备的hctx配置

```bash
$ ls /sys/block/sda/mq/
0    # 单个硬件队列

$ cat /sys/block/sda/mq/0/cpu_list
0-7  # 8个CPU映射到hctx 0

$ cat /sys/block/sda/mq/0/nr_tags
31   # Driver Tag数量

$ cat /sys/block/sda/queue/nr_requests
62   # Internal Tag数量
```

**结构体配置**：
```c
struct blk_mq_hw_ctx hctx_0 = {
    // Tag管理
    .sched_tags = &tags_internal,  // 指向62个Internal Tag的tags
    .tags = &tags_driver,          // 指向31个Driver Tag的tags
    
    // 软件队列映射
    .nr_ctx = 8,                   // 8个软件队列
    .ctxs = {&ctx0, &ctx1, ..., &ctx7},
    .ctx_map = sbitmap(8位),       // 8位位图
    
    // 队列信息
    .queue_num = 0,                // 第0个硬件队列
    .cpumask = 0xFF,               // CPU 0-7
    
    // 调度器接口
    .sched_data = &deadline_data,  // mq-deadline的数据
    .queue = &request_queue,
    
    // 派发队列
    .dispatch = LIST_HEAD_INIT,    // 空链表
    .state = 0,                    // 正常状态
};
```

### 6.2 并发派发场景

```
场景: 300个并发IO请求的处理

初始状态:
- Internal Tag: 62个全部分配完毕
- Driver Tag: 31个可用
- dispatch队列: 空

第1轮派发:
├─ 调度器选择31个请求
├─ 全部成功分配Driver Tag (31个)
├─ 派发到驱动
└─ Internal Tag: 62个仍在使用
   Driver Tag: 31个仍在使用

此时状态:
- 31个请求在硬件执行中
- 31个请求在调度器队列等待
- 238个请求被阻塞在Internal Tag等待(进程D状态)

硬件完成10个IO:
├─ 释放10个Driver Tag
└─ 触发 blk_mq_sched_restart(hctx)

第2轮派发:
├─ 优先处理 dispatch 队列(空)
├─ 调度器选择10个请求
├─ 全部成功分配Driver Tag (10个)
└─ 派发到驱动

...循环直到所有IO完成
```

---

## 七、硬件队列的 CPU 调度

### 7.1 round-robin CPU 选择

```c
// block/blk-mq.c
static int blk_mq_hctx_next_cpu(struct blk_mq_hw_ctx *hctx)
{
    bool tried = false;
    int next_cpu = hctx->next_cpu;

select_cpu:
    // 选择下一个CPU
    next_cpu = cpumask_next_and(next_cpu, hctx->cpumask, cpu_online_mask);
    if (next_cpu >= nr_cpu_ids)
        next_cpu = cpumask_first_and(hctx->cpumask, cpu_online_mask);

    // CPU批次管理
    if (--hctx->next_cpu_batch <= 0) {
        hctx->next_cpu = next_cpu;
        hctx->next_cpu_batch = BLK_MQ_CPU_WORK_BATCH;  // 默认8
    }

    return next_cpu;
}
```

**批次调度的意义**：
```
避免频繁切换CPU导致的缓存失效

不用批次:
CPU0 ──▶ CPU1 ──▶ CPU2 ──▶ CPU0 ──▶ CPU1 ...  (每次切换)

使用批次(batch=8):
CPU0 (处理8次) ──▶ CPU1 (处理8次) ──▶ CPU2 (处理8次) ...
减少上下文切换，提高缓存命中率
```

---

## 八、总结与下篇预告

### 8.1 本文要点

1. **blk_mq_hw_ctx 是派发的核心**
   - 核心字段：`dispatch`, `sched_tags`, `tags`, `ctx_map`, `ctxs`
   - 连接软件队列、调度器和驱动

2. **两种 Tag 的区别**
   - Internal Tag (62个)：request分配时，阻塞等待
   - Driver Tag (31个)：派发时，非阻塞

3. **dispatch 队列的作用**
   - Driver Tag 不足时暂存请求
   - 下次派发时优先处理

4. **ctx_map 位图机制**
   - 高效标记哪些软件队列有请求
   - 避免遍历空队列

### 8.2 派发流程关键点

```
blk_mq_run_hw_queue()
  └─ __blk_mq_sched_dispatch_requests()
      ├─ 1. 处理 hctx->dispatch (已选出但缺Driver Tag的)
      ├─ 2. 从调度器获取新请求
      │     └─ 分配Driver Tag
      └─ 3. 批量派发到驱动
```

### 8.3 下篇预告

**[27-驱动处理与IO完成](./27-驱动处理与IO完成.md)** 将介绍：

- **阶段8：驱动处理**
  - 驱动视角的 request 字段
  - 驱动如何访问 bio 数据
  - queue_rq 回调实现

- **阶段9：IO 完成与清理**
  - request 完成流程
  - 两种 Tag 的释放
  - bio 完成回调

- **附录**
  - 结构体关系速查表
  - 调试技巧（crash/bpftrace）
  - 内存布局分析

---

## 附录：快速参考

### A. blk_mq_hw_ctx 核心字段速查

| 字段 | 作用 | 查看方式 |
|------|------|---------|
| `sched_tags` | Internal Tag池(62) | `/sys/kernel/debug/block/sda/hctx0/sched_tags` |
| `tags` | Driver Tag池(31) | `/sys/kernel/debug/block/sda/hctx0/tags` |
| `dispatch` | 待派发队列 | `/sys/kernel/debug/block/sda/hctx0/dispatched` |
| `ctx_map` | 软件队列位图 | - |
| `ctxs` | 软件队列数组 | `ls /sys/block/sda/mq/0/cpu*/` |

### B. Tag 相关函数

```c
// Internal Tag
blk_mq_get_tag(data)              // 分配（阻塞）
blk_mq_put_tag(sched_tags, ctx, tag)  // 释放

// Driver Tag
blk_mq_get_driver_tag(rq)         // 分配（非阻塞）
blk_mq_put_driver_tag(rq)         // 释放
```

### C. 调试命令

```bash
# 查看hctx状态
cat /sys/kernel/debug/block/sda/hctx0/state
cat /sys/kernel/debug/block/sda/hctx0/run
cat /sys/kernel/debug/block/sda/hctx0/queued

# 查看dispatch队列
cat /sys/kernel/debug/block/sda/hctx0/dispatch

# 查看tags使用情况
cat /sys/kernel/debug/block/sda/hctx0/tags
# 输出:
# nr_tags=31
# nr_reserved_tags=3
# active_queues=1

cat /sys/kernel/debug/block/sda/hctx0/tags_bitmap
# 显示位图状态（哪些tag正在使用）
```

---

**上一篇**: [25-IO调度器结构体详解](./25-IO调度器结构体详解.md)  
**下一篇**: [27-驱动处理与IO完成](./27-驱动处理与IO完成.md)  
**返回**: [系列文章目录](./README.md)
