# ANR问题实战-整机IO/MEM劣化问题分析专项

## 一、问题概述

### 1.1 问题现象
- **时间点**: 2025-10-27 13:38:07-13:38:09
- **ANR数量**: 连续发生4次ANR，时间间隔极短（约1-2秒）
- **影响进程**:
  1. `com.transsion.hilauncher` (13:38:07.089)
  2. `com.android.systemui` (13:38:08.697)
  3. `com.transsion.aivoiceassistant` (13:38:08.825)
  4. `com.android.vending:background` (13:38:08.934)

### 1.2 首个ANR关键信息
- **进程**: com.transsion.hilauncher (PID: 1871)
- **ANR类型**: Input dispatching timed out
- **等待时间**: 5000ms for KeyEvent
- **系统负载**: Load: 52.36 / 23.6 / 17.02 (1分钟/5分钟/15分钟)

## 二、PSI压力数据分析

### 2.1 Memory PSI压力
```
some avg10=57.05 avg60=28.35 avg300=8.25 total=471622661
full avg10=44.72 avg60=20.60 avg300=5.88 total=321772892
```

**分析要点**:
- **avg10=57.05**: 过去10秒内，57%的时间至少有一个任务因内存压力而stall
- **avg60=28.35**: 过去1分钟内，28%的时间存在内存压力
- **avg300=8.25**: 过去5分钟内，8%的时间存在内存压力
- **full avg10=44.72**: 过去10秒内，44%的时间所有任务都因内存压力而stall

**结论**: **内存压力极高**，属于严重的内存压力场景。

### 2.2 CPU PSI压力
```
some avg10=3.97 avg60=7.12 avg300=5.67 total=2389787436
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**分析要点**:
- **avg10=3.97**: CPU压力相对较低
- **full avg10=0.00**: 没有出现所有任务都被CPU压力阻塞的情况
- **avg60=7.12**: 1分钟平均压力略高于10秒平均，说明压力在持续

**结论**: CPU压力存在但不算严重，不是主要根因。

### 2.3 IO PSI压力
```
some avg10=89.35 avg60=45.15 avg300=13.99 total=1090425972
full avg10=62.55 avg60=28.39 avg300=8.42 total=598097838
```

**分析要点**:
- **avg10=89.35**: **过去10秒内，89%的时间至少有一个任务因IO压力而stall** - **这是最严重的问题**
- **avg60=45.15**: 过去1分钟内，45%的时间存在IO压力
- **full avg10=62.55**: 过去10秒内，62%的时间所有任务都因IO压力而stall
- **total=1090425972**: IO stall总时间非常长

**结论**: **IO压力极高，是导致ANR的主要原因**。

### 2.4 PSI压力综合判定

| 压力类型 | avg10 | avg60 | 严重程度 | 判定 |
|---------|-------|-------|---------|------|
| Memory | 57.05 | 28.35 | 严重 | 高 |
| CPU | 3.97 | 7.12 | 中等 | 中 |
| IO | **89.35** | **45.15** | **极严重** | **最高** |

**根因判定**: **IO压力是主要根因，Memory压力是次要因素，两者形成恶性循环**。

## 三、CPU使用情况分析

### 3.1 第一个时间窗口 (3ms to 40125ms later)
```
97% TOTAL: 10% user + 25% kernel + 60% iowait + 0.7% irq + 0.5% softirq
```

**关键发现**:
- **iowait=60%**: CPU有60%的时间在等待IO完成，这是IO瓶颈的明确证据
- **kernel=25%**: 内核态CPU使用率高，说明大量内核线程在工作

**活跃线程分析**:

#### 3.1.1 内存回收相关线程
- **kswapd0 (19%)**: 内核内存回收守护进程，高CPU使用说明系统在积极回收内存
- **kshrink_slabd (18%)**: slab缓存收缩守护进程，配合kswapd进行内存回收

#### 3.1.2 kworker线程（IO相关）
- **kworker/u16:1-devfreq_wq (18%)**: devfreq工作队列，设备频率管理
- **kworker/u16:3-events_unbound (17%)**: 事件处理工作队列
- **kworker/u16:5-events_unbound (13%)**: 事件处理工作队列
- **kworker/u16:0-events_unbound (10%)**: 事件处理工作队列
- **kworker/u17:6-kbase_pm_poweroff_wait (7.8%)**: GPU电源管理
- **kworker/u16:7-loop28 (7.2%)**: **loop28设备相关，可能是存储设备**
- **kworker/u17:4-blk_crypto_wq (6.6%)**: 块设备加密工作队列
- **kworker/u17:7-blk_crypto_wq (5.8%)**: 块设备加密工作队列

**kworker线程说明**:
- `kworker`是Linux内核的工作队列机制，用于异步执行内核任务
- `u16`/`u17`表示unbound工作队列，可以在任意CPU上运行
- 大量kworker线程高CPU使用，说明内核在处理大量IO任务

#### 3.1.3 用户态进程
- **system_server (17%)**: 系统服务进程，包含大量binder线程
- **faults: 26361 minor 24631 major**: 大量缺页异常，说明内存压力导致频繁swap

### 3.2 第二个时间窗口 (20101ms to 20633ms later)
```
98% TOTAL: 16% user + 43% kernel + 37% iowait + 0.9% irq + 0.7% softirq
```

**关键发现**:
- **iowait=37%**: 仍然很高，IO瓶颈持续存在
- **kernel=43%**: 内核态CPU使用率进一步升高

**活跃线程分析**:

#### 3.2.1 内存回收线程
- **kswapd0 (47%)**: CPU使用率翻倍，说明内存回收压力进一步增大
- **kshrink_slabd (18%)**: 持续进行slab收缩

#### 3.2.2 kworker线程
- **kworker/u16:3-loop22 (44%)**: **loop22设备相关，CPU使用率极高**
- **kworker/u16:4+events_unbound (35%)**: 事件处理工作队列
- **kworker/u17:1-blk_crypto_wq (15%)**: 块设备加密
- 多个blk_crypto_wq线程持续运行

**loop设备分析**:
- `loop22`、`loop28`是Linux的loop设备，通常用于：
  - 挂载镜像文件
  - 容器存储
  - 加密存储
- 这些设备的高IO活动可能是IO瓶颈的根源

#### 3.2.3 用户态进程
- **system_server (67%)**: CPU使用率极高
  - **AnrAuxiliaryTas (49%)**: ANR辅助任务线程，正在收集ANR信息
  - 多个binder线程在工作
- **com.transsnet.store (59%)**: 某个应用进程CPU使用率高
- **com.transsion.phoenix:service (35%)**: 系统服务进程
- **faults: 1154 minor 151 major**: 仍然有大量缺页异常

## 四、根因分析

### 4.1 问题链条梳理

```
IO瓶颈 (loop设备/存储设备)
    ↓
