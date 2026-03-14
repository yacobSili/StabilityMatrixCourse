# LMKD 基础

## 📋 学习目标

掌握 LMKD (Low Memory Killer Daemon) 的基本概念和工作原理，能够独立完成基本配置和监控。

## 📚 学习内容

### 第一章：LMKD 入门

#### 1.1 什么是 LMKD？

**LMKD (Low Memory Killer Daemon)** 是 Android 系统中的低内存杀手守护进程，负责监控系统内存压力并在内存不足时终止非关键进程以释放内存。

**核心作用**：
- 监控系统内存压力状态
- 在内存不足时主动终止进程
- 防止系统触发内核级 OOM Killer
- 确保前台应用有足够内存运行

**为什么需要 LMKD？**

1. **移动设备内存有限**
   - 移动设备内存通常较小（4GB-12GB）
   - 多个应用后台驻留会消耗大量内存
   - 需要主动管理内存，避免内存耗尽

2. **防止系统崩溃**
   - 如果内存完全耗尽，内核 OOM Killer 可能杀死关键系统进程
   - 这会导致系统重启或关键服务崩溃
   - LMKD 在内存压力早期就介入，避免这种情况

3. **保证用户体验**
   - 确保前台应用有足够内存
   - 避免应用因内存不足而卡顿或崩溃
   - 提供流畅的用户体验

#### 1.2 LMKD 的历史演进

**Android 早期（Android 4.x 及之前）**：
- 使用内核态的 **Low Memory Killer (LMK)** 驱动
- 位于内核空间：`drivers/staging/android/lowmemorykiller.c`
- 基于内存水位线（minfree）触发
- 配置通过 `/sys/module/lowmemorykiller/parameters/` 接口

**Android 9 (Pie) 及之后**：
- 迁移到用户态的 **LMKD 守护进程**
- 位于用户空间：`system/core/lmkd/`
- 支持 PSI (Pressure Stall Information) 监控
- 更灵活的策略和配置

**演进原因**：
1. **更好的可配置性**：用户态更容易修改和调试
2. **更精确的监控**：PSI 提供更准确的内存压力指标
3. **更灵活的策略**：可以基于多种条件做决策
4. **更好的可维护性**：不需要修改内核代码

#### 1.3 LMKD 与内核 LMK 的区别

| 特性 | 内核 LMK | 用户态 LMKD |
|------|---------|------------|
| **位置** | 内核空间 | 用户空间 |
| **触发机制** | 内存水位线 | PSI / vmpressure |
| **配置方式** | `/sys/module/` | 系统属性 / 配置文件 |
| **可调试性** | 困难 | 容易 |
| **策略灵活性** | 固定 | 可配置 |
| **Android 版本** | ≤ 8.x | ≥ 9.0 |

**当前状态**：
- Android 9+ 默认使用 LMKD
- 内核 LMK 可能仍然存在但通常被禁用
- 可以通过 `ro.lmk.use_psi` 属性控制使用 PSI 还是 vmpressure

---

### 第二章：LMKD 基本工作原理

#### 2.1 LMKD 的基本工作流程

**基本流程**：

```
系统启动
    ↓
LMKD 守护进程启动
    ↓
初始化 PSI/vmpressure 监控
    ↓
进入事件循环，等待内存压力事件
    ↓
收到内存压力事件
    ↓
评估内存压力等级（low/medium/critical）
    ↓
根据压力等级选择目标进程
    ↓
终止目标进程（发送 SIGKILL）
    ↓
记录日志，继续监控
```

**关键步骤详解**：

1. **初始化阶段**
   - 打开 `/proc/pressure/memory`（PSI）或 `/dev/memcg`（vmpressure）
   - 配置内存压力阈值
   - 设置进程优先级（通常使用 `SCHED_FIFO`）

2. **监控阶段**
   - 监听内存压力事件
   - 解析压力等级和指标
   - 评估是否需要采取行动

3. **决策阶段**
   - 根据压力等级确定需要释放的内存
   - 扫描进程列表，计算每个进程的优先级
   - 选择最适合终止的进程

4. **执行阶段**
   - 向目标进程发送 `SIGKILL` 信号
   - 等待进程终止
   - 记录日志

#### 2.2 进程优先级：oom_score_adj

**什么是 oom_score_adj？**

`oom_score_adj` 是系统为每个进程分配的一个值，范围从 **-1000 到 1000**，表示进程的重要性。值越大，进程越容易被 LMKD 优先清理。

**oom_score_adj 的典型值**：

| 进程类型 | oom_score_adj 范围 | 说明 |
|---------|-------------------|------|
| **系统进程** | -1000 | 永远不会被杀死 |
| **前台应用** | 0 | 当前用户正在使用的应用 |
| **可见应用** | 100-200 | 用户可见但不在前台 |
| **感知服务** | 200 | 用户可以感知的服务 |
| **后台应用** | 900-1000 | 后台缓存的应用 |

**查看进程的 oom_score_adj**：

