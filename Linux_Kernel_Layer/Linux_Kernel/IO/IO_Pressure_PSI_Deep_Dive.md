# IO Pressure (PSI) 深度解析

## 📋 文档概述

本文档系统深入地解析 Linux 内核中的 IO PSI（Pressure Stall Information，压力停滞信息）机制，包括其概念、工作原理、接口使用、与 IO 阻塞的关系、实际应用场景以及与 ANR、性能问题的关联。IO PSI 是现代 Linux 内核提供的 IO 压力监控机制，能够量化 IO 压力对系统性能的影响。

## 🎯 学习目标

- 深入理解 IO PSI 的概念和作用
- 掌握 IO PSI 的工作原理和计算方式
- 理解 IO PSI 与 IO 阻塞机制的关系
- 掌握 IO PSI 接口的使用方法
- 能够使用 IO PSI 诊断 IO 压力问题
- 理解 IO PSI 在 ANR 和性能问题分析中的应用
- 能够阅读和分析 IO PSI 相关源码

---

## 第一章：IO PSI 基础

### 1.1 什么是 IO PSI？

**IO PSI（IO Pressure Stall Information，IO 压力停滞信息）** 是 Linux 内核从 4.20 版本开始引入的 PSI 机制的一部分，专门用于量化 IO 资源压力对任务执行的影响。

**核心概念**：

IO PSI 测量的是**IO 压力停滞时间（IO Pressure Stall Time）**，即由于 IO 资源不足导致任务无法执行的时间。

**为什么需要 IO PSI？**

传统的 IO 监控指标（如 `iostat`）只能告诉我们：

- IO 使用率是多少
- IO 吞吐量是多少
- IO 延迟是多少

但无法告诉我们：

- ❌ IO 压力对任务执行的实际影响
- ❌ 任务因为 IO 阻塞被阻塞了多长时间
- ❌ IO 压力是否导致了性能下降或 ANR

**IO PSI 的优势**：

1. **量化影响**
   
   - 直接测量任务因 IO 阻塞的时间
   - 提供可操作的指标

2. **预测性**
   
   - 可以预测即将到来的 IO 压力
   - 提前采取措施

3. **统一接口**
   
   - 与 CPU、内存 PSI 使用统一接口
   - 便于系统级监控

### 1.2 IO PSI 的两种压力类型

**1. some（部分压力）**

- **定义**：至少有一个任务因为 IO 资源不足而被阻塞
- **场景**：部分任务受影响，但系统仍在运行
- **影响**：可能导致延迟增加，但不会完全阻塞

**2. full（完全压力）**

- **定义**：所有非空闲任务都被 IO 阻塞
- **场景**：IO 资源严重不足
- **影响**：系统几乎无法响应，可能导致 ANR、卡顿

**示例理解**：

```
IO 压力场景：

some 压力：
- 进程 A 因为等待磁盘 IO 被阻塞
- 进程 B、C 仍在运行（可能使用 CPU 或内存）
- 系统响应变慢，但仍在工作

full 压力：
- 进程 A、B、C 都因为等待 IO 被阻塞
- 系统几乎无法响应
- 可能导致 ANR、系统卡死
```

### 1.3 IO PSI 在内核中的位置

**IO PSI 与 IO 子系统的关系**：

```
IO 子系统
    ├── VFS（虚拟文件系统）
    ├── Block I/O 层
    ├── IO 调度器
    ├── 存储驱动
    └── PSI（压力监控）← 监控上述机制的影响
```

**IO PSI 的作用**：

- 监控 IO 阻塞对任务的影响
- 监控慢存储设备导致的阻塞
- 监控 IO 竞争导致的延迟
- 提供 IO 压力的量化指标

---

## 第二章：IO PSI 工作原理

### 2.1 IO PSI 的测量机制

**核心思想**：

IO PSI 通过跟踪任务的状态变化来测量 IO 压力停滞时间。

**任务状态**：

```c
// 任务状态（简化）
TASK_RUNNING          // 正在运行或可运行
TASK_UNINTERRUPTIBLE  // 不可中断睡眠（等待 IO）
TASK_INTERRUPTIBLE    // 可中断睡眠
```

**测量逻辑**：

```
任务状态变化
    ↓
检查是否因为 IO 阻塞而进入等待状态
    ↓
记录阻塞时间
    ↓
累积到 IO PSI 统计中
```

