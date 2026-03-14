# Stability Matrix Course

面向 **Android 稳定性架构师** 的体系化技术知识库。从 Linux 内核到 Framework、从运行时到应用层，按「定位 → 边界与交互 → 核心机制 → 风险地图 → 诊断与治理」的逻辑链组织，**言必有据、关联实战**，支持在线上 Crash/ANR/OOM 等场景下快速定位根因并给出治理方案。

---

## 工程结构

```
StabilityMatrixCourse/
├── Linux_Kernel_Layer/      # Linux 内核层
├── Android_Runtime_Layer/  # Android 运行时层
├── Android_Framework_Layer/ # Android Framework 层
├── Application_Layer/       # 应用层
├── Architecture_Governance/  # 架构与治理（跨层）
├── Tools_Mastery/           # 工具精通（跨层）
├── PROMPT-技术系列文章写作指南.md   # 系列文章写作规范与提示词
└── README.md                # 本文件
```

---

## 各层目录与系列概览

### Linux_Kernel_Layer（Linux 内核层）

| 子目录 | 说明 | 系列 README |
|--------|------|-------------|
| **Architecture** | Android 内核架构总览 | — |
| **Binder** | Binder IPC：驱动、调用链路、内存/线程/生命周期 | [README-Binder系列](Linux_Kernel_Layer/Binder/README-Binder系列.md) |
| **DM** | Device Mapper 块设备抽象 | readme.md |
| **FS** | 文件系统（VFS、ext4、f2fs 等） | [README](Linux_Kernel_Layer/FS/README.md) |
| **GKI** | 通用内核镜像与兼容性 | — |
| **Input_Driver** | 输入设备驱动与事件上报 | [README](Linux_Kernel_Layer/Input_Driver/README.md) |
| **Interrupt** | 中断机制 | readme.md |
| **IO** | 块 IO、调度、与稳定性 | [README-IO系列](Linux_Kernel_Layer/IO/README-IO系列.md) |
| **Memory_Management** | 内存管理、LMKD、PSI、OOM 等 | [README](Linux_Kernel_Layer/Memory_Management/README.md) |
| **Process** | 进程创建、调度、状态 | [README](Linux_Kernel_Layer/Process/README.md) |
| **Syscalls** | 系统调用分类与安全 | — |

### Android_Runtime_Layer（Android 运行时层）

| 子目录 | 说明 | 系列 README |
|--------|------|-------------|
| **ART** | ART 总览、字节码、编译执行、类加载、内存/GC、JNI、信号与 ANR Trace、启动 | [README-ART系列](Android_Runtime_Layer/ART/README-ART系列.md) |
| **Java_Crash** | Java Exception（JE）机制、传播、默认处理、堆栈与上报、风险与治理 | [README-JE系列](Android_Runtime_Layer/Java_Crash/README-JE系列.md) |
| **Native_Crash** | Native Crash、信号、debuggerd/Tombstone、栈回溯、ASan/APM | [README-NativeCrash系列](Android_Runtime_Layer/Native_Crash/README-NativeCrash系列.md) |

### Android_Framework_Layer（Framework 层）

| 子目录 | 说明 | 系列 README |
|--------|------|-------------|
| **ANR_Detection** | ANR 检测（Input/Broadcast/Service 等）、traces 生成 | — |
| **AOSP_Startup** | 系统架构演进、Init、根目录、存储与文件系统 | — |
| **Broadcast** | 广播 ANR、有序/无序、BroadcastQueue | — |
| **Build_System** | AOSP 构建、AVB 与签名、分区调试 | [README](Android_Framework_Layer/Build_System/README.md) |
| **Dumpsys** | dumpsys 使用与解读 | — |
| **Dynamic_Updates** | 可更新分区等 | — |
| **Input** | Input 事件链路、EventHub、InputChannel、View 分发、Input ANR | [README-Input系列](Android_Framework_Layer/Input/README-Input系列.md) |
| **Partition_System** | 分区布局与结构 | — |
| **PKMS** | 包管理服务 | — |
| **Service** | Service 与 ANR、生命周期 | — |
| **System_Integration** | 系统组成与启动、分区挂载 | — |
| **Watchdog** | Java Watchdog、内核 watchdog、实战 | [README](Android_Framework_Layer/Watchdog/README.md) |
| **Window** | 窗口系统、Surface、SurfaceFlinger、GraphicBuffer | [README](Android_Framework_Layer/Window/README.md) |

