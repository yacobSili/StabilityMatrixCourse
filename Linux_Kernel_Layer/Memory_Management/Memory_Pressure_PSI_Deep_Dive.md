# Memory Pressure (PSI) 深度解析

## 📋 文档概述

本文档系统深入地解析 Linux 内核中的 PSI（Pressure Stall Information，压力停滞信息）机制，包括其概念、工作原理、接口使用、与内存管理的关系、实际应用场景以及与 ANR、性能问题的关联。PSI 是现代 Linux 内核提供的内存压力监控机制，能够量化内存压力对系统性能的影响。

## 🎯 学习目标

- 深入理解 PSI 的概念和作用
- 掌握 PSI 的工作原理和计算方式
- 理解 PSI 与内存管理机制的关系
- 掌握 PSI 接口的使用方法
- 能够使用 PSI 诊断内存压力问题
- 理解 PSI 在 ANR 和性能问题分析中的应用
- 能够阅读和分析 PSI 相关源码

---

## 第一章：PSI 基础

### 1.1 什么是 PSI？

**PSI（Pressure Stall Information，压力停滞信息）** 是 Linux 内核从 4.20 版本开始引入的一种资源压力监控机制，用于量化系统资源（CPU、内存、IO）压力对任务执行的影响。

**核心概念**：

PSI 测量的是**压力停滞时间（Pressure Stall Time）**，即由于资源不足导致任务无法执行的时间。

**为什么需要 PSI？**

传统的内存监控指标（如 `/proc/meminfo`）只能告诉我们：
- 内存使用了多少
- 有多少空闲内存
- swap 使用了多少

但无法告诉我们：
- ❌ 内存压力对任务执行的实际影响
- ❌ 任务因为内存不足被阻塞了多长时间
- ❌ 内存压力是否导致了性能下降

**PSI 的优势**：

1. **量化影响**
   - 直接测量任务被阻塞的时间
   - 提供可操作的指标

2. **预测性**
   - 可以预测即将到来的资源压力
   - 提前采取措施

3. **统一接口**
   - CPU、内存、IO 使用统一的接口
   - 便于系统级监控

### 1.2 PSI 支持的资源类型

PSI 监控三种系统资源：

1. **CPU 压力（CPU Pressure）**
   - 由于 CPU 资源不足导致的任务等待时间
   - 接口：`/proc/pressure/cpu`

2. **内存压力（Memory Pressure）**
   - 由于内存资源不足导致的任务等待时间
   - 接口：`/proc/pressure/memory`
   - **本文档重点**

3. **IO 压力（IO Pressure）**
   - 由于 IO 资源不足导致的任务等待时间
   - 接口：`/proc/pressure/io`

### 1.3 PSI 的两种压力类型

**1. some（部分压力）**

- **定义**：至少有一个任务因为资源不足而被阻塞
- **场景**：部分任务受影响，但系统仍在运行
- **影响**：可能导致延迟增加，但不会完全阻塞

**2. full（完全压力）**

- **定义**：所有非空闲任务都被阻塞
- **场景**：系统资源严重不足
- **影响**：系统几乎无法响应，可能导致 ANR、卡顿

**示例理解**：

```
内存压力场景：

some 压力：
- 进程 A 因为内存不足被阻塞
- 进程 B、C 仍在运行
- 系统响应变慢，但仍在工作

full 压力：
- 进程 A、B、C 都因为内存不足被阻塞
- 系统几乎无法响应
- 可能导致 ANR、系统卡死
```

### 1.4 PSI 在内核中的位置

**PSI 与内存管理的关系**：

```
内存管理子系统
    ├── 内存分配（Page Allocator）
    ├── 内存回收（kswapd、Direct Reclaim）
    ├── OOM Killer
    └── PSI（压力监控）← 监控上述机制的影响
```

**PSI 的作用**：

