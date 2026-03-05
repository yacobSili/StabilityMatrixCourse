# 面向稳定性的 ART 架构解析系列

## 为什么要写这个系列
jiabo
作为安卓稳定性架构师，我们日常面对的 OOM、ANR、Native Crash、死锁、内存泄漏、启动超时……**90% 的根因最终都指向 ART**。因为所有的 Java/Kotlin 代码——无论是 App 还是 System Server——都跑在 ART 之上。

但市面上关于 ART 的资料，要么是面试八股（浅），要么是纯源码走读（散），没有一套**从架构师视角出发、以稳定性治理为目标、以源码为依据**的体系化材料。

本系列的目标：**让你在遇到任何 ART 相关的线上问题时，能快速定位到对应的子系统、找到对应的源码文件、理解问题的根因机制，并给出治理方案。**

## 系列设计思路

架构师看一个系统，遵循的逻辑链是：

```
它是什么？解决什么问题？（定位）
        ↓
它在系统中处于什么位置？和谁协作？（边界与交互）
        ↓
它内部是怎么运转的？（核心机制）
        ↓
它会在什么地方出问题？（风险地图）
        ↓
出了问题我怎么查？怎么防？（诊断与治理）
```

本系列按照这条逻辑链，分为三个篇章：

---

## 第一篇章：建立全局观（1 篇）

> 核心问题：ART 是什么？从哪来？在哪里？和谁打交道？

### [01-ART总览：稳定性架构师的全局视角](01-ART总览.md)

| 章节 | 内容 | 你将获得 |
| :--- | :--- | :--- |
| **1. ART 是什么** | 一句话定义 ART 的职责边界 | 清晰的概念锚点 |
| **2. 背景与历史演进** | Dalvik → AOT → 混合编译 → Cloud Profile → Mainline，每个阶段解决了什么、又引入了什么稳定性挑战 | 理解"为什么 ART 长成今天这个样子" |
| **3. 架构定位与层级** | ART 在 Android 分层架构中的精确位置（Framework 之下、Native 库之上） | 全局架构图 |
| **4. 为谁服务** | System Server / App 进程 / 系统应用 —— ART 的三类"客户" | 理解 ART 崩溃的影响面（System Server 崩 = 系统重启） |
| **5. 提供什么能力** | 字节码执行、内存管理与 GC、类加载与链接、线程管理、JNI 桥接 —— 五大核心能力及其对应的稳定性问题 | 能力与风险的对照表 |
| **6. 与其他模块的交互** | 与 Zygote（COW/fork）、Binder（线程池/跨进程死锁）、Linux 内核（Signal/mmap/FD）、Framework（反射/启动链路）的深度交互分析 | 疑难杂症的"交叉路口"地图 |
| **7. 源码目录结构** | AOSP `art/` 目录下 12 个核心子目录的职责与稳定性关注点 | 排查问题时的"导航地图" |
| **8. 虚拟机启动流程** | `JNI_CreateJavaVM` → `Runtime::Init`（Heap/Thread/ClassLinker/JIT）→ `Runtime::Start`（BootImage/守护线程），源码级走读 | 理解启动 Crash 和 FinalizerWatchdog TimeoutException 的根因 |

---

## 第二篇章：核心机制深潜（5 篇）

> 核心问题：ART 内部是怎么运转的？每个子系统的源码结构、关键数据结构、核心流程是什么？

### [02-字节码与指令集：Dex 文件的骨骼](02-字节码与指令集.md)

