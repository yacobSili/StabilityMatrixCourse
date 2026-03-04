# kswapd 深度解析

## 📋 文档概述

本文档系统深入地解析 Linux 内核中的 kswapd（Kernel Swap Daemon）机制，包括其工作原理、源码分析、触发机制、调优方法以及实际场景问题分析。

## 🎯 学习目标

- 深入理解 kswapd 的工作原理和触发机制
- 掌握 kswapd 的源码实现细节
- 能够分析 kswapd 相关的性能问题和稳定性问题
- 能够调优 kswapd 参数以优化系统性能
- 能够诊断和解决 kswapd 相关的实际问题

---

## 第一章：kswapd 基础

### 1.1 什么是 kswapd？

**kswapd**（Kernel Swap Daemon）是 Linux 内核中的一个内核线程（kernel thread），负责在后台进行内存回收（memory reclaim），以维持系统的内存平衡。

**核心作用**：
- 在内存压力较低时，主动回收内存页面
- 将不常用的页面交换到 swap 分区
- 释放缓存和缓冲区占用的内存
- 维持系统的可用内存水平

**为什么需要 kswapd？**

1. **预防性内存管理**
   - 在内存压力出现之前就开始回收
   - 避免系统突然陷入内存紧张状态

2. **后台异步回收**
   - 不阻塞用户进程
   - 在后台持续工作

3. **平滑内存压力**
   - 避免内存突然耗尽
   - 提供缓冲时间

### 1.2 kswapd 在内核中的位置

**内存管理子系统架构**：

```
内存管理子系统 (mm/)
├── 页面分配器 (page allocator)
├── SLAB 分配器 (slab allocator)
├── 内存回收机制
│   ├── kswapd (后台回收)
│   ├── direct reclaim (直接回收)
│   └── OOM Killer (最后手段)
└── Swap 机制
```

**kswapd 与其他机制的关系**：

- **kswapd**：后台异步回收，预防性
- **Direct Reclaim**：同步回收，当分配失败时触发
- **OOM Killer**：内存耗尽时的最后手段

### 1.3 kswapd 的基本工作流程

**基本流程**：

```
系统启动
    ↓
创建 kswapd 线程（每个 NUMA 节点一个）
    ↓
kswapd 进入睡眠状态
    ↓
内存压力触发唤醒
    ↓
扫描页面，选择可回收页面
    ↓
回收页面（写入 swap 或释放）
    ↓
检查是否达到目标
    ↓
继续或进入睡眠
```

---

## 第二章：kswapd 工作原理

### 2.1 kswapd 线程的创建

**源码位置**：`mm/vmscan.c`

**创建时机**：
- 系统启动时
- 每个 NUMA 节点创建一个 kswapd 线程
- 线程名称：`kswapd0`, `kswapd1`, ...（对应 NUMA 节点）

**关键代码**（简化版）：

```c
// 启动 kswapd
static int __init kswapd_init(void)
{
    int nid;
    
    for_each_node_state(nid, N_MEMORY)
        kswapd_run(nid);
    
    return 0;
}

// 为特定节点创建 kswapd 线程
int kswapd_run(int nid)
{
    pg_data_t *pgdat = NODE_DATA(nid);
    struct task_struct *kswapd;
    
    kswapd = kthread_run(kswapd, pgdat, "kswapd%d", nid);
    // ...
}
```

### 2.2 kswapd 的唤醒机制

**唤醒条件**：

kswapd 在以下情况下被唤醒：

1. **内存水位低于低水位（low watermark）**
   - 每个 zone 有三个水位：high、low、min
   - 当可用内存低于 low 时，唤醒 kswapd

2. **直接回收失败**
   - 当进程分配内存失败，触发 direct reclaim
   - 如果 direct reclaim 压力大，会唤醒 kswapd 协助

3. **内存碎片化**
   - 当无法分配连续页面时
   - kswapd 可能被唤醒进行内存整理

**水位线（Watermark）**：

```
High Watermark (高水位)
    ↓
    | 正常内存使用区域
    |
Low Watermark (低水位)  ← kswapd 开始工作
    ↓
    | kswapd 回收区域
    |
Min Watermark (最低水位) ← direct reclaim 开始
    ↓
    | 紧急回收区域
```

**查看水位线**：