- 监控内存回收对任务的影响
- 监控 Direct Reclaim 导致的阻塞
- 监控 OOM 前的压力积累
- 提供内存压力的量化指标

---

## 第二章：PSI 工作原理

### 2.1 PSI 的测量机制

**核心思想**：

PSI 通过跟踪任务的状态变化来测量压力停滞时间。

**任务状态**：

```c
// 任务状态（简化）
TASK_RUNNING    // 正在运行或可运行
TASK_UNINTERRUPTIBLE  // 不可中断睡眠（等待资源）
TASK_INTERRUPTIBLE    // 可中断睡眠
```

**测量逻辑**：

```
任务状态变化
    ↓
检查是否因为资源不足而阻塞
    ↓
记录阻塞时间
    ↓
累积到 PSI 统计中
```

### 2.2 PSI 的时间窗口

PSI 使用滑动时间窗口来计算压力：

**三个时间窗口**：

1. **10 秒窗口（avg10）**
   - 短期压力
   - 快速响应变化

2. **60 秒窗口（avg60）**
   - 中期压力
   - 平滑短期波动

3. **300 秒窗口（avg300）**
   - 长期压力
   - 反映趋势

**计算方式**：

```
压力百分比 = (阻塞时间 / 时间窗口) * 100%

例如：
- 10 秒窗口内，任务被阻塞 1 秒
- 压力 = (1 / 10) * 100% = 10%
```

### 2.3 PSI 的触发条件

**内存压力触发**：

PSI 在以下情况记录内存压力：

1. **Direct Reclaim**
   ```c
   // 当进程执行 direct reclaim 时
   if (direct_reclaim) {
       psi_mem_stall_start(...);  // 开始记录压力
       // ... 执行回收
       psi_mem_stall_end(...);    // 结束记录
   }
   ```

2. **kswapd 唤醒**
   - kswapd 被唤醒表示内存压力
   - 记录唤醒到完成的时间

3. **OOM 相关**
   - OOM Killer 触发前的压力积累
   - 内存分配失败导致的阻塞

**关键函数调用**：

```c
// 记录内存压力开始
psi_mem_stall_start(unsigned long *flags)

// 记录内存压力结束
psi_mem_stall_end(unsigned long *flags)

// 更新 PSI 统计
psi_account_memstall()
```

### 2.4 PSI 统计的更新

**更新频率**：

- PSI 统计在任务状态变化时更新
- 使用 per-CPU 统计，减少锁竞争
- 定期聚合到全局统计

**统计结构**：

```c
// 简化结构
struct psi_group {
    // some 压力统计
    u64 some[3];  // avg10, avg60, avg300
    
    // full 压力统计
    u64 full[3];  // avg10, avg60, avg300
    
    // 总阻塞时间
    u64 total[PSI_NONIDLE];
};
```

---

## 第三章：PSI 接口使用

### 3.1 查看 PSI 数据

**内存压力接口**：

```bash
# 查看内存压力
cat /proc/pressure/memory

# 输出示例：
# some avg10=5.23 avg60=2.45 avg300=1.23 total=12345678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**输出解读**：

- **some**：部分压力
  - `avg10=5.23`：过去 10 秒平均 5.23% 的时间有任务被阻塞
  - `avg60=2.45`：过去 60 秒平均 2.45% 的时间有任务被阻塞
  - `avg300=1.23`：过去 300 秒平均 1.23% 的时间有任务被阻塞
  - `total=12345678`：总阻塞时间（微秒）

- **full**：完全压力
  - 格式同 some
  - 如果全为 0，说明没有完全压力

**CPU 和 IO 压力**：

```bash
# CPU 压力
cat /proc/pressure/cpu

# IO 压力
cat /proc/pressure/io
```

### 3.2 PSI 阈值监控

**设置压力阈值**：

PSI 支持设置阈值，当压力超过阈值时触发事件。

**接口文件**：

```bash
# 设置 some 压力阈值（10 秒窗口，阈值 5%）
echo "some 10000 5" > /proc/pressure/memory

