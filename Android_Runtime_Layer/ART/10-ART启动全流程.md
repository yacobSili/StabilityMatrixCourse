# 10-ART 启动全流程：从 app_process 到第一行 Java 代码

## 1. 为什么要深入理解 ART 的启动流程

在第一篇总览中，我们概述了 ART 的启动入口 `JNI_CreateJavaVM` → `Runtime::Init` → `Runtime::Start` 的宏观流程。但那只是冰山一角——ART 的启动并不从 `JNI_CreateJavaVM` 开始，也不在 `Runtime::Start` 结束。

完整的链路是：

```
Linux init 进程
    → 解析 init.rc → 启动 Zygote 服务
        → app_process 可执行文件的 main()
            → AndroidRuntime::start()
                → JNI_CreateJavaVM（创建 ART）
                → ZygoteInit.main()（Java 世界的入口）
                    → 预加载核心类和资源
                    → 启动 System Server
                    → 进入 Socket 循环，等待 fork 请求
                        → fork() → 子进程的 PostFork 初始化
                            → 第一行 App 的 Java 代码开始执行
```

理解这条完整链路，对稳定性架构师至少有三个直接用处：

1. **排查启动阶段 Crash：** 如果 `boot.art` 损坏、预加载类缺失、守护线程启动异常，Zygote 会崩溃 → 系统反复重启。你需要知道在哪一步出了问题。
2. **理解 COW 与内存共享：** Zygote 预加载了什么、fork 后哪些内存是共享的、什么操作会打破 COW，直接影响全局内存策略。
3. **理解 fork 后的初始化差异：** App 进程的 ART 实例和 Zygote 的 ART 实例有什么不同？哪些子系统需要重新初始化？这决定了 App 的冷启动开销。

---

## 2. 从 init 到 Zygote：Linux 世界的启动

### 2.1 init.rc 中的 Zygote 定义

Android 设备开机后，Linux 内核启动的第一个用户态进程是 `init`（PID=1）。`init` 进程解析 `init.rc` 及其导入的 `.rc` 文件，按序启动系统服务。Zygote 的定义通常在：

*   **源码路径：** `system/core/rootdir/init.zygote64_32.rc`（64 位主 + 32 位辅的常见配置）

```rc
# system/core/rootdir/init.zygote64_32.rc
service zygote /system/bin/app_process64 -Xzygote /system/bin \
        --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    ...
```

**关键参数解读：**

| 参数 | 含义 |
| :--- | :--- |
| `/system/bin/app_process64` | 可执行文件路径（这是 ART 启动的真正入口） |
| `-Xzygote` | 传递给 ART 的 VM 选项，标记当前是 Zygote 模式启动 |
| `--zygote` | 告诉 `app_process` 以 Zygote 模式运行 |
| `--start-system-server` | Zygote 启动后自动 fork System Server |
| `socket zygote stream 660` | 创建 Unix Domain Socket，用于 AMS 发送 fork 请求 |
| `onrestart restart audioserver/...` | Zygote 重启时，连带重启依赖它的服务 |

**稳定性架构师视角：** `onrestart` 指令表明 Zygote 崩溃的影响面极大——audioserver、cameraserver、media 等核心服务都会被连带重启。这就是为什么 Zygote 进程（以及 Zygote 中运行的 ART）的稳定性是系统级关键指标。

### 2.2 app_process 的 main()

`app_process`（`app_process64`/`app_process32`）是一个 C++ 可执行文件，它的 `main()` 函数是 ART 启动的真正起点。

*   **源码路径：** `frameworks/base/cmds/app_process/app_main.cpp`

```cpp
// frameworks/base/cmds/app_process/app_main.cpp（简化）
int main(int argc, char* const argv[]) {
    // 1. 创建 AppRuntime（继承自 AndroidRuntime）
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    
    // 2. 解析命令行参数
    bool zygote = false;
    bool startSystemServer = false;
    String8 niceName;
    
    while (i < argc) {
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;  // "zygote64" 或 "zygote"
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        }
        // ...
    }
    
    if (zygote) {
        // 3. Zygote 模式：启动 ART 并进入 ZygoteInit.main()
        runtime.start("com.android.internal.os.ZygoteInit", args, /*zygote=*/true);
    } else {
        // 非 Zygote 模式（如 cmd、am 等命令行工具）
        runtime.start("com.android.internal.os.RuntimeInit", args, /*zygote=*/false);
    }
}
```

`AppRuntime` 继承自 `AndroidRuntime`，核心逻辑在 `AndroidRuntime::start()` 中。

---

## 3. AndroidRuntime::start()：JVM 创建的触发点

### 3.1 整体流程

