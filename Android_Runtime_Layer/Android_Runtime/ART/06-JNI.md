# 06-JNI：Java 与 Native 的边界战争

JNI（Java Native Interface）是 Java 世界与 C++ 世界的边界，也是稳定性的"百慕大三角"。每一次跨越这条边界，都涉及引用转换、线程状态切换、内存管理模型的碰撞。在稳定性架构师的日常中，`SIGSEGV`、`local reference table overflow`、`JNI DETECTED ERROR`、`FindClass returned null` 这些错误信息反复出现，而它们的根因无一例外地埋藏在 JNI 的机制深处。

本篇将从 ART 源码层面，系统剖析 JNI 的五大核心机制，并在每个环节标注稳定性风险点，最后通过两个线上真实案例串联全文。

---

## 1. JavaVM 与 JNIEnv

### 1.1 为什么需要两个结构体

JNI 规范定义了两个核心接口结构体：`JavaVM` 和 `JNIEnv`。初学者常常混淆它们，但对稳定性架构师来说，理解它们的**生命周期和线程归属**是避免一类致命 Bug 的前提。

核心区分：

| 结构体 | 生命周期 | 唯一性 | 用途 |
| :--- | :--- | :--- | :--- |
| `JavaVM` | 与进程相同 | 进程唯一（一个进程只有一个 ART 虚拟机实例） | 获取 `JNIEnv`、Attach/Detach 线程 |
| `JNIEnv` | 与线程相同 | 线程唯一（每个线程有自己独立的 `JNIEnv`） | 调用所有 JNI 函数（FindClass、NewObject、CallMethod 等） |

**为什么这样设计？** 因为 `JNIEnv` 内部维护着大量**线程局部状态**——最重要的是 Local Reference Table（本地引用表）。每个 JNI 函数返回的 Java 对象引用都被记录在调用线程的 Local Reference Table 中，用于防止对象在 Native 代码执行期间被 GC 回收。如果允许 `JNIEnv` 跨线程使用，两个线程同时操作同一张引用表就会产生数据竞争，导致引用表损坏。

### 1.2 ART 中的实现：JavaVMExt 与 JNIEnvExt

ART 对标准 JNI 的 `JavaVM` 和 `JNIEnv` 进行了扩展，分别实现为 `JavaVMExt` 和 `JNIEnvExt`。

*   **源码路径：** `art/runtime/java_vm_ext.h`

```cpp
// art/runtime/java_vm_ext.h (简化)
class JavaVMExt : public JavaVM {
  Runtime* const runtime_;

  // 全局引用表——存储所有 NewGlobalRef 创建的引用
  IndirectReferenceTable globals_ GUARDED_BY(Locks::jni_globals_lock_);

  // 弱全局引用表
  IndirectReferenceTable weak_globals_ GUARDED_BY(Locks::jni_weak_globals_lock_);

  // 已加载的 Native 库列表 (System.loadLibrary 注册的 .so)
  std::unique_ptr<Libraries> libraries_ GUARDED_BY(Locks::jni_libraries_lock_);

  // 当前 Attach 到 VM 的线程列表的检查
  // 用于 CheckJNI 验证 JNIEnv 是否属于当前线程
  bool check_jni_;
};
```

*   **源码路径：** `art/runtime/jni/jni_env_ext.h`

```cpp
// art/runtime/jni/jni_env_ext.h (简化)
class JNIEnvExt : public JNIEnv {
  // 归属线程——JNIEnvExt 与创建它的 Thread 一一绑定
  Thread* const self_;

  // 本地引用表——存储该线程所有 JNI 函数返回的 jobject
  IndirectReferenceTable locals_;

  // 关键区计数器——GetPrimitiveArrayCritical 的嵌套深度
  // 非零时禁止 GC 移动对象
  uint32_t critical_;

  // CheckJNI 的运行时检查函数表
  const JNINativeInterface* unchecked_functions_;
};
```

关键数据结构的关系图：

```
进程 (1 个 JavaVMExt)
    ├── globals_ (IndirectReferenceTable)      ← 所有线程共享，需加锁
    ├── weak_globals_ (IndirectReferenceTable)  ← 所有线程共享，需加锁
    ├── libraries_ (已加载的 .so 列表)
    │
    ├── Thread-1 ── JNIEnvExt-1
    │                  ├── locals_ (IndirectReferenceTable)  ← 线程私有
    │                  └── critical_ = 0
    │
    ├── Thread-2 ── JNIEnvExt-2
    │                  ├── locals_ (IndirectReferenceTable)  ← 线程私有
    │                  └── critical_ = 0
    │
    └── Thread-N ── JNIEnvExt-N
                       └── ...
```

### 1.3 JNIEnv 的获取方式

在 Native 代码中获取 `JNIEnv` 有两种方式，适用场景不同：

**方式一：JNI 函数参数（最常见）**

```cpp
// JNI 函数的第一个参数就是当前线程的 JNIEnv
extern "C" JNIEXPORT void JNICALL
Java_com_example_NativeLib_process(JNIEnv* env, jobject thiz) {
    // env 自动绑定到调用线程，安全使用
    jclass clazz = env->FindClass("com/example/Config");
}
```

**方式二：通过 JavaVM 获取（Native 线程必须）**

```cpp
// 在 Native 创建的 pthread 中
void* native_thread_func(void* arg) {
    JavaVM* vm = GetCachedJavaVM();  // 进程唯一，可安全缓存
    JNIEnv* env = nullptr;

    // AttachCurrentThread 会为当前 pthread 创建一个新的 JNIEnvExt
    // 同时在 ART 的 ThreadList 中注册该线程
    jint result = vm->AttachCurrentThread(&env, nullptr);
    if (result != JNI_OK) {
        // Attach 失败——通常因为 FD 耗尽无法创建线程
        return nullptr;
    }

    // 使用 env 调用 JNI 函数...
    env->FindClass("com/example/Config");

    // 必须 Detach！否则 ART 认为线程仍活跃，GC 会扫描已销毁的栈
    vm->DetachCurrentThread();
    return nullptr;
}
```