**IO 阻塞的识别**：

```c
// 当任务因为 IO 阻塞时
if (task->state == TASK_UNINTERRUPTIBLE && 
    task->in_iowait) {
    // 记录 IO 压力
    psi_io_stall_start(...);
}
```

### 2.2 IO PSI 的时间窗口

IO PSI 使用滑动时间窗口来计算压力：

**三个时间窗口**：

1. **10 秒窗口（avg10）**
   
   - 短期 IO 压力
   - 快速响应变化

2. **60 秒窗口（avg60）**
   
   - 中期 IO 压力
   - 平滑短期波动

3. **300 秒窗口（avg300）**
   
   - 长期 IO 压力
   - 反映趋势

**计算方式**：

```
IO 压力百分比 = (IO 阻塞时间 / 时间窗口) * 100%

例如：
- 10 秒窗口内，任务因 IO 阻塞 1 秒
- 压力 = (1 / 10) * 100% = 10%
```

### 2.3 IO PSI 的触发条件

**IO 压力触发**：

IO PSI 在以下情况记录 IO 压力：

1. **同步 IO 阻塞**
   
   ```c
   // 当进程执行同步 IO 时
   if (sync_io) {
       psi_io_stall_start(...);  // 开始记录压力
       // ... 等待 IO 完成
       psi_io_stall_end(...);    // 结束记录
   }
   ```

2. **等待队列阻塞**
   
   - 进程在 IO 等待队列中等待
   - 等待磁盘 IO 完成

3. **IO 调度延迟**
   
   - IO 请求在调度器中等待
   - 等待存储设备响应

**关键函数调用**：

```c
// 记录 IO 压力开始
psi_io_stall_start(unsigned long *flags)

// 记录 IO 压力结束
psi_io_stall_end(unsigned long *flags)

// 更新 IO PSI 统计
psi_account_io_stall()
```

### 2.4 IO PSI 统计的更新

**更新频率**：

- IO PSI 统计在任务状态变化时更新
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
    u64 total[PSI_IO];
};
```

---

## 第三章：IO PSI 接口使用

### 3.1 查看 IO PSI 数据

**IO 压力接口**：

```bash
# 查看 IO 压力
cat /proc/pressure/io

# 输出示例：
# some avg10=3.45 avg60=1.23 avg300=0.56 total=8765432
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**输出解读**：

- **some**：部分压力
  
  - `avg10=3.45`：过去 10 秒平均 3.45% 的时间有任务被 IO 阻塞
  - `avg60=1.23`：过去 60 秒平均 1.23% 的时间有任务被 IO 阻塞
  - `avg300=0.56`：过去 300 秒平均 0.56% 的时间有任务被 IO 阻塞
  - `total=8765432`：总 IO 阻塞时间（微秒）

- **full**：完全压力
  
  - 格式同 some
  - 如果全为 0，说明没有完全压力

**同时查看三种资源压力**：

```bash
# 查看所有资源压力
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

### 3.2 IO PSI 阈值监控

**设置 IO 压力阈值**：

PSI 支持设置阈值，当压力超过阈值时触发事件。

**接口文件**：

```bash
# 设置 some 压力阈值（10 秒窗口，阈值 5%）
echo "some 10000 5" > /proc/pressure/io

# 设置 full 压力阈值（60 秒窗口，阈值 10%）
echo "full 60000 10" > /proc/pressure/io
```

**参数说明**：

- 第一个参数：`some` 或 `full`
- 第二个参数：时间窗口（微秒）
- 第三个参数：压力阈值（百分比）

**清除阈值**：

```bash
# 清除阈值
echo "" > /proc/pressure/io
```

### 3.3 使用 IO PSI 进行监控

**实时监控脚本**：

```bash
#!/bin/bash
# monitor_io_psi.sh

while true; do
    clear
    echo "=== IO Pressure (PSI) Monitor ==="
    date
    echo ""

    # 读取 IO PSI 数据
    psi_data=$(cat /proc/pressure/io)

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
# io_psi_alert.sh

# 读取压力值
some_avg10=$(cat /proc/pressure/io | grep some | awk '{print $3}' | cut -d= -f2)

# 判断压力等级
if (( $(echo "$some_avg10 > 10" | bc -l) )); then
    echo "⚠️  HIGH IO PRESSURE: $some_avg10%"