*   **源码路径：** `frameworks/base/core/jni/AndroidRuntime.cpp`

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp（简化）
void AndroidRuntime::start(const char* className, 
                            const Vector<String8>& options, 
                            bool zygote) {
    // ======== 阶段一：创建虚拟机 ========
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, isPrimaryZygote) != 0) {
        return;  // 创建失败，进程退出
    }
    
    // 虚拟机创建成功的回调（子类可覆盖）
    onVmCreated(env);
    
    // ======== 阶段二：注册 JNI 方法 ========
    if (startReg(env) < 0) {
        return;
    }
    
    // ======== 阶段三：调用 Java 层入口 ========
    // className = "com.android.internal.os.ZygoteInit"
    jclass startClass = env->FindClass(slashClassName);
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
                                                  "([Ljava/lang/String;)V");
    // 调用 ZygoteInit.main(args)
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
    
    // 正常情况下不会执行到这里
    // ZygoteInit.main() 会进入无限循环（等待 fork 请求）
}
```

### 3.2 阶段一：startVm()——配置 VM 选项并创建 ART

`startVm()` 是连接 Framework 配置和 ART 运行时的桥梁。它读取系统属性、构建 VM 选项列表，然后调用 `JNI_CreateJavaVM`：

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp（简化）
int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, 
                             bool zygote, bool isPrimaryZygote) {
    JavaVMInitArgs initArgs;
    
    // === 堆内存配置 ===
    // 读取 dalvik.vm.heapsize（大堆上限，如 512m）
    parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
    // 读取 dalvik.vm.heapgrowthlimit（普通 App 堆上限，如 256m）
    parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitBuf,
                       "-XX:HeapGrowthLimit=");
    // 读取 dalvik.vm.heapstartsize（初始堆大小，如 8m）
    parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
    
    // === GC 配置 ===
    // 读取 dalvik.vm.gctype（如 "CC" 或 "CMS"）
    parseRuntimeOption("dalvik.vm.gctype", gctypeOptsBuf, "-Xgc:");
    // 前台/后台 GC 阈值
    parseRuntimeOption("dalvik.vm.heaptargetutilization",
                       heaptargetutilizationBuf, "-XX:HeapTargetUtilization=");
    
    // === JIT 配置 ===
    parseRuntimeOption("dalvik.vm.usejit", usejitBuf, "-Xusejit:");
    parseRuntimeOption("dalvik.vm.jitmaxsize", jitmaxsizeOptsBuf,
                       "-Xjitmaxsize:");
    
    // === 镜像文件路径 ===
    // Boot Image 路径（如 /apex/com.android.art/javalib/boot.art）
    parseRuntimeOption("dalvik.vm.boot-image", bootImageBuf, "-Ximage:");
    
    // === 线程配置 ===
    // Zygote 模式下的特殊标志
    if (zygote) {
        addOption("-Xzygote");
    }
    
    // === 创建 ART 虚拟机 ===
    if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed");
        return -1;
    }
    
    return 0;
}
```

**稳定性架构师视角：** 这些系统属性（`dalvik.vm.*`）是调优 ART 行为的关键杠杆。OEM 厂商通常在 `build.prop` 或 `default.prop` 中设置它们。常见的调优点：

| 属性 | 默认值（参考） | 影响 |
| :--- | :--- | :--- |
| `dalvik.vm.heapgrowthlimit` | 256m | 普通 App 的堆上限，超过抛 OOM |
| `dalvik.vm.heapsize` | 512m | `largeHeap=true` App 的堆上限 |
| `dalvik.vm.heaptargetutilization` | 0.75 | 堆利用率目标，越低 GC 越频繁但内存余量越大 |
| `dalvik.vm.usejit` | true | 是否启用 JIT |
| `dalvik.vm.dex2oat-threads` | CPU 核心数 | 后台编译线程数，过高会抢占前台 CPU |

### 3.3 阶段二：startReg()——注册 Framework JNI 方法

ART 虚拟机创建后，Framework 需要将大量的 Native 方法注册到 ART 中，使 Java 代码能调用这些 Native 实现：

```cpp
// frameworks/base/core/jni/AndroidRuntime.cpp（简化）
int AndroidRuntime::startReg(JNIEnv* env) {
    // 注册所有的 Framework JNI 方法
    // 包括：android.os.Binder, android.os.MessageQueue, 
    //       android.view.Surface, android.media.*, ...
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        return -1;
    }
    return 0;
}

// gRegJNI 是一个函数指针数组，每个元素负责注册一组 JNI 方法
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_android_os_Binder),
    REG_JNI(register_android_os_Process),
    REG_JNI(register_android_os_MessageQueue),
    REG_JNI(register_android_view_Surface),
    REG_JNI(register_android_media_AudioTrack),
    // ... 几百个注册函数
};
```

**注意：** 这里注册的是 Android Framework 的 JNI 方法（如 Binder、Surface 等），而非 ART 核心类的 JNI 方法。ART 核心类（如 `java.lang.Thread`、`java.lang.Class`）的 Native 方法在 `Runtime::Init` 阶段就已经通过 `ClassLinker` 注册了。

### 3.4 阶段三：进入 Java 世界

JNI 方法注册完成后，`AndroidRuntime::start()` 通过 JNI 调用 `ZygoteInit.main()`——这标志着**执行流从 C++ 世界跨入 Java 世界**。从此刻起，Zygote 的后续逻辑全部以 Java 代码执行（在 ART 上运行）。

---

## 4. Runtime::Init 详解：各子系统的初始化顺序

`JNI_CreateJavaVM` 调用 `Runtime::Create`，后者调用 `Runtime::Init`。这是 ART 所有子系统初始化的核心函数，代码量超过 500 行。我们按照执行顺序逐一分析。

*   **源码路径：** `art/runtime/runtime.cc`

### 4.1 第一阶段：基础设施初始化

