# ANR 实战案例分析：IO Wait 高导致的 SystemUI ANR

## 📋 案例概述

**ANR 时间**：2025-12-03 10:07:46.658  
**ANR 进程**：com.android.systemui (PID: 30536)  
**ANR 原因**：Input dispatching timed out (NavigationBar0 无响应，等待 5002ms)  
**核心问题**：IO Wait 极高（21%），IO PSI 压力极高（some 78.08%, full 52.78%）

---

## 第一章：日志关键信息提取

### 1.1 ANR 基本信息

```
ANR in com.android.systemui
PID: 30536
Reason: Input dispatching timed out (9598b5 NavigationBar0 (server) is not responding. Waited 5002ms for MotionEvent)
Load: 29.23 / 21.68 / 16.36
```

**关键点**：
- SystemUI 主线程无法处理输入事件
- 系统负载高（1分钟平均 29.23）
- 等待超时 5002ms（超过 5 秒）

### 1.2 PSI 压力数据（核心指标）

#### Memory Pressure
```
some avg10=6.33% avg60=6.32% avg300=5.83%
full avg10=4.59% avg60=4.50% avg300=4.06%
```

**分析**：
- 内存压力中等（some 6.33%）
- full 压力 4.59%，说明有部分任务因内存阻塞
- 压力持续（avg60 和 avg300 相近）

#### CPU Pressure
```
some avg10=11.76% avg60=9.20% avg300=8.94%
full avg10=0.00%
```

**分析**：
- CPU 压力中等（some 11.76%）
- full 压力为 0，说明 CPU 不是主要瓶颈

#### IO Pressure ⚠️ **关键问题**
```
some avg10=78.08% avg60=72.73% avg300=35.96%
full avg10=52.78% avg60=49.30% avg300=21.79%
```

**分析**：
- ⚠️ **IO some 压力 78.08%**：过去 10 秒内，78% 的时间有任务被 IO 阻塞
- ⚠️ **IO full 压力 52.78%**：过去 10 秒内，52% 的时间所有任务都被 IO 阻塞
- 压力极高且持续（avg60 72.73%）
- 长期趋势也在上升（avg300 35.96%）

**结论**：IO 压力是导致 ANR 的**主要原因**。

### 1.3 CPU 使用情况

```
87% TOTAL: 29% user + 33% kernel + 21% iowait + 2% irq + 1.1% softirq
```

**关键发现**：
- **iowait 21%**：CPU 在等待 IO 完成的时间占比 21%
- kernel 占用 33%：内核处理 IO 相关操作
- user 29%：用户态 CPU 使用正常

**iowait 21% 的含义**：
- CPU 空闲但等待 IO 完成的时间占 21%
- 说明 IO 设备是瓶颈，CPU 在等待 IO
- 这是 IO 阻塞的典型表现

**iowait 阈值判断标准**：

iowait 的严重程度判断需要结合多个因素，不能只看单一数值：

**1. 基础阈值（参考值）**：

| iowait 值 | 等级 | 判断标准 | 建议 |
|-----------|------|---------|------|
| 0-5% | 正常 | 正常范围 | 无需关注 |
| 5-10% | 轻微 | 需要关注 | 监控趋势 |
| 10-15% | 中等 | 可能有问题 | 需要分析 |
| 15-20% | 严重 | 明显 IO 阻塞 | 需要处理 |
| >20% | 极严重 | 严重 IO 阻塞 | 紧急处理 |

**2. 结合其他指标判断**：

**关键原则**：iowait 需要与 IO PSI 结合判断，不能孤立看。

- **iowait > 10% + IO PSI some > 20%**：确认 IO 阻塞严重
- **iowait > 15% + IO PSI full > 5%**：确认 IO 阻塞极严重
- **iowait > 20%**：无论其他指标如何，都是严重问题

**本案例分析**：
- iowait 21%：已经超过 20%，属于**极严重**级别
- 结合 IO PSI some 78.08%，full 52.78%：确认 IO 阻塞是主要问题
- **结论**：iowait 21% 已经足够说明 IO 阻塞严重

**3. 系统类型差异**：

- **服务器系统**：iowait > 10% 就需要关注（对延迟敏感）
- **桌面系统**：iowait > 15% 需要关注
- **移动设备（Android）**：iowait > 12% 就需要关注（资源受限，更容易触发 ANR）

**4. 持续时间**：