elif (( $(echo "$some_avg10 > 5" | bc -l) )); then
    echo "⚠️  MEDIUM IO PRESSURE: $some_avg10%"
elif (( $(echo "$some_avg10 > 1" | bc -l) )); then
    echo "ℹ️  LOW IO PRESSURE: $some_avg10%"
else
    echo "✅ NORMAL: $some_avg10%"
fi
```

### 3.4 IO PSI 与系统监控工具集成

**综合监控脚本**：

```bash
#!/bin/bash
# monitor_all_psi.sh

while true; do
    clear
    echo "=== All Resource Pressure (PSI) ==="
    date
    echo ""

    echo "CPU Pressure:"
    cat /proc/pressure/cpu | grep some
    echo ""

    echo "Memory Pressure:"
    cat /proc/pressure/memory | grep some
    echo ""

    echo "IO Pressure:"
    cat /proc/pressure/io | grep some
    echo ""

    sleep 1
done
```

---

## 第四章：IO PSI 与 IO 阻塞的关系

### 4.1 IO PSI 与同步 IO

**同步 IO 的压力记录**：

当进程执行同步 IO 时，IO PSI 会记录阻塞时间：

```c
// 同步读操作路径（简化）
read()
    ↓
vfs_read()
    ↓
// PSI 开始记录
psi_io_stall_start(&flags)
    ↓
// 等待 IO 完成（可能阻塞）
wait_on_page_read(page)
    ↓
// PSI 结束记录
psi_io_stall_end(&flags)
```

**压力计算**：

```
同步 IO 压力 = (IO 等待时间 / 时间窗口) * 100%

如果 IO 等待 100ms，10 秒窗口：
压力 = (0.1 / 10) * 100% = 1%
```

**实际影响**：

- 同步 IO 阻塞进程执行
- IO PSI 量化这种阻塞的影响
- 高同步 IO 压力 → 系统响应变慢

### 4.2 IO PSI 与异步 IO

**异步 IO 的压力记录**：

异步 IO 不直接阻塞进程，但 IO 压力仍然存在：

```c
// 异步 IO 路径（简化）
aio_read()
    ↓
// 提交 IO 请求
submit_bio()
    ↓
// 不阻塞，但 IO 压力仍然记录
// 当 IO 完成时，压力结束
```

**异步 IO 压力的特点**：

- 不直接阻塞用户进程
- 但表示 IO 资源紧张
- 如果异步 IO 压力高，说明存储设备繁忙

### 4.3 IO PSI 与 IO 调度器

**IO 调度器延迟的压力记录**：

IO 请求在调度器中等待时，也会产生压力：

```
IO 请求提交
    ↓
进入 IO 调度器队列
    ↓
等待调度（可能延迟）
    ↓
PSI 记录等待时间
    ↓
IO 请求被调度执行
```

**调度器延迟的影响**：

- 调度器延迟增加 IO 压力
- 不同的调度器策略影响压力
- 队列深度影响压力

### 4.4 IO PSI 与存储设备性能

**存储设备性能与压力的关系**：

```
存储设备性能          IO PSI 压力
─────────────────────────────────
SSD（高性能）          0-1% (低压力)
机械硬盘（中性能）      1-5% (中压力)
慢速存储（低性能）      5-10% (高压力)
网络存储（不稳定）      10%+ (极高压力)
```

**慢存储设备的影响**：

- 慢存储设备导致高 IO 压力
- 高 IO 压力导致系统响应变慢
- 可能导致 ANR

---

## 第五章：IO PSI 源码分析

### 5.1 IO PSI 核心数据结构

**源码位置**：`kernel/sched/psi.c`、`include/linux/psi_types.h`

**关键数据结构**：

```c
// PSI 组（每个 cgroup 或系统级）
struct psi_group {
    // IO 压力统计
    struct psi_stats avg[3];  // avg10, avg60, avg300