```bash
# 查看所有 zone 的水位线
cat /proc/zoneinfo | grep -E "Node|zone|pages free|low|high|min"

# 查看特定节点的水位
cat /proc/zoneinfo | grep -A 20 "Node 0"
```

### 2.3 kswapd 的回收策略

**LRU（Least Recently Used）列表**：

Linux 内核使用 LRU 算法管理页面，将页面分为多个列表：

1. **Active 列表**
   - `active_anon`：活跃的匿名页面（进程堆栈）
   - `active_file`：活跃的文件页面（文件缓存）

2. **Inactive 列表**
   - `inactive_anon`：非活跃的匿名页面
   - `inactive_file`：非活跃的文件页面

**回收优先级**：

```
1. 文件缓存 (inactive_file) ← 最容易回收，无副作用
2. 匿名页面 (inactive_anon) ← 需要交换到 swap
3. 活跃页面 (active) ← 最后选择
```

**回收过程**：

```
kswapd 被唤醒
    ↓
扫描 LRU 列表
    ↓
选择可回收页面
    ↓
文件页面：直接释放（可重新从磁盘读取）
匿名页面：写入 swap 分区
    ↓
更新 LRU 列表
    ↓
检查是否达到回收目标
```

### 2.4 kswapd 的扫描机制

**扫描范围**：

- kswapd 按 zone 进行扫描
- 每个 zone 有独立的 LRU 列表
- 扫描顺序：ZONE_DMA → ZONE_NORMAL → ZONE_HIGHMEM

**扫描参数**：

- **扫描批次**：每次扫描的页面数量
- **扫描速度**：控制扫描的激进程度
- **扫描目标**：需要回收的页面数量

**源码关键函数**：

```c
// kswapd 主函数
static int kswapd(void *p)
{
    pg_data_t *pgdat = (pg_data_t *)p;
    
    for (;;) {
        // 进入睡眠，等待唤醒
        wait_event_interruptible(pgdat->kswapd_wait,
            kswapd_needs_to_run(pgdat));
        
        // 执行回收
        balance_pgdat(pgdat, order, classzone_idx);
    }
}

// 平衡内存
static void balance_pgdat(pg_data_t *pgdat, int order, int classzone_idx)
{
    // 扫描和回收逻辑
    // ...
}
```

---

## 第三章：kswapd 触发机制深入分析

### 3.1 内存水位与 kswapd 触发

**Zone 水位计算**：

每个 zone 的水位线基于以下因素计算：

1. **min_free_kbytes**：系统保留的最小内存（KB）
2. **Zone 大小**：zone 的总页面数
3. **低内存阈值**：系统配置的低内存阈值

**水位计算公式**（简化）：

```
low = min_free_kbytes * zone_size / total_memory
high = low * 1.5 (大约)
min = low / 2 (大约)
```

**查看当前水位**：

```bash
# 查看 min_free_kbytes
cat /proc/sys/vm/min_free_kbytes

# 查看 zone 详细信息
cat /proc/zoneinfo
```

**输出示例**：
```
Node 0, zone   Normal
  pages free     12345
        min      1024      ← 最低水位
        low      2048      ← 低水位（kswapd 触发）
        high     3072      ← 高水位
```

### 3.2 Direct Reclaim 与 kswapd 的协作

**Direct Reclaim（直接回收）**：

当进程分配内存时，如果可用内存不足：

1. **首先尝试 direct reclaim**
   - 同步回收，阻塞分配进程
   - 快速回收少量内存

2. **如果 direct reclaim 压力大**
   - 唤醒 kswapd 协助
   - kswapd 在后台继续回收

**触发条件**：

```c
// 内存分配路径（简化）
__alloc_pages()
    ↓
__alloc_pages_slowpath()
    ↓
__alloc_pages_direct_reclaim()
    ↓
如果压力大 → wake_up_all_kswapds()
```

**压力判断**：

- 回收的页面数量
- 回收耗时
- 系统负载

### 3.3 kswapd 的睡眠与唤醒

**睡眠条件**：

kswapd 在以下情况进入睡眠：

1. 所有 zone 都达到高水位
2. 回收目标已完成
3. 没有可回收的页面

**唤醒机制**：