```cpp
// art/runtime/runtime.cc（简化）
bool Runtime::Init(RuntimeArgumentMap&& runtime_options) {
    // === 1. 解析 VM 选项 ===
    // 从 startVm 传入的 -Xmx、-Xms、-Xgc 等选项中提取配置
    
    // === 2. MemMap 初始化 ===
    // 初始化内存映射基础设施，后续所有的 mmap 操作都通过 MemMap 封装
    MemMap::Init();
    
    // === 3. 初始化 ART 内部的锁体系 ===
    // ART 有一套严格的锁层级（Lock Level），防止死锁
    // 顺序：mutator_lock > thread_list_lock > classlinker_classes_lock > ...
    
    // === 4. 线程管理初始化 ===
    thread_list_ = new ThreadList(thread_suspend_timeout_ns_);
    intern_table_ = new InternTable();
    
    // === 5. 将当前线程（主线程）附加到 ART ===
    // 当前线程就是 app_process 的主线程
    Thread::Startup();
    Thread* self = Thread::Attach("main", false, nullptr, false, true);
    // 从此刻起，当前线程拥有了 Thread 对象和 JNIEnv
```

### 4.2 第二阶段：Heap 创建

```cpp
    // === 6. 创建 Heap（Java 堆） ===
    // 这是最关键的初始化步骤之一
    heap_ = new gc::Heap(
        runtime_options.GetOrDefault(Opt::MemoryInitialSize),   // -Xms（初始大小）
        runtime_options.GetOrDefault(Opt::HeapGrowthLimit),     // heapgrowthlimit
        runtime_options.GetOrDefault(Opt::HeapMinFree),         // 最小空闲空间
        runtime_options.GetOrDefault(Opt::HeapMaxFree),         // 最大空闲空间
        runtime_options.GetOrDefault(Opt::HeapTargetUtilization),// 利用率目标
        foreground_heap_growth_multiplier,
        runtime_options.GetOrDefault(Opt::MemoryMaximumSize),   // -Xmx（最大大小）
        runtime_options.GetOrDefault(Opt::NonMovingSpaceCapacity),
        // GC 类型（CC、CMS 等）
        runtime_options.GetOrDefault(Opt::BackgroundGc),
        // ... 更多配置
    );
```

Heap 构造函数内部会创建多个 Space（内存区域）：

```
Heap 的 Space 布局：
├── Image Space（只读，mmap boot.art）
├── Zygote Space（COW 共享）
├── Main Space / Bump Pointer Space（App 对象分配）
│   └── TLAB（Thread Local Allocation Buffer，无锁快速分配）
├── Large Object Space（>= 12KB 的大对象）
├── Non-Moving Space（不可移动对象，如 Class 元数据）
└── JIT Code Cache（JIT 编译产生的机器码）
```

### 4.3 第三阶段：核心子系统初始化

```cpp
    // === 7. Monitor 初始化（synchronized 锁的底层实现） ===
    monitor_list_ = new MonitorList;
    monitor_pool_ = MonitorPool::Create();
    
    // === 8. ClassLinker 初始化（类加载器） ===
    class_linker_ = new ClassLinker(intern_table_);
    if (is_zygote) {
        // Zygote 模式：从 Boot Image 初始化
        class_linker_->InitFromBootImage(runtime_options, &error_msg);
    } else {
        // 非 Zygote 模式：不使用 Boot Image
        class_linker_->InitWithoutImage(runtime_options, &error_msg);
    }
    
    // === 9. 信号处理初始化 ===
    // 注册 FaultHandler（处理 SIGSEGV/SIGBUS）
    fault_manager.Init();
    // 屏蔽主线程的 SIGQUIT/SIGUSR1（留给 SignalCatcher 处理）
    BlockSignals();
    
    // === 10. JIT 初始化 ===
    if (jit_options_->UseJitCompilation() || jit_options_->GetSaveProfilingInfo()) {
        jit_ = jit::Jit::Create(jit_options_.get(), &error_msg);
        // JIT 编译器在这里被创建，但编译线程尚未启动
        // 编译线程会在 Runtime::Start() 中启动
    }
    
    return true;
}
```

### 4.4 完整初始化顺序总结

以下是 `Runtime::Init` 中各子系统的初始化顺序及其稳定性关注点：

| 顺序 | 子系统 | 源码关键位置 | 稳定性关注 |
| :--- | :--- | :--- | :--- |
| 1 | VM 选项解析 | `ParsedOptions::Parse` | 选项冲突或非法值 → 启动失败 |
| 2 | MemMap 初始化 | `MemMap::Init()` | VMA 限制、mmap 失败 |
| 3 | 锁体系 | `Locks::Init()` | 锁层级违反 → 死锁 |
| 4 | ThreadList | `new ThreadList()` | `thread_suspend_timeout_ns_` 配置 |
| 5 | 主线程 Attach | `Thread::Attach("main")` | 线程栈大小配置 |
| 6 | Heap 创建 | `new gc::Heap(...)` | 堆参数不合理 → OOM 或频繁 GC |
| 7 | InternTable | `new InternTable()` | 字符串常量池 |
| 8 | Monitor | `new MonitorList/Pool` | Monitor 池耗尽 |
| 9 | ClassLinker | `new ClassLinker()` | Boot Image 损坏 → 启动失败 |
| 10 | FaultHandler | `fault_manager.Init()` | 与 APM 信号冲突 |
| 11 | 信号屏蔽 | `BlockSignals()` | SIGQUIT 路由给 SignalCatcher |
| 12 | JIT | `Jit::Create()` | JIT 内存不足 |

---

## 5. Runtime::Start()：让虚拟机真正运转起来