    // 总阻塞时间
    u64 total[PSI_IO];

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

### 5.2 IO PSI 压力记录函数

**psi_io_stall_start**：

```c
// 开始记录 IO 压力
void psi_io_stall_start(unsigned long *flags)
{
    struct rq_flags rf;
    struct rq *rq;

    // 获取当前 CPU 的运行队列
    rq = this_rq_lock_irq(&rf);

    // 设置标志
    *flags = rq->flags;

    // 标记任务在等待 IO
    if (!(rq->flags & RQF_PSI)) {
        rq->flags |= RQF_PSI;
        current->in_iowait = 1;
        psi_task_change(current, 0, TASK_UNINTERRUPTIBLE);
    }

    rq_unlock_irqrestore(rq, &rf);
}
```

**psi_io_stall_end**：

```c
// 结束记录 IO 压力
void psi_io_stall_end(unsigned long *flags)
{
    struct rq_flags rf;
    struct rq *rq;

    rq = this_rq_lock_irq(&rf);

    // 清除标志
    if (*flags & RQF_PSI) {
        rq->flags &= ~RQF_PSI;
        current->in_iowait = 0;
        psi_task_change(current, TASK_UNINTERRUPTIBLE, 0);
    }

    rq_unlock_irqrestore(rq, &rf);
}
```

### 5.3 IO PSI 统计更新

**psi_account_io_stall**：

```c
// 更新 IO 压力统计
void psi_account_io_stall(struct task_struct *task, u64 now)
{
    struct psi_group *group;
    u64 delta;

    // 计算阻塞时间
    delta = now - task->iowait_start;

    // 更新统计
    group = task->psi;
    if (group) {
        group->total[PSI_IO] += delta;

        // 更新平均值
        psi_update_stats(group);
    }
}
```

### 5.4 IO PSI 接口实现

**/proc/pressure/io 读取**：

```c
// 读取 IO PSI 数据
static int psi_io_show(struct seq_file *m, struct psi_group *group)
{
    struct psi_stats *stats = &group->avg[PSI_IO];

    // 输出 some 压力
    seq_printf(m, "some avg10=%u.%02u avg60=%u.%02u avg300=%u.%02u total=%llu\n",
               stats->some[0] / 100, stats->some[0] % 100,
               stats->some[1] / 100, stats->some[1] % 100,
               stats->some[2] / 100, stats->some[2] % 100,
               group->total[PSI_IO]);

    // 输出 full 压力
    seq_printf(m, "full avg10=%u.%02u avg60=%u.%02u avg300=%u.%02u total=%llu\n",
               stats->full[0] / 100, stats->full[0] % 100,
               stats->full[1] / 100, stats->full[1] % 100,
               stats->full[2] / 100, stats->full[2] % 100,
               group->total[PSI_IO_FULL]);

    return 0;
}
```

---

## 第六章：IO PSI 与 ANR 的关系

### 6.1 ANR 与 IO 压力

**ANR 的触发条件**：

Android 系统中，ANR（Application Not Responding）可能由以下原因触发：

1. **主线程阻塞**
   
   - 主线程执行耗时操作
   - 主线程等待锁

2. **IO 阻塞**
   
   - 主线程执行同步 IO
   - 慢存储设备导致 IO 阻塞
   - 文件系统挂载问题

**IO PSI 与 ANR 的关联**：

```
IO 压力上升（IO PSI 上升）
    ↓
主线程因 IO 阻塞
    ↓
主线程无法响应
    ↓
触发 ANR
```

### 6.2 使用 IO PSI 分析 ANR

**ANR 分析流程**：

```bash
# 1. 查看 ANR 发生时的 IO PSI 数据
# （需要从日志或监控系统获取）

# 2. 分析 IO 压力
cat /proc/pressure/io

# 3. 如果 IO PSI 压力高，说明 ANR 可能与 IO 阻塞相关
```

**ANR 日志中的 IO PSI**：

```
ANR in com.example.app
Reason: Executing service com.example.app/.MyService
CPU usage from 0ms to 5000ms:
  ...
IO pressure (PSI):
  some avg10=12.34 avg60=5.67 avg300=2.34
  full avg10=1.23 avg60=0.50 avg300=0.10
```

**分析要点**：

- **some 压力高**：部分任务被 IO 阻塞，可能导致延迟
- **full 压力高**：系统几乎无法响应，很可能导致 ANR
- **压力持续时间**：avg300 反映长期趋势

### 6.3 IO PSI 阈值与 ANR 预防

**设置 IO PSI 阈值预警**：

```bash
# 设置阈值，当 IO 压力超过 10% 时触发事件
echo "some 10000 10" > /proc/pressure/io

# 系统可以监听这个事件，提前采取措施
# 例如：优化 IO 操作、切换到异步 IO 等
```

**Android 中的 IO PSI 使用**：

Android 系统可以使用 IO PSI 来：

- 预测 ANR 风险
- 优化 IO 操作策略
- 调整 IO 调度器参数

---

## 第七章：IO PSI 监控与诊断

### 7.1 IO PSI 监控工具

**基础监控**：

```bash
# 实时监控
watch -n 1 'cat /proc/pressure/io'

# 持续记录
while true; do
    echo "$(date): $(cat /proc/pressure/io)"
    sleep 1
done >> io_psi_log.txt
```

**压力趋势分析**：

```bash
#!/bin/bash
# io_psi_trend.sh

prev_avg10=0
while true; do
    avg10=$(cat /proc/pressure/io | grep some | awk '{print $3}' | cut -d= -f2)

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

### 7.2 IO PSI 与其他指标关联

**IO PSI vs IO 使用率**：

```bash
#!/bin/bash
# io_psi_vs_usage.sh

while true; do
    # IO PSI 压力
    psi=$(cat /proc/pressure/io | grep some | awk '{print $3}' | cut -d= -f2)

