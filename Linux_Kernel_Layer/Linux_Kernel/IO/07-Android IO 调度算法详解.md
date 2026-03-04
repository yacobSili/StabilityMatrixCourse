# Android IO 调度算法详解

## 学习目标

- 理解 Android 设备常用的 IO 调度器
- 掌握各调度器的算法原理和特点
- 学会根据设备类型选择合适的调度器
- 掌握调度器参数的调优方法
- 理解调度器对 IO 性能的影响

## 背景介绍

IO 调度器（I/O Scheduler）是 Linux 内核块设备层的重要组件，负责对 IO 请求进行排序、合并和调度。在 Android 设备上，选择合适的 IO 调度器并正确配置参数，对系统性能、应用响应速度和电池续航都有重要影响。本文深入分析 Android 设备常用的 IO 调度器算法，帮助理解调度器的工作原理和优化方法。

## IO 调度器的作用

### 为什么需要 IO 调度器

**问题场景**：
- 多个进程同时发起 IO 请求
- 请求的扇区位置分散，直接处理效率低
- 需要保证关键请求的延迟上限
- 需要平衡读写的优先级

**调度器的功能**：
1. **请求排序**：按扇区位置排序，减少寻道时间（机械硬盘）
2. **请求合并**：合并相邻扇区的请求，减少 IO 次数
3. **优先级管理**：根据 IO 优先级调度请求
4. **延迟保证**：保证关键请求的延迟上限

### 调度器在 IO 路径中的位置

```mermaid
graph LR
    A[bio 提交] --> B[IO 调度器]
    B --> C[请求排序/合并]
    C --> D[优先级处理]
    D --> E[dispatch 到硬件队列]
    E --> F[设备驱动]
```

## Android 设备常用的 IO 调度器

### 1. none（无调度器）

**特点**：
- 不使用调度器，请求直接进入硬件队列
- 最简单的调度方式，开销最小
- 适合现代 SSD/NVMe/UFS 设备

**适用场景**：
- NVMe SSD
- UFS 存储设备
- 高性能存储设备
- 随机访问性能好的设备

**为什么 SSD 不需要调度器**：
- SSD 没有机械寻道，随机访问性能好
- 调度器的排序和合并对 SSD 意义不大
- 直接处理可以减少延迟

**Android 设备配置**：
```bash
# 查看当前调度器
cat /sys/block/sda/queue/scheduler

# 设置为 none
echo none > /sys/block/sda/queue/scheduler
```

### 2. mq-deadline（多队列截止时间调度器）

**位置**：`block/mq-deadline.c`

**特点**：
- 为每个 IO 请求设置截止时间（deadline）
- 保证请求在截止时间前被处理
- 支持 IO 优先级（RT、BE、IDLE）
- 适合需要延迟保证的场景

**算法原理**：

#### 数据结构

```c
struct deadline_data {
    struct dd_per_prio per_prio[DD_PRIO_COUNT];  // 按优先级分组
    enum dd_data_dir last_dir;                    // 上次处理的方向
    unsigned int batching;                        // 批处理计数
    unsigned int starved;                         // 写请求饥饿计数
    int fifo_expire[DD_DIR_COUNT];                // 读/写截止时间
    int fifo_batch;                               // 批处理大小
    int writes_starved;                           // 写请求饥饿阈值
};

struct dd_per_prio {
    struct rb_root sort_list[DD_DIR_COUNT];       // 按扇区排序的红黑树
    struct list_head fifo_list[DD_DIR_COUNT];    // 按截止时间排序的 FIFO 队列
    struct request *next_rq[DD_DIR_COUNT];       // 下一个请求
};
```

#### 关键参数

**默认值**（`block/mq-deadline.c`）：
```c
static const int read_expire = HZ / 2;      // 读请求截止时间：500ms
static const int write_expire = 5 * HZ;     // 写请求截止时间：5s
static const int prio_aging_expire = 10 * HZ; // 优先级老化时间：10s
static const int writes_starved = 2;         // 写请求饥饿阈值：2次
static const int fifo_batch = 16;            // 批处理大小：16个请求
```

#### 调度算法流程

1. **请求插入**：
   - 根据 IO 优先级分类（RT、BE、IDLE）
   - 插入到对应优先级的红黑树（按扇区排序）
   - 插入到对应优先级的 FIFO 队列（按截止时间排序）

2. **请求调度**：
   - 优先处理高优先级请求（RT > BE > IDLE）
   - 检查截止时间：如果请求超过截止时间，立即处理
   - 批处理：连续处理同一方向的请求（最多 `fifo_batch` 个）
   - 防止饥饿：读请求处理 `writes_starved` 次后，必须处理写请求

3. **优先级老化**：
   - 如果低优先级请求等待超过 `prio_aging_expire`，提升优先级