`Init` 完成后，`JNI_CreateJavaVM` 调用 `Runtime::Start()` 让虚拟机进入可运行状态。

*   **源码路径：** `art/runtime/runtime.cc`

### 5.1 Boot Image 加载

```cpp
// art/runtime/runtime.cc（简化）
bool Runtime::Start() {
    Thread* self = Thread::Current();
    
    // === 1. 从 Boot Image 加载核心类 ===
    // boot.art 是预编译的核心类镜像文件，包含 java.lang.Object、
    // java.lang.String、java.lang.Class 等约 6000+ 核心类的预初始化状态
    if (!GetClassLinker()->InitFromBootImage(runtime_options, &error_msg)) {
        LOG(ERROR) << "Could not create image space with image file '"
                   << GetImageLocation() << "': " << error_msg;
        return false;
    }
```

Boot Image（`boot.art`）是 ART 性能优化的关键：

*   **内容：** 预初始化的 `mirror::Class`、`mirror::String`、`mirror::DexCache` 等核心对象
*   **加载方式：** `mmap` 映射到 Image Space，只读共享
*   **意义：** 避免每次启动都从头加载和初始化核心类，节省数百毫秒的启动时间
*   **风险：** 如果 `boot.art` 损坏或与当前 ART 版本不匹配，`InitFromBootImage` 失败 → Zygote 崩溃 → 系统反复重启

```
Boot Image 的组成（Android 12+, /apex/com.android.art/javalib/）：
├── boot.art           # 对象镜像（Class、String 等预初始化对象）
├── boot.oat           # 编译后的机器码（核心类方法的 AOT 编译结果）
├── boot.vdex          # 原始 Dex 字节码（用于解释器回退）
├── boot-framework.art # Framework 类的镜像
├── boot-framework.oat # Framework 类的编译码
└── ...                # 更多分区
```

### 5.2 启动守护线程

```cpp
    // === 2. 启动 Daemon 线程 ===
    StartDaemonThreads();
    
    return true;
}

void Runtime::StartDaemonThreads() {
    Thread* self = Thread::Current();
    
    // 通过反射调用 java.lang.Daemons.start()
    // 这会启动 4 个守护线程
    JNIEnv* env = self->GetJniEnv();
    jclass daemonsClass = env->FindClass("java/lang/Daemons");
    jmethodID startMethod = env->GetStaticMethodID(daemonsClass, "start", "()V");
    env->CallStaticVoidMethod(daemonsClass, startMethod);
}
```

4 个守护线程的启动顺序和职责在第一篇中已经介绍，这里补充它们的启动细节：

```
Daemons.start()
    ├── HeapTaskDaemon.start()      → 独立 pthread，循环执行 Heap 任务
    ├── ReferenceQueueDaemon.start() → 独立 pthread，处理引用队列
    ├── FinalizerDaemon.start()      → 独立 pthread，执行 finalize()
    └── FinalizerWatchdogDaemon.start() → 独立 pthread，看门狗
```

### 5.3 SignalCatcher 线程创建

在 `Start()` 的后期（守护线程之后），`SignalCatcher` 线程被创建：

```cpp
// Runtime::Start() 末尾（简化）
    // 创建 SignalCatcher 线程
    signal_catcher_ = new SignalCatcher();
    // SignalCatcher 构造函数内部创建 pthread，
    // 开始在 sigwait 循环中等待 SIGQUIT/SIGUSR1
```

至此，`Runtime::Start()` 完成。ART 虚拟机进入完全可运行状态：
*   Heap 已创建，可以分配对象
*   ClassLinker 已初始化，可以加载类
*   GC 守护线程已启动，可以执行垃圾回收
*   JIT 编译线程已就绪，可以编译热点方法
*   SignalCatcher 已启动，可以响应 SIGQUIT

---

## 6. ZygoteInit.main()：Java 世界的第一站

`AndroidRuntime::start()` 通过 JNI 调用 `ZygoteInit.main()` 后，执行流正式进入 Java 层。

*   **源码路径：** `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`

```java
// ZygoteInit.java（简化）
public static void main(String[] argv) {
    ZygoteServer zygoteServer = null;
    
    try {
        // 1. 标记当前处于 Zygote 初始化阶段
        //    此标记影响 ART 的内存分配策略（优先分配到 Zygote Space）
        ZygoteHooks.startZygoteNoThreadCreation();
        
        // 2. 预加载核心类和资源
        preload(bootTimingsTraceLog);
        
        // 3. 预加载完成后执行一次 GC
        //    目的：在 fork 前清理 Zygote 堆中的临时对象，
        //    减少 COW 触发的概率
        gcAndFinalize();
        
        // 4. 允许创建新线程
        ZygoteHooks.stopZygoteNoThreadCreation();
        
        // 5. 创建 Zygote Socket Server
        zygoteServer = new ZygoteServer(isPrimaryZygote);
        
        if (startSystemServer) {
            // 6. fork System Server
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
            if (r != null) {
                // 子进程（System Server）：执行 System Server 的入口方法
                r.run();
                return;
            }
        }
        
        // 7. 父进程（Zygote）：进入 Socket 循环，等待 fork 请求
        caller = zygoteServer.runSelectLoop(abiList);
        
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    }
    
    // 子进程（App）：执行 App 的入口方法
    if (caller != null) {
        caller.run();
    }
}
```

### 6.1 preload()：预加载机制详解

`preload()` 是 Zygote 启动过程中耗时最长的阶段（通常 2-5 秒），也是 COW 内存共享的核心：