- **短暂峰值**（< 1 秒）：可能是正常波动
- **持续高值**（> 5 秒）：需要处理
- **长期高值**（> 30 秒）：严重问题

**本案例**：从 IO PSI 的 avg60=72.73% 可以看出，IO 压力持续了至少 60 秒，说明不是短暂峰值。

**5. 实际影响**：

- **是否导致 ANR**：是（本案例）
- **是否导致系统卡顿**：是
- **是否影响用户体验**：是

**综合判断**：

对于本案例：
- ✅ iowait 21% > 20%（极严重阈值）
- ✅ IO PSI some 78.08% >> 20%（极高）
- ✅ IO PSI full 52.78% >> 5%（极高）
- ✅ 持续时间长（avg60 72.73%）
- ✅ 实际影响：触发 ANR

**结论**：iowait 21% 已经明确表明 IO 阻塞严重，不需要等到更高值。结合 IO PSI 数据，可以100%确认 IO 是主要问题。

### 1.4 高 CPU 占用进程分析

#### 进程 13442/potaa.zzz.world
```
94% 13442/potaa.zzz.world: 66% user + 28% kernel
faults: 38706 minor 41261 major
```

**分析**：
- CPU 占用极高（94%）
- **major faults 41261**：大量页面错误，说明内存不足，频繁从 swap 读取
- 这个进程可能是问题的**触发者**

#### kswapd0
```
42% 103/kswapd0: 0% user + 42% kernel
```

**分析**：
- kswapd0 CPU 占用 42%，说明内存回收压力大
- 与大量 major faults 对应
- 内存不足导致频繁 swap，加剧 IO 压力

#### system_server
```
51% 25707/system_server: 26% user + 24% kernel
faults: 22666 minor 26154 major
```

**分析**：
- system_server 也有大量 major faults（26154）
- 说明系统服务也受内存压力影响

### 1.5 IO 相关线程分析

**大量 blk_crypto_wq 工作线程**：
```
6.5% 6453/kworker/u17:4-blk_crypto_wq
7.1% 20248/kworker/u17:5-blk_crypto_wq
7.5% 20723/kworker/u17:6-blk_crypto_wq
7.6% 21759/kworker/u17:7-blk_crypto_wq
4.4% 10768/kworker/u17:1-blk_crypto_wq
```

**分析**：
- 多个 blk_crypto_wq 线程在工作
- blk_crypto 是块设备加密相关的工作队列
- 说明有**大量加密 IO 操作**在进行
- 加密 IO 会显著增加 IO 延迟

**其他 IO 相关线程**：
```
51% 25495/kworker/u16:4+events_unbound
11% 502/kshrink_slabd
```

---

## 第二章：根因分析

### 2.1 问题链条梳理

**完整的问题链条**：

```
1. 进程 13442 (potaa.zzz.world) 大量分配内存
   ↓
2. 内存不足，触发大量 major faults (41261)
   ↓
3. 需要从 swap 读取数据，产生大量 IO 请求
   ↓
4. 同时有大量加密 IO 操作（blk_crypto_wq）
   ↓
5. 存储设备 IO 队列饱和，IO 延迟急剧增加
   ↓
6. IO PSI 压力飙升（some 78%, full 52%）
   ↓
7. CPU iowait 达到 21%（CPU 等待 IO）
   ↓
8. SystemUI 主线程执行 IO 操作时被阻塞
   ↓
9. 无法处理输入事件，触发 ANR
```

### 2.2 根因确认

#### 根因 1：内存不足导致大量 Swap IO ⚠️ **主要原因**

**证据**：
1. **大量 major faults**：
   - 进程 13442: 41261 major faults
   - system_server: 26154 major faults
   - SystemUI: 9499 major faults

2. **kswapd0 高 CPU**：
   - kswapd0 占用 42% CPU
   - 说明内存回收压力大

3. **内存 PSI 压力**：
   - Memory some: 6.33%
   - Memory full: 4.59%
   - 说明有内存压力

4. **IO PSI 与内存压力的关联**：
   - 内存不足 → 频繁 swap → 大量 IO → IO PSI 压力高

**结论**：内存不足是根本原因，导致大量 swap IO，进而引发 IO 阻塞。

#### 根因 2：加密 IO 操作加剧 IO 压力 ⚠️ **次要原因**

**证据**：
1. **大量 blk_crypto_wq 线程**：
   - 多个 kworker 线程在处理加密 IO
   - 加密 IO 会增加 IO 延迟

