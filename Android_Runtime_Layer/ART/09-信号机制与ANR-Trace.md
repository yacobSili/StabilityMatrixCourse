# 09-信号机制与 ANR Trace：从 SIGQUIT 到 traces.txt 的完整链路

## 1. 为什么架构师必须掌握 SIGQUIT 与 ANR Trace 机制

在前面的文章中，我们讨论过 ART 如何利用 `SIGSEGV` 实现隐式空指针检查（FaultHandler），以及 `sigchain` 如何协调多个信号处理器的优先级。但有一个同等重要甚至更常用的信号，整个系列至今没有展开——**SIGQUIT（信号 3）**。

SIGQUIT 在 Android 中有一个非常特殊的用途：**ANR 时触发 Java 线程堆栈的 dump**。每当你打开一份 `traces.txt`（或 `/data/anr/anr_*` 文件）来排查 ANR 问题时，你看到的那些线程状态、持有的锁、完整的调用栈——这些信息是怎么来的？答案是：

```
AMS 检测到超时 → 向目标进程发送 SIGQUIT → ART 的 SignalCatcher 线程响应
→ 挂起所有 Java 线程 → 遍历每个线程的栈帧 → 输出到文件
```

这条链路涉及 **Framework 层的 ANR 检测、Linux 信号传递、ART 的线程挂起机制、栈帧遍历算法** 四个层次。如果你不理解它，你会遇到以下困境：

*   ANR trace 文件中的线程状态（`RUNNABLE`、`BLOCKED`、`WAITING`、`NATIVE`）到底代表什么？为什么有的线程显示 `RUNNABLE` 但实际在等 I/O？
*   APM 框架通过监听 SIGQUIT 来检测 ANR，为什么有时会漏掉？
*   为什么 `traces.txt` 中有时看不到真正卡住的线程？"Thread suspension timed out" 又意味着什么？
*   如何主动发送 SIGQUIT 来获取运行时快照，辅助排查死锁和卡顿？

本篇将彻底打通这条链路。

---

## 2. SIGQUIT 的语义：它和致命信号有何不同

### 2.1 SIGQUIT 不是 Crash 信号

在 Native Crash 系列中，我们详细介绍了 7 种致命信号（`SIGSEGV`、`SIGABRT`、`SIGBUS`、`SIGFPE`、`SIGILL`、`SIGSYS`、`SIGTRAP`）。这些信号的默认行为是**终止进程并生成 core dump**，它们代表程序遇到了不可恢复的错误。

`SIGQUIT`（信号编号 3）完全不同：

| 特性 | 致命信号（SIGSEGV 等） | SIGQUIT |
| :--- | :--- | :--- |
| **默认行为** | 终止进程 + core dump | 终止进程 + core dump（但 ART 会覆盖） |
| **产生原因** | 硬件异常或程序主动 abort | 外部进程显式发送 |
| **在 ART 中的用途** | 转化为 Java 异常（NPE/SOE） | 触发线程堆栈 dump |
| **是否致命** | 是 | **否**（ART 处理后进程继续运行） |
| **常见触发方** | CPU/MMU 硬件 | AMS、debuggerd、`kill -3` 命令 |

### 2.2 JVM 的传统：kill -3

SIGQUIT 用于 dump 线程堆栈的传统来自标准 JVM。在服务端 Java 开发中，运维人员经常用 `kill -3 <pid>` 向 JVM 进程发送 SIGQUIT，JVM 收到后会将所有线程的堆栈打印到标准输出（或日志文件），用于排查死锁和性能问题。ART 继承了这个传统，但将输出目标改为了 `/data/anr/` 目录下的文件。

### 2.3 ART 中 SIGQUIT 的注册方式

与 `SIGSEGV` 使用 `sigaction` 注册 handler 不同，ART 对 SIGQUIT 采用了**完全不同的处理策略**——不注册信号处理函数，而是通过 `sigwait` 在专门的线程中同步等待。

为什么不用 `sigaction`？因为信号处理函数（signal handler）运行在被中断线程的上下文中，环境极度受限（只能调用 async-signal-safe 函数），无法执行复杂的堆栈 dump 操作。而 `sigwait` 将信号处理转化为普通的线程同步操作，处理函数运行在 SignalCatcher 线程自己的上下文中，可以安全地执行任意操作。

```
sigaction 方式（用于 SIGSEGV）:
  信号 → 内核中断任意线程 → 在被中断线程栈上执行 handler → 环境受限

sigwait 方式（用于 SIGQUIT）:
  信号 → 投递到进程的 pending 信号集 → SignalCatcher 线程的 sigwait() 返回
  → 在 SignalCatcher 线程自己的栈上执行处理逻辑 → 无环境限制
```

### 2.4 ART 信号处理全景表

在深入 SIGQUIT 之前，先建立全局视野：ART 对 Linux 信号的处理分为四种方式，每种对应不同的处理机制和用途。

**方式一：sigwait 同步等待（SignalCatcher 线程处理）**

| 信号 | 编号 | 处理逻辑 | 用途 | 进程是否终止 |
| :--- | :--- | :--- | :--- | :--- |
| **SIGQUIT** | 3 | `HandleSigQuit()` → dump 所有线程栈 | ANR trace、手动 `kill -3` 诊断 | **否** |
| **SIGUSR1** | 10 | `HandleSigUsr1()` → 触发 HPROF 堆 dump | `am dumpheap`、debuggerd 触发堆快照 | **否** |

**方式二：sigaction 注册 handler（FaultHandler + sigchain 管理）**

| 信号 | 编号 | ART 处理逻辑 | 若 ART 不处理则交给 | 进程是否终止 |
| :--- | :--- | :--- | :--- | :--- |
| **SIGSEGV** | 11 | NullPointerHandler → NPE；SuspensionHandler → 线程挂起；StackOverflowHandler → SOE | APM handler 或默认 abort | ART 处理时否；否则是 |
| **SIGBUS** | 7 | 类似 SIGSEGV 的 FaultHandler 链 | APM handler 或默认 abort | ART 处理时否；否则是 |

**方式三：sigchain 管理但 ART 不直接处理（透传给用户 handler）**