### 1.4 稳定性：跨线程传递 JNIEnv → abort

这是 JNI 最常见的致命错误之一。当开启 CheckJNI 时（debug 包默认开启），ART 在每个 JNI 调用入口检查 `env->self_` 是否等于 `Thread::Current()`：

```cpp
// art/runtime/jni/check_jni.cc (概念性)
static void CheckThread(JNIEnv* env) {
    Thread* self = Thread::Current();
    JNIEnvExt* ext = reinterpret_cast<JNIEnvExt*>(env);
    if (UNLIKELY(ext->self_ != self)) {
        // JNIEnv 不属于当前线程！
        JniAbort("JNI ERROR: using JNIEnv* from wrong thread");
    }
}
```

典型的错误模式：

```cpp
// 错误！在线程 A 获取 env，传递给线程 B 使用
JNIEnv* g_env;  // 全局变量缓存 JNIEnv

void thread_a_func(JNIEnv* env) {
    g_env = env;  // 危险：将线程 A 的 env 存为全局
}

void thread_b_func() {
    // 使用线程 A 的 env 在线程 B 中调用 JNI
    // → CheckJNI 检测到不匹配 → JniAbort → 进程终止
    g_env->FindClass("com/example/Foo");
}
```

**即使不开 CheckJNI（release 包），跨线程使用 JNIEnv 也不安全**——Local Reference Table 的并发读写会导致引用表数据结构损坏，最终表现为随机的 `SIGSEGV`，极难定位。

**正确做法：** 永远不要缓存 `JNIEnv*`。在非 JNI 回调线程中，通过缓存 `JavaVM*`（进程唯一，线程安全）+ `AttachCurrentThread` 获取属于当前线程的 `JNIEnv`。

---

## 2. 引用管理

### 2.1 为什么 JNI 需要引用管理

这是 JNI 设计中最精妙也最容易出错的部分。

Java 世界有 GC 自动回收内存，C++ 世界靠手动 `new`/`delete`。当 Native 代码持有一个 Java 对象的引用时，GC 如何知道这个对象"还活着"？答案是 **JNI 引用表**——Native 代码中使用的每一个 `jobject`，都不是指向 Java 对象的直接指针，而是一个**间接引用（Indirect Reference）**，指向引用表中的一个槽位（slot），槽位再指向实际的 Java 对象。

这层间接引用的意义：
1. **GC 可见性**：引用表中的所有槽位都是 GC 的根（GC Root），GC 扫描时会遍历引用表，确保 Native 持有的对象不被回收。
2. **GC 移动安全**：当 CC GC（Concurrent Copying）移动对象时，只需更新引用表中的槽位值，而不需要修改 Native 代码中散落的指针。

### 2.2 IndirectReferenceTable 的底层实现

*   **源码路径：** `art/runtime/indirect_reference_table.h`

```cpp
// art/runtime/indirect_reference_table.h (简化)
class IndirectReferenceTable {
  // 表的类型：Local / Global / Weak Global
  IndirectRefKind kind_;

  // 底层存储：一个 IrtEntry 数组
  // 每个 IrtEntry 包含一个 GcRoot<mirror::Object>（指向 Java 对象）
  // 和一个 serial（版本号，用于检测 stale reference）
  IrtEntry* table_;

  // 当前已使用的槽位数
  size_t segment_state_;

  // 容量上限
  size_t max_entries_;
};
```

`IndirectReferenceTable` 是 ART 实现 JNI 三种引用类型的统一数据结构。每个 `jobject`/`jclass`/`jstring` 在 Native 层看到的值，本质上是 `(table_index << 2) | kind_` 的编码——高位是表索引，低 2 位标识引用类型。

### 2.3 三种引用类型的对比

| 特性 | Local Reference | Global Reference | Weak Global Reference |
| :--- | :--- | :--- | :--- |
| **存储位置** | `JNIEnvExt::locals_` | `JavaVMExt::globals_` | `JavaVMExt::weak_globals_` |
| **生命周期** | JNI 函数返回时自动释放 | 手动调用 `DeleteGlobalRef` 释放 | 手动调用 `DeleteWeakGlobalRef` 释放 |
| **数量上限** | 512（可配置） | 无硬上限（受内存限制） | 无硬上限 |
| **防 GC 回收** | 是 | 是 | 否（对象可被回收，引用变为 null） |
| **线程安全** | 否（线程私有） | 是（全局锁保护） | 是（全局锁保护） |
| **创建方式** | JNI 函数自动返回 | `NewGlobalRef(env, localRef)` | `NewWeakGlobalRef(env, localRef)` |

**Local Reference 的 512 上限机制：**

*   **源码路径：** `art/runtime/jni/jni_env_ext.cc`

```cpp
// art/runtime/jni/jni_env_ext.cc (概念性)
static constexpr size_t kLocalsInitial = 64;    // 初始容量
static constexpr size_t kLocalsMax = 512;       // 上限

bool JNIEnvExt::CheckLocalsValid(size_t entries) {
    if (UNLIKELY(entries > kLocalsMax)) {
        // 触发 local reference table overflow
        std::string msg = StringPrintf(
            "local reference table overflow (max=%zu)", kLocalsMax);
        vm_->JniAbort(msg.c_str());
        return false;
    }
    return true;
}
```

当 Local Reference 的数量超过 512 时，ART 会直接 abort 进程。**这不是一个可以 catch 的异常，而是直接终止。**

### 2.4 Local Reference 的作用域与手动释放