```java
// ZygoteInit.java（简化）
static void preload(TimingsTraceLog bootTimingsTraceLog) {
    // === 1. 预加载 Java 核心类 ===
    preloadClasses();
    
    // === 2. 预加载 Framework 资源 ===
    preloadResources();
    
    // === 3. 预加载共享库 ===
    preloadSharedLibraries();
    
    // === 4. 预加载文本资源 ===
    preloadTextResources();
    
    // === 5. 预加载 WebView（如果需要） ===
    maybePreloadGraphicsDriver();
}
```

**preloadClasses() 的实现：**

```java
// ZygoteInit.java（简化）
private static void preloadClasses() {
    // 读取预加载类列表文件
    InputStream is = new FileInputStream(PRELOADED_CLASSES);
    // 路径：/system/etc/preloaded-classes
    // 该文件包含约 7000+ 个类名，每行一个
    
    BufferedReader br = new BufferedReader(new InputStreamReader(is));
    String line;
    int count = 0;
    while ((line = br.readLine()) != null) {
        line = line.trim();
        if (line.startsWith("#") || line.isEmpty()) continue;
        
        try {
            // 通过 Class.forName 触发类加载
            // 这会调用 ART 的 ClassLinker::DefineClass
            Class.forName(line, true, null);
            count++;
        } catch (ClassNotFoundException e) {
            Log.w(TAG, "Class not found for preloading: " + line);
        }
    }
    Log.i(TAG, "Preloaded " + count + " classes in " + elapsed + "ms.");
}
```

**预加载类列表文件（`/system/etc/preloaded-classes`）的部分内容：**

```
# Classes which are preloaded by com.android.internal.os.ZygoteInit.
# Automatically generated by frameworks/base/tools/preload/WritePreloadedClassFile.java.
android.app.Activity
android.app.Application
android.app.Dialog
android.app.Fragment
android.content.ContentProvider
android.content.Context
android.content.Intent
android.database.sqlite.SQLiteDatabase
android.graphics.Bitmap
android.os.Bundle
android.os.Handler
android.os.Looper
android.os.Message
android.view.View
android.widget.TextView
java.lang.String
java.util.ArrayList
java.util.HashMap
...
```

**稳定性架构师视角：**

1. **预加载的类存储在 Zygote Space 中**，fork 后所有 App 共享这些类对象的物理内存（COW）。如果某个 App 通过反射修改了预加载类的静态字段，会触发 COW，该页的物理内存会被拷贝一份到 App 进程——这是内存"意外上涨"的常见原因。
2. **预加载类列表是 OEM 可定制的**。某些厂商会增删类来优化启动性能或内存。如果增加了不稳定的类（初始化可能抛异常），会导致 Zygote 启动阶段的 Crash。
3. **`Class.forName(line, true, null)` 中的 `true` 表示触发类初始化（执行 `<clinit>`）**。如果某个预加载类的静态初始化块中有 Bug，会导致 Zygote 崩溃。

**preloadResources() 的实现：**

```java
// ZygoteInit.java（简化）
private static void preloadResources() {
    // 加载 Framework 的资源（通过 AssetManager）
    TypedArray ar = mResources.obtainTypedArray(
        com.android.internal.R.array.preloaded_drawables);
    preloadDrawables(ar);
    ar.recycle();
    
    ar = mResources.obtainTypedArray(
        com.android.internal.R.array.preloaded_color_state_lists);
    preloadColorStateLists(ar);
    ar.recycle();
}
```

**preloadSharedLibraries()：**

```java
private static void preloadSharedLibraries() {
    // 预加载几个关键的 Native 库
    System.loadLibrary("android");      // libandroid_runtime.so
    System.loadLibrary("compiler_rt");
    System.loadLibrary("jnigraphics");  // Bitmap JNI
}
```

### 6.2 gcAndFinalize()：fork 前的清理

```java
// ZygoteInit.java
static void gcAndFinalize() {
    // 执行 GC，清理预加载阶段产生的临时对象
    ZygoteHooks.gcAndFinalize();
    
    // 内部实现：
    // 1. System.gc() — 触发一次 Full GC
    // 2. Runtime.getRuntime().runFinalization() — 执行所有待处理的 finalize()
    // 3. System.gc() — 再次 GC，清理 finalize 后释放的对象
}
```

**为什么在 fork 前要 GC？** fork 后的 App 进程通过 COW 共享 Zygote 的物理内存页。如果 Zygote 堆中残留大量临时对象，这些对象的内存页在 App 进程的 GC 中被修改（标记或移动）时会触发 COW，白白浪费物理内存。提前 GC 清理掉这些对象，能最大化 COW 的共享效率。

### 6.3 forkSystemServer()

```java
// ZygoteInit.java（简化）
private static Runnable forkSystemServer(String abiList, 
                                          String socketName,
                                          ZygoteServer zygoteServer) {
    // System Server 的参数
    String[] args = {
        "--setuid=1000",   // system uid
        "--setgid=1000",
        "--setgroups=1001,1002,...",
        "--capabilities=...",
        "--nice-name=system_server",
        "--runtime-args",
        "--target-sdk-version=10000",
        "com.android.server.SystemServer",  // 入口类
    };
    
    // 调用 Zygote.forkSystemServer()
    // 内部调用 nativeForkSystemServer()
    //   → 调用 fork()
    //   → 子进程中调用 Runtime::DidForkFromZygote()
    int pid = Zygote.forkSystemServer(...);
    
    if (pid == 0) {
        // 子进程：System Server
        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }
    
    return null;  // 父进程：继续 Zygote 逻辑
}
```

