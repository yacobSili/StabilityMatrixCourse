# ANR 实战案例分析：IO 阻塞 + 内存压力导致的 SystemUI ANR

## 📋 案例概述

**ANR 时间**：2025-10-27 13:38:08.697  
**ANR 进程**：com.android.systemui (PID: 1652)  
**ANR 原因**：Input dispatching timed out (StatusBar (server) 无响应，等待 5001ms)  
**核心问题**：IO Wait 极高（60%），IO PSI 压力极高（some 85.36%, full 46.97%），内存压力高（some 50.71%, full 38.87%）

---

## 第一章：日志关键信息提取

### 1.1 ANR 基本信息

```
ANR in com.android.systemui
PID: 1652
Reason: Input dispatching timed out (7682d7c StatusBar (server) is not responding. Waited 5001ms for MotionEvent)
Load: 74.47 / 30.6 / 19.47
Frozen: false
```

**关键点**：
- SystemUI 的 StatusBar 无法处理输入事件
- **系统负载极高**：1分钟平均 74.47（远超正常值，正常应 < 4）
- 等待超时 5001ms（超过 5 秒）
- 进程未冻结，说明是响应性问题而非进程被杀死

### 1.2 PSI 压力数据（核心指标）

#### Memory Pressure ⚠️ **严重问题**

```
some avg10=50.71% avg60=38.19% avg300=12.16% total=484823706
full avg10=38.87% avg60=29.16% avg300=9.09% total=332402167
```

**分析**：

**Memory some 压力（部分任务被内存阻塞）**：
- **avg10=50.71%**：过去 10 秒内，50.71% 的时间有任务因内存资源不足而被阻塞
  - ⚠️ **严重**：超过一半的时间都有任务在等待内存
  - 说明系统内存严重不足，频繁触发内存回收
- **avg60=38.19%**：过去 60 秒内，38.19% 的时间有任务被内存阻塞
  - ⚠️ **严重**：中期压力也很高，说明内存压力不是偶发现象
  - 压力持续存在，系统长期处于内存紧张状态
- **avg300=12.16%**：过去 300 秒内，12.16% 的时间有任务被内存阻塞
  - 长期压力中等偏高，已经超过正常阈值（正常应 < 10%）
  - 说明系统长期存在内存压力问题
- **趋势分析**：avg300 (12.16%) → avg60 (38.19%) → avg10 (50.71%)
  - ⚠️ **压力急剧上升**：从长期 12.16% 飙升到短期 50.71%
  - 说明内存压力在近期严重恶化，是导致 IO 阻塞和 ANR 的根源

**Memory full 压力（所有任务都被内存阻塞）**：
- **avg10=38.87%**：过去 10 秒内，38.87% 的时间所有任务都被内存阻塞
  - ⚠️ **严重**：超过 1/3 的时间系统完全无法响应
  - 这是系统卡顿的重要原因，所有任务都在等待内存分配或回收
- **avg60=29.16%**：过去 60 秒内，29.16% 的时间所有任务都被内存阻塞
  - ⚠️ **严重**：中期也有接近 1/3 的时间系统完全阻塞
  - 说明内存阻塞问题持续且严重
- **avg300=9.09%**：过去 300 秒内，9.09% 的时间所有任务都被内存阻塞
  - 长期压力接近正常阈值上限（正常应 < 5%）
  - 说明系统长期存在内存性能问题
- **趋势分析**：avg300 (9.09%) → avg60 (29.16%) → avg10 (38.87%)
  - ⚠️ **压力急剧上升**：从长期 9.09% 飙升到短期 38.87%
  - 说明内存阻塞在近期严重恶化，系统响应能力急剧下降

**关联性分析**：
- **内存压力 → 内存回收**：高内存压力触发 kswapd0 频繁回收内存
- **内存回收 → IO 操作**：页面换出到存储设备产生大量 IO 操作
- **IO 操作 → IO 阻塞**：IO 设备成为瓶颈，导致 IO 阻塞
- **IO 阻塞 → ANR**：所有进程被 IO 阻塞，SystemUI 无法响应

**结论**：内存压力是导致 ANR 的**重要原因之一**。
- Memory some 压力 50.71% 说明超过一半的时间都有任务被内存阻塞
- Memory full 压力 38.87% 说明超过 1/3 的时间系统完全无法响应
- 压力在近期急剧上升，是导致 IO 阻塞和 ANR 的根源