**关键代码**（`block/mq-deadline.c`）：
```c
static struct request *dd_dispatch_request(struct blk_mq_hw_ctx *hctx)
{
    struct deadline_data *dd = hctx->queue->elevator->elevator_data;
    const unsigned long now = jiffies;
    struct request *rq;
    enum dd_prio prio;

    // 1. 检查是否有超过优先级老化时间的请求
    rq = dd_dispatch_prio_aged_requests(dd, now);
    if (rq)
        return rq;

    // 2. 按优先级顺序调度（RT > BE > IDLE）
    for (prio = 0; prio <= DD_PRIO_MAX; prio++) {
        rq = __dd_dispatch_request(dd, &dd->per_prio[prio], now);
        if (rq || dd_queued(dd, prio))
            break;
    }

    return rq;
}
```

**参数调优**：
```bash
# 查看当前参数
cat /sys/block/sda/queue/iosched/read_expire
cat /sys/block/sda/queue/iosched/write_expire
cat /sys/block/sda/queue/iosched/fifo_batch
cat /sys/block/sda/queue/iosched/writes_starved

# 调整参数（单位：毫秒）
echo 250 > /sys/block/sda/queue/iosched/read_expire    # 读请求截止时间
echo 5000 > /sys/block/sda/queue/iosched/write_expire  # 写请求截止时间
echo 8 > /sys/block/sda/queue/iosched/fifo_batch       # 批处理大小
echo 3 > /sys/block/sda/queue/iosched/writes_starved   # 写请求饥饿阈值
```

**适用场景**：
- 需要延迟保证的应用
- 实时性要求高的场景
- 混合读写负载

### 3. kyber（现代 SSD 优化调度器）

**位置**：`block/kyber-iosched.c`

**特点**：
- 基于令牌桶算法
- 自动调整队列深度
- 针对现代 SSD 优化
- 低延迟、高吞吐量

**算法原理**：

#### 令牌桶机制

Kyber 为不同的 IO 类型维护独立的队列和令牌桶：
- **同步读队列**：用户空间的同步读请求
- **同步写队列**：用户空间的同步写请求
- **其他队列**：异步请求等

每个队列有独立的令牌桶，控制队列深度。

#### 自动深度调整

Kyber 根据目标延迟自动调整队列深度：
- 如果延迟低于目标，增加队列深度（提高吞吐量）
- 如果延迟高于目标，减少队列深度（降低延迟）

**关键参数**：
```bash
# 查看当前参数
cat /sys/block/sda/queue/iosched/read_lat_nsec    # 读请求目标延迟（纳秒）
cat /sys/block/sda/queue/iosched/write_lat_nsec   # 写请求目标延迟（纳秒）

# 调整参数
echo 2000000 > /sys/block/sda/queue/iosched/read_lat_nsec   # 2ms
echo 10000000 > /sys/block/sda/queue/iosched/write_lat_nsec  # 10ms
```

**适用场景**：
- 现代 SSD/UFS 设备
- 需要平衡延迟和吞吐量的场景
- Android 设备的默认选择（部分设备）

### 4. bfq（Budget Fair Queueing）

**位置**：`block/bfq-iosched.c`

**特点**：
- Budget Fair Queueing 算法
- 公平调度，保证每个进程的 IO 带宽
- 适合交互式应用
- Android 设备较少使用

**算法原理**：
- 为每个进程分配预算（budget）
- 按预算比例分配 IO 带宽
- 保证低延迟和公平性

**适用场景**：
- 需要公平调度的场景
- 交互式应用
- 多用户系统

## 调度器选择策略

### 根据存储设备类型选择

| 设备类型 | 推荐调度器 | 原因 |
|---------|-----------|------|
| NVMe SSD | none | 随机访问性能好，不需要排序 |
| UFS 3.0/3.1 | none 或 kyber | 高性能，kyber 可优化延迟 |
| eMMC 5.1 | mq-deadline | 需要延迟保证 |
| 机械硬盘 | mq-deadline | 需要排序减少寻道 |

### Android 设备默认配置

**查看当前调度器**：
```bash
# 查看所有可用调度器
cat /sys/block/sda/queue/scheduler

# 输出示例：
# [mq-deadline] kyber none
# 方括号表示当前使用的调度器
```

**常见配置**：
- **高端设备（UFS 3.0+）**：通常使用 `none` 或 `kyber`
- **中端设备（UFS 2.1/eMMC）**：通常使用 `mq-deadline`
- **低端设备（eMMC 5.0）**：通常使用 `mq-deadline`

## 调度器参数调优

### mq-deadline 参数调优

**场景 1：优化读延迟**
```bash
# 减少读请求截止时间
echo 200 > /sys/block/sda/queue/iosched/read_expire  # 200ms

# 减少批处理大小，提高响应性
echo 8 > /sys/block/sda/queue/iosched/fifo_batch
```

**场景 2：优化写吞吐量**
```bash
# 增加写请求截止时间，允许更多合并
echo 10000 > /sys/block/sda/queue/iosched/write_expire  # 10s

# 增加批处理大小
echo 32 > /sys/block/sda/queue/iosched/fifo_batch
```