Local Reference 在一个"Local Frame"中生存。正常的 JNI 函数调用自带一个 Local Frame——函数返回时，该 Frame 中的所有 Local Reference 自动释放。

但在循环中创建大量临时对象时，如果不手动释放，会迅速耗尽 512 的上限：

```cpp
// 危险！循环中不释放 Local Reference
void processLargeArray(JNIEnv* env, jobjectArray array) {
    jsize len = env->GetArrayLength(array);
    for (jsize i = 0; i < len; i++) {
        // 每次循环创建一个新的 Local Reference
        jobject element = env->GetObjectArrayElement(array, i);
        // ... 处理 element ...

        // 如果 len > 512，到第 513 次循环时 → abort!
        // 必须手动释放:
        env->DeleteLocalRef(element);
    }
}
```

另一种方案是使用 `PushLocalFrame` / `PopLocalFrame`：

```cpp
void processLargeArray(JNIEnv* env, jobjectArray array) {
    jsize len = env->GetArrayLength(array);
    for (jsize i = 0; i < len; i++) {
        env->PushLocalFrame(16);  // 开一个容量 16 的新 Frame
        jobject element = env->GetObjectArrayElement(array, i);
        // ... 处理 element ...
        env->PopLocalFrame(nullptr);  // 自动释放 Frame 中所有 Local Ref
    }
}
```

### 2.5 稳定性：Local Reference 溢出与 Global Reference 泄漏

**问题一：`local reference table overflow` Crash**

这是最常见的 JNI Crash 之一。典型日志：

```
JNI ERROR (app bug): local reference table overflow (max=512)
local reference table dump:
  Last 10 entries (of 512):
      511: 0x12e345a0 java.lang.String
      510: 0x12e34580 java.lang.String
      ...
```

高发场景：
- Native 循环中调用 JNI 函数创建对象但未释放
- Native 回调中频繁通过 `FindClass` / `GetMethodID` 获取引用但未清理
- 第三方 so 库内部的 JNI 代码未正确管理引用

**问题二：Global Reference 泄漏 → Java 堆 OOM**

Global Reference 会阻止对象被 GC 回收。如果创建了 `NewGlobalRef` 但忘记 `DeleteGlobalRef`，被引用的 Java 对象及其引用链上的所有对象都无法回收，等效于内存泄漏。

与 Java 层内存泄漏不同的是：**Global Reference 泄漏在常规的 Java 堆 Dump 分析（MAT/LeakCanary）中极难发现**——因为 GC Root 显示为 `JNI Global Reference`，你看到对象被 JNI 持有，但不知道是哪段 Native 代码创建了这个引用。

排查手段：
1. `adb shell dumpsys meminfo <pid>` 查看 `JNI Global References` 的数量是否持续增长。
2. 使用 `GetGlobalReferenceCount` 在关键节点记录 Global Reference 数量的趋势。
3. Debug 模式下开启 CheckJNI，它会在 `DeleteGlobalRef` 时检查引用是否有效。

---

## 3. 关键 JNI 函数源码

### 3.1 为什么要看 JNI 函数的源码

大多数 Native 开发者将 JNI 函数当作黑盒使用：调用 `FindClass` 找类、`GetMethodID` 获取方法、`CallVoidMethod` 执行方法。但在稳定性的视角下，每个 JNI 函数内部都有**隐含的前置条件和异常路径**。不理解这些，就会写出"看起来正确、跑起来随机 Crash"的代码。

*   **源码路径：** `art/runtime/jni/jni_internal.cc`

### 3.2 FindClass：ClassLoader 上下文问题

`FindClass` 是使用频率最高的 JNI 函数，也是问题最多的。

```cpp
// art/runtime/jni/jni_internal.cc (简化)
static jclass FindClass(JNIEnv* env, const char* name) {
    // 1. 获取当前线程
    ScopedObjectAccess soa(env);

    // 2. 确定使用哪个 ClassLoader
    //    关键问题：如果当前是 Native 线程（通过 AttachCurrentThread 附着），
    //    调用栈中没有 Java 帧，ART 无法推断应该使用哪个 ClassLoader
    ObjPtr<mirror::ClassLoader> class_loader = nullptr;

    // 遍历调用栈，找到最近的 Java 帧，使用该帧所属类的 ClassLoader
    // 如果找不到（纯 Native 线程），使用系统 ClassLoader (BootClassLoader)
    ArtMethod* caller = GetCallingMethod(soa);
    if (caller != nullptr) {
        class_loader = caller->GetDeclaringClass()->GetClassLoader();
    } else {
        // Native 线程：只能搜索 BootClassLoader
        // 这意味着 App 自定义的类找不到！
        class_loader = nullptr;  // null = BootClassLoader
    }

    // 3. 通过 ClassLinker 查找类
    ObjPtr<mirror::Class> c = class_linker->FindClass(soa.Self(), descriptor,
                                                        class_loader);
    if (c == nullptr) {
        // 类找不到 → 设置 ClassNotFoundException 到 pending exception
        return nullptr;
    }

    return soa.AddLocalReference<jclass>(c);
}
```

**核心陷阱：Native 线程的 FindClass 只能找到 BootClassLoader 中的类。**

当你从一个纯 Native 线程（`pthread_create` 创建，通过 `AttachCurrentThread` 附着到 ART）调用 `FindClass("com/example/MyClass")`，ART 遍历调用栈找不到任何 Java 帧，于是使用 `BootClassLoader`——它只包含核心类（`java.*`、`android.*`），App 自定义的类不在其中，`FindClass` 返回 `nullptr`。

**正确做法：**