这是理解 ART 一切行为的**最底层基础**——ART 执行的不是 Java 源码，而是 Dex 字节码。

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| **1. 为什么是 Dex 而不是 Class** | 基于寄存器 vs 基于栈的本质区别；Dex 格式如何为移动端优化（合并常量池、减少 I/O） | — | 理解 ART 与 JVM 差异的起点 |
| **2. Dex 文件结构** | Header（magic/checksum/signature）、StringIDs、TypeIDs、ProtoIDs、FieldIDs、MethodIDs、ClassDefs、Data Section 的二进制布局 | `art/libdexfile/dex/dex_file.h` | Dex 损坏导致的 Crash；65536 方法数限制的物理来源（MethodID 索引 16 位） |
| **3. CodeItem 与方法体** | `registers_size_`、`ins_size_`、`insns_[]` 的含义；寄存器分配与方法栈帧的关系 | `art/libdexfile/dex/dex_file_structs.h` | ASM 插桩导致寄存器越界 → VerifyError |
| **4. Dalvik 指令集** | 数据移动、算术运算、对象操作、`invoke-*`（virtual/super/direct/static/interface）五种调用方式的语义差异、控制流 | `art/libdexfile/dex/dex_instruction.h` | `IncompatibleClassChangeError`、`AbstractMethodError` 的本质 |
| **5. 解释器执行循环** | `ExecuteSwitchImpl` 的 while-switch 主循环；指令取码 → 解码 → 分发 → 执行的完整流程 | `art/runtime/interpreter/interpreter_switch_impl.cc` | 如何通过 ANR 堆栈中的 `dex_pc` 定位卡死的具体指令 |

### [03-编译与执行：从字节码到机器码的三条路](03-编译与执行.md)

ART 的"三种执行模式"直接决定了应用的启动速度、运行性能和稳定性。

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| **1. 三种执行模式的全景** | 解释器（Interpreter）、JIT、AOT 各自的工作场景、性能特征和切换时机 | — | 为什么同一个 Bug 在不同机型/不同启动次数下表现不同 |
| **2. 解释器深入** | Mterp（汇编优化）vs Switch Interpreter（C++ 实现）；ShadowFrame 栈帧结构 | `art/runtime/interpreter/interpreter.cc` | StackOverflowError 的两种来源（Java 栈 vs Native 栈） |
| **3. JIT 编译** | 热度计数器（`AddSamples`）→ 阈值触发 → 编译任务（`JitCompileTask`）→ 入口替换（`SetEntryPointFromQuickCompiledCode`）的完整链路；JitCodeCache 的结构与 GC | `art/runtime/jit/jit.cc`、`art/runtime/jit/jit_code_cache.cc` | JIT 线程抢占前台 CPU → Jank；Code Cache 满 → 性能退化 |
| **4. AOT 编译与 dex2oat** | OAT 文件结构（OatHeader、ELF 格式）；`dex2oat` 的编译流程；Vdex/Odex/Art 文件三件套的关系 | `art/dex2oat/dex2oat.cc`、`art/runtime/oat_file.h` | 后台 dex2oat I/O 风暴 → 前台 ANR；OAT 损坏 → SIGBUS |
| **5. PGO 与 Baseline Profile** | Profile 文件的格式与收集时机；Cloud Profile 下发机制；Baseline Profile 的打包与安装时编译 | `art/runtime/jit/profile_saver.cc` | 冷启动耗时波动的根因与治理 |
| **6. 蹦床机制（Trampolines）** | `art_quick_to_interpreter_bridge`、`art_quick_generic_jni_trampoline`、`art_quick_resolution_trampoline` 的作用 | `art/runtime/arch/arm64/quick_entrypoints_arm64.S` | Hook 框架修改 EntryPoint 引发的 Crash |
| **7. OSR（On-Stack Replacement）** | 循环体热点的栈上替换机制；解释器栈帧 → 编译码栈帧的动态切换 | `art/runtime/jit/jit.cc` | OSR 过程中异常/GC 引发的罕见 Native Crash |

### [04-类加载与链接：ClassNotFoundException 到 VerifyError 的全路径](04-类加载与链接.md)

