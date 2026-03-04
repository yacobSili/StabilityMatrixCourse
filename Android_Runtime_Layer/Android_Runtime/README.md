# 03_Native_Runtime（原生库与运行时层）

## 📋 目录说明

本目录专注于 **Android Native库和运行时层**的稳定性问题分析。包括Native代码崩溃、ART虚拟机机制、以及Java异常与运行时的关联。

## 🎯 学习目标

- 深入理解Native崩溃（NE）的机制和分析方法
- 掌握ART虚拟机的运行机制和内存管理
- 理解Java异常与ART运行时的关联

## 📚 目录结构

```
03_Native_Runtime/
├── Native_Crash/     # Native崩溃（NE）分析
├── ART_Runtime/      # ART虚拟机机制
└── Java_Exception/   # Java异常（JE）与ART关联
```

## 🔗 关键主题

### Native_Crash
- Linux信号机制（SIGSEGV、SIGABRT、SIGBUS等）
- 内存管理（栈溢出、堆溢出、内存对齐）
- Tombstone文件分析
- GDB/LLDB调试Native代码

### ART_Runtime
- ART虚拟机架构
- ART内存分配器
- ART GC机制
- JIT和AOT编译

### Java_Exception
- Java异常处理机制
- OOM原理与ART内存管理
- Heap Dump分析
- StackOverflow、ClassNotFoundException等

## 📖 学习建议

1. 深入理解Linux信号机制和内存管理
2. 阅读ART源码，理解虚拟机的实现
3. 使用GDB/LLDB等工具调试Native代码
4. 分析Tombstone和Heap Dump文件

---

**对应Android架构**：Native C/C++ Libraries & Android Runtime 层
