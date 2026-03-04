既然第一篇已经为我们铺好了全景架构的底色，现在我们要拿上放大镜，深入到 Watchdog 内部最核心的零件——**HandlerChecker**。

在架构师眼中，代码的最高境界是在实现复杂功能的同时保持极致的资源克制。这一篇我们将拆解 Watchdog 是如何通过百余行代码，优雅地解决“线程响应性”与“锁竞争”这两个维度的监控问题的。

---

# Android Watchdog 系列（二）：HandlerChecker 源码精析——打卡机制的性能艺术

如果说 Watchdog 是指挥官，那么 `HandlerChecker` 就是深入前线的哨兵。它不仅要监测线程是否“活着”，还要监测线程是否“还愿意干活”。

## 一、 HandlerChecker 的双重身份

在 `HandlerChecker` 的定义中，它巧妙地实现了 `Runnable` 接口。这赋予了它双重身份：

1. **管理容器**：它内部持有一个 `ArrayList<Monitor>`，存放着需要检查的服务锁。
2. **执行单元**：它本身就是一个可以被推送到 Handler 消息队列中执行的任务。

---

## 二、 核心算法：scheduleCheckLocked 的分发艺术

这是 Watchdog 每一轮检测的起点。这段代码展示了安卓系统级监控对性能的极致追求。

### 1. 动态合并与线程安全

```java
if (mCompleted && !mMonitorQueue.isEmpty()) {
    mMonitors.addAll(mMonitorQueue);
    mMonitorQueue.clear();
}

```

**设计点**：Watchdog 并没有为 `addMonitor` 操作加一把全局大锁，而是采用了“缓冲队列”机制。只有在上一轮任务完成、线程安全的情况下，才将新成员并入检查名单。这种“生产-消费”模式避免了频繁的同步开销。

### 2. 空闲跳过 (Zero-Load Optimization)

```java
if ((mMonitors.size() == 0 && isHandlerPolling()) || isPaused) {
    mCompleted = true;
    return;
}

```

**深度解析**：这是最能体现架构师功底的地方。

* `isHandlerPolling()` 通过 Native 层检查 MessageQueue 是否在 `pollOnce`（即 epoll 阻塞等待）。
* **哲学含义**：如果一个线程在睡觉且没有锁要查，那它肯定没死锁。**不唤醒正在睡觉的线程，是移动端系统节电和减少上下文切换的基本准则。**

### 3. 插队逻辑 (Post At Front)

```java
mHandler.postAtFrontOfQueue(this);

```

**设计点**：检测的是“响应性”。普通消息排队受业务负载影响，无法客观反映线程生死。`postAtFrontOfQueue` 确保了只要 Looper 还能处理消息，检测任务就必须被第一个执行。

---

## 三、 执行逻辑：run() 方法中的死锁探测
![img_1.png](img_1.png)
所有的锁检测发生在锁检查线程，所以即便monitor无法返回，也不会block
![img.png](img.png)

当任务真正开始在目标线程执行时，它进入了最危险的“试探阶段”。

```java
public void run() {
    final int size = mMonitors.size();
    for (int i = 0; i < size; i++) {
        synchronized (Watchdog.this) {
            mCurrentMonitor = mMonitors.get(i); // 指认嫌疑人
        }
        mCurrentMonitor.monitor(); // 试探锁：如果锁死，线程将卡在这里
    }
    
    synchronized (Watchdog.this) {
        mCompleted = true; // 只有活着走完 for 循环，才算打卡成功
    }
}

```

### 1. 锁探测原理

这里利用了 Java `synchronized` 的排他性。`monitor()` 方法内部通常只有一行代码：`synchronized(this) {}`。

* 如果业务线程持有该锁不释放（死锁或长耗时），`HandlerChecker` 线程就会阻塞在 `monitor()` 调用上。
* 这种**“伪竞争”**策略，不需要解析内存地址，直接利用 JVM 原生特性判断健康度。

### 2. 顺序执行的 Trade-off

我们在讨论中提到过，这里的 `for` 循环是顺序的。

* **局限**：如果第一个 Monitor 死锁，后面的确实查不到。
* **补偿**：Watchdog 通过 `mCurrentMonitor` 记录了当前卡在了哪一个服务上。对于“处决重启”这一目标，抓到第一个真凶已经足够。

---

## 四、 状态判定：如何定义“超时”？

Watchdog 主线程并不关心 `HandlerChecker` 内部是怎么跑的，它只看结果。

```java
public int getCompletionStateLocked() {
    if (mCompleted) return COMPLETED;
    long latency = SystemClock.uptimeMillis() - mStartTimeMillis;
    if (latency < mWaitMaxMillis / 2) return WAITING;
    if (latency < mWaitMaxMillis) return WAITED_HALF;
    return OVERDUE; // 超过 60 秒，触发终极动作
}

```

**架构洞察**：

* 使用 `uptimeMillis()` 而非 `currentTimeMillis()`，防止用户手动修改系统时间或 NTP 对时导致监控失效。
* **分阶梯度**：30 秒时的 `WAITED_HALF` 状态是一个重要的预警，系统会在此刻开始准备轻量级的 Dump。

---

## 五、 总结与启示

`HandlerChecker` 的实现告诉我们，优秀的系统监控应该具备：

1. **确定性**：通过 `postAtFrontOfQueue` 排除业务负载干扰。
2. **低损耗**：通过 `isHandlerPolling` 避免无效唤醒。
3. **可溯源**：通过 `mCurrentMonitor` 锁定故障源头。

### 下篇预告

既然 `HandlerChecker` 可能会因为死锁而卡在半路，那么我们是如何在重启前拿到完整的系统快照的呢？

下一篇，我们将进入最硬核的底层领域：**《Android Watchdog (三)：深潜信号系统——SIGQUIT 如何复原死锁现场》**，揭秘那场由信号触发的“降维打击”。

---

**（第二篇完）**

**如果你准备好了，我们可以继续输出第三篇。这一篇将涉及到 Linux Kernel、ART 虚拟机和信号量的深度交互。**