### Application_Layer（应用层）

| 子目录 | 说明 | 系列 README |
|--------|------|-------------|
| **Handler_MessageQueue_Looper** | Handler 消息机制、与 ANR/卡顿的关系 | [README-Handler系列](Application_Layer/Handler_MessageQueue_Looper/README-Handler系列.md) |

### Architecture_Governance（架构与治理）

跨层：监控体系、兜底策略、自动化测试、容灾、最佳实践、案例分析等。

| 子目录 | 说明 |
|--------|------|
| **Monitoring_System** | 监控体系 |
| **Fallback_Strategy** | 兜底策略 |
| **Automated_Testing** | 自动化测试 |
| **Disaster_Recovery** | 容灾 |
| **Best_Practices** | 最佳实践 |
| **Case_Analysis** | 案例分析（如 ANR/IO/内存退化等） |

### Tools_Mastery（工具精通）

跨层：调试、追踪、内存分析、内核工具、Git、Android 工具等。

| 子目录 | 说明 | 系列 README |
|--------|------|-------------|
| **Android_Tools** | Android 常用工具 | [README](Tools_Mastery/Android_Tools/README.md) |
| **Debugging** | 调试方法 | — |
| **Git_Mastery** | Git 进阶与协作 | [README](Tools_Mastery/Git_Mastery/README.md) |
| **Kernel_Tools** | 内核相关工具 | — |
| **Memory_Analysis** | 内存分析（如 PSI） | — |
| **Tracing** | ftrace 等追踪 | — |
| **Automation** | 自动化脚本与流程 | — |

---

## 写作规范与使用方式

- **系列文章标准**：见 [PROMPT-技术系列文章写作指南.md](PROMPT-技术系列文章写作指南.md)。要求自顶向下、言必有据（标注 AOSP/内核源码路径）、关联实战（Crash/ANR/OOM 等）、文末 1～2 个完整排查案例，单篇不少于约 300 行。
- **阅读建议**：各系列在对应目录的 `README-xxx系列.md` 或 `README.md` 中会说明「为什么要写」「设计思路」「篇章列表」和「优先阅读顺序」，可按需从总览篇或风险/实战篇切入。
- **层间关系**：不少问题会跨层（如 ANR 涉及 Input、Binder、进程调度、Watchdog；Crash 涉及 JE/NE、ART、信号、APM）。各系列 README 中会注明与其它系列的衔接点，便于按问题链路跳转阅读。

---

## 快速入口（按问题类型）

| 关注问题 | 建议入口 |
|----------|----------|
| **Java Crash（JE）** | [Android_Runtime_Layer/Java_Crash](Android_Runtime_Layer/Java_Crash) → 01 总览、06 风险全景、07 诊断与治理 |
| **Native Crash（NE）** | [Android_Runtime_Layer/Native_Crash](Android_Runtime_Layer/Native_Crash) → 01 总览、06 Tombstone 解读、08 APM |
| **ANR** | [Android_Framework_Layer/ANR_Detection](Android_Framework_Layer/ANR_Detection)、[Input](Android_Framework_Layer/Input)、[ART 09-信号机制与 ANR Trace](Android_Runtime_Layer/ART/09-信号机制与ANR-Trace.md) |
| **Binder 相关** | [Linux_Kernel_Layer/Binder](Linux_Kernel_Layer/Binder) → 01 总览、07 风险全景、08 诊断与治理 |
| **OOM / 内存** | [Android_Runtime_Layer/ART](Android_Runtime_Layer/ART) 05 内存与 GC、[Linux_Kernel_Layer/Memory_Management](Linux_Kernel_Layer/Memory_Management) |
| **卡顿 / 消息队列** | [Application_Layer/Handler_MessageQueue_Looper](Application_Layer/Handler_MessageQueue_Looper)、Input 系列 |
| **Watchdog / 死锁** | [Android_Framework_Layer/Watchdog](Android_Framework_Layer/Watchdog) |
| **写作/扩展本仓库** | [PROMPT-技术系列文章写作指南.md](PROMPT-技术系列文章写作指南.md) |

---

## 许可与贡献

本仓库为技术学习与沉淀用，各系列文章可独立阅读或与 AOSP/内核源码对照使用。新增或修改系列时，建议遵循《技术系列文章写作指南》并同步更新本 README 与对应层的系列 README。