类加载是 ART 最复杂、最容易出错的子系统之一。

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| **1. Android ClassLoader 体系** | BootClassLoader → PathClassLoader → DexClassLoader 的层级关系；`DexPathList` 与 `dexElements` 数组的内部结构 | `libcore/dalvik/src/main/java/dalvik/system/` | 热修复原理（插入 dexElements）；双亲委派被破坏的风险 |
| **2. ClassLinker::DefineClass** | 类加载的完整 Native 流程：查找 → 分配 Class 对象 → 插入 ClassTable（锁竞争）→ 链接 | `art/runtime/class_linker.cc` | ClassTable 锁竞争 → 主线程卡顿；频繁加载 Patch Dex → Non-Moving Space OOM |
| **3. 链接三步骤** | **Verify**（字节码校验，`MethodVerifier`）→ **Prepare**（字段内存布局）→ **Resolve**（符号引用 → 直接引用） | `art/runtime/verifier/method_verifier.cc` | VerifyError 的详细错误码解析；软失败阻止 AOT 编译 → 性能退化 |
| **4. 类初始化与 `<clinit>`** | `<clinit>` 的触发时机（首次 new / 首次静态访问 / 反射）；初始化失败后的 Erroneous 状态 | `art/runtime/class_linker.cc` | `ExceptionInInitializerError` → 后续所有访问抛 `NoClassDefFoundError` 的连锁反应 |
| **5. MultiDex 与 OAT 加载** | `OatFileManager::OpenDexFilesFromOat` 的加载策略；OAT 无效时的回退机制 | `art/runtime/oat_file_manager.cc` | MultiDex 加载 ANR（Dalvik 时代遗留）；OAT 校验失败的回退开销 |

### [05-内存与GC：OOM 的七种死法](05-内存与GC.md)

这是稳定性架构师的**主战场**。

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| **1. Heap 空间划分** | Image Space（只读共享）、Zygote Space（COW）、Allocation Space（App 对象）、Large Object Space（大对象）、Non-Moving Space（Class 元数据）的职责与边界 | `art/runtime/gc/heap.h`、`art/runtime/gc/space/` | 各 Space 的容量配置与 OOM 的关系 |
| **2. 内存分配器** | RosAlloc（TLAB + Run-of-slots + BitMap）vs DlMalloc；TLAB 的无锁分配原理 | `art/runtime/gc/allocator/rosalloc.cc` | TLAB 快速路径 vs 慢速路径的性能差异；碎片化导致的"虚假 OOM" |
| **3. OOM 的多重面孔** | Java Heap OOM、FD OOM、Thread OOM、VSS（虚拟内存）OOM、VMA OOM —— 每种 OOM 的触发条件、日志特征和排查手段 | `art/runtime/gc/heap.cc`、`art/libartbase/base/mem_map.cc` | 区分不同类型 OOM 的实战指南 |
| **4. GC 算法演进** | CMS（标记-清除）→ CC（并发复制，Read Barrier）→ Generational CC（分代）的算法原理、STW 时间对比 | `art/runtime/gc/collector/concurrent_copying.cc`、`art/runtime/read_barrier.h` | CC GC 的读屏障如何影响 Hook 框架（SandHook/Epic） |
| **5. GC 触发时机与日志** | `GC_FOR_ALLOC`（分配失败）、`GC_CONCURRENT`（阈值触发）、`GC_EXPLICIT`（手动 System.gc()）的区分；GC 日志格式解读 | `art/runtime/gc/heap.cc` | 频繁 GC_FOR_ALLOC = 内存抖动的铁证 |
| **6. Finalizer 机制** | `FinalizerDaemon` 的工作原理；`FinalizerWatchdogDaemon` 的 10 秒超时机制；为什么 `finalize()` 是稳定性毒药 | `art/runtime/gc/heap.cc`、`libcore/.../FinalizerDaemon.java` | `TimeoutException` 的根因与治理 |

### [06-JNI：Java 与 Native 的边界战争](06-JNI.md)

JNI 是 Java 世界与 C++ 世界的边界，也是稳定性的"百慕大三角"。

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| **1. JavaVM 与 JNIEnv** | `JavaVMExt`（进程唯一）vs `JNIEnvExt`（线程唯一）的数据结构；为什么 JNIEnv 不能跨线程 | `art/runtime/java_vm_ext.h`、`art/runtime/jni/jni_env_ext.h` | 跨线程传递 JNIEnv → abort |
| **2. 引用管理** | `IndirectReferenceTable` 的底层实现；Local Reference（512 上限）vs Global Reference（无上限但需手动释放）vs Weak Global Reference | `art/runtime/indirect_reference_table.h` | `local reference table overflow` Crash；Global Reference 泄漏 → Java 堆 OOM |
| **3. 关键 JNI 函数源码** | `FindClass`（ClassLoader 上下文问题）、`GetMethodID`（缓存必要性）、`CallVoidMethod`（异常检查）、`RegisterNatives`（Native 方法注册） | `art/runtime/jni/jni_internal.cc` | Native 线程 FindClass 失败；未检查 Exception → 后续 Crash |
| **4. CheckJNI 机制** | CheckJNI 的检查项列表（参数合法性、Critical 区限制、引用类型匹配）；开启方式（`debug.checkjni`） | `art/runtime/jni/check_jni.cc` | Debug 阶段发现隐蔽 JNI Bug 的利器 |
| **5. 线程状态切换** | `kRunnable` ↔ `kNative` 状态切换的开销（SafePoint 检查）；JNI Critical 区的 GC 阻塞问题 | `art/runtime/thread.h` | 高频 JNI 调用的性能开销；`GetPrimitiveArrayCritical` 阻塞 GC → ANR |

