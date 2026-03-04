# Watchdog 系统挂死分析

## 📋 模块概述

Watchdog 是 Android 系统用于监控 SystemServer 健康状态的机制。当系统服务长时间无响应时，Watchdog 会触发系统重启。本模块将深入分析 Watchdog 机制、SystemServer 服务依赖、以及系统级挂死问题的分析方法。

## 🎯 学习目标

- 深入理解 Watchdog 的监控机制和触发条件
- 掌握 SystemServer 服务架构和依赖关系
- 能够分析 Binder 线程池阻塞导致的系统挂死
- 能够从 Kernel 层分析 hung task 问题
- 能够给出系统级稳定性问题的解决方案

## 📚 学习路径

### 01_Theory - Watchdog基础理论

**学习内容**：
- Watchdog 的作用和机制
- SystemServer 架构和服务依赖
- Binder 线程池机制
- Watchdog 监控的线程和服务
- 系统服务之间的依赖关系

**关键概念**：
- **HandlerChecker**: Watchdog 监控的核心组件
- **Monitor**: 被监控的服务接口
- **前台/后台策略**: Watchdog 在不同场景下的监控策略
- **超时时间**: 默认 60 秒（可配置）

**学习产出**：
- Watchdog 原理分析文档
- SystemServer 服务架构图

---

### 02_Framework_SystemServer - SystemServer Watchdog实现

**学习内容**：
- Watchdog 类的实现机制
- `HandlerChecker` 的工作原理
- Watchdog 监控的服务列表
- Watchdog 超时后的处理流程
- SystemServer 重启机制

**关键源码**：
- `frameworks/base/services/core/java/com/android/server/Watchdog.java`
- `HandlerChecker` 类实现
- `Monitor` 接口定义
- SystemServer 服务注册机制

**监控的服务**：
- ActivityManagerService
- WindowManagerService
- PowerManagerService
- PackageManagerService
- 等等

**学习产出**：
- SystemServer Watchdog 源码分析文档
- 服务依赖关系图

---

### 03_Kernel_Hung_Task - Kernel层hung task检测

**学习内容**：
- Kernel hung task 检测机制
- `/proc/sys/kernel/hung_task_timeout_secs` 配置
- dmesg 中的 hung task 日志分析
- RCU 机制与 hung task 的关系
- soft lockup 和 hard lockup

**关键知识点**：
- **hung task**: 处于 D 状态（不可中断睡眠）超过阈值的任务
- **RCU Stall**: RCU 机制导致的系统挂起
- **lockup**: CPU 被长时间占用导致的系统无响应

**Kernel 配置**：
- `CONFIG_DETECT_HUNG_TASK`
- `CONFIG_SOFTLOCKUP_DETECTOR`
- `CONFIG_HARDLOCKUP_DETECTOR`

**学习产出**：
- Kernel hung task 机制分析文档
- dmesg 日志分析指南

---

### 04_Reproduction - Watchdog复现代码

**复现场景**：

1. **SystemServer Watchdog**
   - 系统服务死锁
   - 系统服务长时间阻塞
   - Binder 线程池耗尽

2. **Binder 线程池阻塞**
   - Binder 调用死锁
   - Binder 调用超时
   - Binder 线程被长时间占用

3. **系统服务依赖死锁**
   - 服务 A 等待服务 B，服务 B 等待服务 A
   - 循环依赖导致的死锁

4. **Kernel hung task**
   - IO 操作长时间阻塞（D 状态）
   - 驱动问题导致的挂死

**代码结构**：
```
04_Reproduction/
├── SystemServerWatchdog/    # SystemServer Watchdog场景
├── BinderThreadPoolBlock/    # Binder线程池阻塞场景
├── ServiceDeadlock/          # 服务死锁场景
└── KernelHungTask/           # Kernel hung task场景
```

**学习产出**：
- 各种 Watchdog 场景的复现代码
- 复现步骤说明文档

---

### 05_Analysis_Tools - Watchdog分析工具

**工具学习**：

1. **dmesg 分析**
   - Watchdog 日志提取
   - hung task 日志分析
   - lockup 日志分析

2. **/proc 文件系统分析**
   - `/proc/pid/stat` - 进程状态
   - `/proc/pid/stack` - 进程堆栈
   - `/proc/pid/wchan` - 等待的 channel

3. **Binder 分析工具**
   - `dumpsys binder` - Binder 状态
   - Binder 调用统计
   - Binder 线程状态

4. **系统服务分析**
   - `dumpsys` 命令使用
   - 服务状态检查
   - 服务依赖分析

5. **自动化分析脚本**
   - dmesg 日志解析
   - hung task 自动检测
   - 分析报告生成

**学习产出**：
- 工具使用指南
- 自动化分析脚本
- 分析案例报告

---

### 06_Solution_Governance - 解决方案与治理

**解决方案**：

1. **代码层面**
   - 避免系统服务死锁
   - 优化 Binder 调用
   - 使用超时机制
   - 资源隔离

2. **架构层面**
   - 服务依赖优化
   - 异步处理机制
   - 服务降级策略
   - 容错机制设计

3. **监控与预防**
   - Watchdog 监控体系
   - 系统服务健康检查
   - 实时告警机制
   - 自动化测试

4. **Kernel 层面**
   - 驱动稳定性优化
   - IO 阻塞问题解决
   - RCU 机制优化

**学习产出**：
- Watchdog 解决方案文档
- 系统稳定性监控体系设计
- 最佳实践总结

---

## 🔄 KTAS 学习流程

### Knowledge（知识）
1. 阅读 Watchdog 相关源码
2. 理解 SystemServer 服务架构
3. 学习 Kernel hung task 机制

### Trigger（触发）
1. 在 Playground 中复现各种 Watchdog 场景
2. 收集 dmesg、/proc 等日志

### Analyze（分析）
1. 使用工具分析 Watchdog 日志
2. 定位系统挂死根因
3. 分析服务依赖关系

### Solution（解决）
1. 给出修复方案
2. 设计系统级预防措施
3. 建立监控体系

---

## 📖 推荐资源

### 源码阅读
- AOSP: `frameworks/base/services/core/java/com/android/server/Watchdog.java`
- AOSP: `frameworks/base/services/java/com/android/server/SystemServer.java`
- Linux Kernel: `kernel/watchdog.c`

### 文档
- Android 官方文档：SystemServer
- Linux Kernel 文档：hung task detection

### 工具
- dmesg
- dumpsys
- /proc 文件系统
- ftrace

---

## ✅ 学习检查点

完成本模块学习后，你应该能够：

- [ ] 理解 Watchdog 的完整监控机制
- [ ] 能够从 dmesg 中分析 hung task 问题
- [ ] 能够分析 SystemServer 服务依赖关系
- [ ] 能够复现至少 2 种不同的 Watchdog 场景
- [ ] 能够给出系统级稳定性问题的解决方案

---

**下一步**：完成 Watchdog 模块学习后，进入 `03_Native_Crash_NE` 模块。