2. **加密 IO 的特点**：
   - 需要额外的 CPU 计算（加密/解密）
   - 增加 IO 延迟
   - 在 IO 压力高时，影响更明显

**结论**：加密 IO 操作在 IO 压力高时，进一步加剧了 IO 延迟。

#### 根因 3：IO 设备性能瓶颈 ⚠️ **表现**

**证据**：
1. **IO PSI 压力极高**：
   - some 78.08%，full 52.78%
   - 说明存储设备无法及时处理 IO 请求

2. **iowait 21%**：
   - CPU 在等待 IO 完成
   - 说明 IO 设备是瓶颈

3. **系统负载高**：
   - Load: 29.23 / 21.68 / 16.36
   - 高负载主要由 IO 等待导致

**结论**：存储设备（可能是 eMMC 或较慢的存储）无法承受当前的 IO 负载。

### 2.3 深度质疑：Memory PSI 不高但 kswapd 活跃且 IO 压力极高？

这是一个非常关键的问题，需要深入理解几个机制的本质。

#### 质疑 1：kswapd 一定在内存压力下才工作吗？

**答案：不一定。kswapd 的触发条件更复杂。**

**kswapd 的触发条件**（源码：`mm/vmscan.c`）：

1. **主要触发条件：Zone 水位线**
   ```c
   // kswapd 被唤醒的条件
   static bool kswapd_needs_to_run(pg_data_t *pgdat)
   {
       // 检查是否有 zone 低于低水位
       for (zone = pgdat->node_zones; zone; zone = zone->next) {
           if (zone_watermark_ok(zone, order, ...))
               return true;  // 需要运行
       }
       return false;
   }
   ```
   - 当某个 zone 的空闲内存低于 low watermark 时触发
   - **即使整体内存充足，某个 zone 可能压力大**

2. **其他触发条件**：
   - **Direct Reclaim 压力大时**：即使水位线正常，如果 direct reclaim 频繁，也会唤醒 kswapd
   - **内存碎片化**：即使总内存充足，但无法分配连续大块内存时
   - **文件系统触发**：某些文件系统操作可能触发内存回收
   - **特定内存分配请求**：某些特殊的分配请求（如 DMA）可能触发

**关键点**：
- kswapd 活跃 ≠ 整机内存压力大
- 可能是**局部 zone 压力**或**特定场景触发**

#### 质疑 2：Memory PSI 能否真实反映整机内存压力？

**答案：不能完全反映。Memory PSI 有局限性。**

**Memory PSI 的测量范围**：

```c
// Memory PSI 主要在这些场景记录：
1. Direct Reclaim（直接回收）
   - 进程分配内存失败时触发
   - 同步阻塞进程，记录阻塞时间
   - ✅ 这是 Memory PSI 的主要来源

2. kswapd 的工作
   - kswapd 是后台线程，不直接阻塞用户进程
   - ❌ Memory PSI 几乎不记录 kswapd 的工作时间
   - 只有 kswapd 被唤醒的瞬间可能记录少量压力

3. OOM 相关
   - OOM Killer 触发前的压力积累
   - 但通常发生在 Memory PSI 已经很高之后
```

**Memory PSI 的局限性**：

1. **不测量 kswapd 的工作**：
   - kswapd 是后台异步工作
   - 不阻塞用户进程
   - Memory PSI 无法反映 kswapd 的工作量

2. **不测量内存使用率**：
   - Memory PSI 只测量"阻塞时间"
   - 不测量"内存使用率"
   - 内存可能已经很高，但如果没有 direct reclaim，PSI 可能不高

3. **不测量 swap 活动**：
   - swap 操作本身不直接产生 Memory PSI
   - 但 swap 会产生 IO，导致 IO PSI 高

4. **Zone 级别的差异**：
   - Memory PSI 是系统级别的
   - 某个 zone 可能压力大，但系统级别 PSI 可能不高

**本案例的情况**：

- Memory PSI some 6.33%：说明 direct reclaim 不多（6.33% 的时间有进程被内存阻塞）
- 但 kswapd CPU 42%：说明 kswapd 在**持续工作**
- **结论**：kswapd 工作得好，避免了 direct reclaim，所以 Memory PSI 不高

#### 质疑 3：f2fs.h 中漏合的 patch 可能是什么？

**f2fs.h 文件简介**：

`fs/f2fs/f2fs.h` 是 F2FS（Flash-Friendly File System）文件系统的核心头文件。