```bash
# 查看所有进程的 oom_score_adj
adb shell cat /proc/*/oom_score_adj

# 查看特定进程（例如 PID 1234）
adb shell cat /proc/1234/oom_score_adj

# 查看进程名称和 oom_score_adj
adb shell ps -eo pid,comm,oom_score_adj | head -20
```

**oom_score_adj 的计算**：

系统根据以下因素计算 `oom_score_adj`：
- 进程的优先级（foreground/visible/background）
- 进程的内存使用情况
- 进程的重要性标志（importance）

**示例**：

```bash
# 前台应用（Chrome）
$ adb shell cat /proc/$(pidof com.android.chrome)/oom_score_adj
0

# 后台应用
$ adb shell cat /proc/$(pidof com.example.background)/oom_score_adj
900

# 系统服务
$ adb shell cat /proc/$(pidof system_server)/oom_score_adj
-900
```

#### 2.3 内存压力等级

**压力等级分类**：

LMKD 通常将内存压力分为三个等级：

1. **Low (低压力)**
   - PSI some < 阈值（例如 5%）
   - 轻微的内存压力
   - 通常不需要立即行动

2. **Medium (中等压力)**
   - PSI some 达到中等阈值（例如 10%）
   - 明显的内存压力
   - 可能需要终止一些后台进程

3. **Critical (严重压力)**
   - PSI some 达到严重阈值（例如 20%）
   - 严重的内存压力
   - 需要立即终止进程释放内存

**压力阈值配置**：

通过系统属性配置：

```bash
# 查看当前配置
adb shell getprop | grep lmk

# 常见属性
ro.lmk.psi_partial_stall_ms=70      # PSI some 阈值（毫秒）
ro.lmk.psi_complete_stall_ms=500   # PSI full 阈值（毫秒）
ro.lmk.thrashing_limit=30          # 页面缓存抖动限制（百分比）
```

---

### 第三章：LMKD 基本配置

#### 3.1 系统属性配置

**关键系统属性**：

| 属性名 | 说明 | 默认值 | 影响 |
|--------|------|--------|------|
| `ro.lmk.use_psi` | 是否使用 PSI 监控 | `true` | 监控机制选择 |
| `ro.lmk.psi_partial_stall_ms` | PSI some 阈值（毫秒） | `70` | 低/中压力触发 |
| `ro.lmk.psi_complete_stall_ms` | PSI full 阈值（毫秒） | `500` (Go) / `150` (Non-Go) | 严重压力触发 |
| `ro.lmk.thrashing_limit` | 页面缓存抖动限制（%） | `30` (Go) / `100` (Non-Go) | 抖动检测 |
| `ro.lmk.thrashing_limit_decay` | 抖动阈值衰减（%） | `50` (Go) / `10` (Non-Go) | 抖动恢复 |
| `ro.lmk.kill_heaviest_task` | 是否优先杀最重任务 | `true` (Go) / `false` (Non-Go) | 杀进程策略 |

**查看当前配置**：

```bash
# 查看所有 LMKD 相关属性
adb shell getprop | grep lmk

# 查看特定属性
adb shell getprop ro.lmk.use_psi
adb shell getprop ro.lmk.psi_partial_stall_ms
```

**修改配置**（需要 root 权限）：

```bash
# 临时修改（重启后失效）
adb root
adb shell setprop ro.lmk.psi_partial_stall_ms 100

# 永久修改（需要修改系统属性文件）
# 在设备构建时修改 build.prop 或使用 overlay
```

#### 3.2 传统 minfree 配置（已废弃）

**注意**：Android 9+ 主要使用 PSI，但 minfree 配置可能仍然存在。

**minfree 配置**：

```bash
# 查看 minfree 配置
adb shell cat /sys/module/lowmemorykiller/parameters/minfree

# 格式：18432,23040,27648,32256,36864,46080
# 单位：页面数（通常 4KB/页）
# 含义：6 个内存水位线阈值
```

**minfree 水位线**：

| 水位线 | 说明 | 典型值（4GB 设备） |
|--------|------|------------------|
| 第 1 级 | 后台应用 | ~72MB |
| 第 2 级 | 感知服务 | ~90MB |
| 第 3 级 | 可见应用 | ~108MB |
| 第 4 级 | 前台应用 | ~126MB |
| 第 5 级 | 系统进程 | ~144MB |
| 第 6 级 | 关键进程 | ~180MB |

---

### 第四章：LMKD 监控和调试

#### 4.1 查看 LMKD 日志

**通过 logcat 查看**：

```bash
# 查看所有 LMKD 日志
adb logcat | grep lmkd

# 查看详细的 LMKD 日志
adb logcat -s lmkd:I

# 查看 LMKD 杀进程记录
adb logcat | grep "lowmemorykiller"
```

**典型日志格式**：

```
lmkd: Killing 'com.example.app' (pid 1234), adj 900, 
      to free 50MB, reason: low_psi
```

