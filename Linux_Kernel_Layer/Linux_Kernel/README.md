# 05_Linux_Kernel（Linux内核层）

## 📋 目录说明

本目录专注于 **Linux内核层**的稳定性核心机制。Kernel层是Android系统的核心，负责管理系统资源，并与底层硬件直接交互。

## 🎯 学习目标

- 深入理解Linux内核的核心机制（调度、内存、IO等）
- 掌握Kernel层稳定性问题的分析方法
- 理解Kernel层与上层Framework的交互

## 📚 目录结构

```
05_Linux_Kernel/
├── Architecture/         # Android Kernel架构
├── Process_Scheduling/   # 进程调度机制（CFS、实时调度）
├── Memory_Management/    # 内存管理（OOM Killer、kswapd、LMKD）
├── IO_Blocking/          # IO阻塞机制（block I/O、VFS）
├── FileSystem/           # 文件系统（F2FS、VFS）
├── RCU_Stall/            # RCU Stall问题
└── Kernel_Panic/         # Kernel Panic分析
```

## 🔗 关键主题

### Architecture
- Android Kernel架构概述
- ACK（Android Common Kernel）构建和刷机指南
- Kernel与Android系统的关系

### Process_Scheduling
- CFS调度器（完全公平调度）
- 实时调度（RT调度）
- 进程优先级和nice值
- 调度延迟与ANR的关系
- kworker线程分析

### Memory_Management
- Kernel内存管理（slab、page allocator）
- OOM Killer机制
- LowMemoryKiller（LMKD）
- 内存回收机制（kswapd）
- 内存压缩
- PSI（Pressure Stall Information）
- Zone内存管理

### IO_Blocking
- block I/O机制
- VFS（虚拟文件系统）
- IO调度器
- IO阻塞与ANR的关系
- F2FS文件系统
- PSI IO压力分析

### FileSystem
- 文件系统架构
- VFS核心机制
- IO路径深度分析

### RCU_Stall
- RCU（Read-Copy-Update）机制
- RCU Stall问题分析
- RCU与系统挂死的关系

### Kernel_Panic
- Kernel Panic机制
- Kernel崩溃日志分析
- Kernel调试技巧

## 📖 学习建议

1. 深入阅读Linux Kernel源码
2. 使用ftrace、eBPF等工具分析Kernel行为
3. 理解Kernel层机制对上层稳定性的影响
4. 结合实际问题，分析Kernel层的根因

---

**对应Android架构**：Linux Kernel 层
