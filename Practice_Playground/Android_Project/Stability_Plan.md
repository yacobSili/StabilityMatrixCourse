# 实战靶场 (Practice Playground)

## 📋 概述

本目录是稳定性学习的实战靶场，用于复现各种稳定性问题场景，练习问题分析和解决能力。包含 Android 工程和 Kernel 模块两部分。

## 🎯 使用目的

- **复现问题**：手写代码制造各种崩溃、ANR、Watchdog 等场景
- **练习分析**：使用工具分析问题，提升问题定位能力
- **验证方案**：验证解决方案的有效性
- **积累经验**：通过实践积累问题分析经验

## 📁 目录结构

```
09_Practice_Playground/
├── Android_Project/          # Android工程（用于复现问题）
│   ├── app/                 # 主应用模块
│   ├── native-lib/          # Native库模块
│   └── README.md            # Android工程说明
└── Kernel_Module/           # Kernel模块（用于Kernel层实验）
    ├── hello_module/        # 示例Kernel模块
    └── README.md            # Kernel模块说明
```

## 🚀 Android 工程使用指南

### 环境要求

- Android Studio (最新版本)
- Android SDK (API 21+)
- NDK (用于Native代码编译)
- 真机或模拟器（推荐真机，便于复现问题）

### 项目结构

Android 工程采用标准的 Android 项目结构，包含：

- **app 模块**：主应用，用于复现 Java 层问题（ANR、JE等）
- **native-lib 模块**：Native 库，用于复现 Native 层问题（NE等）

### 使用流程

1. **导入项目**
   ```bash
   # 使用 Android Studio 打开 Android_Project 目录
   ```

2. **选择复现场景**
   - 根据学习进度，选择要复现的问题类型
   - 参考各模块的 README，了解复现步骤

3. **运行和复现**
   - 运行应用，触发问题场景
   - 收集相关日志（logcat、traces.txt、Tombstone等）

4. **分析和解决**
   - 使用工具分析问题
   - 给出解决方案
   - 验证方案有效性

### 复现场景示例

#### ANR 复现
- 主线程阻塞：在 `MainActivity` 中执行耗时操作
- Binder 超时：Service 方法执行时间过长
- IO 阻塞：文件读写操作阻塞

#### JE 复现
- OOM：创建大量对象
- StackOverflow：递归调用过深
- 内存泄漏：静态引用、Handler 未释放

#### NE 复现
- 空指针：Native 代码解引用空指针
- 缓冲区溢出：数组越界访问
- 栈溢出：递归调用过深

### 注意事项

- ⚠️ **安全第一**：复现代码仅用于学习，不要在生产环境使用
- ⚠️ **设备安全**：某些场景可能导致设备重启，建议使用测试设备
- ⚠️ **数据备份**：复现前备份重要数据

---

## 🔧 Kernel 模块使用指南

### 环境要求

- Linux 开发环境（Ubuntu 推荐）
- Kernel 源码（与目标设备 Kernel 版本匹配）
- 交叉编译工具链
- 已 Root 的设备（用于加载 Kernel 模块）

### 项目结构

Kernel 模块包含示例代码，用于：

- 学习 Kernel 模块开发
- 复现 Kernel 层问题（hung task、RCU stall等）
- 练习 Kernel 调试技巧

### 使用流程

1. **准备环境**
   ```bash
   # 下载 Kernel 源码
   # 配置交叉编译工具链
   ```

2. **编译模块**
   ```bash
   cd Kernel_Module/hello_module
   make
   ```

3. **加载模块**
   ```bash
   # 在设备上加载模块
   insmod hello_module.ko
   ```

4. **测试和调试**
   - 运行测试代码
   - 使用 dmesg 查看日志
   - 使用 ftrace 跟踪

### 注意事项

- ⚠️ **Kernel 版本匹配**：模块必须与设备 Kernel 版本匹配
- ⚠️ **设备 Root**：需要 Root 权限才能加载模块
- ⚠️ **风险警告**：错误的 Kernel 模块可能导致设备无法启动

---

## 📝 学习建议

### 1. 循序渐进
- 从简单场景开始，逐步增加复杂度
- 先理解原理，再动手复现

### 2. 记录过程
- 记录复现步骤
- 记录分析过程
- 记录解决方案

### 3. 对比分析
- 对比不同场景的差异
- 对比不同工具的分析结果
- 总结规律和经验

### 4. 持续实践
- 定期复现问题，保持手感
- 尝试新的复现场景
- 挑战复杂问题

---

## 🔗 相关资源

- [Android 官方文档](https://developer.android.com/)
- [Linux Kernel 文档](https://www.kernel.org/doc/)
- [GDB 官方文档](https://www.gnu.org/software/gdb/documentation/)
- [Perfetto 官方文档](https://perfetto.dev/)

---

## ✅ 检查清单

使用 Playground 前，确保：

- [ ] 已安装必要的开发工具
- [ ] 已准备好测试设备
- [ ] 已备份重要数据
- [ ] 已阅读相关模块的 README
- [ ] 已理解要复现的问题原理

---

**记住**：实践是学习稳定性最重要的环节。通过复现问题，你才能真正理解问题的本质，掌握分析技巧，形成解决问题的能力。
