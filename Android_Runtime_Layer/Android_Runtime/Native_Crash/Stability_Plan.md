# Native Crash (NE) 分析

## 📋 模块概述

Native Crash (NE) 是 Android 系统中 Native 层（C/C++）代码崩溃的问题。本模块将深入分析信号机制、内存管理、Tombstone 文件分析，以及从 Framework 层到 Kernel 层的完整崩溃处理流程。

## 🎯 学习目标

- 深入理解 Linux 信号机制和崩溃处理流程
- 掌握内存管理（虚拟内存、内存保护、内存对齐）
- 能够分析 Tombstone 文件定位崩溃根因
- 能够使用 GDB/LLDB 调试 Native 代码
- 能够复现各种 Native 崩溃场景并给出解决方案

## 📚 学习路径

### 01_Theory - 信号机制与内存管理基础

**学习内容**：

1. **Linux 信号机制**
   - 信号的定义和分类
   - 常见崩溃信号：SIGSEGV、SIGABRT、SIGBUS、SIGILL、SIGFPE
   - 信号处理流程（signal handler、sigaction）
   - 信号掩码和信号队列

2. **内存管理基础**
   - 虚拟内存管理
   - 内存映射（mmap）
   - 栈溢出和堆溢出
   - 内存对齐和内存保护
   - 页表机制（MMU）

**关键概念**：
- **SIGSEGV**: 段错误，通常由空指针、内存越界引起
- **SIGABRT**: 程序异常终止，通常由 assert 失败引起
- **SIGBUS**: 总线错误，通常由内存对齐问题引起
- **SIGFPE**: 浮点异常，通常由除零错误引起

**学习产出**：
- 信号机制原理分析文档
- 内存管理基础文档

---

### 02_Framework_Signal - Framework层信号处理

**学习内容**：
- Android 信号处理机制
- Tombstone 生成流程
- crash_dump 服务
- 信号到 Tombstone 的转换
- Native 崩溃上报机制

**关键源码**：
- `system/core/debuggerd/` - crash_dump 服务
- `Tombstone` 生成代码
- 信号处理 handler

**Tombstone 文件内容**：
- 崩溃信号和原因
- 寄存器状态
- 内存映射信息
- 堆栈回溯信息
- 线程信息

**学习产出**：
- Framework 层信号处理机制分析文档
- Tombstone 生成流程分析

---

### 03_Kernel_Signal_Memory - Kernel层信号与内存

**学习内容**：
- Kernel 信号发送机制
- page fault 处理
- 内存保护机制（MMU）
- Kernel 内存管理（slab、page allocator）
- 内存映射与虚拟地址转换

**关键知识点**：
- **page fault**: 访问无效内存时触发
- **MMU**: 内存管理单元，负责虚拟地址到物理地址的转换
- **slab allocator**: Kernel 内存分配器
- **内存保护**: 读、写、执行权限控制

**学习产出**：
- Kernel 层信号与内存机制分析文档
- page fault 处理流程分析

---

### 04_Reproduction - NE复现代码

**复现场景**：

1. **空指针崩溃 (SIGSEGV)**
   - 解引用空指针
   - 访问已释放的内存

2. **内存越界 (SIGSEGV)**
   - 数组越界访问
   - 缓冲区溢出

3. **栈溢出 (SIGSEGV)**
   - 递归调用过深
   - 局部变量过大

4. **堆损坏 (SIGABRT)**
   - 双重释放 (double free)
   - 使用已释放的内存 (use after free)
   - 堆缓冲区溢出

5. **除零错误 (SIGFPE)**
   - 整数除零
   - 浮点除零

6. **内存对齐错误 (SIGBUS)**
   - 未对齐的内存访问

7. **非法指令 (SIGILL)**
   - 执行非法指令

**代码结构**：
```
04_Reproduction/
├── NullPointer/              # 空指针崩溃
├── BufferOverflow/           # 缓冲区溢出
├── StackOverflow/           # 栈溢出
├── HeapCorruption/           # 堆损坏
├── DivideByZero/             # 除零错误
├── MemoryAlignment/          # 内存对齐错误
└── IllegalInstruction/       # 非法指令
```

**学习产出**：
- 各种 NE 场景的复现代码
- 复现步骤说明文档

---

### 05_Analysis_Tools - NE分析工具

**工具学习**：

1. **Tombstone 分析**
   - Tombstone 文件格式
   - 堆栈回溯分析
   - 寄存器状态分析
   - 内存映射分析
   - 关键信息提取

2. **GDB/LLDB 调试**
   - 启动调试会话
   - 设置断点
   - 单步执行
   - 查看变量和内存
   - 分析 core dump

3. **AddressSanitizer (ASan)**
   - 编译时启用 ASan
   - 运行时检测内存问题
   - ASan 报告分析

4. **Valgrind**
   - 内存泄漏检测
   - 内存错误检测
   - 性能分析

5. **自动化分析脚本**
   - Tombstone 自动解析
   - 崩溃模式识别
   - 分析报告生成

**学习产出**：
- Tombstone 分析指南
- GDB/LLDB 使用指南
- ASan 使用指南
- 自动化分析脚本

---

### 06_Solution_Governance - 解决方案与治理

**解决方案**：

1. **代码层面**
   - 空指针检查
   - 边界检查
   - 内存管理规范（RAII、智能指针）
   - 使用安全的内存操作函数

2. **工具层面**
   - 使用 AddressSanitizer 检测
   - 使用静态分析工具
   - 代码审查机制

3. **架构层面**
   - Native 代码隔离
   - 异常处理机制
   - 崩溃恢复策略

4. **监控与预防**
   - Native 崩溃监控
   - 崩溃模式分析
   - 自动化测试
   - 崩溃率监控

**学习产出**：
- NE 解决方案文档
- Native 代码安全编程指南
- 监控体系设计文档

---

## 🔄 KTAS 学习流程

### Knowledge（知识）
1. 学习信号机制和内存管理
2. 阅读 crash_dump 源码
3. 学习 GDB/LLDB 使用

### Trigger（触发）
1. 在 Playground 中复现各种 NE 场景
2. 收集 Tombstone 文件

### Analyze（分析）
1. 分析 Tombstone 文件
2. 使用 GDB/LLDB 调试
3. 使用 ASan 检测内存问题

### Solution（解决）
1. 给出修复方案
2. 设计预防措施
3. 建立监控体系

---

## 📖 推荐资源

### 源码阅读
- AOSP: `system/core/debuggerd/`
- Linux Kernel: `kernel/signal.c`
- Linux Kernel: `mm/` (内存管理)

### 文档
- Linux 信号机制文档
- GDB 官方文档
- AddressSanitizer 文档

### 工具
- GDB/LLDB
- AddressSanitizer
- Valgrind
- Tombstone 解析工具

---

## ✅ 学习检查点

完成本模块学习后，你应该能够：

- [ ] 理解信号机制和内存管理原理
- [ ] 能够从 Tombstone 中快速定位崩溃根因
- [ ] 能够使用 GDB/LLDB 调试 Native 代码
- [ ] 能够使用 ASan 检测内存问题
- [ ] 能够复现至少 5 种不同的 NE 场景
- [ ] 能够给出 NE 的解决方案和预防措施

---

**下一步**：完成 NE 模块学习后，进入 `04_Java_Exception_JE` 模块。