    # IO 使用率（从 iostat）
    util=$(iostat -x 1 1 | grep -A 1 "Device" | tail -1 | awk '{print $NF}')

    # IO 延迟（从 iostat）
    await=$(iostat -x 1 1 | grep -A 1 "Device" | tail -1 | awk '{print $(NF-2)}')

    echo "IO PSI: ${psi}% | IO Util: ${util}% | IO Await: ${await}ms"

    sleep 1
done
```

**IO PSI vs IO 统计**：

```bash
# 查看 IO 统计
cat /proc/diskstats

# 如果 IO 统计显示高 IO，IO PSI 压力通常也高
```

### 7.3 IO PSI 问题诊断

**问题 1：IO PSI 压力持续高**

**症状**：

- IO PSI some 压力 > 10%
- 持续时间长（avg300 也高）

**可能原因**：

1. 慢存储设备
2. IO 竞争激烈
3. IO 调度器配置不当
4. 存储设备故障

**诊断步骤**：

```bash
# 1. 查看 IO 使用情况
iostat -x 1

# 2. 查看 IO 延迟
iostat -x 1 | grep await

# 3. 查看 IO 队列深度
iostat -x 1 | grep avgqu-sz

# 4. 查看进程 IO
iotop

# 5. 检查存储设备健康
smartctl -a /dev/sda
```

**问题 2：IO PSI full 压力出现**

**症状**：

- IO PSI full 压力 > 0%
- 系统几乎无法响应

**严重性**：

- ⚠️ 非常严重
- 可能导致 ANR、系统卡死

**诊断**：

```bash
# 1. 立即查看 IO PSI
cat /proc/pressure/io

# 2. 查看 IO 状态
iostat -x 1

# 3. 查看进程状态
ps aux | grep -E "D|Z"  # 查找阻塞进程

# 4. 检查存储设备
dmesg | grep -i "io error"
```

**问题 3：IO PSI 压力突然上升**

**症状**：

- IO PSI 压力短时间内大幅上升
- 系统响应变慢

**可能原因**：

1. 大量 IO 操作启动
2. 存储设备性能下降
3. IO 密集型应用启动
4. 存储设备故障

**诊断**：

```bash
# 1. 查看最近启动的进程
ps aux --sort=start_time | tail -10

# 2. 查看 IO 增长快的进程
iotop -o

# 3. 查看 IO 操作类型
iostat -x 1 | grep -E "r/s|w/s"

# 4. 检查存储设备
dmesg | tail -50
```

---

## 第八章：IO PSI 实际应用场景

### 场景 1：Android 系统 ANR 分析

**问题**：

应用频繁 ANR，需要分析是否与 IO 阻塞相关。

**分析步骤**：

```bash
# 1. 获取 ANR 发生时的 IO PSI 数据
# （从系统日志或监控系统）

# 2. 分析 IO PSI 压力
# 如果 some avg10 > 10%，说明 IO 压力高

# 3. 关联分析
# - IO PSI 压力高 + 同步 IO → 主线程 IO 阻塞
# - IO PSI 压力高 + 慢存储 → 存储设备问题
# - IO PSI 压力高 + IO 竞争 → 多个进程竞争 IO
```

**解决方案**：

- 如果主线程 IO 阻塞：使用异步 IO 或后台线程
- 如果存储设备慢：优化存储或使用更快的存储
- 如果 IO 竞争：优化 IO 操作，减少竞争

### 场景 2：服务器性能优化

**问题**：

服务器响应变慢，需要找出瓶颈。

**使用 IO PSI 分析**：

```bash
# 1. 同时监控三种资源压力
watch -n 1 'cat /proc/pressure/{cpu,memory,io}'

# 2. 找出压力最高的资源
# 如果 IO 压力最高，重点优化 IO

# 3. 分析 IO 压力趋势
# 如果压力持续上升，需要采取措施
```

**优化策略**：

- **IO 压力高**：优化 IO 操作、使用 SSD、调整 IO 调度器
- **CPU 压力高**：优化算法、增加 CPU
- **内存压力高**：增加内存、优化应用

### 场景 3：存储设备性能评估

**问题**：

评估不同存储设备的性能差异。

**使用 IO PSI 监控**：

```bash
# 1. 在不同存储设备上执行相同 IO 操作
# 2. 监控 IO PSI 压力
# 3. 对比压力值

# SSD
cat /proc/pressure/io
# some avg10=0.5%  # 低压力

# 机械硬盘
cat /proc/pressure/io
# some avg10=5.0%  # 中等压力

# 网络存储
cat /proc/pressure/io
# some avg10=15.0%  # 高压力
```

### 场景 4：预测性 IO 管理

**使用 IO PSI 预测 IO 问题**：

```bash
#!/bin/bash
# predictive_io.sh

# 监控 IO PSI 趋势
prev_avg10=0
while true; do
    avg10=$(cat /proc/pressure/io | grep some | awk '{print $3}' | cut -d= -f2)

