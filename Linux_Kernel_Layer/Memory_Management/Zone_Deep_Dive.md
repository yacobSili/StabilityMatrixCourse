# Zone 深度解析

## 📋 文档概述

本文档系统深入地解析 Linux 内核中的 Zone（内存域）机制，包括其概念、类型、结构、分配策略、水位线机制以及与内存管理其他组件的关系。Zone 是 Linux 内核内存管理的核心概念，理解 Zone 是深入理解内存管理的基础。

## 🎯 学习目标

- 深入理解 Zone 的概念和作用
- 掌握三种 Zone 类型的特点和使用场景
- 理解 Zone 的数据结构和属性
- 掌握 Zone 的水位线机制
- 理解 Zone 与内存分配的关系
- 能够分析 Zone 相关的内存问题
- 掌握 Zone 的监控和诊断方法

---

## 第一章：Zone 基础

### 1.1 什么是 Zone？

**Zone（内存域）** 是 Linux 内核将物理内存按照用途和特性划分的逻辑区域。每个 Zone 代表一类具有特定属性的内存页面集合。

**为什么需要 Zone？**

1. **硬件限制**
   - 某些设备只能访问特定地址范围的内存（DMA）
   - 32 位系统地址空间限制（HIGHMEM）

2. **内存特性差异**
   - 不同区域的内存有不同的访问特性
   - 需要区分可用的内存类型

3. **管理效率**
   - 将内存分类管理，提高分配效率
   - 便于实现不同的内存策略

### 1.2 Zone 的类型

Linux 内核定义了三种主要的 Zone 类型：

**1. ZONE_DMA（DMA 区域）**

- **地址范围**：0-16MB（或 0-4GB，取决于架构）
- **用途**：直接内存访问（DMA）设备使用
- **特点**：
  - 地址范围固定
  - 某些设备只能访问此范围
  - 通常较小，资源稀缺

**2. ZONE_NORMAL（普通区域）**

- **地址范围**：16MB-896MB（32 位）或更大（64 位）
- **用途**：内核和用户进程的常规内存
- **特点**：
  - 内核可以直接映射（线性映射）
  - 性能最好
  - 是主要的内存区域

**3. ZONE_HIGHMEM（高端内存区域）**

- **地址范围**：896MB 以上（32 位系统）
- **用途**：32 位系统上超过 896MB 的内存
- **特点**：
  - 64 位系统通常不需要（地址空间足够）
  - 32 位系统上需要动态映射
  - 主要用于用户空间

**Zone 类型总结**：

| Zone 类型 | 地址范围（32位） | 主要用途 | 64位系统 |
|-----------|----------------|---------|---------|
| ZONE_DMA | 0-16MB | DMA 设备 | 存在 |
| ZONE_NORMAL | 16MB-896MB | 常规内存 | 主要区域 |
| ZONE_HIGHMEM | >896MB | 高端内存 | 通常不存在 |

### 1.3 Zone 在内核中的位置

**内存管理层次结构**：

```
系统内存
    ↓
NUMA 节点 (Node)
    ↓
Zone (内存域)
    ↓
页面 (Page)
```

**多级结构**：

```
pg_data_t (节点)
    ├── node_zones[] (Zone 数组)
    │   ├── ZONE_DMA
    │   ├── ZONE_NORMAL
    │   └── ZONE_HIGHMEM
    └── node_zonelists[] (Zone 列表，用于分配)
```

---

## 第二章：Zone 的数据结构

### 2.1 zone 结构体详解

**源码位置**：`include/linux/mmzone.h`

**关键结构体**：

```c
struct zone {
    // 水位线
    unsigned long watermark[NR_WMARK];
    
    // 空闲页面管理
    long lowmem_reserve[MAX_NR_ZONES];
    struct per_cpu_pageset __percpu *pageset;
    
    // 空闲页面统计
    unsigned long       pages_min;      // 最低水位
    unsigned long       pages_low;      // 低水位
    unsigned long       pages_high;      // 高水位
    unsigned long       pages_spanned;   // Zone 总页面数
    unsigned long       pages_present;   // 实际存在的页面数
    unsigned long       pages_managed;   // 被伙伴系统管理的页面
    
    // LRU 列表
    struct lruvec      lruvec;
    
    // 统计信息
    atomic_long_t      vm_stat[NR_VM_ZONE_STAT_ITEMS];
    
    // 其他属性
    unsigned long       zone_start_pfn;  // Zone 起始页面帧号
    unsigned long       zone_end_pfn;    // Zone 结束页面帧号
    // ...
};
```