# 设置 full 压力阈值（60 秒窗口，阈值 10%）
echo "full 60000 10" > /proc/pressure/memory
```

**参数说明**：

- 第一个参数：`some` 或 `full`
- 第二个参数：时间窗口（微秒）
- 第三个参数：压力阈值（百分比）

**清除阈值**：

```bash
# 清除阈值
echo "" > /proc/pressure/memory
```

### 3.3 使用 PSI 进行监控

**实时监控脚本**：

```bash
#!/bin/bash
# monitor_psi.sh

while true; do
    clear
    echo "=== Memory Pressure (PSI) Monitor ==="
    date
    echo ""
    
    # 读取 PSI 数据
    psi_data=$(cat /proc/pressure/memory)
    
    # 解析 some 压力
    some_avg10=$(echo "$psi_data" | grep some | awk '{print $3}' | cut -d= -f2)
    some_avg60=$(echo "$psi_data" | grep some | awk '{print $4}' | cut -d= -f2)
    some_avg300=$(echo "$psi_data" | grep some | awk '{print $5}' | cut -d= -f2)
    
    # 解析 full 压力
    full_avg10=$(echo "$psi_data" | grep full | awk '{print $3}' | cut -d= -f2)
    full_avg60=$(echo "$psi_data" | grep full | awk '{print $4}' | cut -d= -f2)
    full_avg300=$(echo "$psi_data" | grep full | awk '{print $5}' | cut -d= -f2)
    
    echo "Some Pressure:"
    echo "  avg10:  ${some_avg10}%"
    echo "  avg60:  ${some_avg60}%"
    echo "  avg300: ${some_avg300}%"
    echo ""
    echo "Full Pressure:"
    echo "  avg10:  ${full_avg10}%"
    echo "  avg60:  ${full_avg60}%"
    echo "  avg300: ${full_avg300}%"
    
    sleep 1
done
```

**压力等级判断**：

```bash
#!/bin/bash
# psi_alert.sh

# 读取压力值
some_avg10=$(cat /proc/pressure/memory | grep some | awk '{print $3}' | cut -d= -f2)

# 判断压力等级
if (( $(echo "$some_avg10 > 10" | bc -l) )); then
    echo "⚠️  HIGH PRESSURE: $some_avg10%"
elif (( $(echo "$some_avg10 > 5" | bc -l) )); then
    echo "⚠️  MEDIUM PRESSURE: $some_avg10%"
elif (( $(echo "$some_avg10 > 1" | bc -l) )); then
    echo "ℹ️  LOW PRESSURE: $some_avg10%"
else
    echo "✅ NORMAL: $some_avg10%"
fi
```

### 3.4 PSI 与系统监控工具集成

**使用 systemd 监控**：

```ini
# /etc/systemd/system/psi-monitor.service
[Unit]
Description=PSI Memory Pressure Monitor

[Service]
Type=simple
ExecStart=/usr/local/bin/psi-monitor.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

**Prometheus 集成**：

```bash
# 导出 PSI 指标到 Prometheus
# 可以使用 node_exporter 或自定义 exporter
```

---

## 第四章：PSI 与内存管理的关系

### 4.1 PSI 与 Direct Reclaim

**Direct Reclaim 的压力记录**：

当进程执行 Direct Reclaim 时，PSI 会记录阻塞时间：

```c
// 内存分配路径（简化）
__alloc_pages()
    ↓
__alloc_pages_slowpath()
    ↓
__alloc_pages_direct_reclaim()
    ↓
// PSI 开始记录
psi_mem_stall_start(&flags)
    ↓
// 执行回收（可能阻塞）
perform_reclaim()
    ↓
// PSI 结束记录
psi_mem_stall_end(&flags)
```

**压力计算**：

```
Direct Reclaim 压力 = (回收耗时 / 时间窗口) * 100%

如果回收耗时 100ms，10 秒窗口：
压力 = (0.1 / 10) * 100% = 1%
```