#### CPU Pressure

```
some avg10=24.90% avg60=9.97% avg300=6.34% total=2393427261
full avg10=0.00% avg60=0.00% avg300=0.00% total=0
```

**分析**：

**CPU some 压力（部分任务被 CPU 阻塞）**：
- **avg10=24.90%**：过去 10 秒内，24.9% 的时间有任务因 CPU 资源不足而被阻塞
  - 说明短期内有 CPU 竞争，但不算严重
  - 可能是内存回收和 IO 操作导致的 CPU 使用增加
- **avg60=9.97%**：过去 60 秒内，9.97% 的时间有任务被 CPU 阻塞
  - 中期压力较低，说明 CPU 压力是短期突增
- **avg300=6.34%**：过去 300 秒内，6.34% 的时间有任务被 CPU 阻塞
  - 长期压力很低，说明系统 CPU 资源整体充足
- **趋势分析**：avg300 (6.34%) → avg60 (9.97%) → avg10 (24.90%)
  - 压力在近期明显上升，但长期来看 CPU 不是瓶颈

**CPU full 压力（所有任务都被 CPU 阻塞）**：
- **avg10=0.00%**：过去 10 秒内，没有出现所有任务都被 CPU 阻塞的情况
- **avg60=0.00%**：过去 60 秒内，没有出现所有任务都被 CPU 阻塞的情况
- **avg300=0.00%**：过去 300 秒内，没有出现所有任务都被 CPU 阻塞的情况

**结论**：
- CPU 资源充足，不是导致 ANR 的主要原因
- CPU 压力的短期上升可能是内存回收和 IO 操作的副作用
- **CPU full 压力为 0** 说明即使有 CPU 竞争，也没有达到完全阻塞的程度

#### IO Pressure ⚠️ **关键问题**

```
some avg10=85.36% avg60=57.98% avg300=19.32% total=1108768035
full avg10=46.97% avg60=36.52% avg300=11.90% total=609698204
```

**分析**：

**IO some 压力（部分任务被 IO 阻塞）**：
- **avg10=85.36%**：过去 10 秒内，85.36% 的时间有任务因 IO 阻塞而无法执行
  - ⚠️ **极其严重**：超过 80% 的时间都有 IO 阻塞
  - 说明 IO 设备是严重瓶颈，几乎所有时间都有任务在等待 IO
- **avg60=57.98%**：过去 60 秒内，57.98% 的时间有任务被 IO 阻塞
  - ⚠️ **严重**：中期压力也很高，说明 IO 阻塞不是偶发现象
  - 压力持续存在，系统长期处于 IO 阻塞状态
- **avg300=19.32%**：过去 300 秒内，19.32% 的时间有任务被 IO 阻塞
  - 长期压力中等，但已经超过正常阈值（正常应 < 10%）
- **趋势分析**：avg300 (19.32%) → avg60 (57.98%) → avg10 (85.36%)
  - ⚠️ **压力急剧上升**：从长期 19.32% 飙升到短期 85.36%
  - 说明 IO 阻塞在近期严重恶化，是导致 ANR 的直接原因

**IO full 压力（所有任务都被 IO 阻塞）**：
- **avg10=46.97%**：过去 10 秒内，46.97% 的时间所有任务都被 IO 阻塞
  - ⚠️ **极其严重**：接近一半的时间系统完全无法响应
  - 这是系统卡顿和 ANR 的直接表现
- **avg60=36.52%**：过去 60 秒内，36.52% 的时间所有任务都被 IO 阻塞
  - ⚠️ **严重**：中期也有超过 1/3 的时间系统完全阻塞
  - 说明 IO 阻塞问题持续且严重
- **avg300=11.90%**：过去 300 秒内，11.90% 的时间所有任务都被 IO 阻塞
  - 长期压力已经超过正常阈值（正常应 < 5%）
  - 说明系统长期存在 IO 性能问题
- **趋势分析**：avg300 (11.90%) → avg60 (36.52%) → avg10 (46.97%)
  - ⚠️ **压力急剧上升**：从长期 11.90% 飙升到短期 46.97%
  - 说明 IO 阻塞在近期严重恶化，系统响应能力急剧下降

**对比分析**：
- **IO some vs Memory some**：85.36% vs 50.71%
  - IO 压力比内存压力更高，说明 IO 是更严重的瓶颈