### 2.2 水位线（Watermark）详解

**三种水位线**：

1. **min（最低水位）**
   - `pages_min` 或 `watermark[WMARK_MIN]`
   - 紧急情况下的最低内存
   - 低于此值会触发 direct reclaim

2. **low（低水位）**
   - `pages_low` 或 `watermark[WMARK_LOW]`
   - kswapd 开始工作的阈值
   - 低于此值会唤醒 kswapd

3. **high（高水位）**
   - `pages_high` 或 `watermark[WMARK_HIGH]`
   - 理想的内存水平
   - 达到此值 kswapd 可以停止工作

**水位线关系**：

```
High Watermark (高水位)
    ↓
    | 正常内存使用区域
    | kswapd 停止工作
    |
Low Watermark (低水位)  ← kswapd 开始工作
    ↓
    | kswapd 回收区域
    |
Min Watermark (最低水位) ← direct reclaim 开始
    ↓
    | 紧急回收区域
    | 可能触发 OOM
```

**水位线计算**：

```c
// 简化版计算逻辑
min = min_free_kbytes * zone_size / total_memory
low = min * 1.5
high = min * 2.0
```

**查看水位线**：

```bash
# 查看所有 zone 的水位线
cat /proc/zoneinfo | grep -E "Node|zone|pages free|low|high|min"

# 输出示例：
# Node 0, zone   Normal
#   pages free     12345
#         min      1024
#         low      2048
#         high     3072
```

### 2.3 Zone 的统计信息

**vm_stat 数组**：

Zone 维护大量的统计信息，存储在 `vm_stat[]` 数组中：

**关键统计项**：

- **NR_FREE_PAGES**：空闲页面数
- **NR_ALLOC_BATCH**：批量分配页面数
- **NR_INACTIVE_ANON**：非活跃匿名页面
- **NR_ACTIVE_ANON**：活跃匿名页面
- **NR_INACTIVE_FILE**：非活跃文件页面
- **NR_ACTIVE_FILE**：活跃文件页面
- **NR_UNEVICTABLE**：不可回收页面
- **NR_MLOCK**：mlock 锁定的页面

**查看统计信息**：

```bash
# 查看 zone 统计
cat /proc/zoneinfo

# 查看全局统计
cat /proc/vmstat

# 查看特定统计项
cat /proc/vmstat | grep nr_free_pages
```

---

## 第三章：Zone 与内存分配

### 3.1 Zone 分配顺序（Zonelist）

**什么是 Zonelist？**

Zonelist 定义了内存分配时的 Zone 搜索顺序。当需要分配内存时，内核按照 Zonelist 的顺序在各个 Zone 中查找可用内存。

**Zonelist 类型**：

1. **ZONELIST_FALLBACK（回退列表）**
   - 包含所有节点的所有 Zone
   - 当前节点优先，然后回退到其他节点
   - 默认使用此列表

2. **ZONELIST_NOFALLBACK（不回退列表）**
   - 只包含当前节点的 Zone
   - 用于 NUMA 系统的特定策略

**Zonelist 构建规则**：

```
Zonelist 顺序（简化）：
1. 当前节点的 ZONE_NORMAL
2. 当前节点的 ZONE_DMA
3. 其他节点的 ZONE_NORMAL
4. 其他节点的 ZONE_DMA
5. 当前节点的 ZONE_HIGHMEM（如果存在）
6. 其他节点的 ZONE_HIGHMEM（如果存在）
```

**查看 Zonelist**：

```bash
# 查看节点的 zonelist
cat /proc/zoneinfo | grep -A 50 "Node"
```

### 3.2 内存分配流程

**分配路径**：

```
__alloc_pages()
    ↓
__alloc_pages_nodemask()
    ↓
get_page_from_freelist()
    ↓
遍历 Zonelist
    ↓
检查 Zone 水位线
    ↓
从 Zone 分配页面
```

**关键函数**：