```cpp
// 在 JNI_OnLoad（有正确的 ClassLoader 上下文）时缓存需要的 jclass
static jclass g_myclass = nullptr;

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6);

    jclass localRef = env->FindClass("com/example/MyClass");
    // 转为 Global Reference，跨越 JNI_OnLoad 的 Local Frame 生命周期
    g_myclass = reinterpret_cast<jclass>(env->NewGlobalRef(localRef));
    env->DeleteLocalRef(localRef);

    return JNI_VERSION_1_6;
}

// 在 Native 线程中使用缓存的 Global Reference
void native_thread_work(JNIEnv* env) {
    // 安全：g_myclass 是 Global Reference，不受 ClassLoader 上下文影响
    jmethodID mid = env->GetMethodID(g_myclass, "process", "()V");
    // ...
}
```

### 3.3 GetMethodID：缓存的必要性

```cpp
// art/runtime/jni/jni_internal.cc (简化)
static jmethodID GetMethodID(JNIEnv* env, jclass clazz, const char* name,
                              const char* sig) {
    ScopedObjectAccess soa(env);
    // 1. 解码 jclass → mirror::Class
    ObjPtr<mirror::Class> klass = soa.Decode<mirror::Class>(clazz);

    // 2. 确保类已初始化（可能触发 <clinit>）
    if (!Runtime::Current()->GetClassLinker()->EnsureInitialized(
            soa.Self(), klass, true, true)) {
        return nullptr;
    }

    // 3. 在类的方法列表及继承链中查找匹配的方法
    //    这涉及字符串比较（方法名 + 签名），开销不小
    ArtMethod* method = klass->FindClassMethod(name, sig, image_pointer_size);

    if (method == nullptr || method->IsStatic()) {
        // 方法找不到或类型不匹配
        ThrowNoSuchMethodError(...);
        return nullptr;
    }

    return jni::EncodeArtMethod(method);
}
```

**为什么必须缓存 jmethodID：**

1. `GetMethodID` 内部涉及**字符串匹配**（遍历方法名和签名），虽然有 DexCache 加速，但首次查找仍有可观开销。
2. `EnsureInitialized` 可能触发 `<clinit>` 执行——如果类尚未初始化，这个"查方法"的操作可能隐含地执行了类的静态初始化代码，耗时不可预测。
3. `jmethodID` 在 ART 中是稳定的（它本质上是 `ArtMethod*` 的编码），只要类不被卸载，ID 就不会失效。

**高频调用场景下的性能差异：**

```cpp
// 不缓存：每帧调用 GetMethodID → 每帧都做字符串匹配
void onDrawFrame(JNIEnv* env, jobject renderer) {
    jclass clazz = env->GetObjectClass(renderer);
    jmethodID mid = env->GetMethodID(clazz, "render", "(I)V");  // 每帧查找
    env->CallVoidMethod(renderer, mid, frameCount);
    env->DeleteLocalRef(clazz);
}

// 缓存：初始化时查找一次，后续直接使用
static jmethodID g_render_mid = nullptr;
void initRenderer(JNIEnv* env, jobject renderer) {
    jclass clazz = env->GetObjectClass(renderer);
    g_render_mid = env->GetMethodID(clazz, "render", "(I)V");
    env->DeleteLocalRef(clazz);
}
void onDrawFrame(JNIEnv* env, jobject renderer) {
    env->CallVoidMethod(renderer, g_render_mid, frameCount);  // 零查找开销
}
```

### 3.4 CallVoidMethod：异常检查

```cpp
// art/runtime/jni/jni_internal.cc (概念性)
static void CallVoidMethod(JNIEnv* env, jobject obj, jmethodID mid, ...) {
    ScopedObjectAccess soa(env);
    // 1. 解码参数
    ObjPtr<mirror::Object> receiver = soa.Decode<mirror::Object>(obj);
    ArtMethod* method = jni::DecodeArtMethod(mid);

    // 2. 调用方法（可能抛出 Java 异常）
    InvokeVirtualOrInterfaceWithVarArgs(soa, receiver, method, args);

    // 注意：如果 Java 方法内部抛出了异常，
    // 异常不会自动传播到 Native 层！
    // 它被存储在 Thread::pending_exception_ 中。
    // 如果你不检查，后续的 JNI 调用会因为 pending exception 而失败。
}
```

**关键规则：每次 JNI 调用后必须检查异常。**

```cpp
void callJavaMethod(JNIEnv* env, jobject obj) {
    env->CallVoidMethod(obj, g_process_mid);

    // 必须检查！
    if (env->ExceptionCheck()) {
        // Java 方法抛出了异常
        env->ExceptionDescribe();  // 打印到 logcat
        env->ExceptionClear();     // 清除异常，恢复正常状态
        return;
    }

    // 如果不检查异常就继续调用 JNI 函数：
    // ART 检测到 pending exception 未处理 → JniAbort
    // 日志：JNI ERROR: JNI call made with exception pending
    env->CallVoidMethod(obj, g_another_mid);  // 这里会 abort!
}
```

这是 Native 开发中**最常被忽略的错误模式**。Java 层的异常不会自动变成 C++ 的 `try-catch`，也不会导致 Native 函数提前返回。如果你不主动检查和处理，异常就像一颗"沉默的地雷"——当下一次 JNI 调用触发 CheckJNI 的前置检查时，进程直接 abort。

### 3.5 RegisterNatives：Native 方法注册

`RegisterNatives` 允许在运行时将 Java 的 `native` 方法绑定到 C/C++ 函数，替代默认的 JNI 命名规则查找（`Java_包名_类名_方法名`）。