- **IO full vs Memory full**：46.97% vs 38.87%
  - IO 完全阻塞的时间比内存完全阻塞的时间更长
  - 说明 IO 阻塞对系统响应的影响更大

**结论**：IO 压力是导致 ANR 的**主要原因**。
- IO some 压力 85.36% 说明几乎所有时间都有任务被 IO 阻塞
- IO full 压力 46.97% 说明接近一半的时间系统完全无法响应
- 压力在近期急剧上升，是导致 ANR 的直接原因

### 1.3 CPU 使用情况（ANR 前 40 秒）

```
97% TOTAL: 10% user + 25% kernel + 60% iowait + 0.7% irq + 0.5% softirq
```

**关键发现**：
- ⚠️ **iowait 60%**：CPU 在等待 IO 完成的时间占比 60%（**极其严重**）
- kernel 占用 25%：内核处理 IO 相关操作
- user 10%：用户态 CPU 使用较低（说明 CPU 资源充足，但被 IO 阻塞）

**iowait 60% 的含义**：
- CPU 空闲但等待 IO 完成的时间占 60%
- 说明 IO 设备是严重瓶颈，CPU 在等待 IO
- 这是 IO 阻塞的典型表现，**远超正常阈值（>20% 为极严重）**

---

## 第二章：CPU 使用统计详细分析

### 2.1 时间段说明

ANR 日志包含两个时间段的 CPU 统计：

1. **ANR 前时间段**：从 41015ms 到 893ms 前（约 40 秒）
   - 这是 ANR 发生前的系统状态
   - 用于分析导致 ANR 的根本原因

2. **ANR 后时间段**：从 128ms 到 1319ms 后（约 1.2 秒）
   - 这是 ANR 发生后的系统状态
   - 用于分析 ANR 发生时的系统响应

### 2.2 ANR 前时间段（41015ms 到 893ms 前）进程分析

#### 2.2.1 内存回收相关进程

**kswapd0 (PID: 103) - 19% CPU**

```
19% 103/kswapd0: 0% user + 19% kernel
```

**进程说明**：
- `kswapd0` 是内核的内存回收守护进程
- 当系统内存不足时，kswapd0 会主动回收内存
- 19% 的 CPU 使用说明内存回收压力很大

**分析**：
- 结合内存 PSI（some 50.71%, full 38.87%），说明系统内存严重不足
- kswapd0 在频繁进行内存回收操作
- 内存回收会导致页面换出到存储设备，产生大量 IO 操作

**关联性**：
- 内存压力 → kswapd0 活跃 → 产生 IO 操作 → IO 阻塞 → ANR

**kshrink_slabd (PID: 528) - 13% CPU（ANR 后时间段）**

```
13% 528/kshrink_slabd: 0% user + 13% kernel
```

**进程说明**：
- `kshrink_slabd` 是内核的 slab 缓存收缩守护进程
- 负责回收内核对象缓存（如 dentry、inode 等）
- 13% 的 CPU 使用说明 slab 缓存回收压力大

**分析**：
- 内存压力导致需要回收 slab 缓存
- slab 缓存回收也会产生 IO 操作

#### 2.2.2 IO 相关 kworker 进程

**kworker/u16:1-devfreq_wq (PID: 16874) - 18% CPU**

```
18% 16874/kworker/u16:1-devfreq_wq: 0% user + 18% kernel
```

**进程说明**：
- `kworker` 是内核工作队列的工作线程
- `devfreq_wq` 是设备频率管理的工作队列
- 用于动态调整设备（如 GPU、DDR）的工作频率

**分析**：
- 18% 的 CPU 使用说明设备频率管理任务繁忙
- 可能与内存压力导致的设备性能调整有关

**kworker/u16:3-events_unbound (PID: 14061) - 17% CPU**

```
17% 14061/kworker/u16:3-events_unbound: 0% user + 17% kernel
```

**进程说明**：
- `events_unbound` 是未绑定 CPU 的事件处理工作队列
- 用于处理各种异步事件，包括文件系统事件、设备事件等

**分析**：
- 17% 的 CPU 使用说明有大量异步事件需要处理
- 可能与文件系统操作、设备 I/O 相关

**kworker/u16:5-events_unbound (PID: 18175) - 13% CPU**

```
13% 18175/kworker/u16:5-events_unbound: 0% user + 13% kernel
```

**分析**：
- 同样是事件处理工作队列，说明系统中有大量异步事件积压