---

## 7. fork 后的世界：App 进程的 PostFork 初始化

当 AMS 请求启动一个新 App 时，Zygote 通过 `fork()` 创建子进程。fork 后，子进程继承了 Zygote 的整个内存空间（通过 COW 共享），但**某些 ART 子系统需要重新初始化**。

### 7.1 Zygote.forkAndSpecialize

*   **源码路径：** `frameworks/base/core/java/com/android/internal/os/Zygote.java`

```java
// Zygote.java（简化）
static int forkAndSpecialize(int uid, int gid, int[] gids,
                              int runtimeFlags, ...) {
    // 1. fork 前的准备
    ZygoteHooks.preFork();
    // 内部调用 Runtime::PreZygoteFork()
    //   → 暂停 HeapTaskDaemon
    //   → 暂停 JIT 线程
    //   → 等待所有 GC 任务完成
    
    // 2. 执行 fork
    int pid = nativeForkAndSpecialize(uid, gid, gids, ...);
    
    // 3. fork 后的初始化
    if (pid == 0) {
        // 子进程
        ZygoteHooks.postForkChild(runtimeFlags, ...);
        // 内部调用 Runtime::PostZygoteFork() / DidForkFromZygote()
    } else {
        // 父进程
        ZygoteHooks.postForkParent();
    }
    
    return pid;
}
```

### 7.2 PreFork：fork 前的准备

为什么 fork 前需要特殊准备？因为 `fork()` 只复制调用线程，其他线程在子进程中不存在。如果 HeapTaskDaemon 正在执行 GC、JIT 线程正在编译，fork 后这些操作在子进程中处于未知状态，可能导致数据损坏。

*   **源码路径：** `art/runtime/runtime.cc`

```cpp
// art/runtime/runtime.cc（简化）
void Runtime::PreZygoteFork() {
    // 1. 等待并暂停 Heap 任务（GC）
    heap_->PreZygoteFork();
    
    // 2. 暂停 JIT 线程池
    if (jit_ != nullptr) {
        jit_->PreZygoteFork();
    }
    
    // 3. 暂停 SignalCatcher
    // fork 后子进程需要创建自己的 SignalCatcher
}
```

### 7.3 DidForkFromZygote：子进程的重新初始化

fork 后，子进程（App 进程）调用 `DidForkFromZygote` 重新初始化需要独立运行的子系统：

*   **源码路径：** `art/runtime/runtime.cc`

```cpp
// art/runtime/runtime.cc（简化）
void Runtime::DidForkFromZygote(JNIEnv* env, NativeBridgeAction action,
                                 const char* isa) {
    // === 1. 标记不再是 Zygote ===
    is_zygote_ = false;
    
    // === 2. 重新创建 SignalCatcher ===
    // Zygote 的 SignalCatcher 线程在 fork 后不存在于子进程中
    // 必须创建新的 SignalCatcher，否则 SIGQUIT 无法处理
    signal_catcher_ = new SignalCatcher();
    
    // === 3. 重新启动 Heap 任务线程 ===
    heap_->PostForkChildAction(self);
    // 内部：
    //   - 创建新的 HeapTaskProcessor 线程
    //   - 配置 GC 触发阈值（可能与 Zygote 不同）
    
    // === 4. 重新启动 JIT ===
    if (jit_ != nullptr) {
        jit_->PostForkChildAction(is_system_server, in_jit_zygote_pool);
        // 内部：
        //   - 创建 JIT 编译线程池
        //   - 如果有 Profile 文件，加载 Profile 引导的热点方法列表
        //   - 设置 JIT Code Cache 大小
    }
    
    // === 5. 切换 Heap Space ===
    // 将 Zygote Space "冻结"（标记为 COW 共享区域）
    // 后续 App 的对象分配将使用独立的 Allocation Space
    heap_->CreateMainMallocSpace(...);
    
    // === 6. 重置线程优先级和 cgroup ===
    // App 进程的主线程优先级和 cgroup 由 AMS 根据 App 的重要性设置
    
    // === 7. 允许 Native Bridge（如果需要，用于 x86 模拟 ARM） ===
    if (action == NativeBridgeAction::kInitialize) {
        InitNativeBridge(env);
    }
}
```

### 7.4 fork 前后的子系统状态对比

| 子系统 | Zygote 中的状态 | fork 后子进程 | 重新初始化内容 |
| :--- | :--- | :--- | :--- |
| **Heap** | 包含预加载对象的 Zygote Space | COW 共享 Zygote Space + 独立的 Main Space | 创建 Main Space 和 TLAB |
| **ClassLinker** | 已加载约 7000+ 核心类 | 继承所有已加载的类（共享） | 无需重新初始化 |
| **ThreadList** | 包含 Zygote 的线程 | 只有 fork 的线程（通常是主线程） | 后续按需创建新线程 |
| **GC 守护线程** | HeapTask/Reference/Finalizer 已运行 | 不存在（fork 不复制其他线程） | 重新创建所有守护线程 |
| **JIT** | JIT 编译线程可能在运行 | 不存在 | 重新创建 JIT 编译线程池 |
| **SignalCatcher** | 正在 sigwait 循环中 | 不存在 | 重新创建 SignalCatcher 线程 |
| **FaultHandler** | 已注册 SIGSEGV handler | 继承（信号 handler 跨 fork 保留） | 无需重新初始化 |
| **InternTable** | 包含预加载的字符串常量 | COW 共享 | 后续新增的字符串独立 |