    # 如果压力持续上升
    if (( $(echo "$avg10 > $prev_avg10 + 2" | bc -l) )); then
        echo "⚠️  WARNING: IO pressure rising: $avg10%"

        # 预测：如果继续上升，可能触发 ANR
        if (( $(echo "$avg10 > 15" | bc -l) )); then
            echo "🚨 ALERT: High IO pressure, ANR risk!"
            # 可以触发预防措施
        fi
    fi

    prev_avg10=$avg10
    sleep 5
done
```

---

## 第九章：IO PSI 调优

### 9.1 IO PSI 阈值配置

**设置合理的阈值**：

```bash
# 根据系统特点设置阈值

# 服务器环境（对延迟敏感）
echo "some 10000 5" > /proc/pressure/io   # 10秒窗口，5%阈值
echo "full 10000 1" > /proc/pressure/io   # 10秒窗口，1%阈值

# 桌面环境（可以容忍一定延迟）
echo "some 60000 10" > /proc/pressure/io  # 60秒窗口，10%阈值
echo "full 60000 5" > /proc/pressure/io   # 60秒窗口，5%阈值
```

**阈值选择原则**：

- **时间窗口**：越短越敏感，但可能误报
- **压力阈值**：越低越敏感，但可能频繁触发
- **根据实际需求调整**

### 9.2 IO PSI 与 IO 参数调优

**降低 IO PSI 压力的方法**：

1. **使用更快的存储设备**
   
   - SSD 比机械硬盘快得多
   - 减少 IO 等待时间

2. **优化 IO 操作**
   
   - 使用异步 IO
   - 批量 IO 操作
   - 减少不必要的 IO

3. **调整 IO 调度器**
   
   ```bash
   # 查看当前调度器
   cat /sys/block/sda/queue/scheduler
   
   # 切换到合适的调度器
   echo mq-deadline > /sys/block/sda/queue/scheduler
   ```

4. **优化应用**
   
   - 减少同步 IO
   - 使用缓存减少 IO
   - 优化 IO 模式

### 9.3 IO PSI 监控系统集成

**集成到监控系统**：

```bash
# Prometheus exporter 示例（伪代码）
while true; do
    psi_data=$(cat /proc/pressure/io)
    some_avg10=$(echo "$psi_data" | grep some | awk '{print $3}' | cut -d= -f2)

    # 导出到 Prometheus
    echo "io_pressure_some_avg10 $some_avg10" | \
        curl -X POST http://prometheus:9091/metrics/job/psi