**kworker/u16:0-events_unbound (PID: 6958) - 10% CPU**

```
10% 6958/kworker/u16:0-events_unbound: 0% user + 10% kernel
```

**分析**：
- 多个 events_unbound 工作线程都在高负载运行
- 说明系统事件处理能力不足，事件积压严重

**kworker/u17:6-kbase_pm_poweroff_wait (PID: 7296) - 7.8% CPU**

```
7.8% 7296/kworker/u17:6-kbase_pm_poweroff_wait: 0% user + 7.8% kernel
```

**进程说明**：
- `kbase_pm_poweroff_wait` 是 Mali GPU 驱动的工作队列
- 用于处理 GPU 电源管理和关闭等待

**分析**：
- GPU 相关的电源管理任务，可能与内存压力导致的 GPU 操作有关

**kworker/u16:7-loop28 (PID: 9853) - 7.2% CPU**

```
7.2% 9853/kworker/u16:7-loop28: 0% user + 7.2% kernel
```

**进程说明**：
- `loop28` 是 loop 设备（用于挂载镜像文件）的工作队列
- 可能与文件系统操作、存储设备相关

**分析**：
- 7.2% 的 CPU 使用说明有 loop 设备相关的 IO 操作

**kworker/u17:4-blk_crypto_wq (PID: 5922) - 6.6% CPU**

```
6.6% 5922/kworker/u17:4-blk_crypto_wq: 0% user + 6.6% kernel
```

**进程说明**：
- `blk_crypto_wq` 是块设备加密的工作队列
- 用于处理存储设备的加密/解密操作

**分析**：
- 6.6% 的 CPU 使用说明有大量加密 IO 操作
- 加密操作会增加 IO 延迟，加剧 IO 阻塞

**kworker/u17:7-blk_crypto_wq (PID: 17304) - 5.8% CPU**

```
5.8% 17304/kworker/u17:7-blk_crypto_wq: 0% user + 5.8% kernel
```

**分析**：
- 多个 blk_crypto_wq 工作线程都在运行
- 说明加密 IO 操作频繁，可能是 IO 阻塞的重要原因

#### 2.2.3 系统服务进程

**system_server (PID: 1108) - 17% CPU**

```
17% 1108/system_server: 8.5% user + 8.8% kernel / faults: 26361 minor 24631 major
```

**进程说明**：
- `system_server` 是 Android 系统服务的核心进程
- 包含 ActivityManager、WindowManager、PackageManager 等关键服务

**关键指标**：
- **faults: 26361 minor 24631 major**
  - minor faults：页面未在内存中，需要从 swap 或文件加载（26,361 次）
  - major faults：需要从存储设备读取数据（24,631 次）
  - **major faults 极高**，说明大量数据需要从存储设备读取

**分析**：
- 17% 的 CPU 使用正常，但 major faults 极高
- 说明 system_server 在频繁进行 IO 操作
- 可能与内存压力导致的页面换出有关

#### 2.2.4 其他进程

**com.android.systemui (PID: 1652) - 主进程（ANR 进程）**

在 ANR 前时间段，systemui 的 CPU 使用未单独列出，说明：
- systemui 在 ANR 前可能被阻塞，无法获得 CPU 时间
- 或者 CPU 使用较低，但被 IO 阻塞

### 2.3 ANR 后时间段（128ms 到 1319ms 后）进程分析

#### 2.3.1 内存回收进程（更活跃）

**kswapd0 (PID: 103) - 63% CPU**

```
63% 103/kswapd0: 0% user + 63% kernel
```

**分析**：
- CPU 使用从 19% 飙升到 63%
- 说明 ANR 发生后，内存回收压力进一步增大
- 可能是 ANR 处理过程中产生了更多内存需求

#### 2.3.2 system_server（ANR 处理）

**system_server (PID: 1108) - 116% CPU**

```
116% 1108/system_server: 65% user + 51% kernel / faults: 1683 minor 160 major
faults: 1683 minor 160 major
```

**关键指标**：
- **116% CPU**：多核系统中，一个进程可以超过 100%（使用了多个 CPU 核心）
- **faults: 1683 minor 160 major**：在 1.2 秒内发生了大量页面错误
- 说明 system_server 在 ANR 处理过程中需要大量内存和 IO

**子线程分析**：

```
49% 10707/AnrAuxiliaryTas: 21% user + 28% kernel
```