| 信号 | 编号 | sigchain 行为 | 典型场景 |
| :--- | :--- | :--- | :--- |
| **SIGABRT** | 6 | sigchain 拦截后按链分发，最终走默认 abort | ART 内部 `LOG(FATAL)` / `CHECK` 失败；APM 捕获 Native Crash |
| **SIGFPE** | 8 | sigchain 拦截后按链分发 | 浮点异常（ART 中极少触发，整数除零在 Dex 层已检查） |
| **SIGTRAP** | 5 | sigchain 拦截后按链分发 | 断点（debugger 使用）、`__builtin_trap()` |
| **SIGPIPE** | 13 | ART 在启动时设为 **SIG_IGN**（忽略） | 防止 Socket/Pipe 写端关闭时意外终止进程 |

**方式四：ART 不干预，走 Linux 内核默认行为**

| 信号 | 编号 | 默认行为 | 典型场景 |
| :--- | :--- | :--- | :--- |
| **SIGKILL** | 9 | 立即终止进程（**不可捕获、不可忽略**） | `kill -9`、LMK（Low Memory Killer）杀进程、`forceStopPackage` |
| **SIGSTOP** | 19 | 暂停进程（**不可捕获**） | debugger 附加进程 |
| **SIGTERM** | 15 | 终止进程 | 正常的进程终止请求（但 Android 中更常用 SIGKILL） |
| **SIGCONT** | 18 | 恢复被暂停的进程 | 配合 SIGSTOP 使用 |

**一个常见误解：** 很多人以为 `kill -9`（SIGKILL）和 `kill -3`（SIGQUIT）的区别只是"信号编号不同"。实际上它们属于完全不同的类别——SIGKILL 不可捕获，进程立即死亡；SIGQUIT 被 ART 的 SignalCatcher 捕获后进程继续正常运行。

### 2.5 实用技巧：手动 kill -3 排查线上问题

理解了 SIGQUIT 的非致命性后，它就变成了一个**强大的诊断工具**：你可以随时对任意 Java 进程拍一张"线程快照"，且不影响进程的正常运行。

**使用场景：**

```bash
# 场景一：排查 system_server 中的死锁
# system_server 的 PID 通常可以通过 ps 获取
adb shell ps -A | grep system_server
# u0_a0  3699  1234  ... system_server
adb shell kill -3 3699
# 然后从 /data/anr/ 目录获取 dump 文件

# 场景二：排查某个 App 的线程卡顿（无需等 ANR 超时）
adb shell ps -A | grep com.example.myapp
adb shell kill -3 <pid>

# 场景三：使用 debuggerd 获取更详细的信息（包含 Native 栈）
adb shell debuggerd -b <pid>
```

**手动 `kill -3` 与 ANR 触发的对比：**

| 维度 | ANR 触发的 SIGQUIT | 手动 `kill -3` |
| :--- | :--- | :--- |
| **触发源** | AMS 超时检测后自动发送 | 人工执行命令 |
| **发送目标** | 多个进程（ANR 进程 + system_server + 其他关键进程） | 只发给指定的一个进程 |
| **附带动作** | ANR 对话框、DropBox 记录、可能杀进程 | 无任何附带动作 |
| **ART 侧行为** | **完全相同** | **完全相同** |
| **进程存活** | 可能被杀（用户点"关闭"或后台策略） | 一定不会被杀 |
| **输出位置** | `/data/anr/` 目录 | `/data/anr/` 目录（相同） |

底层机制完全一样：都是 SIGQUIT → SignalCatcher → `HandleSigQuit()` → `SuspendAll` → dump → `ResumeAll`。所以当你需要排查死锁或卡顿但又不想等 ANR 超时时，主动发送 `kill -3` 是最直接的手段。

---

## 3. SignalCatcher 线程：ART 的信号守卫

### 3.1 创建时机

SignalCatcher 线程在 `Runtime::Start()` 阶段被创建，而非 `Runtime::Init()` 阶段。这是因为 SignalCatcher 依赖于 Runtime 的大部分子系统（ThreadList、Heap、ClassLinker 等）已经初始化完毕。

*   **源码路径：** `art/runtime/signal_catcher.cc`、`art/runtime/runtime.cc`

```cpp
// art/runtime/runtime.cc（简化）
bool Runtime::Start() {
  // ... BootImage 加载、守护线程启动 ...
  
  // 创建 SignalCatcher 线程
  signal_catcher_ = new SignalCatcher();
  
  return true;
}
```

### 3.2 初始化与信号屏蔽

SignalCatcher 在构造函数中创建一个新的 pthread，并在新线程的入口函数中配置信号掩码：

```cpp
// art/runtime/signal_catcher.cc（简化）
SignalCatcher::SignalCatcher() {
  // 创建一个新线程来处理信号
  CHECK_PTHREAD_CALL(pthread_create, 
      (&pthread_, &attr, &SignalCatcher::Run, this), "signal catcher thread");
}

void* SignalCatcher::Run(void* arg) {
  Runtime* runtime = Runtime::Current();
  
  // 将当前线程附加到 ART Runtime
  runtime->AttachCurrentThread("Signal Catcher", true, 
                                runtime->GetSystemThreadGroup(), ...);
  
  Thread* self = Thread::Current();
  
  // 配置信号掩码：只等待 SIGQUIT 和 SIGUSR1
  sigset_t mask;
  sigemptyset(&mask);
  sigaddset(&mask, SIGQUIT);   // ANR dump
  sigaddset(&mask, SIGUSR1);   // 强制 GC（debuggerd 使用）
  
  while (true) {
    int signal_number = WaitForSignal(self, mask);
    if (signal_number == SIGQUIT) {
      HandleSigQuit();
    } else if (signal_number == SIGUSR1) {
      HandleSigUsr1();  // 触发堆 dump（HPROF）
    }
  }
}
```

**关键设计点：**