```c
// 检查是否需要运行 kswapd
static bool kswapd_needs_to_run(pg_data_t *pgdat)
{
    // 检查是否有 zone 低于低水位
    for (zone = pgdat->node_zones; zone; zone = zone->next) {
        if (zone_watermark_ok(zone, order, ...))
            return true;
    }
    return false;
}

// 唤醒 kswapd
void wakeup_kswapd(struct zone *zone, int order, enum zone_type classzone_idx)
{
    if (!waitqueue_active(&zone->zone_pgdat->kswapd_wait))
        return;
    
    if (kswapd_needs_to_run(zone->zone_pgdat))
        wake_up_interruptible(&zone->zone_pgdat->kswapd_wait);
}
```

---

## 第四章：kswapd 源码分析

### 4.1 kswapd 主函数分析

**源码位置**：`mm/vmscan.c::kswapd()`

**完整流程**：

```c
static int kswapd(void *p)
{
    unsigned int alloc_order, reclaim_order;
    unsigned int classzone_idx = MAX_NR_ZONES - 1;
    pg_data_t *pgdat = (pg_data_t *)p;
    struct task_struct *tsk = current;
    
    // 设置线程属性
    tsk->flags |= PF_MEMALLOC | PF_KSWAPD;
    set_freezable();
    
    // 主循环
    for (;;) {
        bool ret;
        
        // 进入睡眠，等待唤醒条件
        ret = wait_event_freezable(
            pgdat->kswapd_wait,
            kswapd_needs_to_run(pgdat) || kthread_should_stop());
        
        if (kthread_should_stop())
            break;
        
        // 计算回收参数
        alloc_order = reclaim_order = pgdat->kswapd_order;
        classzone_idx = pgdat->classzone_idx;
        
        // 重置
        pgdat->kswapd_order = 0;
        pgdat->classzone_idx = MAX_NR_ZONES - 1;
        
        // 执行内存平衡（回收）
        balance_pgdat(pgdat, alloc_order, classzone_idx);
    }
    
    return 0;
}
```

**关键点分析**：

1. **PF_MEMALLOC 标志**
   - 允许 kswapd 在内存紧张时分配内存
   - 避免 kswapd 自身触发回收

2. **等待机制**
   - `wait_event_freezable`：可被冻结的等待
   - 支持系统休眠

3. **回收参数**
   - `alloc_order`：分配阶数
   - `classzone_idx`：zone 索引

### 4.2 balance_pgdat 函数分析

**函数作用**：平衡节点的内存，执行实际的回收工作

**关键逻辑**：

```c
static void balance_pgdat(pg_data_t *pgdat, int order, int classzone_idx)
{
    int end_zone = 0;
    unsigned long nr_soft_reclaimed;
    unsigned long nr_soft_scanned;
    struct scan_control sc = {
        .gfp_mask = GFP_KERNEL,
        .order = order,
        .priority = DEF_PRIORITY,
        .may_writepage = !laptop_mode,
        .may_unmap = 1,
        .may_swap = 1,
    };
    
    // 扫描所有 zone
    for (i = 0; i <= end_zone; i++) {
        struct zone *zone = pgdat->node_zones + i;
        
        // 检查是否需要回收
        if (!populated_zone(zone))
            continue;
        
        // 执行回收
        sc.nr_reclaimed = 0;
        shrink_zone(zone, &sc);
    }
}
```

### 4.3 LRU 扫描机制

**shrink_zone 函数**：

```c
static void shrink_zone(struct zone *zone, struct scan_control *sc)
{
    unsigned long nr[NR_LRU_LISTS];
    unsigned long targets[NR_LRU_LISTS];
    unsigned long nr_to_scan;
    enum lru_list lru;
    
    // 计算需要扫描的页面数
    for_each_evictable_lru(lru) {
        nr[lru] = zone_page_state(zone, NR_LRU_BASE + lru);
        targets[lru] = nr[lru] >> sc->priority;
    }
    
    // 扫描 LRU 列表
    while (nr[LRU_INACTIVE_ANON] || nr[LRU_ACTIVE_FILE] ||
           nr[LRU_INACTIVE_FILE]) {
        unsigned long nr_anon, nr_file, percentage;
        
        // 计算扫描比例
        // ...
        
        // 扫描匿名页面
        if (nr_anon)
            shrink_list(LRU_INACTIVE_ANON, nr_anon, &sc);
        
        // 扫描文件页面
        if (nr_file)
            shrink_list(LRU_INACTIVE_FILE, nr_file, &sc);
    }
}
```