**进程说明**：
- `AnrAuxiliaryTas` 是 ANR 辅助任务线程
- 用于收集 ANR 相关信息（如 traces、PSI 数据等）

**分析**：
- 49% 的 CPU 使用说明 ANR 处理任务繁重
- 28% kernel 时间可能与 IO 操作相关（写入 traces 文件等）

#### 2.3.3 systemui（ANR 进程）

**com.android.systemui (PID: 1652) - 77% CPU**

```
77% 1652/com.android.systemui: 62% user + 15% kernel / faults: 1874 minor 1084 major
```

**关键指标**：
- **faults: 1874 minor 1084 major**：在 1.2 秒内发生了大量页面错误
- **major faults 1084**：需要从存储设备读取数据（极高）
- 说明 systemui 在尝试恢复时，需要大量从存储设备加载数据

**分析**：
- 77% 的 CPU 使用说明 systemui 在努力恢复
- 但 major faults 极高，说明被 IO 阻塞严重
- 这是典型的 IO 阻塞导致的 ANR

#### 2.3.4 其他高 CPU 进程

**kworker/u16:5-events_unbound (PID: 18175) - 58% CPU**

```
58% 18175/kworker/u16:5-events_unbound: 0% user + 58% kernel
```

**分析**：
- CPU 使用从 13% 飙升到 58%
- 说明 ANR 发生后，事件处理压力进一步增大

**com.zhiliaoapp.musically.go (PID: 10634) - 34% CPU**

```
34% 10634/com.zhiliaoapp.musically.go: 4.9% user + 29% kernel / faults: 388 minor 84 major
```

**分析**：
- 第三方应用（可能是抖音）
- 29% kernel 时间和 major faults 84，说明也在进行 IO 操作
- 可能与文件读写、网络 IO 相关

**kworker/u16:3-writeback (PID: 14061) - 14% CPU**

```
14% 14061/kworker/u16:3-writeback: 0% user + 14% kernel
```

**进程说明**：
- `writeback` 是页面回写工作队列
- 负责将脏页写回存储设备

**分析**：
- 14% 的 CPU 使用说明有大量脏页需要写回
- 这是内存压力导致的典型表现

**kworker/2:3H-kblockd (PID: 25371) - 12% CPU**

```
12% 25371/kworker/2:3H-kblockd: 0% user + 12% kernel
```

**进程说明**：
- `kblockd` 是块设备工作队列
- 用于处理块设备的 IO 请求

**分析**：
- 12% 的 CPU 使用说明块设备 IO 操作频繁
- 这是 IO 阻塞的直接原因

**kworker/3:169H-kblockd (PID: 27064) - 16% CPU**

```
16% 27064/kworker/3:169H-kblockd: 0% user + 16% kernel
```

**分析**：
- 多个 kblockd 工作线程都在高负载运行
- 说明块设备 IO 请求积压严重

---

## 第三章：根因分析总结

### 3.1 问题链条

```
内存压力高 (PSI some 50.71%, full 38.87%)
    ↓
kswapd0 频繁回收内存 (19% → 63% CPU)
    ↓
大量页面换出到存储设备
    ↓
产生大量 IO 操作
    ↓
IO 设备成为瓶颈 (iowait 60%, IO PSI some 85.36%)
    ↓
所有进程被 IO 阻塞
    ↓
SystemUI 无法响应输入事件
    ↓
ANR 发生
```

### 3.2 核心问题

1. **内存压力严重**
   - PSI memory some: 50.71%
   - PSI memory full: 38.87%
   - 导致 kswapd0 频繁进行内存回收

2. **IO 阻塞严重**
   - iowait: 60%（远超正常阈值）
   - PSI IO some: 85.36%
   - PSI IO full: 46.97%
   - 大量 kworker 进程在进行 IO 操作

3. **系统负载极高**
   - Load: 74.47 / 30.6 / 19.47
   - 说明系统资源严重不足

4. **页面错误频繁**
   - system_server: 24,631 major faults
   - systemui: 1,084 major faults（ANR 后）
   - 说明大量数据需要从存储设备读取

### 3.3 关键进程角色

| 进程 | CPU 使用 | 角色 | 影响 |
|------|---------|------|------|
| kswapd0 | 19% → 63% | 内存回收 | 产生 IO 操作 |
| kworker (多个) | 5-18% | IO 处理 | 直接导致 IO 阻塞 |
| system_server | 17% → 116% | 系统服务 | major faults 极高 |
| systemui | 0% → 77% | ANR 进程 | 被 IO 阻塞无法响应 |

