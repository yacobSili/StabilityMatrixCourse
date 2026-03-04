# 内核 Watchdog 与 watchdogd

## 学习目标

- 理解内核 Watchdog 的职责：soft/hard lockup 检测与 `/dev/watchdog` 设备
- 掌握用户态 **watchdogd** 如何打开、配置并周期喂 `/dev/watchdog`
- 理解“未喂狗”时的后果：内核触发整机重启
- 与 [process/19-用户态与内核态深入解析](../../process/19-用户态与内核态深入解析.md) 中“watchdog/*：检测 CPU 死锁”的呼应

## 一、内核 Watchdog 概述

### 位置与职责

- **源码位置**：Linux 内核中一般为 `kernel/watchdog.c`，以及各平台提供的 `/dev/watchdog` 设备驱动。
- **职责**：  
  1. **Soft/Hard lockup 检测**：检测是否有 CPU 长时间不调度、长时间关中断等，发现则报错或触发重启（视配置）。  
  2. **看门狗设备**：管理 `/dev/watchdog`；若在设定时间内未收到用户态“喂狗”（如 write），则触发重启或 panic。

与用户态/内核态边界的关系可参考 [process/19-用户态与内核态深入解析](../../process/19-用户态与内核态深入解析.md) 中关于“watchdog/*：检测 CPU 死锁”的提及：内核 watchdog 运行在内核态，用于检测调度器或 CPU 长时间不进展的问题。

### Soft lockup 与 Hard lockup（概念）

- **Soft lockup**：某 CPU 上长时间没有调度到其他任务（例如某内核线程或进程长时间占用 CPU 且不让出），内核 watchdog 线程检测到“该 CPU 长时间不进展”后报错或触发恢复。
- **Hard lockup**：通常指某 CPU 长时间关中断或无法响应中断，导致调度器无法在该 CPU 上运行；NMI 或类似机制可检测并报错。

具体实现与配置因内核版本和平台而异；Android 使用的 GKI/内核与主线内核的 watchdog 子系统一致，详见内核文档与 `kernel/watchdog.c`。

### `/dev/watchdog` 设备

- **作用**：用户态通过打开并**周期性地向该设备写数据**（“喂狗”）表示“用户态仍在运行”；若在设定超时时间内未收到 write，内核认为用户态可能卡死，触发**整机重启**（或 panic，视驱动与配置）。
- **典型用法**：  
  - `open("/dev/watchdog", O_RDWR)`  
  - `ioctl(fd, WDIOC_SETTIMEOUT, &timeout_sec)` 设置超时（秒）  
  - 循环：`write(fd, "", 1); sleep(interval);`  
  - 若进程退出或停止 write，超时后内核看门狗触发重启。

## 二、watchdogd 守护进程

### 位置与启动

- **源码位置**：  
  - 独立模块：`system/core/watchdogd/`（若存在）  
  - 或集成在 init：`system/core/init/watchdogd.cpp`（视 AOSP 版本）
- **启动**：由 **Init** 在启动早期按 init.rc 配置启动；启动时间会记录在属性 **ro.boottime.watchdogd** 中，可用于分析冷启动耗时，见 [17-Android冷启动时间线属性详解](../17-Android冷启动时间线属性详解.md)。

### 工作流程

1. **打开设备**：`open("/dev/watchdog", O_RDWR)` 获取 fd。  
2. **设置超时**：通过 `WDIOC_SETTIMEOUT` ioctl 设置看门狗超时时间，通常为 `interval + margin`（例如默认 interval 与 margin 各 10 秒，则超时 20 秒）。  
3. **周期喂狗**：进入循环，每隔一定时间（如 10 秒）执行一次 `write(fd, "", 1)`，然后 `sleep(interval)`。  
4. **超时调整**：若通过 `WDIOC_GETTIMEOUT` 发现内核返回的超时与预期不同，可调整喂狗间隔，保证在超时前完成喂狗。

若 watchdogd 崩溃、被杀死或整个用户态卡死无法调度到 watchdogd，则无人对 `/dev/watchdog` 进行 write；内核在超时后触发**整机重启**，作为最后一道防线。

### 与 Java Watchdog 的区分

- **watchdogd**：只负责“代表用户态”定期喂 `/dev/watchdog`；不监控 system_server 内具体服务。  
- **Java Watchdog**：只监控 system_server 内服务/线程；不直接操作 `/dev/watchdog`。  
- 若 **system_server 卡死**但其他进程（包括 watchdogd）仍能运行，watchdogd 会继续喂狗，**不会**触发整机重启；此时由 **Java Watchdog** 超时后杀 system_server，Init 重启 system_server。  
- 若**整个用户态卡死**（包括 watchdogd 得不到调度），则无人喂狗，**内核**看门狗超时后**整机重启**。

## 三、小结

- **内核 Watchdog**：`kernel/watchdog.c` 及 `/dev/watchdog` 驱动；负责 soft/hard lockup 检测与“未喂狗”超时；超时后整机重启（或 panic）。  
- **watchdogd**：用户态守护进程，由 Init 启动；打开 `/dev/watchdog`、设置超时、周期 write 喂狗；若停止喂狗，内核看门狗超时导致整机重启。  
- **与 process/19 的呼应**：内核 watchdog 用于检测 CPU/调度器长时间不进展（死锁/卡死），详见 [19-用户态与内核态深入解析](../../process/19-用户态与内核态深入解析.md)。

下一篇文章将介绍 [Watchdog 超时排查与调优](06-Watchdog超时排查与调优.md)，包括典型日志、trace 与常见根因。