**实际影响**：

- Direct Reclaim 阻塞进程执行
- PSI 量化这种阻塞的影响
- 高 Direct Reclaim 压力 → 系统响应变慢

### 4.2 PSI 与 kswapd

**kswapd 的压力记录**：

kswapd 虽然不直接阻塞进程，但内存压力会影响系统：

```c
// kswapd 被唤醒表示内存压力
wakeup_kswapd()
    ↓
// 记录压力开始
psi_mem_stall_start(...)
    ↓
// kswapd 执行回收
balance_pgdat()
    ↓
// 记录压力结束
psi_mem_stall_end(...)
```

**kswapd 压力的特点**：

- 不直接阻塞用户进程
- 但表示内存压力存在
- 如果 kswapd 持续工作，说明内存紧张

### 4.3 PSI 与 OOM Killer

**OOM 前的压力积累**：

在 OOM Killer 触发前，PSI 会记录压力积累：

```
内存持续紧张
    ↓
PSI 压力逐渐上升
    ↓
压力达到阈值（可配置）
    ↓
触发 OOM Killer
```

**使用 PSI 预测 OOM**：

```bash
# 监控压力趋势
# 如果压力持续上升，可能即将触发 OOM

some_avg10=$(cat /proc/pressure/memory | grep some | awk '{print $3}' | cut -d= -f2)

if (( $(echo "$some_avg10 > 20" | bc -l) )); then
    echo "⚠️  WARNING: High memory pressure, OOM risk!"
fi
```

### 4.4 PSI 与内存水位线

**水位线与压力的关系**：

```
内存水位线状态          PSI 压力
─────────────────────────────────
High (高水位)          0% (无压力)
Low (低水位)           1-5% (低压力)
Min (最低水位)         5-10% (中压力)
低于 Min               10%+ (高压力)
```

**水位线触发回收 → PSI 记录压力**：

```
内存低于低水位
    ↓
唤醒 kswapd / 触发 Direct Reclaim
    ↓
PSI 记录压力时间
    ↓
压力值上升
```

---

## 第五章：PSI 源码分析

### 5.1 PSI 核心数据结构

**源码位置**：`kernel/sched/psi.c`、`include/linux/psi_types.h`

**关键数据结构**：

```c
// PSI 组（每个 cgroup 或系统级）
struct psi_group {
    // 压力统计
    struct psi_stats avg[3];  // avg10, avg60, avg300
    
    // 总阻塞时间
    u64 total[PSI_NONIDLE];
    
    // 阈值配置
    struct psi_trigger *trigger;
    
    // 锁
    struct mutex stat_lock;
};

// PSI 统计
struct psi_stats {
    u32 some[3];  // some 压力（avg10, avg60, avg300）
    u32 full[3];  // full 压力（avg10, avg60, avg300）
};
```

### 5.2 PSI 压力记录函数

**psi_mem_stall_start**：

```c
// 开始记录内存压力
void psi_mem_stall_start(unsigned long *flags)
{
    struct rq_flags rf;
    struct rq *rq;
    
    // 获取当前 CPU 的运行队列
    rq = this_rq_lock_irq(&rf);
    
    // 设置标志
    *flags = rq->flags;
    
    // 记录压力开始
    if (!(rq->flags & RQF_PSI)) {
        rq->flags |= RQF_PSI;
        psi_task_change(current, 0, TASK_UNINTERRUPTIBLE);
    }
    
    rq_unlock_irqrestore(rq, &rf);
}
```

**psi_mem_stall_end**：

```c
// 结束记录内存压力
void psi_mem_stall_end(unsigned long *flags)
{
    struct rq_flags rf;
    struct rq *rq;
    
    rq = this_rq_lock_irq(&rf);
    
    // 清除标志
    if (*flags & RQF_PSI) {
        rq->flags &= ~RQF_PSI;
        psi_task_change(current, TASK_UNINTERRUPTIBLE, 0);
    }
    
    rq_unlock_irqrestore(rq, &rf);
}
```