```cpp
// art/runtime/jni/jni_internal.cc (简化)
static jint RegisterNatives(JNIEnv* env, jclass clazz,
                             const JNINativeMethod* methods, jint nMethods) {
    ScopedObjectAccess soa(env);
    ObjPtr<mirror::Class> klass = soa.Decode<mirror::Class>(clazz);

    for (jint i = 0; i < nMethods; ++i) {
        const char* name = methods[i].name;
        const char* sig = methods[i].signature;
        void* fnPtr = methods[i].fnPtr;

        // 在类中查找匹配的 native 方法
        ArtMethod* method = klass->FindClassMethod(name, sig, image_pointer_size);

        if (method == nullptr || !method->IsNative()) {
            // 方法找不到或不是 native → 注册失败
            ThrowNoSuchMethodError(...);
            return JNI_ERR;
        }

        // 将 Native 函数指针设置为方法的入口点
        method->SetEntryPointFromJni(fnPtr);
    }

    return JNI_OK;
}
```

`RegisterNatives` 的优势是无需遵循冗长的 JNI 命名规则，且可以在运行时动态切换 Native 实现（用于 Native Hook）。

**稳定性风险：**

- **签名不匹配**：如果 `JNINativeMethod` 中的签名（如 `"(Ljava/lang/String;)V"`）与 Java 声明不一致，`RegisterNatives` 会失败并抛出 `NoSuchMethodError`。但如果签名"看起来一样"但参数类型宽度不同（如 `jint` vs `jlong`），注册可能成功但调用时参数解析错误，导致不可预测的行为。
- **so 未加载时调用 native 方法**：如果 Java 层在 `System.loadLibrary` 完成之前就调用了 native 方法，ART 找不到绑定的 Native 函数，会抛出 `UnsatisfiedLinkError`。

---

## 4. CheckJNI 机制

### 4.1 为什么需要 CheckJNI

JNI 是一套基于 C 语言的接口，没有编译期类型检查、没有自动内存管理、没有空安全。一个拼写错误的方法签名、一个被提前释放的引用、一个在 Critical 区调用的分配函数——这些 Bug 在 release 版本中可能只是偶尔的 `SIGSEGV`（难以复现），但在 CheckJNI 开启时会被立即检测并报错。

**CheckJNI 是 ART 内置的 JNI 调用合法性检查器**——它通过函数表替换（将 `JNINativeInterface` 的函数指针从"直接实现"替换为"检查 + 直接实现"的包装版本），在每个 JNI 调用前后插入大量的前置和后置检查。

### 4.2 CheckJNI 的检查项

*   **源码路径：** `art/runtime/jni/check_jni.cc`

```cpp
// art/runtime/jni/check_jni.cc (概念性)
// 以 CallVoidMethod 为例，CheckJNI 包装版本：
static void CheckCallVoidMethod(JNIEnv* env, jobject obj, jmethodID mid, ...) {
    ScopedObjectAccess soa(env);

    // ① 检查 JNIEnv 归属线程
    CheckThread(env);

    // ② 检查 obj 不为 null 且是有效的引用
    CheckInstance(soa, obj);

    // ③ 检查 mid 不为 null 且对应的方法确实属于 obj 的类
    CheckMethodID(soa, mid, obj);

    // ④ 检查没有未处理的 pending exception
    CheckExceptionNotPending(soa);

    // ⑤ 检查不在 Critical 区中（GetPrimitiveArrayCritical 后不应调用其他 JNI 函数）
    CheckNonCritical(env);

    // 通过所有检查 → 调用实际实现
    baseEnv(env)->CallVoidMethodV(env, obj, mid, args);
}
```

CheckJNI 的完整检查项列表：

| 检查项 | 检测的问题 | 常见错误 |
| :--- | :--- | :--- |
| **线程归属** | JNIEnv 是否属于当前线程 | 跨线程传递 JNIEnv |
| **引用有效性** | jobject 是否为有效引用（未被释放、未越界） | 使用已 `DeleteLocalRef` 的引用 |
| **引用类型匹配** | 需要 Global Ref 的地方是否传了 Local Ref | `SetStaticObjectField` 传了 Local Ref（函数返回后失效） |
| **异常状态** | 是否有未处理的 pending exception | CallMethod 后未检查异常 |
| **参数合法性** | 方法签名与实际参数是否匹配 | `CallIntMethod` 调用了返回 void 的方法 |
| **Critical 区限制** | 是否在 GetPrimitiveArrayCritical 后调用了分配类 JNI 函数 | Critical 区内调用 NewObject / FindClass |
| **UTF-8 合法性** | 传入的字符串是否是合法的 Modified UTF-8 | 包含 `\0` 中间字节的字符串 |
| **数组边界** | 数组访问是否越界 | `GetIntArrayRegion` 的 start+len 超过数组长度 |
| **返回值检查** | 返回给 Java 层的值是否与方法签名匹配 | native 方法声明返回 String 但实际返回了 null 之外的非 String 引用 |

### 4.3 开启方式

CheckJNI 默认在 **debug 版本的 App** 中开启（`debuggable=true`）。在 release 版本中关闭以避免性能开销。

手动开启方式：

```bash
# 全局开启（影响所有进程，需要重启）
adb shell setprop debug.checkjni 1

# 针对特定 App 开启
adb shell setprop debug.checkjni.com.example.myapp 1

# 通过 dalvik.vm.checkjni 属性（需要 root）
adb shell setprop dalvik.vm.checkjni true
```

### 4.4 稳定性：Debug 阶段发现隐蔽 JNI Bug 的利器

**最佳实践：在 CI/CD 流程中，所有涉及 JNI 的自动化测试必须开启 CheckJNI。** 许多 JNI Bug 在正常运行时不会立即表现为 Crash（引用恰好还没被回收、GC 恰好没有移动对象），但在高负载或特定 GC 时序下会突然暴露。CheckJNI 能在 Bug 发生的第一时间捕获，并提供精确的错误描述，大幅降低排查成本。

典型的 CheckJNI 错误日志示例：

```
JNI DETECTED ERROR IN APPLICATION: use of deleted local reference 0x75
  from void com.example.NativeLib.process()
  Call to GetObjectClass
    in call to GetObjectClass
```