**F2FS 与内存管理的关系**：

F2FS 是 Android 设备常用的文件系统，与内存管理有密切关系：

1. **Page Cache 使用**：
   - F2FS 使用 Page Cache 缓存文件数据
   - 文件操作会影响内存使用

2. **内存回收触发**：
   - F2FS 的某些操作可能触发内存回收
   - 例如：大量文件写入 → 产生脏页 → 可能触发回收

3. **可能的 patch 内容**（推测）：
   - 与内存压力相关的优化
   - 与 kswapd 唤醒相关的改进
   - 与 IO 阻塞相关的修复
   - 与脏页写回相关的优化

**需要确认的信息**：

1. **patch 的具体内容**：
   - 是什么 patch？
   - 修复了什么问题？
   - 与内存压力或 IO 相关吗？

2. **patch 的影响**：
   - 是否会影响 kswapd 的触发？
   - 是否会影响 Memory PSI 的测量？
   - 是否会影响 IO 行为？

#### 重新审视本案例

**基于质疑的重新分析**：

1. **Memory PSI 6.33% 的含义**：
   - ✅ 确实说明 direct reclaim 不多
   - ❌ 但不能说明整机内存压力不大
   - ❌ 不能说明 kswapd 不应该工作

2. **kswapd CPU 42% 的可能原因**：
   - ✅ 可能是内存压力（某个 zone 压力大）
   - ✅ 可能是内存碎片化
   - ✅ 可能是文件系统触发（F2FS？）
   - ✅ 可能是其他因素（需要更多信息）

3. **IO PSI 78% 的可能原因**：
   - ✅ 可能是 kswapd 的 swap IO
   - ✅ 可能是其他 IO 操作（加密 IO、文件 IO）
   - ✅ 可能是存储设备性能问题
   - ❌ 不能确定一定是 kswapd 导致的

**需要更多信息才能确定**：

1. **Zone 级别的内存状态**：
   ```bash
   cat /proc/zoneinfo
   # 查看各个 zone 的水位线和空闲内存
   ```

2. **内存使用详情**：
   ```bash
   cat /proc/meminfo
   # 查看详细的内存使用情况
   ```

3. **Swap 使用情况**：
   ```bash
   free -h
   cat /proc/swaps
   # 查看 swap 的使用
   ```

4. **IO 统计信息**：
   ```bash
   cat /proc/diskstats
   iostat -x 1
   # 查看 IO 的详细统计
   ```

5. **f2fs.h 的 patch 内容**：
   - 需要知道具体是什么 patch
   - 才能判断是否相关

#### 更谨慎的结论

**基于现有日志，我们只能确定**：

1. ✅ **IO PSI 极高**（78% some, 52% full）：这是事实
2. ✅ **iowait 高**（21%）：这是事实
3. ✅ **kswapd 活跃**（42% CPU）：这是事实
4. ✅ **Memory PSI 中等**（6.33% some）：这是事实
5. ✅ **大量 major faults**：这是事实

**但我们不能确定**：

1. ❌ kswapd 活跃是否一定由内存压力导致
2. ❌ IO PSI 高是否一定由 kswapd 的 swap IO 导致
3. ❌ Memory PSI 不高是否说明内存没问题
4. ❌ f2fs.h 的 patch 是否相关

**需要更多信息**：

- Zone 级别的内存状态
- Swap 使用情况
- IO 统计详情
- f2fs.h patch 的具体内容
- 系统配置参数（min_free_kbytes、swappiness 等）

**建议**：在获得更多信息之前，保持开放态度，不要强行绑定因果关系。

### 2.4 为什么 SystemUI 会 ANR？

**SystemUI ANR 的直接原因**：

1. **SystemUI 主线程需要执行 IO 操作**：
   - 可能是读取配置文件
   - 可能是写入日志
   - 可能是访问资源文件

2. **IO 阻塞导致主线程无法响应**：
   - 主线程执行同步 IO 操作
   - 由于 IO 压力极高（78% some, 52% full），IO 操作被阻塞
   - 主线程无法处理输入事件

3. **Input 事件超时**：
   - NavigationBar0 等待 5002ms 无响应
   - 触发 ANR

**关键点**：SystemUI 本身不是问题的根源，而是**受害者**。真正的问题是系统级的 IO 阻塞。

---

## 第三章：问题验证

### 3.1 需要确认的信息