### 5.3 PSI 统计更新

**psi_account_memstall**：

```c
// 更新内存压力统计
void psi_account_memstall(struct task_struct *task, u64 now)
{
    struct psi_group *group;
    u64 delta;
    
    // 计算阻塞时间
    delta = now - task->memstall_start;
    
    // 更新统计
    group = task->psi;
    if (group) {
        group->total[PSI_MEM] += delta;
        
        // 更新平均值
        psi_update_stats(group);
    }
}
```

### 5.4 PSI 接口实现

**/proc/pressure/memory 读取**：

```c
// 读取 PSI 数据
static int psi_show(struct seq_file *m, struct psi_group *group,
                    enum psi_res res)
{
    struct psi_stats *stats = &group->avg[res];
    
    // 输出 some 压力
    seq_printf(m, "some avg10=%u.%02u avg60=%u.%02u avg300=%u.%02u total=%llu\n",
               stats->some[0] / 100, stats->some[0] % 100,
               stats->some[1] / 100, stats->some[1] % 100,
               stats->some[2] / 100, stats->some[2] % 100,
               group->total[PSI_MEM]);
    
    // 输出 full 压力
    seq_printf(m, "full avg10=%u.%02u avg60=%u.%02u avg300=%u.%02u total=%llu\n",
               stats->full[0] / 100, stats->full[0] % 100,
               stats->full[1] / 100, stats->full[full[1] % 100,
               stats->full[2] / 100, stats->full[2] % 100,
               group->total[PSI_MEM_FULL]);
    
    return 0;
}
```

---

## 第六章：PSI 与 ANR 的关系

### 6.1 ANR 与内存压力

**ANR 的触发条件**：

Android 系统中，ANR（Application Not Responding）可能由以下原因触发：

1. **主线程阻塞**
   - 主线程执行耗时操作
   - 主线程等待锁

2. **内存压力**
   - 内存不足导致 GC 频繁
   - Direct Reclaim 阻塞主线程
   - 内存分配失败

**PSI 与 ANR 的关联**：

```
内存压力上升（PSI 上升）
    ↓
Direct Reclaim 阻塞主线程
    ↓
主线程无法响应
    ↓
触发 ANR
```

### 6.2 使用 PSI 分析 ANR

**ANR 分析流程**：

```bash
# 1. 查看 ANR 发生时的 PSI 数据
# （需要从日志或监控系统获取）

# 2. 分析内存压力
cat /proc/pressure/memory

# 3. 如果 PSI 压力高，说明 ANR 可能与内存压力相关
```

**ANR 日志中的 PSI**：

```
ANR in com.example.app
Reason: Executing service com.example.app/.MyService
CPU usage from 0ms to 5000ms:
  ...
Memory pressure (PSI):
  some avg10=15.23 avg60=8.45 avg300=3.21
  full avg10=2.10 avg60=0.50 avg300=0.10
```

**分析要点**：

- **some 压力高**：部分任务被阻塞，可能导致延迟
- **full 压力高**：系统几乎无法响应，很可能导致 ANR
- **压力持续时间**：avg300 反映长期趋势

### 6.3 PSI 阈值与 ANR 预防

**设置 PSI 阈值预警**：

```bash
# 设置阈值，当压力超过 10% 时触发事件
echo "some 10000 10" > /proc/pressure/memory

# 系统可以监听这个事件，提前采取措施
# 例如：触发 GC、杀死低优先级进程等
```

**Android 中的 PSI 使用**：

Android 系统可以使用 PSI 来：
- 预测 ANR 风险
- 提前触发 LowMemoryKiller
- 优化内存分配策略

---

## 第七章：PSI 监控与诊断

### 7.1 PSI 监控工具

**基础监控**：