    sleep 10
done
```

---

## 第十章：IO PSI 与其他监控机制对比

### 10.1 IO PSI vs 传统 IO 指标

**传统指标（iostat）**：

```
优点：
- 详细的 IO 使用信息
- 历史数据丰富
- 设备级别的统计

缺点：
- 无法量化对性能的影响
- 无法预测问题
- 无法直接关联到任务阻塞
```

**IO PSI**：

```
优点：
- 量化性能影响
- 预测性问题
- 直接关联任务阻塞
- 统一接口

缺点：
- 需要内核 4.20+
- 数据相对简单
- 无法看到具体设备
```

**结合使用**：

- IO PSI 用于快速判断压力
- iostat 用于详细分析设备

### 10.2 IO PSI vs IO 延迟

**IO 延迟（iostat await）**：

- 测量 IO 请求的平均等待时间
- 设备级别的指标
- 不直接反映对任务的影响

**IO PSI**：

- 测量任务因 IO 阻塞的时间
- 系统级别的指标
- 直接反映对任务的影响

**互补关系**：

- IO 延迟高 → IO PSI 压力通常也高
- IO PSI 可以量化延迟对系统的影响

### 10.3 IO PSI vs IO 使用率

**IO 使用率（iostat %util）**：

- 测量存储设备的繁忙程度
- 0-100% 的范围
- 不反映对任务的影响

**IO PSI**：

- 测量任务阻塞时间
- 直接反映对任务的影响
- 可以预测性能问题

**关系**：

- IO 使用率高 → IO PSI 压力可能高
- 但 IO PSI 更直接反映性能影响

---

## 附录

### A. IO PSI 相关内核参数

| 参数        | 路径                     | 说明      |
| --------- | ---------------------- | ------- |
| IO PSI 接口 | /proc/pressure/io      | IO 压力接口 |
| IO PSI 阈值 | /proc/pressure/io (写入) | 设置压力阈值  |

### B. IO PSI 相关源码文件

- `kernel/sched/psi.c`：PSI 核心实现
- `include/linux/psi_types.h`：PSI 类型定义
- `include/linux/sched/psi.h`：PSI 接口定义
- `block/`：Block I/O 中的 PSI 调用
- `fs/`：文件系统中的 PSI 调用

### C. IO PSI 诊断命令速查

```bash
# 查看 IO 压力
cat /proc/pressure/io

# 查看所有资源压力
cat /proc/pressure/{cpu,memory,io}

# 设置阈值
echo "some 10000 5" > /proc/pressure/io

# 清除阈值
echo "" > /proc/pressure/io

# 结合 iostat 分析
iostat -x 1
cat /proc/pressure/io
```

### D. IO PSI 压力等级参考

| 压力值    | 等级  | 影响         | 建议   |
| ------ | --- | ---------- | ---- |
| 0-1%   | 正常  | 无影响        | 无需处理 |
| 1-5%   | 低   | 轻微延迟       | 监控   |
| 5-10%  | 中   | 明显延迟       | 优化   |
| 10-20% | 高   | 严重延迟       | 立即处理 |
| >20%   | 极高  | 可能 ANR/OOM | 紧急处理 |

### E. IO PSI 学习资源

- Linux 内核文档：`Documentation/accounting/psi.rst`
- PSI 论文：Pressure Stall Information
- 内核源码：`kernel/sched/psi.c`
- Block I/O 文档：`Documentation/block/`

---

## 📝 学习检查点

完成 IO PSI 学习后，您应该能够：

- [ ] 理解 IO PSI 的概念和作用
- [ ] 理解 IO PSI 的工作原理和计算方式
- [ ] 掌握 IO PSI 接口的使用方法
- [ ] 能够使用 IO PSI 诊断 IO 压力问题
- [ ] 理解 IO PSI 与 ANR 的关系
- [ ] 能够分析 IO PSI 数据并找出问题
- [ ] 能够阅读 IO PSI 相关源码
- [ ] 能够将 IO PSI 集成到监控系统

---

**下一步**：完成 IO PSI 理论学习后，可以：

1. 在实际系统中观察 IO PSI 数据
2. 分析 ANR 问题中的 IO PSI 数据
3. 将 IO PSI 集成到监控和告警系统
4. 结合其他 IO 机制，形成完整的 IO 问题分析能力