为了更准确地分析，需要以下信息：

1. **存储设备信息**：
   ```bash
   # 存储设备类型和性能
   cat /proc/partitions
   cat /sys/block/*/queue/rotational  # 是否为机械硬盘
   ```

2. **Swap 使用情况**：
   ```bash
   # Swap 使用情况
   free -h
   cat /proc/swaps
   ```

3. **IO 统计信息**：
   ```bash
   # IO 统计
   cat /proc/diskstats
   iostat -x 1
   ```

4. **进程 13442 的详细信息**：
   ```bash
   # 这是什么进程？
   ps aux | grep 13442
   cat /proc/13442/cmdline
   ```

5. **内存使用情况**：
   ```bash
   # 内存使用详情
   cat /proc/meminfo
   ```

### 3.2 基于现有日志的推断

**推断 1：存储设备可能是 eMMC 或较慢的存储**

**理由**：
- IO PSI 压力极高（78%）
- 在 Android 设备上，eMMC 性能有限
- 大量 swap IO + 加密 IO 会压垮慢速存储

**推断 2：Swap 空间可能不足或性能差**

**理由**：
- 大量 major faults 说明频繁 swap
- 如果 swap 在慢速存储上，会加剧 IO 压力

**推断 3：进程 13442 可能是内存泄漏或异常进程**

**理由**：
- CPU 占用 94%
- major faults 41261（极高）
- 可能是异常进程或内存泄漏

---

## 第四章：解决方案

### 4.1 短期解决方案（应急）

#### 方案 1：杀死异常进程

```bash
# 如果进程 13442 是异常进程，可以杀死
kill -9 13442

# 或者通过系统机制限制
echo -1000 > /proc/13442/oom_score_adj
```

**效果**：立即释放内存，减少 swap IO。

#### 方案 2：清理内存

```bash
# 触发内存回收
echo 3 > /proc/sys/vm/drop_caches

# 或者通过 LowMemoryKiller
# 系统会自动杀死低优先级进程
```

**效果**：释放内存，减少内存压力。

### 4.2 中期解决方案（优化）

#### 方案 1：优化内存使用

1. **增加物理内存**（如果可能）
2. **优化应用内存使用**：
   - 减少内存泄漏
   - 优化内存分配策略
   - 及时释放不需要的内存

3. **调整内存参数**：
   ```bash
   # 增加 min_free_kbytes，提前触发回收
   echo 131072 > /proc/sys/vm/min_free_kbytes
   
   # 降低 swappiness，减少 swap 使用
   echo 10 > /proc/sys/vm/swappiness
   ```

#### 方案 2：优化 IO 性能

1. **使用更快的存储设备**（如果可能）
2. **优化 IO 调度器**：
   ```bash
   # 查看当前调度器
   cat /sys/block/*/queue/scheduler
   
   # 切换到 mq-deadline（适合大多数场景）
   echo mq-deadline > /sys/block/sda/queue/scheduler
   ```

3. **减少加密 IO**（如果可能）：
   - 评估是否真的需要全盘加密
   - 考虑使用文件级加密替代

#### 方案 3：优化 SystemUI

1. **避免主线程 IO**：
   - 所有 IO 操作移到后台线程
   - 使用异步 IO

2. **使用缓存**：
   - 缓存配置文件
   - 减少文件 IO

### 4.3 长期解决方案（架构）

#### 方案 1：监控和预警

1. **设置 PSI 阈值预警**：
   ```bash
   # 当 IO PSI 超过 20% 时预警
   echo "some 10000 20" > /proc/pressure/io
   ```

2. **监控内存压力**：
   - 监控 major faults
   - 监控 swap 使用
   - 提前预警

#### 方案 2：系统级优化

1. **内存管理优化**：
   - 优化 LowMemoryKiller 阈值
   - 优化内存回收策略

2. **IO 管理优化**：
   - 优化 IO 调度器参数
   - 优化预读和写回参数

---

## 第五章：预防措施

### 5.1 监控指标

**关键监控指标**：

1. **IO PSI**：
   - some > 10%：需要关注
   - some > 20%：需要处理
   - full > 5%：严重问题

2. **iowait**（需要结合 IO PSI 判断）：
   - **5-10%**：需要关注，监控趋势
   - **10-15%**：可能有问题，需要分析
   - **15-20%**：明显 IO 阻塞，需要处理
   - **>20%**：严重 IO 阻塞，紧急处理
   - **>12% + IO PSI some > 20%**：确认 IO 阻塞严重（移动设备）
   - **>15% + IO PSI full > 5%**：确认 IO 阻塞极严重