```bash
# 实时监控
watch -n 1 'cat /proc/pressure/memory'

# 持续记录
while true; do
    echo "$(date): $(cat /proc/pressure/memory)"
    sleep 1
done >> psi_log.txt
```

**压力趋势分析**：

```bash
#!/bin/bash
# psi_trend.sh

prev_avg10=0
while true; do
    avg10=$(cat /proc/pressure/memory | grep some | awk '{print $3}' | cut -d= -f2)
    
    if (( $(echo "$avg10 > $prev_avg10" | bc -l) )); then
        trend="↑ 上升"
    elif (( $(echo "$avg10 < $prev_avg10" | bc -l) )); then
        trend="↓ 下降"
    else
        trend="→ 稳定"
    fi
    
    echo "$(date): $avg10% $trend"
    prev_avg10=$avg10
    sleep 5
done
```

### 7.2 PSI 与其他指标关联

**PSI vs 内存使用率**：

```bash
#!/bin/bash
# psi_vs_memory.sh

while true; do
    # PSI 压力
    psi=$(cat /proc/pressure/memory | grep some | awk '{print $3}' | cut -d= -f2)
    
    # 内存使用率
    mem_used=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100}')
    
    # Swap 使用率
    swap_used=$(free | grep Swap | awk '{printf "%.1f", $3/$2 * 100}')
    
    echo "PSI: ${psi}% | Mem: ${mem_used}% | Swap: ${swap_used}%"
    
    sleep 1
done
```

**PSI vs Direct Reclaim**：

```bash
# 查看 Direct Reclaim 统计
cat /proc/vmstat | grep pgsteal_direct

# 如果 Direct Reclaim 多，PSI 压力通常也高
```

### 7.3 PSI 问题诊断

**问题 1：PSI 压力持续高**

**症状**：
- PSI some 压力 > 10%
- 持续时间长（avg300 也高）

**可能原因**：
1. 内存不足
2. 内存泄漏
3. swap 性能差

**诊断步骤**：

```bash
# 1. 查看内存使用
free -h
cat /proc/meminfo

# 2. 查看 swap 活动
vmstat 1

# 3. 查看内存回收
cat /proc/vmstat | grep -E "pgscan|pgsteal"

# 4. 查看进程内存使用
ps aux --sort=-%mem | head -10
```

**问题 2：PSI full 压力出现**

**症状**：
- PSI full 压力 > 0%
- 系统几乎无法响应

**严重性**：
- ⚠️ 非常严重
- 可能导致 ANR、系统卡死

**诊断**：

```bash
# 1. 立即查看 PSI
cat /proc/pressure/memory

# 2. 查看内存状态
free -h

# 3. 查看 OOM 相关信息
dmesg | grep -i oom

# 4. 查看进程状态
ps aux | grep -E "D|Z"  # 查找阻塞进程
```

**问题 3：PSI 压力突然上升**

**症状**：
- PSI 压力短时间内大幅上升
- 系统响应变慢

**可能原因**：
1. 新进程启动，大量分配内存
2. 内存泄漏进程
3. 内存密集型操作

**诊断**：

```bash
# 1. 查看最近启动的进程
ps aux --sort=start_time | tail -10

# 2. 查看内存增长快的进程
# 使用 top 或 htop 观察

# 3. 查看内存分配失败
dmesg | grep -i "allocation failed"
```

---

## 第八章：PSI 实际应用场景

### 场景 1：Android 系统 ANR 分析

**问题**：

应用频繁 ANR，需要分析是否与内存压力相关。

**分析步骤**：

```bash
# 1. 获取 ANR 发生时的 PSI 数据
# （从系统日志或监控系统）

# 2. 分析 PSI 压力
# 如果 some avg10 > 10%，说明内存压力高

# 3. 关联分析
# - PSI 压力高 + Direct Reclaim 多 → 内存不足
# - PSI 压力高 + swap 活跃 → swap 性能问题
# - PSI 压力高 + 内存使用正常 → 内存碎片化
```