---

## 第三篇章：横向对比与落地实战（2 篇）

> 核心问题：ART 为什么做了这些选择？作为架构师我该怎么用这些知识？

### [07-ART与JVM：设计哲学的分野](07-ART与JVM.md)

| 章节 | 内容 | 你将获得 |
| :--- | :--- | :--- |
| **1. 指令集对比** | 基于寄存器（Dalvik）vs 基于栈（JVM）的性能差异与设计取舍 | 理解为什么 Android 选择自研而非复用 |
| **2. 内存管理对比** | CMS/G1/ZGC vs CC/GenCC；TLAB 实现差异；分代策略差异 | OOM 排查思路的跨平台迁移 |
| **3. 编译策略对比** | C1/C2 分层编译 vs 解释器/JIT/AOT 混合编译；为什么 JVM 没有标准 AOT（直到 GraalVM） | 理解 ART 对"启动速度"的极致追求 |
| **4. 类加载对比** | JVM 的 PermGen/Metaspace vs ART 的 Non-Moving Space；ClassLoader 隔离差异 | 热修复/插件化在两个平台上可行性差异的根因 |
| **5. 监控工具对比** | JMX/JFR/Arthas vs JVMTI/Perfetto/Simpleperf | 工具选型参考 |

### [08-演进与未来：架构师的下半场](08-演进与未来.md)

| 章节 | 内容 | 你将获得 |
| :--- | :--- | :--- |
| **1. Mainline 模块化** | ART 从系统固件剥离为 APEX 模块；库路径变化（`/system/lib64/` → `/apex/com.android.art/`）；版本检测策略 | 适配 Mainline 的实操指南 |
| **2. 异常处理与信号机制** | `FaultHandler`（隐式空指针/栈溢出）、`sigchain`（信号链管理）、`UncaughtExceptionHandler` 的完整 Crash 捕获链路 | 理解 APM 框架（xCrash/Breakpad）与 ART 共存的原理与冲突 |
| **3. 监控与诊断基础设施** | JVMTI（Android 8.0+）的能力边界（内存分配监控/GC 回调/方法进入退出）；ART Method Hook 的三种流派（EntryPoint 替换/Inline Hook/Trampoline）；Systrace/Perfetto 的 ART 埋点 | APM 体系建设的技术选型依据 |
| **4. Hook 框架深度分析** | Epic/SandHook/Pine 的实现原理；CC GC 读屏障对 Hook 的影响；Android 版本兼容性矩阵 | Hook 方案选型与风险评估 |
| **5. 未来趋势** | Cloud Profile 2.0（设备差异化下发）；硬件加速 GC；Kotlin/Native 与 ART 的关系 | 技术前瞻 |

---

## 阅读建议

**如果你时间有限，优先阅读：**
1.  **01 总览** — 建立全局观，这是所有后续文章的地基。
2.  **05 内存与 GC** — OOM 是稳定性的头号敌人，这是实战价值最高的一篇。
3.  **06 JNI** — Native Crash 的高发区，架构师必须掌握。

**如果你要系统学习，按顺序阅读 01 → 08。** 每篇文章的设计逻辑是：
```
背景与定义（是什么、为什么）
    → 架构与交互（在哪里、和谁打交道）
        → 核心机制与源码（怎么工作的，源码在哪）
            → 稳定性风险点（会在哪里出问题）
                → 实战案例（线上真实问题的排查过程）
```
