# Watchdog 超时排查与调优

## 学习目标

- 能根据典型日志识别 Watchdog 超时（含 half_watchdog 与完整 watchdog）
- 能在 trace 中定位 watchdog.monitor、BinderThreadMonitor 等关键线程与阻塞点
- 理解常见根因：主线程/关键线程持锁过久、Binder 阻塞、服务主线程做重活
- 掌握基本排查步骤与可调参数（如有），并与 process、window、input 等系列交叉引用

## 一、典型日志

### 1. 完整 Watchdog 超时（杀 system_server）

- **`*** WATCHDOG KILLING SYSTEM PROCESS ***`**：表示 Watchdog 已判定超时并即将杀死 system_server。
- **`SERVICE TIMEOUT`**：表示某被监控的服务/线程在超时时间内未完成检查，通常伴随被阻塞的 Handler 或 Monitor 名称与时长。

示例（含义类似）：

```text
*** WATCHDOG KILLING SYSTEM PROCESS ***
SERVICE TIMEOUT in com.android.server...
Blocked in monitor com.android.server.Watchdog$BinderThreadMonitor on monitor thread (watchdog.monitor) for 64s
```

### 2. Half-watchdog（未杀进程）

- **half_watchdog**：表示已超过“pre-watchdog”时间（一般为完整超时的一半，如 30 秒），但尚未超过完整超时（如 60 秒）；此时通常只打日志/事件，**不杀** system_server。
- 在 trace 或 event 中可能看到类似：

```text
half_watchdog
  subject: "Blocked in monitor com.android.server.Watchdog$BinderThreadMonitor on monitor thread (watchdog.monitor) for 15s"
```

若后续该线程仍不完成检查，下一轮会升级为完整 **watchdog** 并杀进程。

### 3. 日志中的关键信息

- **被阻塞的线程名**：如 `monitor thread (watchdog.monitor)`、`main thread`、`foreground thread` 等，对应 [04-Java-Watchdog设计与实现](04-Java-Watchdog设计与实现.md) 中的 HandlerChecker 名称。
- **被阻塞的 Monitor**：如 `Watchdog$BinderThreadMonitor` 表示 Binder 线程可用性检查阻塞；其他服务名表示该服务的 `monitor()` 未在超时内返回。
- **阻塞时长**：如 `for 64s`、`for 15s`，可用于与超时阈值对照。

## 二、Trace 中的关键点

### 1. 线程名与栈

- **watchdog.monitor**：执行所有通过 `addMonitor()` 注册的 Monitor（含 BinderThreadMonitor 及各服务的 `monitor()`）；若 Watchdog 报“Blocked in monitor ... on monitor thread (watchdog.monitor)”，说明该线程在某个 `monitor()` 调用中阻塞。
- **BinderThreadMonitor**：内部调用 `Binder.blockUntilThreadAvailable()`；若 Binder 线程池被占满或长时间不释放，会卡在这里。
- **main / foreground / i/o / display / animation 等**：对应各 HandlerChecker；若某线程长时间持锁或执行重逻辑，其对应 HandlerChecker 会 OVERDUE。

在 systrace/perfetto 或 Java 栈 dump 中：

- 找到名为 `watchdog` 的线程（Watchdog 主循环）和名为 `watchdog.monitor` 的线程。
- 查看 **watchdog.monitor** 的栈：若停在某 `monitor()` 或 `blockUntilThreadAvailable()`，即阻塞点。
- 结合 Binder 事务与锁竞争（见 [process](../../process/)、[window](../../window/) 系列）分析谁占用了 Binder 或锁。

### 2. 与 Binder 的关联

Watchdog 超时常伴随 **Binder 拥堵**：某进程向 system_server 发起大量同步 Binder 调用，或 system_server 内某线程持锁导致 Binder 线程无法处理新请求，进而 `Binder.blockUntilThreadAvailable()` 一直等不到可用线程。在 trace 或 `dumpsys binder` 中可看到：

- 大量 pending 或长时间未完成的 transaction（如本仓库 `binder/logs/SWT_JBT_TRACES` 中 `elapsed 59717ms`、`elapsed 68296ms` 等）。
- 被阻塞在 `blockUntilThreadAvailable` 的 watchdog.monitor 栈。