1. **SignalCatcher 是一个标准的 ART 管理线程**（通过 `AttachCurrentThread` 注册到 Runtime），拥有自己的 `Thread` 对象和 `JNIEnv`。在 `traces.txt` 中，你会看到它显示为 `"Signal Catcher"` 线程。
2. **使用 `sigwait` 而非 `sigaction`**，将信号处理转化为同步的线程等待，避免了 async-signal-safe 的限制。
3. **同时监听两个信号：** `SIGQUIT` 用于 ANR trace dump，`SIGUSR1` 用于触发 HPROF 堆 dump（通常由 `debuggerd` 或 `am dumpheap` 使用）。

### 3.3 信号掩码与全进程的协作

一个进程中所有线程共享 pending 信号集，但每个线程有自己的信号掩码（signal mask）。为了确保 SIGQUIT 只被 SignalCatcher 线程接收，ART 在所有其他线程中都屏蔽了 SIGQUIT：

*   **源码路径：** `art/runtime/runtime.cc`

```cpp
// art/runtime/runtime.cc（简化）
// 在 Runtime::Init 中，主线程屏蔽 SIGQUIT
void Runtime::BlockSignals() {
  sigset_t sigset;
  sigemptyset(&sigset);
  sigaddset(&sigset, SIGQUIT);
  sigaddset(&sigset, SIGUSR1);
  pthread_sigmask(SIG_BLOCK, &sigset, nullptr);
  // 后续所有由此线程 fork/clone 的子线程都继承这个信号掩码
  // 只有 SignalCatcher 线程会在自己的 Run() 中通过 sigwait 解除屏蔽
}
```

这保证了当 AMS 向进程发送 SIGQUIT 时，信号不会中断正在工作的业务线程或 GC 线程，而是精确地被 SignalCatcher 线程接收。

### 3.4 WaitForSignal：等待信号的核心循环

```cpp
// art/runtime/signal_catcher.cc（简化）
int SignalCatcher::WaitForSignal(Thread* self, const sigset_t& mask) {
  // 将线程状态切换为 kWaiting，允许 GC 进行
  ScopedThreadStateChange tsc(self, ThreadState::kWaiting);
  
  int signal_number;
  // sigwait 会阻塞当前线程，直到 mask 中的某个信号到达
  int rc = sigwait(&mask, &signal_number);
  if (rc != 0) {
    PLOG(FATAL) << "sigwait failed";
  }
  
  return signal_number;
}
```

**稳定性架构师视角：** 注意 `ScopedThreadStateChange` 将线程状态切换为 `kWaiting`。这很关键——如果 SignalCatcher 处于 `kRunnable` 状态等待信号，GC 执行 `SuspendAll` 时会一直等待 SignalCatcher 到达 SafePoint，而 SignalCatcher 可能正卡在 `sigwait` 上无法响应，造成死锁。切换为 `kWaiting` 后，GC 知道这个线程是安全的，不需要等待它。

---

## 4. ANR 完整链路：从 AMS 超时到 traces.txt

### 4.1 ANR 的四种触发场景

ANR（Application Not Responding）由 `ActivityManagerService`（AMS）检测。AMS 会在以下四种场景中设置超时定时器：

| 场景 | 超时阈值 | 检测机制 | 源码路径 |
| :--- | :--- | :--- | :--- |
| **Input 事件** | 5 秒 | `InputDispatcher` 等待 App 消费事件 | `frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp` |
| **Broadcast 接收** | 前台 10 秒 / 后台 60 秒 | AMS 发送广播后启动定时器 | `frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java` |
| **Service 生命周期** | 前台 20 秒 / 后台 200 秒 | AMS 启动 Service 后设置超时 | `frameworks/base/services/core/java/com/android/server/am/ActiveServices.java` |
| **ContentProvider** | 10 秒 | 发布 Provider 超时 | `frameworks/base/services/core/java/com/android/server/am/ContentProviderHelper.java` |

### 4.2 ANR 触发后的 Framework 流程

以 Input ANR 为例，当超时发生后：

```
InputDispatcher 检测到 App 5 秒未响应
    ↓
InputDispatcher 通知 AMS
    ↓
AMS.appNotResponding()（ProcessErrorStateRecord.appNotResponding）
    ↓
收集需要 dump 的进程 pid 列表（当前进程 + System Server + 关键进程）
    ↓
调用 StackTracesDumpHelper.dumpStackTraces()
    ↓
对每个 pid：Process.sendSignal(pid, Process.SIGNAL_QUIT)
    ↓
等待目标进程完成 dump（通过文件锁或超时机制）
    ↓
将 dump 结果写入 /data/anr/ 目录
    ↓
弹出 ANR 对话框（或直接杀进程）
```

*   **源码路径：** `frameworks/base/services/core/java/com/android/server/am/ProcessErrorStateRecord.java`

```java
// ProcessErrorStateRecord.java（简化）
void appNotResponding(String activityShortComponentName, 
                       ApplicationInfo aInfo, ...) {
    // 1. 收集需要 dump 的进程 PID
    ArrayList<Integer> firstPids = new ArrayList<>();
    firstPids.add(mApp.getPid());  // ANR 进程本身
    // 加入 System Server 和其他关键进程
    
    // 2. 触发堆栈 dump
    File tracesFile = StackTracesDumpHelper.dumpStackTraces(
        firstPids, nativePids, extraPids, ...);
    
    // 3. 记录到 DropBox
    mService.addErrorToDropBox("anr", mApp, ...);
    
    // 4. 弹出 ANR 对话框或直接杀进程
    if (mService.mUserController.getCurrentUser().isBgRestricted()) {
        mService.mHandler.post(() -> mService.mAppErrors.killAppAtUserRequestLocked(mApp));
    } else {
        // 弹出对话框
    }
}
```

### 4.3 发送 SIGQUIT

`StackTracesDumpHelper.dumpStackTraces()` 最终通过 `Process.sendSignal()` 向目标进程发送 SIGQUIT：

*   **源码路径：** `frameworks/base/services/core/java/com/android/server/am/StackTracesDumpHelper.java`

```java
// StackTracesDumpHelper.java（简化）
public static File dumpStackTraces(ArrayList<Integer> firstPids, ...) {
    // Android 9+: 通过 tombstoned 收集 traces
    // Android 8及以下: 直接写入 /data/anr/traces.txt
    
    for (int pid : firstPids) {
        // 向目标进程发送 SIGQUIT
        Process.sendSignal(pid, Process.SIGNAL_QUIT);
        
        // 等待目标进程完成 dump（超时 20 秒）
        waitForDumpCompleted(pid, timeout);
    }
    
    return tracesFile;
}
```