---

## 第五章：kswapd 监控与诊断

### 5.1 监控 kswapd 活动

**查看 kswapd 线程**：

```bash
# 查看 kswapd 线程
ps aux | grep kswapd

# 查看 kswapd 的 CPU 使用
top -H -p $(pgrep kswapd)

# 查看 kswapd 的详细统计
cat /proc/vmstat | grep -E "pgscan|pgsteal|kswapd"
```

**关键指标**：

- **pgscan_kswapd**：kswapd 扫描的页面数
- **pgsteal_kswapd**：kswapd 回收的页面数
- **kswapd_steal**：kswapd 从 direct reclaim 偷取的页面
- **pgscan_direct**：direct reclaim 扫描的页面数

**查看内存回收统计**：

```bash
# 查看详细的回收统计
cat /proc/vmstat | grep -E "pgscan|pgsteal|pgmajfault"

# 输出示例：
# pgscan_kswapd_normal 123456
# pgsteal_kswapd_normal 12345
# pgscan_direct_normal 67890
# pgsteal_direct_normal 6789
```

### 5.2 使用 ftrace 跟踪 kswapd

**启用跟踪**：

```bash
# 启用函数跟踪
echo function > /sys/kernel/debug/tracing/current_tracer

# 设置过滤（只跟踪 kswapd 相关）
echo kswapd* > /sys/kernel/debug/tracing/set_ftrace_filter
echo balance_pgdat >> /sys/kernel/debug/tracing/set_ftrace_filter
echo shrink_zone >> /sys/kernel/debug/tracing/set_ftrace_filter

# 开始跟踪
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 触发 kswapd（例如：分配大量内存）
# ...

# 停止跟踪
echo 0 > /sys/kernel/debug/tracing/tracing_on

# 查看跟踪结果
cat /sys/kernel/debug/tracing/trace
```

### 5.3 使用 perf 分析 kswapd

**性能分析**：

```bash
# 记录 kswapd 的性能数据
perf record -g -p $(pgrep kswapd) sleep 10

# 查看报告
perf report

# 查看调用图
perf report --call-graph
```

**火焰图分析**：

```bash
# 生成火焰图数据
perf record -g -p $(pgrep kswapd)
perf script | stackcollapse-perf.pl | flamegraph.pl > kswapd.svg
```

### 5.4 诊断工具

**vmstat**：

```bash
# 持续监控
vmstat 1

# 关键指标：
# si: swap in (从 swap 读入)
# so: swap out (写入 swap)
# 如果 si/so 持续大于 0，说明 swap 活跃
```

**sar**：

```bash
# 安装 sysstat
# 查看内存和 swap 统计
sar -r 1
sar -S 1  # swap 统计
```

**/proc/meminfo**：

```bash
# 查看详细内存信息
cat /proc/meminfo

# 关键指标：
# MemFree: 空闲内存
# MemAvailable: 可用内存（估算）
# SwapTotal: swap 总大小
# SwapFree: swap 空闲大小
# Active(anon): 活跃匿名页面
# Inactive(anon): 非活跃匿名页面
# Active(file): 活跃文件页面
# Inactive(file): 非活跃文件页面
```

---

## 第六章：kswapd 调优

### 6.1 关键参数

**min_free_kbytes**：

```bash
# 查看当前值
cat /proc/sys/vm/min_free_kbytes

# 设置值（需要 root）
echo 65536 > /proc/sys/vm/min_free_kbytes

# 永久设置
echo "vm.min_free_kbytes = 65536" >> /etc/sysctl.conf
sysctl -p
```

**作用**：
- 控制最低水位线
- 影响 kswapd 的触发时机
- 值越大，kswapd 越早触发

**调优建议**：
- 默认值：`sqrt(lowmem_kbytes * 16)`
- 建议范围：系统内存的 1-3%
- 太小：kswapd 触发晚，可能频繁 direct reclaim
- 太大：浪费内存，kswapd 过早触发

**swappiness**：

```bash
# 查看当前值（0-100）
cat /proc/sys/vm/swappiness

# 设置值
echo 60 > /proc/sys/vm/swappiness
```

