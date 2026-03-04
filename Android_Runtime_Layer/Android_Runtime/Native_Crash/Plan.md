# Native Crash 基础理论

## 📋 目录说明

本目录用于存放 Native Crash 相关的基础理论知识、信号机制和内存管理原理。

## 📚 学习内容

1. **Linux 信号机制**
   - 信号的定义和分类
   - 常见崩溃信号：SIGSEGV、SIGABRT、SIGBUS、SIGILL、SIGFPE
   - 信号处理流程（signal handler、sigaction）

2. **内存管理基础**
   - 虚拟内存管理
   - 内存映射（mmap）
   - 栈溢出和堆溢出
   - 内存对齐和内存保护

## 📝 学习产出

- 信号机制原理分析文档
- 内存管理基础文档
- 关键概念总结

## 🔗 关键概念

- **SIGSEGV**: 段错误，通常由空指针、内存越界引起
- **SIGABRT**: 程序异常终止，通常由 assert 失败引起
- **SIGBUS**: 总线错误，通常由内存对齐问题引起

---

**提示**：理解信号机制和内存管理是分析 Native Crash 的基础。