```c
// 内存分配入口
struct page *__alloc_pages(gfp_t gfp_mask, unsigned int order,
                          struct zonelist *zonelist);

// 从 freelist 获取页面
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
                       const struct alloc_context *ac);
```

**分配标志（gfp_mask）的影响**：

- **GFP_KERNEL**：可以从所有 Zone 分配
- **GFP_DMA**：只能从 ZONE_DMA 分配
- **GFP_HIGHMEM**：可以从 ZONE_HIGHMEM 分配
- **__GFP_HIGH**：可以忽略水位线限制

### 3.3 Zone 的伙伴系统（Buddy System）

**什么是伙伴系统？**

伙伴系统是 Zone 中管理空闲页面的算法，将空闲页面组织成不同大小的块（order）。

**Order 概念**：

- Order 0：1 个页面（4KB）
- Order 1：2 个页面（8KB）
- Order 2：4 个页面（16KB）
- Order 10：1024 个页面（4MB）

**伙伴系统结构**：

```c
struct zone {
    // 空闲页面列表（按 order 组织）
    struct free_area free_area[MAX_ORDER];
    // ...
};

struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  // 空闲页面链表
    unsigned long nr_free;                       // 空闲块数量
};
```

**分配过程**：

```
请求分配 order N 的块
    ↓
检查 free_area[N] 是否有空闲块
    ↓
有：直接分配
    ↓
无：从 free_area[N+1] 分割
    ↓
继续向上查找，直到找到或失败
```

**查看伙伴系统状态**：

```bash
# 查看 buddy 信息
cat /proc/buddyinfo

# 输出示例：
# Node 0, zone   Normal
#   0    1    2    3    4    5    6    7    8    9   10
# 1234 567 890 123 45  6   7   8   9   0   1
# 表示各 order 的空闲块数量
```

---

## 第四章：Zone 的水位线机制

### 4.1 水位线的计算

**计算基础**：

水位线基于 `min_free_kbytes` 参数计算，该参数定义了系统保留的最小内存。

**计算公式**（简化）：

```c
// 伪代码
total_free_pages = sum(all_zones_free_pages);
min_free_kbytes = sqrt(total_free_pages * 16);  // 默认计算

for each zone:
    zone->pages_min = (min_free_kbytes * zone->pages_present) / total_pages;
    zone->pages_low = zone->pages_min * 1.5;
    zone->pages_high = zone->pages_min * 2.0;
```

**实际计算考虑因素**：

1. Zone 大小比例
2. 系统总内存
3. 低内存阈值
4. 可配置参数

**查看和设置 min_free_kbytes**：

```bash
# 查看当前值
cat /proc/sys/vm/min_free_kbytes

# 设置值（需要 root）
echo 65536 > /proc/sys/vm/min_free_kbytes

# 永久设置
echo "vm.min_free_kbytes = 65536" >> /etc/sysctl.conf
sysctl -p
```

### 4.2 水位线检查

**检查函数**：

```c
// 检查 zone 是否有足够内存
bool zone_watermark_ok(struct zone *z, unsigned int order,
                      unsigned long mark, int classzone_idx,
                      unsigned long alloc_flags);

// 检查所有 zone
bool zone_watermark_ok_safe(struct zone *z, unsigned int order,
                           unsigned long mark, int classzone_idx);
```

**检查逻辑**：

```c
// 简化逻辑
free_pages = zone_page_state(zone, NR_FREE_PAGES);
required = watermark[mark] + (1 << order);  // 水位 + 请求大小

if (free_pages >= required)
    return true;  // 有足够内存
else
    return false;  // 内存不足，需要回收
```

### 4.3 水位线触发机制

**触发 direct reclaim**：

```c
// 内存分配时检查
if (!zone_watermark_ok(zone, order, WMARK_MIN, ...)) {
    // 触发 direct reclaim
    page = __alloc_pages_direct_reclaim(...);
}
```

**触发 kswapd**：

```c
// 检查是否需要唤醒 kswapd
if (!zone_watermark_ok(zone, order, WMARK_LOW, ...)) {
    // 唤醒 kswapd
    wakeup_kswapd(zone, order, classzone_idx);
}
```

**水位线状态判断**：