**作用**：
- 控制匿名页面和文件页面的回收比例
- 0：优先回收文件页面
- 100：优先回收匿名页面（交换到 swap）
- 默认：60

**调优建议**：
- **服务器**：10-30（减少 swap 使用）
- **桌面系统**：60（平衡）
- **数据库服务器**：1-10（避免 swap）

### 6.2 其他相关参数

**dirty_ratio / dirty_background_ratio**：

```bash
# 控制脏页写回
cat /proc/sys/vm/dirty_ratio          # 默认 20%
cat /proc/sys/vm/dirty_background_ratio  # 默认 10%
```

**vfs_cache_pressure**：

```bash
# 控制文件缓存的回收倾向
cat /proc/sys/vm/vfs_cache_pressure   # 默认 100
# 值越大，越倾向于回收文件缓存
```

**zone_reclaim_mode**：

```bash
# NUMA 系统的 zone 回收模式
cat /proc/sys/vm/zone_reclaim_mode
# 0: 禁用
# 1: 启用本地回收
```

### 6.3 调优策略

**场景 1：减少 swap 使用**

```bash
# 降低 swappiness
echo 10 > /proc/sys/vm/swappiness

# 增加 min_free_kbytes（提前触发回收）
echo 131072 > /proc/sys/vm/min_free_kbytes
```

**场景 2：提高响应速度**

```bash
# 增加 min_free_kbytes（提前回收）
echo 262144 > /proc/sys/vm/min_free_kbytes

# 降低 dirty_ratio（减少写回延迟）
echo 10 > /proc/sys/vm/dirty_ratio
```

**场景 3：数据库服务器**

```bash
# 最小化 swap
echo 1 > /proc/sys/vm/swappiness

# 增加保留内存
echo 262144 > /proc/sys/vm/min_free_kbytes

# 禁用 zone reclaim（如果是 NUMA）
echo 0 > /proc/sys/vm/zone_reclaim_mode
```

---

## 第七章：kswapd 问题诊断

### 7.1 kswapd CPU 占用过高

**症状**：
- kswapd 进程 CPU 使用率持续很高
- 系统响应变慢
- 内存持续紧张

**可能原因**：

1. **内存不足**
   - 系统内存太小
   - 应用程序占用过多内存

2. **swap 配置不当**
   - swap 空间不足
   - swap 设备性能差（机械硬盘）

3. **内存泄漏**
   - 应用程序内存泄漏
   - 内核模块内存泄漏

**诊断步骤**：

```bash
# 1. 检查内存使用
free -h
cat /proc/meminfo

# 2. 检查 swap 使用
swapon -s
cat /proc/swaps

# 3. 检查 kswapd 活动
cat /proc/vmstat | grep kswapd

# 4. 检查内存泄漏
# 使用 valgrind 或 AddressSanitizer 检查应用程序
# 检查内核日志中的 OOM 相关信息
dmesg | grep -i oom
```

**解决方案**：

1. **增加内存或 swap**
2. **优化应用程序内存使用**
3. **调整 kswapd 参数**
4. **排查内存泄漏**

### 7.2 kswapd 无法回收内存

**症状**：
- kswapd 持续运行但内存不释放
- 系统内存持续紧张
- 可能触发 OOM

**可能原因**：

1. **所有页面都被锁定**
   - mlock() 锁定的页面
   - 内核页面无法回收

2. **swap 空间已满**
   - 无法交换匿名页面

3. **内存碎片化**
   - 无法分配连续页面
   - kswapd 无法有效回收

**诊断步骤**：

```bash
# 1. 检查锁定的内存
cat /proc/meminfo | grep -i locked

# 2. 检查 swap 使用
free -h
swapon -s

# 3. 检查内存碎片
cat /proc/buddyinfo

# 4. 检查可回收页面
cat /proc/meminfo | grep -E "Active|Inactive"
```

### 7.3 kswapd 与 direct reclaim 频繁切换

**症状**：
- kswapd 和 direct reclaim 交替工作
- 系统性能下降
- 内存压力持续

**可能原因**：

1. **min_free_kbytes 设置不当**
   - 太小：kswapd 触发晚
   - 太大：频繁触发但回收不足

2. **内存分配模式**
   - 大量小内存分配
   - 分配速度超过回收速度

**解决方案**：