**注意：** 从 Android 9 开始，traces 的收集通过 `tombstoned` 守护进程中转。ART 的 SignalCatcher 不再直接写文件，而是通过 socket 连接 `tombstoned`，由 `tombstoned` 分配输出 fd 并管理文件路径（`/data/anr/anr_*`）。

### 4.4 ART 侧的响应：HandleSigQuit

当 SIGQUIT 到达目标进程后，SignalCatcher 线程的 `sigwait` 返回，进入 `HandleSigQuit()`：

*   **源码路径：** `art/runtime/signal_catcher.cc`

```cpp
// art/runtime/signal_catcher.cc（简化）
void SignalCatcher::HandleSigQuit() {
  Runtime* runtime = Runtime::Current();
  
  std::ostringstream os;
  // 1. 输出 ART 构建信息和进程信息
  os << "\n----- pid " << getpid() << " at " << GetTimestamp() << " -----\n";
  os << "Cmd line: " << GetCmdLine() << "\n";
  os << "Build fingerprint: " << runtime->GetFingerprint() << "\n";
  os << "ABI: " << GetInstructionSetString(runtime->GetInstructionSet()) << "\n";
  
  // 2. 核心：dump Runtime 的所有信息
  runtime->DumpForSigQuit(os);
  
  os << "----- end " << getpid() << " -----\n";
  
  // 3. 输出到文件
  Output(os.str());
}
```

### 4.5 Runtime::DumpForSigQuit：dump 什么内容

`DumpForSigQuit` 是整个 dump 流程的调度中心，它依次 dump Runtime 的各个子系统：

*   **源码路径：** `art/runtime/runtime.cc`

```cpp
// art/runtime/runtime.cc（简化）
void Runtime::DumpForSigQuit(std::ostream& os) {
  // 1. 最重要：dump 所有线程的堆栈
  //    这一步会 SuspendAll，是耗时最长的部分
  GetThreadList()->DumpForSigQuit(os);
  
  // 2. dump ClassLinker 信息（已加载的类数量等）
  GetClassLinker()->DumpForSigQuit(os);
  
  // 3. dump InternTable（字符串常量池状态）
  GetInternTable()->DumpForSigQuit(os);
  
  // 4. dump JNI 全局引用信息
  GetJavaVM()->DumpForSigQuit(os);
  
  // 5. dump Heap/GC 信息
  GetHeap()->DumpForSigQuit(os);
  
  // 6. dump JIT 编译状态
  if (GetJit() != nullptr) {
    GetJit()->DumpForSigQuit(os);
  }
  
  // 7. dump 被追踪的 ArtMethod（用于采样 profiling）
  TrackedAllocators::Dump(os);
  
  os << "\n";
}
```

---

## 5. 线程挂起机制：SuspendAll 的实现

### 5.1 为什么需要挂起线程

dump 线程堆栈时，需要读取每个线程的栈帧、寄存器状态、持有的锁信息。如果线程还在运行，栈帧随时在变化，读到的数据会不一致甚至导致 Crash。因此，ART 必须先**挂起所有 Java 线程**，使它们停在一个一致性状态（SafePoint），然后再进行遍历。

### 5.2 SuspendAll 的流程

*   **源码路径：** `art/runtime/thread_list.cc`

```cpp
// art/runtime/thread_list.cc（简化）
void ThreadList::SuspendAll(const char* cause, bool long_suspend) {
  Thread* self = Thread::Current();
  
  // 1. 获取 thread_suspend_count_lock_（互斥锁）
  MutexLock mu(self, *Locks::thread_suspend_count_lock_);
  
  // 2. 设置全局挂起标志
  //    将 pending_threads 计数设为当前 Runnable 线程数
  int num_runnable = 0;
  for (Thread* thread : list_) {
    if (thread != self && thread->GetState() == kRunnable) {
      ++num_runnable;
      // 递增每个线程的 suspend_count_
      thread->IncrementSuspendCount(self);
      // 修改线程的 TLS 标志位，使其在下一个 SafePoint 检查时挂起
      thread->AtomicSetFlag(ThreadFlag::kSuspendRequest);
    }
  }
  
  // 3. 翻转 Polling Page（使编译码中的 SafePoint 检查触发故障）
  FlipPollingPage();
  
  // 4. 等待所有 Runnable 线程到达 SafePoint 并挂起
  //    超时时间由 thread_suspend_timeout_ns_ 控制
  while (pending_threads.load() > 0) {
    if (WaitTimeout(thread_suspend_timeout_ns_)) {
      // 超时！有线程无法挂起
      LOG(FATAL) << "Thread suspension timed out";
      // 这会导致进程 abort
    }
  }
}
```

### 5.3 SafePoint：线程在哪里响应挂起请求

设置了挂起标志后，线程不会立即停下——它必须运行到一个 **SafePoint（安全点）** 才会检查标志并挂起。SafePoint 的位置取决于线程当前的执行模式：

**解释器模式：**

*   **源码路径：** `art/runtime/interpreter/interpreter_switch_impl.cc`

解释器在执行每条 Dex 指令的循环中，会调用 `TestAllFlags()` 检查挂起标志：

```cpp
// art/runtime/interpreter/interpreter_switch_impl.cc（简化）
template<bool do_access_check, bool transaction_active>
void ExecuteSwitchImpl(Thread* self, const CodeItemDataAccessor& accessor,
                       ShadowFrame& shadow_frame, ...) {
  while (true) {
    // 取下一条 Dex 指令
    const Instruction* inst = Instruction::At(dex_pc_ptr);
    
    // ★ SafePoint 检查：每条指令（或每 N 条指令）检查线程标志
    if (UNLIKELY(self->TestAllFlags())) {
      CheckSuspend(self);  // 如果有挂起请求，线程在此处阻塞
    }
    
    // 解码并执行指令
    switch (inst->Opcode()) {
      case Instruction::INVOKE_VIRTUAL:
        // ...
    }
  }
}
```