```bash
# 脚本：检查 zone 水位状态
#!/bin/bash
cat /proc/zoneinfo | awk '
/Node/ {node=$2}
/zone/ {zone=$2}
/pages free/ {free=$3}
/min/ {min=$2}
/low/ {low=$2}
/high/ {high=$2}
{
    if (free && min && low && high) {
        if (free >= high) status="正常";
        else if (free >= low) status="低水位";
        else if (free >= min) status="最低水位";
        else status="紧急";
        print "Node " node ", zone " zone ": " status " (free=" free ", min=" min ", low=" low ", high=" high ")";
    }
}'
```

---

## 第五章：NUMA 与 Zone

### 5.1 NUMA 架构中的 Zone

**NUMA（Non-Uniform Memory Access）**：

在 NUMA 系统中，每个 CPU 节点有本地内存，访问本地内存快，访问远程内存慢。

**NUMA 与 Zone 的关系**：

```
系统
├── Node 0 (CPU 0-3)
│   ├── ZONE_DMA
│   ├── ZONE_NORMAL
│   └── ZONE_HIGHMEM
└── Node 1 (CPU 4-7)
    ├── ZONE_DMA
    ├── ZONE_NORMAL
    └── ZONE_HIGHMEM
```

**每个节点独立管理**：

- 每个 NUMA 节点有独立的 Zone
- 每个节点有独立的 kswapd
- 内存分配优先使用本地节点

### 5.2 Zone 的本地性

**本地 Zone 优先**：

```c
// Zonelist 构建时，本地 Zone 在前
zonelist = {
    // 本地节点
    node_zonelists[ZONE_NORMAL],  // 本地 ZONE_NORMAL
    node_zonelists[ZONE_DMA],     // 本地 ZONE_DMA
    // 远程节点（回退）
    remote_node_zonelists[...],
};
```

**查看 NUMA 拓扑**：

```bash
# 查看 NUMA 拓扑
numactl --hardware

# 查看各节点内存
numastat

# 查看 zone 分布
cat /proc/zoneinfo | grep -E "Node|zone|pages"
```

### 5.3 Zone Reclaim Mode

**zone_reclaim_mode 参数**：

控制当本地 Zone 内存不足时的行为。

```bash
# 查看当前值
cat /proc/sys/vm/zone_reclaim_mode

# 值说明：
# 0: 禁用，允许从其他节点分配
# 1: 启用，优先回收本地 Zone
# 2: 写回脏页
# 4: 交换页面
```

**使用场景**：

- **禁用（0）**：允许跨节点分配，避免本地内存压力
- **启用（1）**：保持内存本地性，提高性能

---

## 第六章：Zone 的监控与诊断

### 6.1 查看 Zone 信息

**/proc/zoneinfo**：

最详细的 Zone 信息源。

```bash
# 查看所有 zone
cat /proc/zoneinfo

# 查看特定节点的 zone
cat /proc/zoneinfo | grep -A 30 "Node 0"

# 查看关键信息
cat /proc/zoneinfo | grep -E "Node|zone|pages free|min|low|high|spanned|present"
```

**输出解读**：

```
Node 0, zone   Normal
  pages free     1234567        ← 空闲页面数
        min       10240          ← 最低水位
        low       20480          ← 低水位
        high      30720          ← 高水位
        spanned   1048576        ← Zone 总页面数（包括空洞）
        present   1048576        ← 实际存在的页面数
        managed   1040000        ← 被伙伴系统管理的页面
```

### 6.2 监控 Zone 水位变化

**实时监控脚本**：

```bash
#!/bin/bash
# monitor_zone_watermark.sh

while true; do
    clear
    echo "=== Zone Watermark Monitor ==="
    date
    echo ""
    
    cat /proc/zoneinfo | awk '
    /Node/ {node=$2}
    /zone/ {zone=$2}
    /pages free/ {free=$3}
    /min/ {min=$2}
    /low/ {low=$2}
    /high/ {high=$2}
    {
        if (free && min && low && high) {
            ratio = (free / high) * 100;
            printf "Node %s, zone %-10s: Free=%8d (%.1f%% of high)\n", 
                node, zone, free, ratio;
        }
    }'
    
    sleep 1
done
```

### 6.3 Zone 压力监控

**内存压力接口**（Linux 5.2+）：

