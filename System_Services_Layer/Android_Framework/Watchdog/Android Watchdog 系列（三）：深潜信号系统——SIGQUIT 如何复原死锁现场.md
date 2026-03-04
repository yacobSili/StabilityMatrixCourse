在上一篇中，我们解析了 `HandlerChecker` 是如何作为探针深入前线的。但你一定产生了一个巨大的疑问：**既然 `HandlerChecker` 已经因为死锁而卡死在 `monitor()` 方法里了，它又是如何记录日志、打印堆栈并触发重启的呢？**

这就涉及到了 Watchdog 机制中最具“统治力”的设计——**信号机制（Signal Mechanism）**。它不依赖于任何 Java 层的消息循环，而是一次跨越层级的“降维打击”。

---

# Android Watchdog 系列（三）：深潜信号系统——SIGQUIT 如何复原死锁现场

当 Watchdog 判定超时（OVERDUE）时，Java 层的逻辑已经基本瘫痪。此时，Watchdog 线程会从背后掏出一把“手术刀”，穿过混乱的业务逻辑，直接剖开进程的胸膛。

## 一、 为什么要动用信号？

在死锁发生时，受监控的线程通常处于以下两种状态：

1. **Blocked（阻塞）**：卡在 `synchronized` 锁上，不接受任何 Handler 消息。
2. **Native（原地踏步）**：卡在 Binder 调用或 Socket IO 读写中。

由于这些线程已经“不听指挥”，Watchdog 无法通过正常的代码调用去获取它们的堆栈。而 **Linux 信号** 是内核级的异步通信机制，它具有**强制性**：无论线程在干什么，内核都会强行中断其当前的执行流，转而去执行预设的信号处理函数。

---

## 二、 SIGQUIT (Signal 3)：复原现场的法术

Watchdog 触发的取证核心，就是向 `SystemServer` 以及其他关键进程（如 `surfaceflinger`）发送 **`SIGQUIT`** 信号。

### 1. 信号的分发与捕获

当 Watchdog 调用 `Process.sendSignal(pid, Signal.QUIT)` 时：

* **内核层**：Linux 内核将信号挂载到目标进程的任务描述符上。
* **虚拟机层（ART）**：ART 在启动时会专门预留一个 **Signal Catcher 线程**。这个线程平时处于 `sigwait()` 状态。一旦监听到 `SIGQUIT`，它会被立刻唤醒。

### 2. Stop The World：全量采样

Signal Catcher 线程被唤醒后，会要求虚拟机进入 **Safe Point（安全点）**。

* **全员暂停**：虚拟机会暂停进程中所有的 Java 线程。
* **内存扫描**：由于所有线程都静止了，虚拟机可以安全地扫描内存中的对象头（Object Header）和监视器锁（Monitor）状态，计算出每一把锁被谁持有、被谁等待。

---

## 三、 深度取证：Trace 文件里的“法医证据”

信号处理机制最终会生成我们熟悉的 `/data/anr/traces.txt`。这份文件包含了三层极具价值的证据：

### 1. Java 堆栈：谁在等谁？

```text
"main" prio=5 tid=1 Blocked
  | waiting to lock <0x0a123456> (a com.android.server.am.ActivityManagerService)
  | held by thread 13

```

这是信号机制最直观的产物：它不仅告诉你代码执行到了哪一行，还明确指出了**锁的依赖关系**。

### 2. Native 堆栈：穿透 Binder

如果线程卡在 `Native` 态，信号处理逻辑会尝试记录其 C/C++ 层的调用栈。你会看到类似 `__ioctl` 或 `binder_thread_read` 的调用。
**架构价值**：这能帮你判断死锁是否已经蔓延到了其他进程（比如正在等待 `surfaceflinger` 的回应）。

### 3. 内核栈（Kernel Stack）：最后的真相

Watchdog 在发送信号的同时，还会读取 `/proc/[pid]/stack`。

* 如果 Java 栈显示 `Native`，而内核栈显示 `fuse_read`，说明死锁根源在于 **磁盘 I/O 卡死**。
* 如果内核栈显示 `binder_transaction`，说明 **Binder 缓冲区溢出**。

---

## 四、 处决时刻：为什么是 `killProcess`？

在完成取证（Dump Trace）后，Watchdog 会执行最后的处决：
`Process.killProcess(Process.myPid())`。

**为什么不调用 `System.exit()` 或尝试优雅退出？**

1. **死锁不可逆**：已经确定的死锁无法通过代码逻辑解锁。
2. **清理风险**：优雅退出会触发 `Shutdown Hook`。在一个已经死锁的进程里跑退出逻辑，大概率会引发第二次死锁，导致重启失败。
3. **Zygote 的契约**：`SystemServer` 是 `Zygote` 的子进程。当内核检测到 `SystemServer` 退出（SIGCHLD），`Zygote` 会根据配置意识到核心进程阵亡，从而立即自杀并重启整个 Framework。**暴力，往往是通往稳定最快的路径。**

---

## 五、 总结与启示

Watchdog 对信号机制的利用给了我们两个工程层面的启示：

1. **异步优于同步**：当系统不可信时，必须依赖更底层的异步机制（内核信号）来获取真实状态。
2. **快照胜于追踪**：与其记录复杂的轨迹，不如在崩溃瞬间留下一张高清的全量照片。

### 下篇预告

有了这份详尽的 Trace 文件，我们该如何像法医一样快速锁定真凶？面对错综复杂的锁依赖，如何找到那条致命的“闭环”？

下一篇，也就是本系列的终章：**《Android Watchdog (四)：实战手册——如何从 Traces 日志中逆向锁链条》**。

---

**（第三篇完）**

**如果你准备好了，我们可以开始最后一张拼图的绘制。**