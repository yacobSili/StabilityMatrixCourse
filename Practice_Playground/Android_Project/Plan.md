# ANR 复现代码

## 📋 目录说明

本目录用于存放各种 ANR 场景的复现代码，用于练习问题分析和解决。

## 📚 复现场景

1. **主线程阻塞 ANR**
   - 主线程执行耗时操作
   - 主线程死锁
   - 主线程等待锁

2. **Binder 调用超时 ANR**
   - Service 方法执行时间过长
   - Binder 线程池阻塞

3. **IO 阻塞 ANR**
   - 文件读写阻塞
   - 网络 IO 阻塞
   - 数据库操作阻塞

4. **Broadcast ANR**
   - BroadcastReceiver 执行时间过长

5. **Service ANR**
   - Service 启动超时
   - Service 方法执行超时

## 📝 代码结构

每个场景应该有：
- 复现代码
- 复现步骤说明
- 预期结果
- 分析方法

---

**提示**：复现问题后，记得收集 traces.txt 和 logcat 日志进行分析。