```bash
# 查看内存压力
cat /proc/pressure/memory

# 输出示例：
# some avg10=0.00 avg60=0.00 avg300=0.00 total=12345
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**压力指标说明**：

- **some**：部分进程因内存压力而阻塞
- **full**：所有进程因内存压力而阻塞
- **avg10/60/300**：过去 10/60/300 秒的平均压力
- **total**：总阻塞时间（微秒）

### 6.4 Zone 碎片化分析

**查看碎片化**：

```bash
# 查看 buddy 信息（碎片化指标）
cat /proc/buddyinfo

# 分析脚本
cat /proc/buddyinfo | awk '
BEGIN {
    print "Node Zone Order0 Order1 Order2 Order3 Order4 Order5 Order6 Order7 Order8 Order9 Order10";
}
/Node/ {
    node=$2; zone=$3;
    getline;
    orders=$0;
    print node, zone, orders;
}'
```

**碎片化判断**：

- 如果低 order（0-3）的空闲块多，高 order（4+）的空闲块少，说明碎片化严重
- 碎片化会导致无法分配大块连续内存

---

## 第七章：Zone 相关问题诊断

### 7.1 Zone 内存耗尽

**症状**：

- 某个 Zone 的内存完全耗尽
- 无法从该 Zone 分配内存
- 可能触发 OOM

**诊断**：

```bash
# 1. 查看各 Zone 的空闲内存
cat /proc/zoneinfo | grep -E "Node|zone|pages free"

# 2. 查看 Zone 的水位线
cat /proc/zoneinfo | grep -E "min|low|high"

# 3. 查看内存分配失败统计
cat /proc/vmstat | grep allocstall

# 4. 查看 OOM 相关信息
dmesg | grep -i oom
```

**可能原因**：

1. **内存泄漏**
   - 应用程序或内核模块泄漏内存
   - 内存无法释放回 Zone

2. **内存碎片化**
   - 虽然有空闲页面，但无法组成连续块
   - 查看 `/proc/buddyinfo`

3. **Zone 配置不当**
   - ZONE_DMA 太小
   - 大量 DMA 分配请求

**解决方案**：

1. 排查内存泄漏
2. 增加物理内存
3. 调整 Zone 配置
4. 使用内存压缩（如果支持）

### 7.2 ZONE_DMA 耗尽

**特殊场景**：

ZONE_DMA 通常很小（16MB），容易被耗尽。

**诊断**：

```bash
# 查看 ZONE_DMA 状态
cat /proc/zoneinfo | grep -A 20 "DMA"

# 查看 DMA 分配失败
dmesg | grep -i "dma allocation failed"
```

**解决方案**：

1. **增加 ZONE_DMA 大小**（如果硬件支持）
   - 修改内核启动参数
   - `dma_zone_size=XX`

2. **优化 DMA 使用**
   - 减少 DMA 缓冲区大小
   - 及时释放 DMA 内存

3. **使用 CMA（Contiguous Memory Allocator）**
   - 为 DMA 提供连续内存
   - 配置 CMA 区域

### 7.3 Zone 水位线异常

**症状**：

- Zone 水位线设置不合理
- kswapd 过早或过晚触发
- 频繁 direct reclaim

**诊断**：

```bash
# 1. 查看当前水位线
cat /proc/zoneinfo | grep -E "Node|zone|min|low|high"

# 2. 查看 min_free_kbytes
cat /proc/sys/vm/min_free_kbytes

# 3. 计算水位线比例
# 检查是否合理（low 应该是 min 的 1.5 倍，high 是 2 倍）
```

**调整**：

```bash
# 调整 min_free_kbytes
# 建议值：系统内存的 1-3%
echo 131072 > /proc/sys/vm/min_free_kbytes  # 128MB

# 验证效果
cat /proc/zoneinfo | grep -E "min|low|high"
```

### 7.4 NUMA 系统中的 Zone 不平衡

**症状**：

- 某个节点的 Zone 内存紧张
- 其他节点内存充足
- kswapd 在特定节点持续工作

**诊断**：

```bash
# 1. 查看各节点内存分布
numastat

# 2. 查看各节点的 Zone 状态
cat /proc/zoneinfo | grep -E "Node|pages free"

# 3. 查看进程的内存绑定
numactl --show
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

3. **使用内存交错（interleave）**
   ```bash
   numactl --interleave=all ./application
   ```