**JIT/AOT 编译码模式：**

编译后的机器码不包含解释器那样的逐指令检查。ART 使用 **Polling Page** 机制：

1. ART 在进程中 `mmap` 一个特殊的内存页（Polling Page）
2. 编译器在生成的机器码中，在方法入口、循环回边（backward branch）等关键位置插入对 Polling Page 的读取指令
3. 正常情况下 Polling Page 可读，读取指令是空操作（近零开销）
4. 当需要 `SuspendAll` 时，ART 将 Polling Page 的权限改为不可读（`mprotect(PROT_NONE)`）
5. 线程执行到 Polling Page 读取指令时触发 `SIGSEGV`，ART 的 `SuspensionHandler` 捕获后将线程挂起

```
正常运行时:
  编译码 → 读 Polling Page → 正常返回 → 继续执行
  
SuspendAll 时:
  编译码 → 读 Polling Page → SIGSEGV!
      → SuspensionHandler 捕获
      → 线程进入挂起状态
      → 等待 ResumeAll 唤醒
```

**Native 代码执行中的线程：**

处于 `kNative` 状态的线程（正在执行 JNI 代码或系统调用）不需要被挂起。因为 Native 代码不操作 Java 堆上的对象引用（通过 JNI 间接访问时有引用表保护），GC 和 dump 操作是安全的。这些线程会在从 Native 返回到 Java 代码时（JNI 调用返回的 `JniMethodEnd`），检查挂起标志并自行挂起。

### 5.4 挂起超时：Thread suspension timed out

如果某个线程迟迟不到达 SafePoint，`SuspendAll` 会在等待 `thread_suspend_timeout_ns_`（通常 4-10 秒）后超时，触发致命错误：

```
F/art: Thread suspension timed out
F/art: Runnable thread <thread_name> not responding to suspend request
```

**常见原因：**

| 原因 | 机制 | 排查线索 |
| :--- | :--- | :--- |
| **长时间 JNI 调用** | 线程在 `kNative` 状态执行耗时的 Native 代码，返回前无法响应 | trace 中线程卡在 JNI 方法上 |
| **编译码中缺少 SafePoint** | 极端情况下编译器未在长循环中插入 SafePoint 检查 | 线程卡在热循环中 |
| **内核调度问题** | 线程被内核长时间抢占，无法执行到 SafePoint | dmesg 中有调度延迟日志 |
| **死循环** | Native 层或 JNI 层的死循环 | CPU 占用异常高 |

---

## 6. Java 栈 dump 的实现：从栈帧到文本

### 6.1 线程 dump 的入口

所有线程挂起后，`ThreadList::DumpForSigQuit` 遍历线程列表，对每个线程调用 `Thread::Dump()`：

*   **源码路径：** `art/runtime/thread_list.cc`

```cpp
// art/runtime/thread_list.cc（简化）
void ThreadList::DumpForSigQuit(std::ostream& os) {
  {
    // 挂起所有线程
    ScopedSuspendAll ssa(__FUNCTION__);
    
    os << "DALVIK THREADS (" << list_.size() << "):\n";
    
    for (const auto& thread : list_) {
      thread->Dump(os, /* dump_native_stack */ true,
                   /* force_dump_stack */ false);
    }
  }
  // ScopedSuspendAll 析构时自动 ResumeAll
}
```

### 6.2 Thread::Dump 的输出内容

每个线程的 dump 包含以下信息：

*   **源码路径：** `art/runtime/thread.cc`

```cpp
// art/runtime/thread.cc（简化）
void Thread::Dump(std::ostream& os, bool dump_native_stack, ...) {
  // 1. 线程头信息
  DumpState(os);
  // 输出格式：
  // "main" prio=5 tid=1 Runnable
  //   | group="main" sCount=0 ucsCount=0 flags=0 obj=0x... self=0x...
  //   | sysTid=12345 nice=0 cgrp=default sched=0/0 handle=0x...
  //   | state=R schedstat=( ... ) utm=... stm=... core=... HZ=100
  //   | stack=0x...-0x... stackSize=...
  //   | held mutexes= "mutator lock"(shared held)
  
  // 2. Java 栈帧
  DumpJavaStack(os);
  // 输出格式：
  //   at android.os.MessageQueue.nativePollOnce(Native method)
  //   at android.os.MessageQueue.next(MessageQueue.java:335)
  //   at android.os.Looper.loopOnce(Looper.java:161)
  //   - waiting on <0x... > (a android.os.MessageQueue)
  //   at android.os.Looper.loop(Looper.java:288)
  //   at android.app.ActivityThread.main(ActivityThread.java:7839)
  
  // 3. Native 栈帧（可选）
  if (dump_native_stack) {
    DumpNativeStack(os);
    // 使用 libunwindstack 进行 Native 栈回溯
  }
}
```

### 6.3 Java 栈帧的遍历：StackVisitor

ART 使用 `StackVisitor` 模式遍历线程的 Java 调用栈。Java 线程的栈上混合存在两种栈帧：

*   **ShadowFrame（解释器栈帧）：** 解释器为每个方法创建的帧，包含 `dex_pc_`（当前执行的 Dex 指令偏移）、`vregs_[]`（虚拟寄存器数组）
*   **QuickFrame（编译码栈帧）：** JIT/AOT 编译后的方法使用标准的 CPU 栈帧，通过 `OatQuickMethodHeader` 中的映射表回溯到 Dex 信息

*   **源码路径：** `art/runtime/stack.cc`

```cpp
// art/runtime/stack.cc（简化）
class StackVisitor {
 public:
  void WalkStack(bool include_transitions) {
    for (;;) {
      if (cur_shadow_frame_ != nullptr) {
        // 解释器栈帧：直接从 ShadowFrame 读取信息
        ArtMethod* method = cur_shadow_frame_->GetMethod();
        uint32_t dex_pc = cur_shadow_frame_->GetDexPC();
        
        // 回调给 Dump 逻辑
        bool should_continue = VisitFrame();
        
        cur_shadow_frame_ = cur_shadow_frame_->GetLink();
      } else if (cur_quick_frame_ != nullptr) {
        // 编译码栈帧：通过 OatQuickMethodHeader 查找方法
        ArtMethod* method = *cur_quick_frame_;
        uint32_t dex_pc = GetDexPc();  // 从 PC 映射表中查找
        
        bool should_continue = VisitFrame();
        
        // 根据栈帧大小跳到上一帧
        cur_quick_frame_ = reinterpret_cast<ArtMethod**>(
            reinterpret_cast<uint8_t*>(cur_quick_frame_) + frame_size);
      } else {
        break;
      }
    }
  }
};
```