可结合 Binder 与进程调度（[process/18-Android Binder与进程调试](../../process/18-Android%20Binder与进程调试.md)）分析调用链与优先级。

## 三、常见根因

| 类型 | 表现 | 方向 |
|------|------|------|
| **主线程/关键线程持锁过久** | 某 HandlerChecker 对应线程（如 main、foreground）在超时内未执行完 post 的 Runnable；栈上可见某锁 wait 或长时间运行逻辑 | 检查该线程栈、锁持有者；避免在主线程或关键服务线程做重活或长时间持锁 |
| **Binder 阻塞** | 日志/trace 显示 BinderThreadMonitor、`blockUntilThreadAvailable`；Binder 线程池满或某 Binder 调用长时间不返回 | 查 Binder transaction 耗时、调用方与接收方；优化同步调用、避免 system_server 内阻塞 |
| **某服务 monitor() 卡住** | 阻塞在某个具体 Monitor 实现类（非 BinderThreadMonitor）；栈为该服务的 `monitor()` | 检查该服务 `monitor()` 实现（如是否拿锁、是否调 Binder）；简化 monitor 逻辑，避免阻塞 |
| **长时间安装/升级等** | 若未对对应线程设置更长超时，大包安装等可能触发误杀 | 确认是否已用自定义超时（见 [04-Java-Watchdog设计与实现](04-Java-Watchdog设计与实现.md)）；必要时为长耗时操作申请更长超时或 pause 检查 |

## 四、排查步骤建议

1. **确认类型**：从 logcat/trace 区分是 half_watchdog（仅告警）还是完整 watchdog（已杀 system_server）。
2. **定位阻塞点**：根据日志中的 “Blocked in monitor … on … thread … for Xs” 确定是哪个线程、哪个 Monitor；在对应时间点的 trace 中查该线程栈。
3. **分析栈与锁**：看该线程在等什么锁、在做什么调用；若为 Binder，结合 `dumpsys binder` 与 transaction 耗时分析调用链。
4. **结合上下游**：若涉及 WMS/IMS/AMS，可参考 [window/07-WindowManagerService架构详解](../../window/07-WindowManagerService架构详解.md)、[input/10-InputManagerService详解](../../input/10-InputManagerService详解.md) 理解服务职责与线程模型，判断是否为锁顺序或主线程耗时导致。
5. **可调参数**：部分版本支持通过属性或设置调节 Watchdog 超时（如 `mWatchdogTimeoutMillis` 的配置）；修改需谨慎，避免掩盖真实卡顿或引入误杀。

## 五、与各系列的关系

- **process**：用户态/内核态、Binder 调用与调度；Watchdog 超时常与某线程调度、持锁或 Binder 阻塞相关，见 [18-Android Binder与进程调试](../../process/18-Android%20Binder与进程调试.md)、[19-用户态与内核态深入解析](../../process/19-用户态与内核态深入解析.md)。
- **window**：WMS 实现 `Watchdog.Monitor`，其主线程或关键线程若持锁过久会影响 Watchdog 检查，见 [07-WindowManagerService架构详解](../../window/07-WindowManagerService架构详解.md)。
- **input**：IMS 实现 `Watchdog.Monitor` 并注册到 Watchdog；输入链路阻塞可能间接导致 Binder 或主线程拥堵，见 [10-InputManagerService详解](../../input/10-InputManagerService详解.md)。

## 六、小结

- **日志**：关注 `*** WATCHDOG KILLING SYSTEM PROCESS ***`、`SERVICE TIMEOUT`、half_watchdog/watchdog 的 subject（线程名、Monitor 名、时长）。
- **Trace**：重点看 **watchdog.monitor** 与各 Handler 线程栈；Binder 事务耗时与 `blockUntilThreadAvailable` 常与 BinderThreadMonitor 超时相关。
- **根因**：主线程/关键线程持锁过久、Binder 阻塞、某服务 `monitor()` 卡住、长耗时操作未申请更长超时等。
- **排查**：先区分 half/完整 watchdog，再根据线程与 Monitor 定位栈与锁，结合 Binder 与各服务文档分析并优化。

本系列文章到此结束；整体索引与学习路径见 [README](README.md)。