---

## 第八章：Zone 源码分析

### 8.1 Zone 初始化

**源码位置**：`mm/page_alloc.c`

**初始化流程**：

```c
// 系统启动时初始化
void __init free_area_init_nodes(unsigned long *max_zone_pfn)
{
    // 为每个节点初始化 zone
    for_each_online_node(nid) {
        pg_data_t *pgdat = NODE_DATA(nid);
        free_area_init_node(nid, NULL, ...);
    }
}

// 初始化节点
static void __init free_area_init_node(int nid, ...)
{
    // 初始化各个 zone
    for (j = 0; j < MAX_NR_ZONES; j++) {
        struct zone *zone = pgdat->node_zones + j;
        free_area_init_core(pgdat, zone, j, ...);
    }
}
```

### 8.2 Zone 水位线初始化

**计算水位线**：

```c
static void __setup_per_zone_wmarks(void)
{
    unsigned long pages_min = min_free_kbytes >> (PAGE_SHIFT - 10);
    unsigned long lowmem_pages = 0;
    struct zone *zone;
    unsigned long flags;
    
    // 计算总低内存页面
    for_each_zone(zone) {
        if (!is_highmem(zone))
            lowmem_pages += zone->pages_present;
    }
    
    // 为每个 zone 计算水位线
    for_each_zone(zone) {
        u64 tmp;
        
        tmp = (u64)pages_min * zone->pages_present;
        do_div(tmp, lowmem_pages);
        
        zone->watermark[WMARK_MIN] = tmp;
        zone->watermark[WMARK_LOW] = tmp * 5 / 4;
        zone->watermark[WMARK_HIGH] = tmp * 3 / 2;
        
        zone->pages_min = zone->watermark[WMARK_MIN];
        zone->pages_low = zone->watermark[WMARK_LOW];
        zone->pages_high = zone->watermark[WMARK_HIGH];
    }
}
```

### 8.3 Zone 分配路径

**关键函数调用链**：

```c
// 分配入口
__alloc_pages(gfp_t gfp_mask, unsigned int order, ...)
    ↓
__alloc_pages_nodemask(gfp_mask, order, nodemask, ...)
    ↓
get_page_from_freelist(gfp_mask, order, alloc_flags, ...)
    ↓
// 遍历 zonelist
for_each_zone_zonelist_nodemask(zone, z, zonelist, ...)
    ↓
// 检查水位线
zone_watermark_ok(zone, order, mark, ...)
    ↓
// 从伙伴系统分配
__rmqueue(zone, order, migratetype, ...)
```

**zone_watermark_ok 实现**：

```c
bool zone_watermark_ok(struct zone *z, unsigned int order,
                      unsigned long mark, int classzone_idx,
                      unsigned long alloc_flags)
{
    long free_pages = zone_page_state(z, NR_FREE_PAGES);
    long min = mark;
    int o;
    
    // 检查是否有足够的空闲页面
    if (free_pages <= min + z->lowmem_reserve[classzone_idx])
        return false;
    
    // 如果需要大块内存，检查高 order 的空闲块
    if (order) {
        for (o = order; o < MAX_ORDER; o++) {
            struct free_area *area = &z->free_area[o];
            int mt;
            
            if (!area->nr_free)
                continue;
            
            for (mt = 0; mt < MIGRATE_TYPES; mt++) {
                if (!list_empty(&area->free_list[mt]))
                    return true;
            }
        }
        return false;
    }
    return true;
}
```

---

## 第九章：Zone 与内存回收

### 9.1 Zone 与 kswapd

**kswapd 按 Zone 工作**：

```c
// kswapd 扫描所有 zone
static void balance_pgdat(pg_data_t *pgdat, int order, int classzone_idx)
{
    int end_zone = 0;
    struct scan_control sc = { ... };
    
    // 遍历所有 zone
    for (i = 0; i <= end_zone; i++) {
        struct zone *zone = pgdat->node_zones + i;
        
        if (!populated_zone(zone))
            continue;
        
        // 对每个 zone 执行回收
        shrink_zone(zone, &sc);
    }
}
```

**Zone 的 LRU 列表**：

每个 Zone 维护自己的 LRU 列表：