```bash
# 调整 min_free_kbytes
# 根据系统负载和内存使用模式调整
echo 131072 > /proc/sys/vm/min_free_kbytes

# 监控效果
watch -n 1 'cat /proc/vmstat | grep -E "pgscan|pgsteal"'
```

---

## 第八章：实际场景问题分析

### 8.1 场景：kswapd 导致系统卡顿

**问题描述**：
- 系统在内存压力大时出现卡顿
- kswapd CPU 占用高
- 用户进程响应慢

**分析思路**：

1. **确认 kswapd 活动**
   ```bash
   # 查看 kswapd CPU 使用
   top -H -p $(pgrep kswapd)
   
   # 查看回收统计
   cat /proc/vmstat | grep kswapd
   ```

2. **分析内存使用模式**
   ```bash
   # 查看内存分布
   cat /proc/meminfo
   
   # 查看进程内存使用
   ps aux --sort=-%mem | head -20
   ```

3. **检查 swap 性能**
   ```bash
   # 查看 swap I/O
   iostat -x 1 | grep -i swap
   
   # 如果 swap 在机械硬盘上，I/O 会成为瓶颈
   ```

**解决方案**：

1. **优化 swap 设备**
   - 使用 SSD 作为 swap
   - 增加 swap 空间

2. **调整回收策略**
   - 降低 swappiness（减少 swap 使用）
   - 增加 min_free_kbytes（提前回收）

3. **优化应用程序**
   - 减少内存使用
   - 优化内存分配模式

### 8.2 场景：kswapd 无法满足内存需求

**问题描述**：
- kswapd 持续运行但内存仍不足
- 频繁触发 direct reclaim
- 最终触发 OOM Killer

**分析步骤**：

```bash
# 1. 检查内存压力
cat /proc/pressure/memory

# 2. 检查回收效率
cat /proc/vmstat | grep -E "pgscan|pgsteal"
# 如果 pgsteal/pgscan 比例低，说明回收效率低

# 3. 检查可回收页面
cat /proc/meminfo | grep -E "Active|Inactive"
# 如果大部分是 Active，说明页面都在使用中
```

**可能原因**：

1. **工作集（working set）太大**
   - 应用程序实际需要的内存超过物理内存
   - kswapd 无法回收正在使用的页面

2. **内存泄漏**
   - 内存持续增长
   - 回收速度跟不上泄漏速度

3. **swap 空间不足**
   - 无法交换匿名页面

### 8.3 场景：NUMA 系统中的 kswapd 问题

**问题描述**：
- 多 NUMA 节点系统
- 某个节点的 kswapd 持续工作
- 其他节点内存充足

**分析**：

```bash
# 1. 查看 NUMA 拓扑
numactl --hardware

# 2. 查看各节点内存使用
numastat

# 3. 查看各节点的 kswapd
ps aux | grep kswapd
# 可能有 kswapd0, kswapd1 等

# 4. 查看各节点的内存统计
cat /proc/zoneinfo | grep -A 10 "Node"
```

**解决方案**：

1. **启用 zone_reclaim_mode**
   ```bash
   echo 1 > /proc/sys/vm/zone_reclaim_mode
   ```

2. **使用 numactl 绑定进程**
   ```bash
   numactl --membind=0 --cpunodebind=0 ./application
   ```

3. **调整内存分配策略**
   - 使用 interleave 策略
   - 或绑定到特定节点

---

## 第九章：kswapd 源码阅读指南

### 9.1 关键文件

**主要源码文件**：

1. **mm/vmscan.c**
   - kswapd 主函数
   - balance_pgdat
   - shrink_zone
   - LRU 扫描逻辑

2. **mm/page_alloc.c**
   - 内存分配
   - direct reclaim
   - 唤醒 kswapd

3. **mm/swap.c**
   - swap 机制
   - 页面交换

4. **include/linux/mmzone.h**
   - zone 结构定义
   - 水位线定义

5. **include/linux/vmstat.h**
   - 统计信息定义

### 9.2 关键数据结构

**pg_data_t（节点）**：

```c
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];
    struct zonelist node_zonelists[MAX_ZONELISTS];
    int nr_zones;
    wait_queue_head_t kswapd_wait;
    struct task_struct *kswapd;
    int kswapd_order;
    enum zone_type kswapd_classzone_idx;
    // ...
} pg_data_t;
```