### 6.4 从 dex_pc 到源码行号

遍历栈帧时，每个帧提供的是 `dex_pc`（Dex 字节码的偏移量）。要在 trace 中显示 `MessageQueue.java:335` 这样的源码行号，需要查询 Dex 文件中的 **Debug Info**：

*   **源码路径：** `art/libdexfile/dex/dex_file.cc`

Dex 文件的 `debug_info_item` 使用一种紧凑的状态机编码，记录了 `dex_pc` 到源码行号的映射。ART 的 `DexFile::DecodeDebugPositionInfo()` 方法解码这个映射，将 `dex_pc` 转换为源文件名和行号。

**稳定性架构师视角：** 如果 App 经过了 ProGuard/R8 混淆，行号映射会被干扰。使用 `-keepattributes SourceFile,LineNumberTable` 保留行号信息对 ANR 排查至关重要。如果 trace 中只显示行号为 0 或 Unknown，很可能是这个属性被混淆器移除了。

---

## 7. traces.txt 格式深度解读

### 7.1 文件位置的演变

| Android 版本 | 路径 | 管理方式 |
| :--- | :--- | :--- |
| Android 8 及之前 | `/data/anr/traces.txt` | ART 直接写入，后续 dump 追加或覆盖 |
| Android 9+ | `/data/anr/anr_<timestamp>_<pid>` | 通过 `tombstoned` 守护进程管理，每次 ANR 生成独立文件 |
| Android 11+ | `/data/anr/trace_<N>` | tombstoned 管理，自动轮转（保留最近 N 份） |

### 7.2 线程状态的含义

trace 中线程名称后面的状态关键字对应 ART 内部的 `ThreadState` 枚举：

*   **源码路径：** `art/runtime/thread_state.h`

| trace 中的状态 | ART 内部状态 | 含义 | 排查启示 |
| :--- | :--- | :--- | :--- |
| `Runnable` | `kRunnable` | 线程正在执行 Java/Native 代码 | 如果主线程是 Runnable 却 ANR，说明在做耗时计算或等 Native 调用返回 |
| `Sleeping` | `kTimedWaiting` (sleep) | `Thread.sleep()` | 不应出现在主线程上 |
| `Waiting` | `kWaiting` | `Object.wait()` 无超时 | 检查等待的 Monitor 被谁持有 |
| `TimedWaiting` | `kTimedWaiting` | `Object.wait(timeout)` | 检查超时时间是否合理 |
| `Blocked` | `kBlocked` | 等待进入 `synchronized` 块 | **锁竞争**，检查谁持有锁 |
| `Native` | `kNative` | 正在执行 JNI/Native 代码 | 主线程在 Native = 可能在等 I/O、Binder、锁 |
| `WaitingForGcToComplete` | `kWaitingForGcToComplete` | 等待 GC 完成 | GC 耗时过长导致的 ANR |
| `WaitingPerformingGc` | `kWaitingPerformingGc` | 正在执行 GC | GC 线程自身的状态 |
| `Suspended` | `kSuspended` | 被 GC 或 debugger 挂起 | 正常 dump 时的状态，非异常 |

### 7.3 锁信息解读

trace 中的锁信息是排查 ANR 的关键线索：

```
"main" prio=5 tid=1 Blocked
  ...
  at com.example.App.doSomething(App.java:42)
  - waiting to lock <0x0a1b2c3d> (a java.lang.Object) held by thread 15
  at com.example.App.handleMessage(App.java:30)
  ...

"Worker-3" prio=5 tid=15 Runnable
  ...
  at com.example.App.heavyWork(App.java:100)
  - locked <0x0a1b2c3d> (a java.lang.Object)
  ...
```

关键格式：
*   `waiting to lock <地址> (a 类名) held by thread N`：当前线程正在等待获取指定 Monitor，该 Monitor 被线程 N 持有
*   `locked <地址> (a 类名)`：当前线程持有指定 Monitor
*   `waiting on <地址> (a 类名)`：当前线程在该对象上调用了 `wait()`

### 7.4 一份完整 trace 的结构

```
----- pid 12345 at 2024-01-15 10:30:45.123 -----
Cmd line: com.example.myapp
Build fingerprint: 'google/oriole/oriole:13/...'
ABI: 'arm64'
Build type: optimized
Zygote loaded classes=12345 post zygote classes=6789

suspend all histogram:  Sum: 1.234ms 99% C.I. 0.010ms-0.500ms ...

DALVIK THREADS (25):

"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 ucsCount=0 flags=1 obj=0x... self=0x...
  | sysTid=12345 nice=-10 cgrp=top-app sched=0/0 handle=0x...
  | state=S schedstat=( 123456789 23456789 1234 ) utm=100 stm=23 core=3 HZ=100
  | stack=0x7ff...-0x7ff... stackSize=8388608KB
  | held mutexes=
  at com.example.App.doWork(App.java:42)
  - waiting to lock <0x0a1b2c3d> (a java.lang.Object) held by thread 15
  at android.os.Handler.handleCallback(Handler.java:942)
  at android.os.Handler.dispatchMessage(Handler.java:99)
  at android.os.Looper.loopOnce(Looper.java:201)
  at android.os.Looper.loop(Looper.java:288)
  at android.app.ActivityThread.main(ActivityThread.java:7839)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1003)

"Signal Catcher" daemon prio=10 tid=6 Runnable
  | group="system" sCount=0 ...
  | ...
  (本线程就是正在执行 dump 的 SignalCatcher)

"FinalizerDaemon" daemon prio=10 tid=8 Waiting
  | group="system" ...
  at java.lang.Object.wait(Native method)
  - waiting on <0x...> (a java.lang.ref.ReferenceQueue)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:188)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:209)
  at java.lang.ref.Finalization$FinalizerDaemon.runInternal(...)
  
...

----- end 12345 -----
```