3. **major faults**：
   - 单个进程 > 1000/秒：异常
   - 系统总 major faults 持续高：内存不足

4. **内存 PSI**：
   - some > 5%：需要关注
   - full > 2%：需要处理

**iowait 判断的注意事项**：

1. **不能孤立看 iowait**：
   - iowait 高可能是正常现象（如果 CPU 使用率低）
   - 需要结合 CPU 使用率和 IO PSI 判断

2. **移动设备更敏感**：
   - Android 设备资源受限
   - iowait > 12% 就需要关注
   - 因为更容易触发 ANR

3. **持续时间很重要**：
   - 短暂峰值可能是正常波动
   - 持续高值才是问题

4. **实际影响是最终判断**：
   - 如果已经触发 ANR，说明问题严重
   - 不需要纠结具体阈值

### 5.2 告警规则

```bash
#!/bin/bash
# anr_prevention_monitor.sh

# 监控 IO PSI
io_some=$(cat /proc/pressure/io | grep some | awk '{print $3}' | cut -d= -f2)
io_full=$(cat /proc/pressure/io | grep full | awk '{print $3}' | cut -d= -f2)

# 监控 iowait
iowait=$(top -bn1 | grep "Cpu(s)" | awk '{print $10}' | cut -d'%' -f1)

# 告警
if (( $(echo "$io_some > 20" | bc -l) )); then
    echo "⚠️  ALERT: High IO pressure: $io_some%"
    # 触发预防措施
fi

if (( $(echo "$io_full > 5" | bc -l) )); then
    echo "🚨 CRITICAL: Full IO pressure: $io_full%"
    # 紧急处理
fi

if (( $(echo "$iowait > 10" | bc -l) )); then
    echo "⚠️  ALERT: High iowait: $iowait%"
fi
```

---

## 第六章：经验总结

### 6.1 关键发现

1. **IO PSI 是诊断 IO 问题的关键指标**
   - 78% 的 some 压力和 52% 的 full 压力明确指向 IO 问题
   - 比传统的 iostat 更能反映对系统的影响

2. **内存不足会引发 IO 问题**
   - 大量 swap IO 会压垮存储设备
   - 需要同时关注内存和 IO

3. **加密 IO 会加剧 IO 压力**
   - 在 IO 压力高时，加密 IO 的影响更明显
   - 需要权衡安全性和性能

4. **SystemUI ANR 可能是系统问题的表现**
   - SystemUI 本身可能没有问题
   - 需要从系统层面分析

### 6.2 分析方法

1. **优先看 PSI 数据**
   - PSI 能快速定位问题资源
   - 本案例中 IO PSI 78% 直接指向 IO 问题

2. **结合多个指标**
   - PSI + iowait + major faults
   - 综合判断问题链条

3. **关注异常进程**
   - 进程 13442 的异常行为是触发点
   - 需要分析异常进程

### 6.3 最佳实践

1. **避免主线程 IO**
   - 所有 IO 操作应该在后台线程
   - 使用异步 IO

2. **监控系统资源**
   - 设置 PSI 阈值预警
   - 提前发现问题

3. **优化内存使用**
   - 减少内存泄漏
   - 及时释放内存

4. **优化存储性能**
   - 使用更快的存储设备
   - 优化 IO 调度器

---

## 附录：需要进一步确认的信息

为了更准确地分析，建议提供以下信息：

1. **存储设备信息**：
   - 设备类型（eMMC/UFS/SSD）
   - 性能参数

2. **Swap 配置**：
   - Swap 大小
   - Swap 位置（分区/文件）
   - Swap 使用情况

3. **进程 13442 的详细信息**：
   - 进程名称和用途
   - 内存使用情况
   - 是否为异常进程

4. **系统配置**：
   - 物理内存大小
   - 内存参数配置
   - IO 调度器配置

5. **ANR 发生前的系统状态**：
   - 是否有大量应用启动
   - 是否有大量数据读写
   - 系统负载变化趋势

---

**总结**：这是一个典型的**内存不足 → 大量 swap IO → IO 阻塞 → ANR** 的问题链条。IO PSI 数据（78% some, 52% full）明确指向 IO 问题是主要原因，而内存不足是根本原因。需要从内存管理和 IO 优化两个方向解决。