---

## 第四章：后续分析建议

### 4.1 立即需要收集的信息

#### 4.1.1 内存相关

1. **内存使用情况**
   ```bash
   # 查看内存使用详情
   adb shell dumpsys meminfo
   adb shell cat /proc/meminfo
   ```

2. **内存回收统计**
   ```bash
   # 查看内存回收统计
   adb shell cat /proc/vmstat | grep -E "pgmajfault|pgpgin|pgpgout|pswpin|pswpout"
   ```

3. **进程内存详情**
   ```bash
   # 查看 systemui 和 system_server 的内存使用
   adb shell dumpsys meminfo 1652  # systemui
   adb shell dumpsys meminfo 1108  # system_server
   ```

#### 4.1.2 IO 相关

1. **IO 统计信息**
   ```bash
   # 查看 IO 统计
   adb shell cat /proc/diskstats
   adb shell iostat -x 1 10
   ```

2. **IO 等待队列**
   ```bash
   # 查看 IO 等待队列长度
   adb shell cat /sys/block/*/queue/nr_requests
   ```

3. **存储设备性能**
   ```bash
   # 查看存储设备信息
   adb shell cat /proc/partitions
   adb shell lsblk
   ```

#### 4.1.3 进程详情

1. **systemui 线程堆栈**
   ```bash
   # 获取 systemui 的 traces
   adb shell kill -3 1652
   adb pull /data/anr/traces.txt
   ```

2. **kswapd0 调用栈**
   ```bash
   # 查看 kswapd0 的调用栈
   adb shell cat /proc/103/stack
   ```

3. **kworker 调用栈**
   ```bash
   # 查看高 CPU 的 kworker 调用栈
   adb shell cat /proc/16874/stack  # kworker/u16:1-devfreq_wq
   adb shell cat /proc/14061/stack  # kworker/u16:3-events_unbound
   ```

### 4.2 深入分析方向

#### 4.2.1 内存压力根因分析

1. **内存泄漏检查**
   - 使用 `dumpsys meminfo` 查看各进程内存使用趋势
   - 检查是否有进程内存持续增长
   - 使用 `MAT` 或 `LeakCanary` 分析内存泄漏

2. **内存分配分析**
   - 使用 `dumpsys meminfo --unreachable` 查看不可达对象
   - 分析大对象分配（Large Object Allocation）
   - 检查是否有异常的内存分配模式

3. **内存回收效率**
   - 分析 kswapd0 的回收效率
   - 检查是否有内存碎片化问题
   - 评估内存压缩（zram/zswap）的效果

#### 4.2.2 IO 阻塞根因分析

1. **IO 操作类型分析**
   - 使用 `strace` 或 `ftrace` 跟踪 IO 操作
   - 分析是读操作还是写操作导致阻塞
   - 检查是否有大量小 IO 操作

2. **存储设备性能**
   - 测试存储设备的随机读写性能
   - 检查是否有存储设备降频或节流
   - 评估存储设备的健康状态

3. **文件系统分析**
   - 检查文件系统类型（ext4/f2fs）
   - 分析文件系统的碎片化情况
   - 评估文件系统的性能参数

#### 4.2.3 系统负载分析

1. **负载来源分析**
   - 使用 `top` 或 `htop` 查看实时进程 CPU 使用
   - 分析哪些进程消耗了最多 CPU
   - 检查是否有异常进程

2. **调度分析**
   - 使用 `ftrace` 分析进程调度延迟
   - 检查是否有进程调度异常
   - 分析 CPU 频率调节（CPUFreq）的影响

### 4.3 监控和预防

#### 4.3.1 实时监控指标

1. **PSI 监控**
   ```bash
   # 持续监控 PSI 指标
   watch -n 1 'cat /proc/pressure/memory && cat /proc/pressure/io && cat /proc/pressure/cpu'
   ```

2. **IO 监控**
   ```bash
   # 监控 IO 等待
   watch -n 1 'iostat -x 1 1'
   ```

3. **内存监控**
   ```bash
   # 监控内存使用
   watch -n 1 'free -h && cat /proc/meminfo | grep -E "MemAvailable|SwapFree"'
   ```

#### 4.3.2 告警阈值设置