**稳定性架构师视角：**

1. **fork 后 GC 守护线程的重建是 App 冷启动耗时的一部分**。特别是 `FinalizerDaemon` 和 `FinalizerWatchdogDaemon`，它们的启动晚于预期可能导致 `finalize()` 积压。
2. **SignalCatcher 的重建意味着 fork 后的短暂窗口内 SIGQUIT 无法被处理**。如果在这个窗口内发生 ANR 并发送 SIGQUIT，trace dump 可能失败。
3. **JIT 在 fork 后需要"冷启动"**。Zygote 积累的 JIT Profile 不会传递给子进程（Profile 是 per-process 的），子进程的 JIT 需要重新收集热点信息。这是 App 前几次启动可能比后续启动更慢的原因之一。

---

## 8. 完整启动时序图

将以上所有阶段串联，ART 的完整启动时序如下：

```
Linux init (PID=1)
│
├─ 解析 init.zygote64_32.rc
│
└─ 启动 app_process64
   │
   ├─ main() [app_main.cpp]
   │   ├─ 解析参数：--zygote --start-system-server
   │   └─ runtime.start("ZygoteInit", args)
   │
   ├─ AndroidRuntime::start() [AndroidRuntime.cpp]
   │   ├─ startVm()
   │   │   ├─ 读取 dalvik.vm.* 系统属性
   │   │   ├─ 构建 VM 选项列表
   │   │   └─ JNI_CreateJavaVM()
   │   │       ├─ Runtime::Create()
   │   │       │   └─ Runtime::Init()
   │   │       │       ├─ MemMap::Init()
   │   │       │       ├─ ThreadList 创建
   │   │       │       ├─ Thread::Attach("main")
   │   │       │       ├─ Heap 创建（Spaces + GC 策略）
   │   │       │       ├─ ClassLinker 创建 + Boot Image 加载
   │   │       │       ├─ FaultHandler 注册
   │   │       │       ├─ BlockSignals()（屏蔽 SIGQUIT）
   │   │       │       └─ JIT::Create()
   │   │       └─ Runtime::Start()
   │   │           ├─ InitFromBootImage()
   │   │           ├─ StartDaemonThreads()（HeapTask/Ref/Finalizer/Watchdog）
   │   │           └─ new SignalCatcher()
   │   ├─ startReg()
   │   │   └─ 注册 Framework JNI 方法（Binder/Surface/Media/...）
   │   └─ CallStaticVoidMethod(ZygoteInit.main)
   │
   └─ ZygoteInit.main() [ZygoteInit.java]
       ├─ preloadClasses()    （加载 7000+ 核心类）
       ├─ preloadResources()  （加载 Framework 资源）
       ├─ preloadSharedLibraries() （加载 libandroid.so 等）
       ├─ gcAndFinalize()     （fork 前 GC 清理）
       ├─ forkSystemServer()
       │   ├─ PreZygoteFork()  （暂停 GC/JIT/SignalCatcher）
       │   ├─ fork()
       │   ├─ [子进程] DidForkFromZygote()
       │   │   ├─ new SignalCatcher()
       │   │   ├─ Heap::PostForkChildAction()
       │   │   ├─ JIT::PostForkChildAction()
       │   │   └─ SystemServer.main() → 启动所有系统服务
       │   └─ [父进程] PostForkParent() → 恢复 GC/JIT
       │
       └─ runSelectLoop()     （Zygote 进入 Socket 循环）
           │
           └─ 收到 AMS 的 fork 请求
               ├─ PreZygoteFork()
               ├─ fork()
               ├─ [子进程] DidForkFromZygote()
               │   ├─ new SignalCatcher()
               │   ├─ Heap/JIT 重新初始化
               │   └─ ActivityThread.main() → App 的第一行 Java 代码
               └─ [父进程] 继续等待下一个 fork 请求
```

---

## 9. 稳定性风险地图：启动阶段的常见问题

### 9.1 Boot Image 损坏

**触发条件：** OTA 升级中断、磁盘 I/O 错误、存储空间不足导致 `boot.art` 文件不完整或校验失败。

**表现：** Zygote 反复启动失败，系统进入 bootloop。Logcat 中可见：

```
E/art: Failed to open boot image '/apex/com.android.art/javalib/boot.art': 
  Image checksum mismatch
F/art: Runtime aborting --- ...
```

**排查：** 检查 Boot Image 文件的 magic number 和 checksum。如果是 Mainline 更新导致，回滚 ART APEX 模块。

### 9.2 预加载类初始化失败

**触发条件：** OEM 在预加载类列表中加入了自定义类，该类的 `<clinit>` 依赖了运行时才可用的服务（如 PackageManagerService），在 Zygote 阶段这些服务尚未启动。

**表现：** Zygote 启动时 Crash，异常类型通常是 `ExceptionInInitializerError`。

**排查：** 检查预加载类列表（`/system/etc/preloaded-classes`）中的自定义类，确认其 `<clinit>` 不依赖运行时服务。

### 9.3 fork 后 SignalCatcher 未就绪

**触发条件：** App 进程 fork 后立即遇到 ANR，此时 SignalCatcher 线程尚未创建完成。