### 7.5 关键字段解释

| 字段 | 含义 |
| :--- | :--- |
| `tid` | ART 内部的线程 ID（不是 Linux 的 TID） |
| `sysTid` | Linux 系统的线程 TID（可用于 `strace`、`perf` 等工具） |
| `sCount` | 线程被挂起的次数（suspend count） |
| `ucsCount` | 用户代码触发的挂起次数（user code suspend count） |
| `nice` | Linux nice 值（-20 到 19，值越小优先级越高） |
| `cgrp` | CGroup 分组（`top-app` 表示前台应用，优先级最高） |
| `utm` / `stm` | 用户态 / 内核态 CPU 时间（单位：jiffies） |
| `held mutexes` | 当前线程持有的 ART 内部锁（如 `"mutator lock"`） |

---

## 8. APM 对 SIGQUIT 的利用与冲突

### 8.1 APM 监听 ANR 的三种策略

由于 ANR 不像 Crash 那样有明确的回调接口，APM 框架通常采用以下策略来检测和采集 ANR 信息：

**策略一：FileObserver 监听 `/data/anr/`**

```java
FileObserver observer = new FileObserver("/data/anr/", FileObserver.CLOSE_WRITE) {
    @Override
    public void onEvent(int event, @Nullable String path) {
        // 检测到新的 trace 文件写入完成
        // 读取文件内容并上报
    }
};
observer.startWatching();
```

*   **优点：** 实现简单，不干扰 ART 的信号处理
*   **缺点：** Android 10+ 权限收紧，普通 App 无法读取 `/data/anr/`；文件观察存在时序问题

**策略二：监听 SIGQUIT 信号**

APM 框架通过 `sigaction` 注册 SIGQUIT handler，在 handler 中采集主线程堆栈和系统信息，然后将信号转发给 ART 的原始 handler：

```cpp
// APM 框架的 SIGQUIT 监听（伪代码）
static struct sigaction original_sigquit_action;

void apm_sigquit_handler(int sig, siginfo_t* info, void* context) {
    // 1. 检测主线程是否卡顿（通过 Watchdog/主线程消息队列检查）
    if (isMainThreadBlocked()) {
        // 2. 采集现场信息
        collectAnrInfo();
    }
    
    // 3. 转发给 ART 的 SignalCatcher
    //    注意：不能直接调用 original handler，
    //    因为 ART 使用的是 sigwait 而非 sigaction
    //    需要将信号重新投递给进程
}
```

*   **优点：** 能在 ART dump 之前/同时采集信息
*   **缺点：** 与 ART 的 sigchain 存在潜在冲突（见下文）

**策略三：主线程 Watchdog + 主动 dump**

最稳健的方案：在后台线程定期向主线程 Handler 发送心跳消息，如果超时未收到响应，认为主线程卡顿，主动向自身进程发送 SIGQUIT 触发 dump：

```java
// Watchdog 检测到主线程卡顿后
Process.sendSignal(Process.myPid(), 3);  // 发送 SIGQUIT
// 等待 dump 完成后读取 trace 文件
```

*   **优点：** 不仅能捕获 AMS 触发的 ANR，还能捕获"准 ANR"（卡顿但未达到 ANR 阈值）
*   **缺点：** 增加了不必要的 dump 开销；需要控制发送频率

### 8.2 SIGQUIT 与 sigchain 的交互

ART 的 `sigchain` 主要管理 `SIGSEGV`、`SIGBUS` 等致命信号。SIGQUIT 的处理策略不同——它通过 `sigwait` 在 SignalCatcher 线程中等待，而非通过 `sigaction` 注册 handler。

这意味着：

1. **如果 APM 通过 `sigaction` 注册了 SIGQUIT handler**，这个 handler 会在 sigchain 的管理下被保存为用户 handler。但因为 SignalCatcher 使用 `sigwait`（绕过了 handler 机制），APM 的 handler 可能不会被调用——信号被 `sigwait` "消费"掉了。

2. **APM 需要使用 `sigwait` 或 `signalfd` 来与 SignalCatcher "竞争" SIGQUIT**，但这会导致信号被 APM 线程截获，SignalCatcher 收不到，ART 无法完成 dump。

3. **推荐做法：** APM 不要拦截 SIGQUIT 本身，而是通过监控主线程消息队列的响应时间来检测卡顿，发现问题后主动 dump 堆栈（通过 `Thread.getAllStackTraces()` 或自行发送 SIGQUIT）。

### 8.3 常见冲突与解决方案

| 冲突场景 | 表现 | 解决方案 |
| :--- | :--- | :--- |
| APM 通过 `sigaction` 注册 SIGQUIT handler | Handler 可能不被调用（被 `sigwait` 抢先消费） | 改用 Watchdog + 主动 dump 策略 |
| APM 使用 `sigwait` 等待 SIGQUIT | ART SignalCatcher 收不到信号，`traces.txt` 生成失败 | 收到信号后先采集 APM 数据，再重新发送 SIGQUIT 给进程 |
| APM 过于频繁地主动发送 SIGQUIT | 频繁 `SuspendAll` 导致全进程卡顿 | 限制发送频率（如每分钟最多 1 次），且仅在检测到卡顿时触发 |
| 多个 APM 框架同时监听 SIGQUIT | 信号被其中一个消费，其他收不到 | 统一 ANR 监控入口，避免多框架竞争 |

---

## 9. 实战案例

### 9.1 案例一：traces.txt 中主线程 Blocked，排查锁竞争导致的 ANR

**现象：** 线上某电商 App 在大促期间 ANR 率飙升 5 倍。ANR 类型为 Input Dispatch Timeout（5 秒无响应）。

**分析 traces.txt：**