**场景 3：平衡读写**
```bash
# 调整写请求饥饿阈值
echo 4 > /sys/block/sda/queue/iosched/writes_starved
```

### kyber 参数调优

**场景：优化延迟**
```bash
# 设置更严格的目标延迟
echo 1000000 > /sys/block/sda/queue/iosched/read_lat_nsec   # 1ms
echo 5000000 > /sys/block/sda/queue/iosched/write_lat_nsec  # 5ms
```

**场景：优化吞吐量**
```bash
# 设置更宽松的目标延迟
echo 5000000 > /sys/block/sda/queue/iosched/read_lat_nsec   # 5ms
echo 20000000 > /sys/block/sda/queue/iosched/write_lat_nsec  # 20ms
```

## 实战案例：Android 设备 IO 调度器配置优化

### 案例 1：应用启动慢

**问题**：应用启动时 IO 延迟高，启动时间长。

**分析**：
- 当前使用 `mq-deadline`，读请求截止时间为 500ms
- 启动时大量小文件读取，需要更低的延迟

**解决方案**：
```bash
# 1. 切换到 kyber 调度器（自动调整延迟）
echo kyber > /sys/block/sda/queue/scheduler

# 2. 或者优化 mq-deadline 参数
echo mq-deadline > /sys/block/sda/queue/scheduler
echo 200 > /sys/block/sda/queue/iosched/read_expire  # 降低读延迟
echo 4 > /sys/block/sda/queue/iosched/fifo_batch      # 减少批处理
```

### 案例 2：后台写回影响前台 IO

**问题**：后台写回占用 IO，导致前台应用响应慢。

**分析**：
- 写请求截止时间过长（5s），允许大量写请求积累
- 读请求被写请求阻塞

**解决方案**：
```bash
# 1. 减少写请求截止时间
echo 2000 > /sys/block/sda/queue/iosched/write_expire  # 2s

# 2. 增加写请求饥饿阈值，优先处理读请求
echo 1 > /sys/block/sda/queue/iosched/writes_starved
```

### 案例 3：Kernel 升级后 IO 性能下降

**问题**：Kernel 从 r2 升级到 r3 后，IO 性能下降。

**分析**：
- 调度器默认参数可能改变
- 调度器选择可能改变

**解决方案**：
```bash
# 1. 检查当前调度器和参数
cat /sys/block/sda/queue/scheduler
cat /sys/block/sda/queue/iosched/*

# 2. 对比 r2 和 r3 的默认配置
# 3. 根据设备类型调整调度器和参数
# 4. 测试性能，找到最优配置
```

## 调度器性能监控

### 查看调度器统计

**mq-deadline**：
```bash
# 查看各优先级的统计
cat /sys/block/sda/queue/iosched/prio_aging_expire
```

**kyber**：
```bash
# 查看延迟统计（需要内核支持）
cat /sys/block/sda/queue/iosched/read_lat_nsec
cat /sys/block/sda/queue/iosched/write_lat_nsec
```

### 使用 iostat 监控

```bash
# 监控 IO 延迟和吞吐量
iostat -x 1

# 关键指标：
# await: 平均等待时间
# svctm: 平均服务时间
# %util: 设备利用率
```

## 总结

### 核心要点

1. **调度器选择**：
   - SSD/UFS：`none` 或 `kyber`
   - eMMC/机械硬盘：`mq-deadline`

2. **算法特点**：
   - `mq-deadline`：截止时间保证，支持优先级
   - `kyber`：自动调整，平衡延迟和吞吐量
   - `bfq`：公平调度，保证带宽分配

3. **参数调优**：
   - 根据应用场景调整参数
   - 平衡延迟和吞吐量
   - 监控性能变化

### 关键概念

- **IO 调度器**：对 IO 请求进行排序、合并和调度的机制
- **截止时间**：请求必须被处理的最晚时间
- **批处理**：连续处理同一方向的多个请求
- **优先级**：不同优先级请求的处理顺序

### 下一步学习

- [08-Android 整机 IO 压力分析与占用定位](08-Android 整机 IO 压力分析与占用定位.md) - 学习如何分析 IO 性能问题
- [09-Android 设备常见 IO 问题与排查](09-Android 设备常见 IO 问题与排查.md) - 掌握 IO 问题的排查方法
- [10-Android IO 性能优化实战](10-Android IO 性能优化实战.md) - 学习 IO 性能优化实践

## 参考资料

- Linux 内核源码：`block/mq-deadline.c` - deadline 调度器实现
- Linux 内核源码：`block/kyber-iosched.c` - kyber 调度器实现
- Linux 内核源码：`block/bfq-iosched.c` - bfq 调度器实现
- Linux 内核文档：`Documentation/block/deadline-iosched.rst` - deadline 调度器文档
- Linux 内核文档：`Documentation/block/bfq-iosched.rst` - bfq 调度器文档

## 更新记录

- 2026-01-27：初始创建，包含 Android IO 调度算法详解