**表现：** AMS 发送的 SIGQUIT 导致进程以默认行为退出（core dump），而非正常 dump 堆栈。表现为 ANR 但无 traces 文件。

**排查：** 检查 ANR 时间与进程创建时间的差值，如果极短（< 100ms），可能命中此窗口。这属于极端边界情况，通常不需要修复。

### 9.4 FinalizerWatchdog 超时

**触发条件：** Zygote 启动阶段的 `gcAndFinalize()` 或 App 启动初期，`FinalizerDaemon` 执行某个对象的 `finalize()` 超过 10 秒。

**表现：** `java.util.concurrent.TimeoutException` → 进程 abort。

**排查：** 检查 `finalize()` 方法中的 I/O 操作、锁等待。优先消除所有自定义类的 `finalize()`，用 `Cleaner`（Java 9+）或 `try-with-resources` 替代。

---

## 10. 实战案例

### 10.1 案例一：App 冷启动耗时波动排查

**现象：** 某金融 App 反馈冷启动耗时波动极大。同一台设备，有时 1.2 秒，有时 3.5 秒。特征：首次安装后前几次启动慢，使用几天后变快，但清除数据后又变慢。

**分析思路：**

1. **怀疑 JIT Profile 影响。** 根据 ART 混合编译机制，前几次启动没有 Profile，走解释+JIT；积累 Profile 后，后台 dex2oat 编译热点方法，后续启动可以直接走 AOT 编译码。
2. **验证：** 通过 `adb shell cmd package compile -m speed-profile -f com.example.app` 触发强制编译，启动耗时稳定在 1.2 秒。清除编译产物（`adb shell cmd package compile --reset com.example.app`）后恢复到 3.5 秒。
3. **根因确认：** 该 App 没有配置 Baseline Profile，冷启动路径的方法全部走解释器执行。

**修复方案：**

1. **添加 Baseline Profile：** 使用 Macrobenchmark 库录制启动路径的方法签名，生成 `baseline-prof.txt`，随 APK 一起打包。安装时系统自动对这些方法进行 AOT 编译。
2. **效果：** 配置 Baseline Profile 后，首次启动耗时从 3.5 秒降至 1.5 秒，与后续启动的差距从 2.3 秒缩小到 0.3 秒。

### 10.2 案例二：Zygote 反复崩溃导致系统 bootloop

**现象：** 某 OEM 厂商的工程样机在刷入新系统后反复重启，Logcat 中大量 Zygote 崩溃日志。

**分析 Logcat：**

```
E/Zygote: ... ExceptionInInitializerError ...
E/Zygote: Caused by: java.lang.NullPointerException:
  Attempt to invoke virtual method 'String android.content.pm.PackageManager
  .getInstallerPackageName(String)' on a null object reference
E/Zygote:   at com.oem.security.DeviceVerifier.<clinit>(DeviceVerifier.java:45)
E/Zygote:   at java.lang.Class.forName0(Native Method)
E/Zygote:   at java.lang.Class.forName(Class.java:454)
E/Zygote:   at com.android.internal.os.ZygoteInit.preloadClasses(ZygoteInit.java:...)
```

**根因分析：**

1. OEM 在预加载类列表中添加了自定义类 `com.oem.security.DeviceVerifier`
2. 该类的 `<clinit>`（静态初始化块）中调用了 `context.getPackageManager()`
3. 在 Zygote 预加载阶段，`PackageManagerService` 尚未启动（它在 System Server 中启动），因此 `getPackageManager()` 返回 null
4. `NullPointerException` → `ExceptionInInitializerError` → Zygote 崩溃 → 系统重启 → 再次崩溃 → bootloop

**修复方案：**

1. **从预加载类列表中移除 `DeviceVerifier`**，改为在 App 进程中按需加载。
2. **`DeviceVerifier` 的 `<clinit>` 不应依赖运行时服务。** 将 `PackageManager` 的调用改为懒加载（首次使用时初始化）。

---

## 11. 总结

本篇从 `init.rc` 出发，完整追踪了 ART 从诞生到第一行 App Java 代码执行的全流程：

1. **Linux init → app_process → AndroidRuntime::start()**：这是从 Linux 世界到 Java 世界的桥梁。`startVm()` 读取 `dalvik.vm.*` 系统属性配置 ART，`startReg()` 注册 Framework JNI 方法。
2. **JNI_CreateJavaVM → Runtime::Init**：按严格顺序初始化 ART 的 12+ 个子系统——MemMap、ThreadList、Heap、ClassLinker、FaultHandler、JIT 等。任何一步失败都意味着 Zygote 崩溃。
3. **Runtime::Start**：加载 Boot Image、启动 4 个守护线程和 SignalCatcher，ART 进入可运行状态。
4. **ZygoteInit.main()**：预加载 7000+ 核心类和 Framework 资源（这是 COW 内存共享的基础），然后进入 Socket 循环等待 fork 请求。
5. **fork 后的 DidForkFromZygote**：子进程重新创建 SignalCatcher、GC 守护线程、JIT 编译线程——这些在 fork 中丢失的线程必须重建，否则 App 进程的 ART 无法正常工作。

理解这条完整链路，你就能精确定位启动阶段的各类异常：Boot Image 校验失败在哪一步、预加载类异常在哪一步、守护线程超时在哪一步。这些都是系统级稳定性问题的排查基础。