这条日志直接告诉你：在 `NativeLib.process()` 方法中，调用 `GetObjectClass` 时传入了一个已经被删除的 Local Reference（0x75）。有了这个信息，定位到 Native 源码中 `DeleteLocalRef` 后仍使用该引用的位置就很容易了。

---

## 5. 线程状态切换

### 5.1 为什么 JNI 涉及线程状态切换

这是 JNI 性能开销的核心来源，也是 `GetPrimitiveArrayCritical` 阻塞 GC 导致 ANR 的根因。

在 ART 中，每个 Java 线程在任意时刻处于以下状态之一：

| 状态 | 含义 | GC 行为 |
| :--- | :--- | :--- |
| `kRunnable` | 正在执行 Java 代码或 JNI 代码中操作 Java 对象 | GC 不能移动对象，需要线程到达 SafePoint 才能暂停 |
| `kNative` | 正在执行不涉及 Java 对象的 Native 代码 | GC 可以并发执行，无需等待该线程 |
| `kBlocked` | 等待 Monitor 锁 | GC 可以并发执行 |
| `kWaiting` | 调用了 `Object.wait()` | GC 可以并发执行 |
| `kSuspended` | 被 GC 挂起 | GC 正在进行 |

**关键规则：只有当所有 `kRunnable` 线程都到达 SafePoint 时，GC 才能进入 STW（Stop-The-World）阶段。** 如果有一个线程长时间处于 `kRunnable` 状态不到达 SafePoint，GC 就会一直等待，其他需要分配内存的线程全部阻塞——最终导致 ANR。

### 5.2 JNI 调用中的状态切换

*   **源码路径：** `art/runtime/thread.h`

每次 Java 调用 Native 方法时，线程状态经历如下切换：

```
Java 代码执行中 (kRunnable)
    │
    ▼ 进入 JNI 方法
    ├── 保存 Java 栈帧信息
    ├── 切换到 kNative 状态  ← ScopedThreadSuspension
    │     此时 GC 可以并发执行，无需等待该线程
    │
    ▼ 执行 Native 代码
    │   （纯 C++ 代码，不操作 Java 对象）
    │
    ▼ 需要操作 Java 对象（调用 JNI 函数如 CallMethod/GetField）
    ├── 切换回 kRunnable 状态  ← ScopedObjectAccess
    │     需要检查 SafePoint：如果 GC 正在等待，先暂停自己
    │
    ▼ JNI 函数完成
    ├── 切换回 kNative 状态
    │
    ▼ Native 方法返回
    ├── 切换回 kRunnable 状态
    │     再次检查 SafePoint
    └── 继续执行 Java 代码
```

```cpp
// art/runtime/thread.h (概念性)
class Thread {
    // 线程状态
    volatile ThreadState state_;

    // 切换到 kNative（允许 GC 并发）
    void TransitionFromRunnableToNative() {
        // 1. 设置状态为 kNative
        state_ = kNative;
        // 2. 释放"mutator lock"——GC 通过这个锁来暂停所有 Runnable 线程
        Locks::mutator_lock_->SharedUnlock(this);
    }

    // 从 kNative 切换回 kRunnable（操作 Java 对象前必须）
    void TransitionFromNativeToRunnable() {
        // 1. 获取 mutator lock 的共享锁
        //    如果 GC 正在进行（持有独占锁），这里会阻塞等待
        Locks::mutator_lock_->SharedLock(this);
        // 2. 设置状态为 kRunnable
        state_ = kRunnable;
        // 3. 检查 SafePoint 标志
        CheckSuspend();
    }
};
```

### 5.3 状态切换的性能开销

每次 `kRunnable` ↔ `kNative` 切换涉及：
1. **mutator_lock 的获取/释放**：这是一个读写锁操作（共享锁），在无竞争时约 20-50ns，在 GC 竞争时可能阻塞毫秒级。
2. **SafePoint 检查**：检查 `suspend_count_` 标志，在无挂起请求时仅需一次内存读取。
3. **栈帧记录**：需要更新线程的 `managed_stack_` 指针。

单次切换约 50-100ns，看似微不足道。但在高频 JNI 调用场景下（如渲染管线每帧调用数百次 JNI），累计开销可达 **10-50μs/帧**，占 16ms 帧预算的 0.6%-3%。

**优化策略：** 批量化 JNI 调用。将多次细粒度的 `GetIntField`/`SetIntField` 合并为一次 `GetIntArrayRegion`/`SetIntArrayRegion`，减少状态切换次数。

### 5.4 JNI Critical 区与 GC 阻塞

`GetPrimitiveArrayCritical` / `ReleasePrimitiveArrayCritical` 是 JNI 提供的一种高性能数组访问方式。它返回 Java 数组的**直接内存指针**（而非副本），避免了数据拷贝。但代价是：**在 Critical 区期间，GC 不能移动任何对象**（因为 Native 代码持有的是直接指针，移动会导致指针失效）。

*   **源码路径：** `art/runtime/jni/jni_env_ext.h`

```cpp
// JNIEnvExt 中的 Critical 区计数器
uint32_t critical_;

void* GetPrimitiveArrayCritical(JNIEnv* env, jarray array, jboolean* isCopy) {
    JNIEnvExt* ext = reinterpret_cast<JNIEnvExt*>(env);
    ext->critical_++;  // 进入 Critical 区
    // 返回数组数据的直接指针（无拷贝）
    return array->GetRawData();
}

void ReleasePrimitiveArrayCritical(JNIEnv* env, jarray array, void* carray, jint mode) {
    JNIEnvExt* ext = reinterpret_cast<JNIEnvExt*>(env);
    ext->critical_--;  // 退出 Critical 区
}
```

