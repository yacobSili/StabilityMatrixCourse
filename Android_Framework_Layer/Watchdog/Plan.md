# Kernel 层 hung task 检测

## 📋 目录说明

本目录用于存放 Kernel 层 hung task 检测机制的分析文档和学习笔记。

## 📚 学习内容

- Kernel hung task 检测机制
- `/proc/sys/kernel/hung_task_timeout_secs` 配置
- dmesg 中的 hung task 日志分析
- RCU 机制与 hung task 的关系
- soft lockup 和 hard lockup

## 📝 学习产出

- Kernel hung task 机制分析文档
- dmesg 日志分析指南
- 相关 Kernel 配置说明

## 🔗 关键知识点

- **hung task**: 处于 D 状态（不可中断睡眠）超过阈值的任务
- **RCU Stall**: RCU 机制导致的系统挂起
- **lockup**: CPU 被长时间占用导致的系统无响应

---

**提示**：理解 Kernel 层机制有助于分析系统级挂死问题。