| 指标 | 正常值 | 警告阈值 | 严重阈值 | 紧急阈值 |
|------|--------|---------|---------|---------|
| PSI Memory some | < 10% | 10-20% | 20-40% | > 40% |
| PSI Memory full | < 5% | 5-10% | 10-20% | > 20% |
| PSI IO some | < 10% | 10-30% | 30-60% | > 60% |
| PSI IO full | < 5% | 5-15% | 15-30% | > 30% |
| iowait | < 5% | 5-10% | 10-20% | > 20% |
| Load (1min) | < 4 | 4-8 | 8-16 | > 16 |

#### 4.3.3 预防措施

1. **内存优化**
   - 优化应用内存使用，减少内存泄漏
   - 调整 LMK（Low Memory Killer）参数
   - 启用内存压缩（zram/zswap）

2. **IO 优化**
   - 优化文件系统参数（如 ext4 的 journal 模式）
   - 减少不必要的 IO 操作
   - 使用 IO 调度器优化（如 deadline、noop）

3. **系统优化**
   - 限制后台进程数量
   - 优化进程优先级
   - 调整 CPU 频率策略

---

## 第五章：问题定位检查清单

### 5.1 内存问题检查

- [ ] 检查系统总内存使用率（应 < 80%）
- [ ] 检查各进程内存使用（是否有异常增长）
- [ ] 检查 swap 使用情况（swap 使用率高说明内存不足）
- [ ] 检查内存回收统计（pgmajfault、pswpout 等）
- [ ] 检查是否有内存泄漏（使用 MAT 或 LeakCanary）

### 5.2 IO 问题检查

- [ ] 检查 iowait 值（应 < 10%）
- [ ] 检查 IO PSI 指标（some 应 < 30%，full 应 < 15%）
- [ ] 检查磁盘 IO 使用率（应 < 80%）
- [ ] 检查 IO 等待队列长度
- [ ] 检查存储设备性能（随机读写速度）
- [ ] 检查文件系统类型和参数

### 5.3 进程问题检查

- [ ] 检查 kswapd0 CPU 使用（应 < 10%）
- [ ] 检查 kworker 进程数量和 CPU 使用
- [ ] 检查 system_server major faults（应 < 1000/秒）
- [ ] 检查 ANR 进程的线程堆栈
- [ ] 检查是否有异常进程消耗资源

### 5.4 系统负载检查

- [ ] 检查系统负载（Load 1min 应 < CPU 核心数）
- [ ] 检查 CPU 使用率分布（user/kernel/iowait）
- [ ] 检查进程调度延迟
- [ ] 检查 CPU 频率调节策略

---

## 第六章：总结

### 6.1 问题本质

这是一个典型的**资源竞争导致的 ANR**：
- **内存不足** → 触发内存回收 → 产生大量 IO 操作
- **IO 设备成为瓶颈** → 所有进程被 IO 阻塞
- **SystemUI 无法响应** → ANR 发生

### 6.2 关键指标

| 指标 | 值 | 状态 |
|------|-----|------|
| PSI Memory some | 50.71% | ⚠️ 严重 |
| PSI Memory full | 38.87% | ⚠️ 严重 |
| PSI IO some | 85.36% | ⚠️ 极严重 |
| PSI IO full | 46.97% | ⚠️ 极严重 |
| iowait | 60% | ⚠️ 极严重 |
| Load (1min) | 74.47 | ⚠️ 极严重 |

### 6.3 解决方向

1. **短期**：优化内存使用，减少内存压力
2. **中期**：优化 IO 性能，减少 IO 阻塞
3. **长期**：建立监控体系，预防类似问题

### 6.4 经验总结

1. **PSI 指标是判断资源压力的关键**
   - Memory PSI > 30% 需要关注
   - IO PSI > 50% 需要紧急处理

2. **iowait 是 IO 阻塞的直接指标**
   - iowait > 20% 说明 IO 严重阻塞
   - 需要结合 PSI IO 指标综合判断

3. **major faults 反映 IO 压力**
   - major faults 高说明大量数据需要从存储设备读取
   - 需要优化内存使用，减少页面换出

4. **kworker 进程是 IO 操作的执行者**
   - 多个 kworker 高 CPU 使用说明 IO 操作频繁
   - 需要分析 kworker 的调用栈，找出 IO 操作来源

---

**文档创建时间**：2025-10-27  
**ANR 发生时间**：2025-10-27 13:38:08.697  
**分析状态**：初步分析完成，待深入调查
