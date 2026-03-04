# Android 稳定性与 ACK 学习工程

本工程按 **安卓系统架构分层** 组织学习内容，融合原 ACKLearning 与 stabilityLearning，从应用层到底层内核与工具，面向稳定性工程师与架构师成长。

## 架构分层（自上而下）

| 层级 | 目录 | 说明 |
|------|------|------|
| 1 | [Application_Layer](Application_Layer/) | 应用层：Android Apps、Privileged Apps、Device Manufacturer Apps、Android API |
| 2 | [System_Services_Layer](System_Services_Layer/) | 系统服务层：System Services、Android Framework、System API |
| 3 | [Android_Runtime_Layer](Android_Runtime_Layer/) | 安卓运行时层：Android Runtime（ART、Binder、JE/NE 等） |
| 4 | [HAL_And_Lower_Services_Layer](HAL_And_Lower_Services_Layer/) | 硬件抽象层及守护进程：HAL、System Services and Daemons |
| 5 | [Linux_Kernel_Layer](Linux_Kernel_Layer/) | Linux 内核层：进程、内存、Block、FS、IO、中断、系统调用等 |

## 跨层目录

- [Tools_Mastery](Tools_Mastery/)：调试、追踪、Git、Android 工具等
- [Architecture_Governance](Architecture_Governance/)：监控、兜底、测试、容灾、最佳实践与案例分析
- [Practice_Playground](Practice_Playground/)：实战与复现场景

## 来源说明

- 内容来自 **ACKLearning** 与 **stabilityLearning** 的 Markdown 按主题重组，未改动 **common**（ACK 源码目录）。
- 各层内按知识点再分子目录，详见各层目录下 `README.md`。