大量IO等待 (iowait 60%+)
    ↓
进程无法及时响应 (等待IO完成)
    ↓
内存压力增大 (进程无法及时释放内存)
    ↓
kswapd0/kworker大量工作 (内存回收+IO处理)
    ↓
CPU资源被内核线程占用
    ↓
用户进程无法获得CPU时间片
    ↓
ANR发生 (Input dispatching timeout)
```

### 4.2 核心问题

#### 4.2.1 IO瓶颈（主要根因）
1. **IO PSI压力89.35%**: 系统89%的时间在等待IO
2. **iowait 60%+**: CPU大量时间等待IO完成
3. **loop设备高活动**: loop22、loop28设备可能是瓶颈点
4. **blk_crypto_wq高活动**: 块设备加密可能加剧IO延迟

**可能原因**:
- 存储设备性能差（eMMC/UFS老化或质量问题）
- 存储设备加密导致额外IO开销
- loop设备上的文件系统性能问题
- 大量小IO请求导致IO队列拥塞

#### 4.2.2 内存压力（次要根因，但加剧问题）
1. **Memory PSI压力57.05%**: 内存压力高
2. **kswapd0高CPU使用**: 积极进行内存回收
3. **大量major faults**: 频繁swap，说明内存不足
4. **内存压力导致进程无法及时释放资源**

**与IO的关联**:
- 内存不足 → 频繁swap → 增加IO压力
- IO慢 → 进程阻塞 → 无法及时释放内存 → 内存压力增大
- **形成恶性循环**

### 4.3 为什么连续4个ANR？

1. **系统级资源竞争**: IO和内存压力是系统级的，影响所有进程
2. **级联效应**: 第一个ANR（hilauncher）触发系统服务处理，进一步消耗资源
3. **时间窗口重叠**: 4个ANR发生在1-2秒内，说明系统在这个时间窗口内完全无法响应
4. **关键进程受影响**: systemui、launcher等关键进程同时受影响

## 五、关键线程详解

### 5.1 kswapd0
- **作用**: 内核内存回收守护进程
- **高CPU原因**: 系统内存压力大，需要积极回收内存
- **与ANR关系**: 间接相关，内存回收本身不会直接导致ANR，但会消耗CPU资源

### 5.2 kworker线程
- **作用**: 内核工作队列，处理异步内核任务（主要是IO相关）
- **高CPU原因**: 
  - 大量IO请求需要处理
  - loop设备、块设备加密等IO操作
- **与ANR关系**: **直接相关**，IO瓶颈导致进程无法及时响应

### 5.3 loop设备相关线程
- **kworker/u16:7-loop28**: loop28设备的工作队列
- **kworker/u16:3-loop22**: loop22设备的工作队列
- **可能用途**:
  - 容器存储（如果使用了容器技术）
  - 加密存储卷
  - 镜像文件挂载
- **问题**: 这些设备的IO性能可能是瓶颈

### 5.4 blk_crypto_wq线程
- **作用**: 块设备加密工作队列
- **高CPU原因**: 存储加密导致每个IO都需要加解密操作
- **影响**: 增加IO延迟，加剧IO瓶颈

### 5.5 system_server进程
- **AnrAuxiliaryTas**: ANR辅助任务线程，用于收集ANR信息
- **高CPU原因**: 正在处理ANR，需要收集大量系统信息
- **影响**: 进一步消耗系统资源

## 六、问题判定结论

### 6.1 主要根因：IO瓶颈
- **证据1**: IO PSI压力89.35%（极高）
- **证据2**: iowait 60%+（CPU大量时间等待IO）
- **证据3**: 大量kworker线程处理IO任务
- **证据4**: loop设备、blk_crypto_wq高活动

### 6.2 次要根因：内存压力
- **证据1**: Memory PSI压力57.05%（高）
- **证据2**: kswapd0高CPU使用（47%）
- **证据3**: 大量major faults（频繁swap）

### 6.3 问题性质
- **系统级问题**: 不是单个应用问题，而是整机资源瓶颈
- **IO-MEM恶性循环**: IO慢导致内存压力，内存压力加剧IO压力
- **级联ANR**: 系统级问题导致多个关键进程同时ANR

## 七、分析方法和工具

### 7.1 PSI数据分析方法
1. **关注avg10值**: 反映最近10秒的压力，最敏感
2. **对比三个维度**: Memory、CPU、IO，找出压力最大的
3. **full vs some**: 
   - `some`: 至少一个任务stall
   - `full`: 所有任务都stall（更严重）
4. **阈值判断**:
   - avg10 > 50%: 严重压力
   - avg10 > 80%: 极严重压力

### 7.2 CPU使用分析
1. **iowait指标**: 直接反映IO瓶颈
   - iowait > 30%: IO瓶颈明显
   - iowait > 50%: IO瓶颈严重
2. **内核线程分析**: 
   - kswapd0: 内存回收
   - kworker: IO处理
   - 高CPU使用说明对应资源压力大
3. **faults分析**:
   - minor faults: 正常缺页
   - major faults: 需要从swap读取，说明内存不足

### 7.3 线程识别方法
1. **进程名+线程名**: 如`kworker/u16:3-loop22`
2. **工作队列命名**: 
   - `devfreq_wq`: 设备频率管理
   - `events_unbound`: 事件处理
   - `blk_crypto_wq`: 块设备加密
   - `loopXX`: loop设备
3. **PID分析**: 通过PID可以进一步追踪线程

## 八、排查建议

### 8.1 立即排查项
1. **存储设备健康检查**
   - 检查eMMC/UFS健康状态
   - 检查存储设备性能（IOPS、延迟）
   - 检查是否有坏块

2. **loop设备分析**
   - 检查loop22、loop28的用途
   - 检查这些设备上的文件系统性能
   - 检查是否有大量IO在这些设备上

3. **存储加密影响**
   - 评估blk_crypto的性能影响
   - 考虑是否可以使用硬件加密加速
   - 评估加密算法的性能

4. **内存使用分析**
   - 检查内存使用情况（/proc/meminfo）
   - 检查swap使用情况
   - 分析哪些进程占用内存多

### 8.2 长期优化项
1. **IO优化**
   - 优化IO调度策略
   - 减少不必要的IO操作
   - 优化小IO合并
   - 考虑使用IO限流

2. **内存优化**
   - 减少内存占用
   - 优化内存回收策略
   - 减少swap使用

3. **监控告警**
   - 建立PSI压力监控
   - 设置IO/MEM压力阈值告警
   - 建立ANR根因自动分析

## 九、总结

### 9.1 问题本质
这是一个**系统级IO瓶颈问题**，导致整机性能劣化，进而引发级联ANR。

### 9.2 分析要点
1. **PSI数据是核心**: IO PSI 89.35%明确指向IO瓶颈
2. **iowait是关键指标**: 60%+的iowait证实IO瓶颈
3. **线程分析提供细节**: kworker、loop设备、blk_crypto等提供具体线索
4. **系统级视角**: 不是单个应用问题，需要系统级优化

### 9.3 判定标准
- **IO PSI > 80% + iowait > 50%** → IO瓶颈导致ANR
- **Memory PSI > 50% + kswapd0高CPU** → 内存压力导致ANR
- **两者同时存在** → IO-MEM恶性循环，需要同时优化

### 9.4 分析方法论
1. **先看PSI**: 快速定位压力类型
2. **再看iowait**: 确认IO瓶颈
3. **分析线程**: 找出具体瓶颈点
4. **系统视角**: 理解问题链条
5. **综合判定**: 确定主要根因和次要因素

---

## 附录：相关文档索引

- [PSI与ANR深度关联解析](../05_Kernel_Stability_Core/07_Personal_Summary/PSI与ANR深度关联解析：CPU、MEM、IO压力下的根因判定.md)
- [Memory Pressure PSI Deep Dive](../05_Kernel_Stability_Core/02_Memory_Management/Memory_Pressure_PSI_Deep_Dive.md)
- [IO Pressure PSI Deep Dive](../05_Kernel_Stability_Core/03_IO_Blocking/IO_Pressure_PSI_Deep_Dive.md)
- [kswapd Deep Dive](../05_Kernel_Stability_Core/02_Memory_Management/kswapd_Deep_Dive.md)
- [kworker Deep Dive](../05_Kernel_Stability_Core/01_Process_Scheduling/kworker_Deep_Dive.md)