当任何线程的 `critical_` 大于 0 时，CC GC 的并发复制阶段（Concurrent Copying Phase）必须等待该线程退出 Critical 区。如果 Native 代码在 Critical 区内执行了耗时操作（如写文件、网络请求），GC 会被阻塞，其他所有需要分配内存的线程也随之阻塞。

### 5.5 稳定性：高频 JNI 调用的性能开销与 Critical 区 GC 阻塞

**问题一：高频 JNI 调用的性能开销**

音频/视频处理类 App 常见此问题。AudioTrack 的回调中每帧（约 5-20ms 间隔）需要从 Java 获取配置参数并写入音频数据，涉及大量 JNI 调用。每次调用的 `kNative` ↔ `kRunnable` 切换累计导致音频处理线程的延迟增加，表现为音频卡顿。

治理方案：
- 将配置参数一次性批量传入 Native 层缓存，减少逐字段的 JNI 调用。
- 使用 `GetDirectBufferAddress` 获取 `DirectByteBuffer` 的直接指针，在 Native 侧直接操作内存，避免反复跨越 JNI 边界。

**问题二：`GetPrimitiveArrayCritical` 阻塞 GC → ANR**

```
典型的因果链：
Thread-A: GetPrimitiveArrayCritical(largeArray)
             ↓ 在 Critical 区内执行文件写入（200ms）
             ↓ 此时 GC 被触发（内存不足）
GC Thread: 需要 STW，等待 Thread-A 退出 Critical 区
             ↓ 等待 200ms
Thread-B (Main Thread): 需要 new Object()
             ↓ 等待 GC 完成才能分配内存
             ↓ 等待 200ms+
             ↓ 累计等待超过 5 秒 → ANR!
```

治理方案：
- **绝不在 Critical 区内执行 I/O 操作**。先将数据拷贝到 Native 缓冲区（`GetByteArrayRegion`），退出 Critical 区后再写文件。
- 如果必须使用 Critical 区访问大数组，确保操作耗时在微秒级（仅内存拷贝）。
- 使用 `GetByteArrayRegion` / `SetByteArrayRegion` 替代 Critical 操作——虽然涉及一次数据拷贝，但不会阻塞 GC。

---

## 稳定性实战案例

### 案例 1：Global Reference 泄漏导致渐进性 OOM —— 图片 SDK 的"慢性毒药"

**现象：** 某社交 App 在用户连续浏览图片 30-60 分钟后崩溃，日志为 `java.lang.OutOfMemoryError: Failed to allocate a xxx-byte allocation`。但通过 MAT 分析 Java 堆 Dump，发现堆占用只有 180MB（上限 256MB），理论上还有 76MB 的空间，且没有明显的大对象泄漏。

**分析过程：**

1. 注意到 `dumpsys meminfo` 中 `JNI Global References` 的数量异常：
   ```
   App Summary:
       JNI Global References: 48721 (正常值应在 500-5000)
   ```
   每浏览一张图片增加约 50 个 Global Reference，30 分钟浏览了约 1000 张，累积 5 万个。

2. 每个 Global Reference 指向的 Java 对象及其引用链都不会被 GC 回收。追踪发现这些 Global Reference 指向的是 `android.graphics.Bitmap` 的 backing `byte[]`——一个 500KB 的 Bitmap 对应一个 Global Reference，5 万个意味着 **约 25GB 的不可回收的虚拟对象引用**（实际 Bitmap 像素数据在 Native 堆，但 Java 侧的 `Bitmap` 对象本身也约 100 字节 × 48721 ≈ 5MB 不可回收）。

3. 定位到问题代码：图片 SDK 在 Native 层通过 `NewGlobalRef` 持有 `Bitmap` 对象的引用用于异步渲染，但在渲染完成的回调中忘记调用 `DeleteGlobalRef`。SDK 使用的是 Native 线程池执行渲染，回调通过 `postToMainThread` 执行，当主线程繁忙时回调被延迟，而 Native 线程已经继续处理下一张图片并创建新的 Global Reference。

4. 虽然单个 `Bitmap` Java 对象只有约 100 字节，但 `Bitmap.mNativePtr` 指向的 Native 内存（通过 `NativeAllocationRegistry` 注册）无法被 GC 回收。随着 Global Reference 的累积，越来越多的 Native 内存无法释放，最终 `malloc` 失败导致 OOM。

**根因：** 图片 SDK 的 Native 层在异步渲染完成后未释放 `NewGlobalRef` 创建的 Bitmap Global Reference。

**修复方案：**

1. 在渲染完成的回调中严格保证 `DeleteGlobalRef` 的执行——即使回调被延迟或异常中断，也通过 RAII 封装确保释放：
   ```cpp
   class ScopedGlobalRef {
       JNIEnv* env_;
       jobject ref_;
   public:
       ScopedGlobalRef(JNIEnv* env, jobject ref) : env_(env), ref_(ref) {}
       ~ScopedGlobalRef() { if (ref_) env_->DeleteGlobalRef(ref_); }
       jobject get() { return ref_; }
   };
   ```

2. 添加 Global Reference 数量的运行时监控，当超过 5000 时触发报警并输出引用分布的快照日志。

3. 在 CI 自动化测试中，添加图片浏览场景的压力测试用例（连续浏览 500 张图片），运行结束后断言 Global Reference 增量不超过 100。

**教训：** Global Reference 泄漏是一种"慢性毒药"——不会立即 Crash，而是随时间缓慢累积，最终在某个不可预测的时刻引爆 OOM。常规的 Java 堆分析工具很难发现它，必须结合 `dumpsys meminfo` 的 JNI Global References 数量和 Native 内存分析来排查。

---

### 案例 2：GetPrimitiveArrayCritical 阻塞 GC 导致主线程 ANR

**现象：** 某音视频编辑 App 在导出视频时偶发 ANR，概率约 2%。ANR 类型为 `Input dispatching timed out`，主线程堆栈：