**解决方案**：

- 如果内存不足：增加内存或优化应用
- 如果 swap 性能差：使用 SSD 作为 swap
- 如果内存碎片化：调整内存参数

### 场景 2：服务器性能优化

**问题**：

服务器响应变慢，需要找出瓶颈。

**使用 PSI 分析**：

```bash
# 1. 同时监控三种资源压力
watch -n 1 'cat /proc/pressure/{cpu,memory,io}'

# 2. 找出压力最高的资源
# 如果内存压力最高，重点优化内存

# 3. 分析压力趋势
# 如果压力持续上升，需要采取措施
```

**优化策略**：

- **内存压力高**：增加内存、优化应用、调整 swappiness
- **CPU 压力高**：优化算法、增加 CPU
- **IO 压力高**：优化 IO、使用 SSD

### 场景 3：容器内存限制

**问题**：

容器因内存限制导致性能下降。

**使用 PSI 监控**：

```bash
# 在容器内查看 PSI
# （需要内核支持 cgroup v2 和 PSI）

# 1. 查看容器内存压力
cat /sys/fs/cgroup/memory.pressure

# 2. 设置阈值预警
echo "some 10000 5" > /sys/fs/cgroup/memory.pressure

# 3. 当压力超过阈值时，可以：
# - 触发容器内 GC
# - 限制容器内存使用
# - 迁移容器到其他节点
```

### 场景 4：预测性内存管理

**使用 PSI 预测内存问题**：

```bash
#!/bin/bash
# predictive_memory.sh

# 监控 PSI 趋势
prev_avg10=0
while true; do
    avg10=$(cat /proc/pressure/memory | grep some | awk '{print $3}' | cut -d= -f2)
    
    # 如果压力持续上升
    if (( $(echo "$avg10 > $prev_avg10 + 2" | bc -l) )); then
        echo "⚠️  WARNING: Memory pressure rising: $avg10%"
        
        # 预测：如果继续上升，可能触发 OOM
        if (( $(echo "$avg10 > 15" | bc -l) )); then
            echo "🚨 ALERT: High memory pressure, OOM risk!"
            # 可以触发预防措施
        fi
    fi
    
    prev_avg10=$avg10
    sleep 5
done
```

---

## 第九章：PSI 调优

### 9.1 PSI 阈值配置

**设置合理的阈值**：

```bash
# 根据系统特点设置阈值

# 服务器环境（对延迟敏感）
echo "some 10000 5" > /proc/pressure/memory   # 10秒窗口，5%阈值
echo "full 10000 1" > /proc/pressure/memory   # 10秒窗口，1%阈值

# 桌面环境（可以容忍一定延迟）
echo "some 60000 10" > /proc/pressure/memory  # 60秒窗口，10%阈值
echo "full 60000 5" > /proc/pressure/memory   # 60秒窗口，5%阈值
```

**阈值选择原则**：

- **时间窗口**：越短越敏感，但可能误报
- **压力阈值**：越低越敏感，但可能频繁触发
- **根据实际需求调整**

### 9.2 PSI 与内存参数调优

**降低 PSI 压力的方法**：

1. **增加物理内存**
   - 最直接有效

2. **调整 swappiness**
   ```bash
   # 降低 swap 使用，减少回收压力
   echo 10 > /proc/sys/vm/swappiness
   ```

3. **调整 min_free_kbytes**
   ```bash
   # 提前触发回收，避免突然压力
   echo 131072 > /proc/sys/vm/min_free_kbytes
   ```

4. **优化应用程序**
   - 减少内存使用
   - 及时释放内存
   - 避免内存泄漏

### 9.3 PSI 监控系统集成

**集成到监控系统**：