```
"main" prio=5 tid=1 Blocked
  at com.example.cache.DiskCache.get(DiskCache.java:87)
  - waiting to lock <0x0d4e5f6a> (a java.lang.Object) held by thread 22
  at com.example.image.ImageLoader.loadFromCache(ImageLoader.java:145)
  at com.example.ui.ProductAdapter.onBindViewHolder(ProductAdapter.java:62)
  at androidx.recyclerview.widget.RecyclerView$Adapter.onBindViewHolder(RecyclerView.java:7254)
  ...

"OkHttp Dispatcher-3" prio=5 tid=22 Runnable
  at com.example.cache.DiskCache.put(DiskCache.java:123)
  - locked <0x0d4e5f6a> (a java.lang.Object)
  at com.example.cache.DiskCache.writeToFile(DiskCache.java:156)
  at java.io.FileOutputStream.write(FileOutputStream.java:...)
  ...
```

**根因分析：**

1. 主线程（tid=1）在 `RecyclerView.onBindViewHolder` 中调用了磁盘缓存的 `get()` 方法
2. `get()` 方法需要获取一个对象锁（`<0x0d4e5f6a>`）
3. 该锁被线程 22（OkHttp 线程）持有——线程 22 正在执行磁盘写入（`FileOutputStream.write`）
4. 大促期间图片下载量暴增，磁盘写入队列积压，单次写入耗时增加，锁持有时间变长
5. 主线程等锁超过 5 秒 → Input ANR

**修复方案：**

1. **读写分离：** 将 `DiskCache` 的锁从单一的互斥锁改为 `ReadWriteLock`，读操作（`get`）使用共享读锁，写操作（`put`）使用独占写锁。读-读不互斥，大幅减少主线程等待概率。
2. **异步化：** 主线程不直接访问磁盘缓存，改为先查内存缓存，miss 后异步加载。
3. **写入限流：** 对磁盘写入操作设置 QPS 限制，避免大促期间写入风暴。

### 9.2 案例二：Thread suspension timed out 导致进程 abort

**现象：** 线上某音视频 App 出现大量 Native Crash，Tombstone 中的 abort message 为 `Thread suspension timed out`。Crash 通常发生在 GC 触发时。

**分析 Tombstone：**

```
Abort message: 'Thread suspension timed out: Thread[14,"AudioDecoder"] 
  suspended=0 state=Runnable'
```

结合 logcat：

```
W/art: Long JNI call detected: 12340 ms (thread: AudioDecoder, 
  method: com.example.codec.NativeDecoder.decode)
```

**根因分析：**

1. `AudioDecoder` 线程通过 JNI 调用了一个 Native 解码方法 `NativeDecoder.decode()`
2. 该 Native 方法内部存在一个紧密循环，处理大量音频帧数据，单次调用耗时超过 10 秒
3. 线程处于 `kRunnable` 状态但实际在执行 Native 代码时，不会检查挂起标志
4. **注意区分：** 正常的 JNI 调用会在入口将线程状态切换为 `kNative`，GC 不需要等待它。但如果 Native 代码中有回调到 Java 的操作（如调用 `CallVoidMethod`），线程状态会短暂切回 `kRunnable`，然后又回到 Native——如果恰好在 `kRunnable` 期间 GC 请求 `SuspendAll`，但线程随后又进入了长时间的 Native 循环……
5. 实际根因：该 Native 方法内部多次调用 `GetByteArrayElements` 和 `ReleaseByteArrayElements`（每次调用都涉及 `kRunnable` ↔ `kNative` 切换），在最后一次 `GetByteArrayElements` 后（状态为 `kRunnable`）进入了长时间的数据处理循环，GC 等待超时后 abort。

**修复方案：**

1. **减少 JNI 边界穿越：** 将多次的 `GetByteArrayElements/Release` 合并为一次，将大块数据一次性拷贝到 Native 缓冲区后在 Native 侧处理。
2. **主动释放 GC 压力：** 在长时间 Native 计算前，显式调用 `env->MonitorEnter/Exit` 或使用 `GetPrimitiveArrayCritical`（注意 Critical 区间内禁止任何 JNI 调用，且会阻塞 GC）。更好的做法是将线程状态手动切换为 `kNative`。
3. **限制单次解码帧数：** 将大批量解码拆分为小批次，每个批次之间插入 SafePoint 检查的时机。

---

## 10. 总结

本篇从 SIGQUIT 的语义出发，完整拆解了 ANR trace dump 的全链路：

1. **ART 对信号的处理分为四种方式**：sigwait 同步等待（SIGQUIT/SIGUSR1）、sigaction+FaultHandler（SIGSEGV/SIGBUS）、sigchain 透传（SIGABRT/SIGFPE/SIGTRAP）、以及不干预走默认行为（SIGKILL/SIGTERM 等）。理解这张全景图是排查一切信号相关问题的基础。
2. **SIGQUIT 不是 Crash 信号**，它在 ART 中被重新定义为"触发线程堆栈 dump"的指令，进程收到后不会终止。手动 `kill -3 <pid>` 与 ANR 触发的底层机制完全一样，是排查死锁和卡顿的利器。
3. **SignalCatcher 线程**是 ART 专门用于处理 SIGQUIT 的守卫线程，使用 `sigwait` 同步等待信号，避免了 async-signal-safe 的限制。
4. **ANR 的完整链路**是：AMS 检测超时 → `Process.sendSignal(SIGQUIT)` → SignalCatcher 响应 → `Runtime::DumpForSigQuit` → `SuspendAll` 挂起所有线程 → 遍历栈帧 → 输出到 `/data/anr/` 文件。
5. **SuspendAll 依赖 SafePoint 机制**：解释器通过 `TestAllFlags` 逐指令检查，编译码通过 Polling Page 触发 SIGSEGV 实现，Native 代码在返回 Java 时检查。
6. **traces.txt 中的线程状态、锁信息、栈帧**是排查 ANR 根因的核心证据，理解其含义是架构师的基本功。
7. **APM 对 SIGQUIT 的监听需要小心处理**与 ART SignalCatcher 的竞争关系，推荐使用 Watchdog + 主动 dump 策略。

理解这条链路后，你不仅能更准确地解读 `traces.txt`，还能建设更健壮的 ANR 监控体系——这正是稳定性架构师的核心竞争力。