```
at android.graphics.Bitmap.createBitmap (Bitmap.java:1042)
at com.example.editor.ThumbnailView.onDraw (ThumbnailView.java:87)
```

主线程卡在 `Bitmap.createBitmap`——这是一个正常的内存分配操作，耗时应该在微秒级，为何会导致 ANR？

**分析过程：**

1. 检查 `traces.txt` 中 GC 相关线程的状态：
   ```
   "HeapTaskDaemon" daemon waiting for:
       thread 27 (native_encoder) to exit JNI critical region
   ```
   GC 正在等待 `native_encoder` 线程退出 JNI Critical 区。

2. 查看 `native_encoder` 线程的堆栈：
   ```
   #0  write(fd=42, buf=0x7a2c000, count=4194304) at libc.so
   #1  com_example_encoder_NativeEncoder_writeFrame at native_encoder.cc:234
   ```
   该线程在 Critical 区内执行了 4MB 的文件写入操作。

3. 还原完整因果链：
   - `native_encoder` 线程调用 `GetPrimitiveArrayCritical` 获取视频帧数据数组的直接指针（避免拷贝以提升性能）。
   - 在 Critical 区内，将帧数据直接写入文件（`write` 系统调用）。在 HDD/eMMC 设备上，4MB 的写入可能需要 **50-200ms**。
   - 在此期间，GC 被触发（其他线程的内存分配请求达到阈值），GC 需要 STW 但 `native_encoder` 处于 Critical 区，GC 只能等待。
   - 主线程的 `Bitmap.createBitmap` 需要分配内存，但分配器发现堆空间不足，需要等 GC 完成后回收空间。
   - GC 等 `native_encoder` → `native_encoder` 在写文件 → 主线程等 GC → 累计等待超过 5 秒 → ANR。

4. 该 Bug 偶发的原因：只有当 `native_encoder` 的文件写入恰好与 GC 触发和主线程内存分配同时发生时才会 ANR。在高端设备（SSD/UFS 存储）上写入极快，时序窗口极小；但在中低端设备（eMMC 存储）上，写入延迟放大了时序窗口，概率升高。

**根因：** 视频编码器在 `GetPrimitiveArrayCritical` 的 Critical 区内执行了耗时的磁盘 I/O 操作，阻塞了 GC，间接导致主线程 ANR。

**修复方案：**

1. 将帧数据拷贝与文件写入分离：
   ```cpp
   // 修复前（Critical 区内写文件）
   void* data = env->GetPrimitiveArrayCritical(frameArray, nullptr);
   write(fd, data, dataSize);  // 危险！Critical 区内 I/O
   env->ReleasePrimitiveArrayCritical(frameArray, data, 0);

   // 修复后（先拷贝到 Native 缓冲区，再写文件）
   void* data = env->GetPrimitiveArrayCritical(frameArray, nullptr);
   memcpy(nativeBuffer, data, dataSize);  // 仅内存拷贝，微秒级
   env->ReleasePrimitiveArrayCritical(frameArray, data, 0);
   write(fd, nativeBuffer, dataSize);  // Critical 区外写文件，不影响 GC
   ```

2. 对于不需要极致性能的场景，直接使用 `GetByteArrayRegion` 替代 `GetPrimitiveArrayCritical`——虽然多一次拷贝，但彻底消除 GC 阻塞风险。

3. 添加 Critical 区耗时监控：在 `GetPrimitiveArrayCritical` 和 `ReleasePrimitiveArrayCritical` 的封装层中记录耗时，超过 1ms 输出警告日志。

**教训：** `GetPrimitiveArrayCritical` 是一把双刃剑——它消除了数据拷贝的开销，但将 GC 的命运交到了 Native 代码手中。一旦在 Critical 区内执行了任何可能阻塞的操作（I/O、网络、锁等待），就等于给整个进程的内存分配系统安装了一颗不定时炸弹。**在 Code Review 中，任何 `GetPrimitiveArrayCritical` 的使用都应该被标记为 HIGH RISK 并逐行审查 Critical 区内的代码。**

---

## 6. 总结

本篇从"JavaVM 与 JNIEnv 的生命周期 → 引用管理的三种模型 → 关键 JNI 函数的源码语义 → CheckJNI 检查机制 → 线程状态切换与 GC 交互"五个维度，系统剖析了 JNI 的核心机制和稳定性风险。

**核心要点：**

1. **JNIEnv 是线程私有的**，跨线程传递会导致 abort。正确做法是缓存 `JavaVM` + `AttachCurrentThread`。
2. **Local Reference 有 512 上限**，循环中必须手动释放或使用 `PushLocalFrame`/`PopLocalFrame`。**Global Reference 无上限但需手动释放**，泄漏会导致 Java 对象和关联的 Native 内存都无法回收。
3. **Native 线程的 `FindClass` 只能找到 BootClassLoader 的类**——App 自定义类必须在有正确 ClassLoader 上下文时（如 `JNI_OnLoad`）缓存为 Global Reference。
4. **每次 JNI 调用后必须检查 `ExceptionCheck()`**——Java 异常不会自动传播到 Native 层，忽略 pending exception 会导致后续 JNI 调用 abort。
5. **CheckJNI 是发现隐蔽 JNI Bug 的利器**，CI/CD 流程中的 JNI 相关测试必须开启。
6. **`kRunnable` ↔ `kNative` 状态切换有 50-100ns 的开销**，高频 JNI 调用应批量化。**`GetPrimitiveArrayCritical` 在 Critical 区内阻塞 GC**，严禁在 Critical 区执行 I/O 或其他耗时操作。

下一篇，我们将跨越 ART 与 JVM 的边界，通过**横向对比**，理解 ART 为什么做了这些设计选择——从指令集、GC 算法、编译策略到类加载模型，全面剖析两者的设计哲学分野。