**zone（内存域）**：

```c
struct zone {
    unsigned long watermark[NR_WMARK];  // 水位线
    unsigned long nr_free_pages;        // 空闲页面
    struct lruvec lruvec;              // LRU 列表
    // ...
};
```

**scan_control（扫描控制）**：

```c
struct scan_control {
    unsigned long nr_to_reclaim;       // 需要回收的页面数
    gfp_t gfp_mask;                    // 分配标志
    int order;                         // 分配阶数
    int priority;                      // 扫描优先级
    unsigned int may_writepage:1;      // 可以写回
    unsigned int may_unmap:1;          // 可以取消映射
    unsigned int may_swap:1;           // 可以交换
    // ...
};
```

### 9.3 阅读路径建议

**入门路径**：

1. `kswapd()` 函数：了解主流程
2. `balance_pgdat()`：了解回收逻辑
3. `shrink_zone()`：了解 zone 扫描
4. `shrink_list()`：了解 LRU 扫描

**深入路径**：

1. LRU 列表管理
2. 页面回收算法
3. swap 机制
4. 内存压缩（如果支持）

---

## 第十章：实战案例分析

### 案例 1：Android 系统中的 kswapd 问题

**场景**：
- Android 设备内存紧张
- kswapd 频繁触发
- 应用启动慢，ANR 增加

**分析**：

```bash
# 1. 查看系统内存
adb shell dumpsys meminfo

# 2. 查看 kswapd 活动
adb shell cat /proc/vmstat | grep kswapd

# 3. 查看 LowMemoryKiller 日志
adb shell dmesg | grep -i "lowmemorykiller"
```

**Android 特殊机制**：

- **LowMemoryKiller**：Android 特有的内存回收机制
- **lmk** 与 kswapd 协作
- 当内存低于阈值时，lmk 会杀死进程

### 案例 2：服务器内存压力问题

**场景**：
- 服务器运行数据库
- 内存使用率高
- kswapd 持续工作影响性能

**诊断**：

```bash
# 1. 监控内存和 swap
watch -n 1 'free -h && swapon -s'

# 2. 监控 kswapd
watch -n 1 'ps aux | grep kswapd'

# 3. 分析内存使用
cat /proc/meminfo
```

**优化方案**：

1. 增加物理内存
2. 优化数据库配置
3. 调整 kswapd 参数
4. 使用 swap 分区（SSD）

---

## 附录

### A. 相关内核参数速查

| 参数 | 路径 | 说明 |
|------|------|------|
| min_free_kbytes | /proc/sys/vm/min_free_kbytes | 最小保留内存 |
| swappiness | /proc/sys/vm/swappiness | swap 使用倾向 |
| dirty_ratio | /proc/sys/vm/dirty_ratio | 脏页比例阈值 |
| vfs_cache_pressure | /proc/sys/vm/vfs_cache_pressure | 文件缓存回收倾向 |
| zone_reclaim_mode | /proc/sys/vm/zone_reclaim_mode | NUMA zone 回收模式 |

### B. 常用诊断命令

```bash
# 内存概览
free -h

# 详细内存信息
cat /proc/meminfo

# Zone 信息
cat /proc/zoneinfo

# 内存统计
cat /proc/vmstat

# Swap 信息
swapon -s
cat /proc/swaps

# kswapd 进程
ps aux | grep kswapd
top -H -p $(pgrep kswapd)
```

### C. 推荐阅读资源

- Linux 内核源码：`mm/vmscan.c`
- 《深入理解 Linux 内核》- 内存管理章节
- 《Linux Kernel Development》- Memory Management
- Kernel.org 文档：Memory Management

---

## 📝 学习笔记模板

### 问题分析模板

```markdown
## 问题描述
- 现象：
- 影响：

## 环境信息
- 内核版本：
- 内存大小：
- swap 配置：

## 诊断过程
1. 检查项：
2. 发现：

## 根因分析
- 原因：
- 证据：

## 解决方案
- 措施：
- 效果：

## 经验总结
```

---

**下一步**：完成理论学习后，可以：
1. 阅读内核源码
2. 在实际环境中观察 kswapd 行为
3. 分析真实问题场景
4. 提出具体问题，深入讨论