**日志字段说明**：
- `Killing 'xxx'`：被终止的进程名称
- `pid 1234`：进程 ID
- `adj 900`：oom_score_adj 值
- `to free 50MB`：预期释放的内存
- `reason`：终止原因（low_psi, minfree, thrashing 等）

#### 4.2 监控 PSI 压力

**查看 PSI 数据**：

```bash
# 查看内存压力
adb shell cat /proc/pressure/memory

# 输出示例：
# some avg10=5.23 avg60=2.45 avg300=1.23 total=12345678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**PSI 输出解读**：
- `some`：部分压力（至少一个任务被阻塞）
- `full`：完全压力（所有任务都被阻塞）
- `avg10/60/300`：过去 10/60/300 秒的平均压力百分比
- `total`：总阻塞时间（微秒）

**实时监控脚本**：

```bash
#!/bin/bash
# monitor_psi.sh

while true; do
    clear
    echo "=== PSI Memory Pressure ==="
    adb shell cat /proc/pressure/memory
    echo ""
    echo "=== LMKD Logs (last 5) ==="
    adb logcat -d | grep lmkd | tail -5
    sleep 2
done
```

#### 4.3 查看进程内存使用

**使用 dumpsys meminfo**：

```bash
# 查看系统内存概览
adb shell dumpsys meminfo

# 查看特定进程内存
adb shell dumpsys meminfo com.example.app

# 查看所有进程内存（按 PSS 排序）
adb shell dumpsys meminfo | grep -A 1 "Total PSS"
```

**关键指标**：
- **PSS (Proportional Set Size)**：进程实际占用的物理内存
- **RSS (Resident Set Size)**：进程占用的物理内存（包含共享）
- **USS (Unique Set Size)**：进程独占的物理内存

#### 4.4 检查 LMKD 进程状态

**查看 LMKD 进程**：

```bash
# 查看 LMKD 进程信息
adb shell ps -A | grep lmkd

# 查看 LMKD 的 oom_score_adj（应该是 -1000）
adb shell cat /proc/$(pidof lmkd)/oom_score_adj

# 查看 LMKD 的调度策略（应该是 SCHED_FIFO）
adb shell cat /proc/$(pidof lmkd)/sched
```

---

### 第五章：常见问题排查

#### 5.1 LMKD 没有启动

**症状**：
- `ps -A | grep lmkd` 没有输出
- 内存压力时没有进程被杀死

**排查步骤**：

```bash
# 1. 检查 LMKD 服务状态
adb shell getprop init.svc.lmkd

# 2. 查看系统日志
adb logcat | grep lmkd

# 3. 检查 SELinux 策略
adb shell getenforce
adb shell dmesg | grep avc | grep lmkd
```

**解决方案**：
- 检查设备配置是否启用了 LMKD
- 检查 SELinux 策略是否允许 LMKD 运行
- 查看系统日志中的错误信息

#### 5.2 LMKD 杀进程过于频繁

**症状**：
- 应用频繁被杀死
- 用户抱怨应用无法保持后台运行

**排查步骤**：

```bash
# 1. 查看 LMKD 日志频率
adb logcat | grep "Killing" | wc -l

# 2. 检查 PSI 压力
adb shell cat /proc/pressure/memory

# 3. 检查系统内存使用
adb shell dumpsys meminfo | head -20
```

**可能原因**：
- PSI 阈值设置过低
- 系统内存不足
- 内存泄漏导致内存持续增长

**解决方案**：
- 提高 PSI 阈值（增加 `ro.lmk.psi_partial_stall_ms`）
- 检查是否有内存泄漏
- 优化应用内存使用

#### 5.3 LMKD 不杀进程

**症状**：
- 内存压力很高但进程没有被杀死
- 系统卡顿或 ANR

**排查步骤**：

```bash
# 1. 检查 LMKD 是否运行
adb shell ps -A | grep lmkd

# 2. 检查 PSI 监控是否正常
adb shell cat /proc/pressure/memory

# 3. 检查进程的 oom_score_adj
adb shell ps -eo pid,comm,oom_score_adj | sort -k3 -n
```

**可能原因**：
- LMKD 没有正确监控 PSI
- 所有进程的 oom_score_adj 都很低
- PSI 阈值设置过高

**解决方案**：
- 检查 PSI 接口是否可用
- 降低 PSI 阈值
- 检查进程优先级设置

---

## 📝 学习检查点

完成基础学习后，您应该能够：

- [ ] 理解 LMKD 的基本概念和作用
- [ ] 理解 LMKD 与内核 LMK 的区别
- [ ] 掌握 oom_score_adj 的含义和使用
- [ ] 能够配置基本的 LMKD 参数
- [ ] 能够监控 LMKD 的运行状态
- [ ] 能够查看 PSI 压力数据
- [ ] 能够排查基本的 LMKD 问题

---

**下一步**：完成基础学习后，可以进入 `02_Advanced` 学习进阶内容，深入理解 LMKD 的源码实现和高级配置。