```c
struct zone {
    struct lruvec lruvec;  // LRU 列表
    // ...
};

struct lruvec {
    struct list_head lists[NR_LRU_LISTS];  // 各种 LRU 列表
    // ...
};
```

### 9.2 Zone 的回收优先级

**回收顺序**：

1. **文件页面**（inactive_file）
   - 最容易回收
   - 可以直接释放
   - 需要时可以从磁盘重新读取

2. **匿名页面**（inactive_anon）
   - 需要交换到 swap
   - 有 I/O 开销
   - 但可以释放物理内存

3. **活跃页面**（active）
   - 最后选择
   - 需要先移动到 inactive

**Zone 间回收**：

- kswapd 会平衡各个 Zone
- 优先回收内存压力大的 Zone
- 保持各 Zone 的水位线

---

## 第十章：实际应用场景

### 场景 1：32 位系统的 ZONE_HIGHMEM

**问题**：

32 位系统地址空间有限（4GB），需要 ZONE_HIGHMEM 来使用超过 896MB 的内存。

**特点**：

- ZONE_HIGHMEM 需要动态映射
- 访问性能略低于 ZONE_NORMAL
- 主要用于用户空间

**64 位系统**：

- 地址空间足够大
- 通常不需要 ZONE_HIGHMEM
- ZONE_NORMAL 可以覆盖所有内存

### 场景 2：嵌入式系统的 ZONE_DMA

**问题**：

嵌入式系统通常内存较小，ZONE_DMA 可能占用较大比例。

**优化**：

1. 减少 ZONE_DMA 大小（如果硬件允许）
2. 优化 DMA 使用
3. 使用 CMA

### 场景 3：服务器系统的 Zone 配置

**大型服务器**：

- 通常有大量内存
- 多个 NUMA 节点
- 需要合理配置各 Zone

**监控重点**：

- 各 Zone 的水位线
- Zone 间的平衡
- 内存分配失败率

---

## 附录

### A. Zone 相关内核参数

| 参数 | 路径 | 说明 |
|------|------|------|
| min_free_kbytes | /proc/sys/vm/min_free_kbytes | 最小保留内存，影响水位线 |
| zone_reclaim_mode | /proc/sys/vm/zone_reclaim_mode | NUMA zone 回收模式 |
| lowmem_reserve_ratio | /proc/sys/vm/lowmem_reserve_ratio | Zone 保留内存比例 |

### B. 常用诊断命令

```bash
# Zone 详细信息
cat /proc/zoneinfo

# Buddy 系统状态（碎片化）
cat /proc/buddyinfo

# 内存统计
cat /proc/vmstat

# NUMA 信息
numactl --hardware
numastat

# 内存压力
cat /proc/pressure/memory
```

### C. Zone 相关数据结构速查

```c
// Zone 结构
struct zone {
    unsigned long watermark[NR_WMARK];  // 水位线
    struct free_area free_area[MAX_ORDER];  // 伙伴系统
    struct lruvec lruvec;  // LRU 列表
    atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];  // 统计
    // ...
};

// 节点结构
struct pg_data_t {
    struct zone node_zones[MAX_NR_ZONES];  // Zone 数组
    struct zonelist node_zonelists[MAX_ZONELISTS];  // 分配列表
    // ...
};
```

### D. 源码文件位置

- `include/linux/mmzone.h`：Zone 结构定义
- `mm/page_alloc.c`：Zone 初始化和分配
- `mm/vmscan.c`：Zone 回收
- `mm/memory.c`：Zone 相关操作

---

## 📝 学习检查点

完成 Zone 学习后，您应该能够：

- [ ] 理解 Zone 的概念和三种类型
- [ ] 理解 Zone 的数据结构和属性
- [ ] 理解水位线机制和计算
- [ ] 理解 Zone 与内存分配的关系
- [ ] 能够查看和分析 Zone 状态
- [ ] 能够诊断 Zone 相关问题
- [ ] 理解 NUMA 系统中的 Zone
- [ ] 能够阅读 Zone 相关源码

---

**下一步**：完成 Zone 理论学习后，可以：
1. 阅读内核源码，深入理解实现细节
2. 在实际系统中观察 Zone 行为
3. 分析 Zone 相关的实际问题
4. 结合 kswapd 学习，理解完整的内存回收机制