```bash
# Prometheus exporter 示例（伪代码）
while true; do
    psi_data=$(cat /proc/pressure/memory)
    some_avg10=$(echo "$psi_data" | grep some | awk '{print $3}' | cut -d= -f2)
    
    # 导出到 Prometheus
    echo "memory_pressure_some_avg10 $some_avg10" | \
        curl -X POST http://prometheus:9091/metrics/job/psi
    
    sleep 10
done
```

---

## 第十章：PSI 与其他监控机制对比

### 10.1 PSI vs 传统指标

**传统指标（/proc/meminfo）**：

```
优点：
- 详细的内存使用信息
- 历史数据丰富

缺点：
- 无法量化对性能的影响
- 无法预测问题
```

**PSI**：

```
优点：
- 量化性能影响
- 预测性问题
- 统一接口

缺点：
- 需要内核 4.20+
- 数据相对简单
```

**结合使用**：

- PSI 用于快速判断压力
- 传统指标用于详细分析

### 10.2 PSI vs OOM Score

**OOM Score**：

- 用于 OOM Killer 选择进程
- 反映进程的内存使用
- 静态指标

**PSI**：

- 反映系统级内存压力
- 动态指标
- 可以预测 OOM

**互补关系**：

- PSI 监控系统压力
- OOM Score 用于选择进程

### 10.3 PSI vs Memory Pressure Events

**Memory Pressure Events（旧机制）**：

- Android 早期使用
- 基于内存水位线
- 不够精确

**PSI**：

- 更精确的压力测量
- 量化阻塞时间
- 现代标准

---

## 附录

### A. PSI 相关内核参数

| 参数 | 路径 | 说明 |
|------|------|------|
| PSI 接口 | /proc/pressure/memory | 内存压力接口 |
| PSI 阈值 | /proc/pressure/memory (写入) | 设置压力阈值 |

### B. PSI 相关源码文件

- `kernel/sched/psi.c`：PSI 核心实现
- `include/linux/psi_types.h`：PSI 类型定义
- `include/linux/sched/psi.h`：PSI 接口定义
- `mm/page_alloc.c`：内存分配中的 PSI 调用
- `mm/vmscan.c`：内存回收中的 PSI 调用

### C. PSI 诊断命令速查

```bash
# 查看内存压力
cat /proc/pressure/memory

# 查看 CPU 压力
cat /proc/pressure/cpu

# 查看 IO 压力
cat /proc/pressure/io

# 设置阈值
echo "some 10000 5" > /proc/pressure/memory

# 清除阈值
echo "" > /proc/pressure/memory
```

### D. PSI 压力等级参考

| 压力值 | 等级 | 影响 | 建议 |
|--------|------|------|------|
| 0-1% | 正常 | 无影响 | 无需处理 |
| 1-5% | 低 | 轻微延迟 | 监控 |
| 5-10% | 中 | 明显延迟 | 优化 |
| 10-20% | 高 | 严重延迟 | 立即处理 |
| >20% | 极高 | 可能 ANR/OOM | 紧急处理 |

### E. PSI 学习资源

- Linux 内核文档：`Documentation/accounting/psi.rst`
- PSI 论文：Pressure Stall Information
- 内核源码：`kernel/sched/psi.c`

---

## 📝 学习检查点

完成 PSI 学习后，您应该能够：

- [ ] 理解 PSI 的概念和作用
- [ ] 理解 PSI 的工作原理和计算方式
- [ ] 掌握 PSI 接口的使用方法
- [ ] 能够使用 PSI 诊断内存压力问题
- [ ] 理解 PSI 与 ANR 的关系
- [ ] 能够分析 PSI 数据并找出问题
- [ ] 能够阅读 PSI 相关源码
- [ ] 能够将 PSI 集成到监控系统

---

**下一步**：完成 PSI 理论学习后，可以：
1. 在实际系统中观察 PSI 数据
2. 分析 ANR 问题中的 PSI 数据
3. 将 PSI 集成到监控和告警系统
4. 结合其他内存管理机制，形成完整的内存问题分析能力
