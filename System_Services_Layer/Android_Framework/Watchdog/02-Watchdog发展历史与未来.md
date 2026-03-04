# Watchdog 发展历史与未来

## 学习目标

- 了解 Android Watchdog 的起源与在 AOSP 中的引入时间
- 掌握超时策略的演进：从固定 1 分钟到可变超时、检查间隔等
- 理解当前默认与可配置超时的大致形态
- 了解可能的未来方向：可观测性、策略可调、与系统健康/PSI 结合等

## 一、起源与引入

### 时间线

- **约 2008 年**：Android 源码树中已出现 Watchdog 相关实现，用于在 system_server 内检测核心服务是否无响应。
- **早期定位**：作为 SystemServer 的“最后一根保险丝”，当关键服务线程长时间不处理消息或持锁过久时，主动杀 system_server，由 Init 重启，避免整机假死。

内核侧 soft/hard lockup 检测与 `/dev/watchdog` 来自 Linux 内核的看门狗子系统；用户态 **watchdogd** 在 Android 启动早期由 Init 拉起，负责定期向 `/dev/watchdog` 喂狗，二者与 Java Watchdog 分层配合，形成“应用/系统/内核”多级防护。

## 二、超时策略演进

### 早期：固定 1 分钟

- 早期 Java Watchdog 对**所有**被监控的线程采用**统一的 1 分钟超时**。
- 逻辑简单，但容易产生误杀：某些合法长耗时操作（如大包安装、大量文件操作）可能超过 1 分钟，导致本不该触发的 Watchdog 超时。

### KitKat 及以后：可变超时与检查间隔

- **KitKat（Android 4.4）及之后**：引入**可变超时**机制，不同 Handler/服务可以配置不同的超时时间，而不是全局固定 1 分钟。
- **目的**：支持长时间操作（如数 GB 应用安装）而不误触发 Watchdog，同时仍对“本当快速响应”的线程保持较短超时。
- **检查间隔**：Watchdog 线程本身的检查周期演进为约 **30 秒** 一次（不同版本可能略有差异），即每 30 秒对已注册的 HandlerChecker/Monitor 做一轮检查；单个 Handler 的超时时间可以大于 30 秒，但不会小于检查间隔带来的“最小可见超时”。

### 当前形态概览

- **Java Watchdog**：  
  - 以约 30 秒为周期进行一轮检查。  
  - 各 HandlerChecker 可有各自超时（如 60 秒或更长），Monitor 通过 `monitor()` 做轻量健康检查。  
  - 超时后：打栈、写 "SERVICE TIMEOUT" 等日志、杀 system_server。
- **内核/watchdogd**：  
  - watchdogd 按配置间隔（如 10 秒）向 `/dev/watchdog` 写数据喂狗；若停止喂狗，内核在设定超时后触发重启。  
  - 内核 soft/hard lockup 检测与喂狗相互独立，用于发现调度器或 CPU 长时间不进展的问题。

具体常量与配置以 AOSP 当前源码为准（如 `frameworks/base/services/core/java/com/android/server/Watchdog.java`），不同 Android 版本可能微调。

## 三、未来方向（简述）

以下为基于常见需求的合理演进方向，并非 AOSP 已承诺的路线图，仅供建立“未来可能怎样”的认知：

- **可观测性增强**：在超时前后记录更结构化的诊断信息（如 trace、PSI 快照、锁依赖），便于定位持锁、Binder 阻塞、主线程耗时等根因，与 [06-Watchdog超时排查与调优](06-Watchdog超时排查与调优.md) 中的排查手段形成闭环。
- **策略可调**：在保持安全的前提下，允许通过属性或配置对不同服务/线程的超时时间、是否允许多次“宽限”再杀进程等进行调优，兼顾稳定性与对长耗时任务的容忍度。
- **与系统健康/PSI 结合**：将 Watchdog 与内存/IO/CPU 压力（如 PSI）结合，在触发杀进程或打栈时附带资源压力信息，帮助区分“单纯卡死”与“资源争用导致的卡顿”。
- **分层与降级**：继续明确“应用 ANR → 系统 Watchdog → 内核重启”的分层，避免过度依赖整机重启，优先通过杀 system_server 由 Init 重启恢复。

实际演进以 AOSP 与各厂商实现为准。

## 四、小结

- **起源**：约 2008 年 AOSP 中引入，用于检测 system_server 内核心服务无响应并触发恢复。
- **超时演进**：从固定 1 分钟 → KitKat 起可变超时、约 30 秒检查间隔，以支持长耗时操作并减少误杀。
- **未来**：可观测性、策略可调、与 PSI/系统健康结合、分层恢复等方向有助于更好排查与更可控的恢复策略。

下一篇文章将介绍 [多层 Watchdog 架构](03-多层Watchdog架构.md)，明确内核 Watchdog、watchdogd 与 Java Watchdog 三层如何配合。
