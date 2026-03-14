# ART 虚拟机内存管理

## 学习目标

- 理解 ART 堆的结构和组织
- 掌握 ART 垃圾回收（GC）机制
- 了解 JNI 内存管理
- 理解 Android 应用内存优化策略

## 一、ART 概述

### 1.1 ART vs Dalvik

| 特性 | ART | Dalvik |
|-----|-----|--------|
| 编译方式 | AOT + JIT | JIT |
| 启动时间 | 较快（预编译） | 较慢 |
| GC | 并发复制 | 标记清除 |
| 内存效率 | 更好 | 一般 |
| 引入版本 | Android 5.0 | Android 1.0 |

### 1.2 ART 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Android Application                       │
│                    (Java/Kotlin Code)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      ART Runtime                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │   Class     │ │   Memory    │ │    JIT      │           │
│  │   Loader    │ │   Manager   │ │   Compiler  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │     GC      │ │   Thread    │ │   Monitor   │           │
│  │   System    │ │   Manager   │ │             │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Native Libraries                          │
│         (libc, libm, OpenGL, ...)                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、ART 堆结构

### 2.1 堆空间布局

```
ART 堆内存布局（Android 12+）：

┌─────────────────────────────────────────────────────────────┐
│                        ART Heap                              │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                 Region Space                           │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐         │  │
│  │  │Region 0│ │Region 1│ │Region 2│ │  ...   │         │  │
│  │  │ 256KB  │ │ 256KB  │ │ 256KB  │ │        │         │  │
│  │  │(Thread │ │(Bump   │ │(Large  │ │        │         │  │
│  │  │ Local) │ │Pointer)│ │Object) │ │        │         │  │
│  │  └────────┘ └────────┘ └────────┘ └────────┘         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐          │
│  │    Image Space      │  │    Zygote Space     │          │
│  │  (boot.art/oat)     │  │   (Shared w/fork)   │          │
│  │  Pre-initialized    │  │                     │          │
│  │  objects            │  │                     │          │
│  └─────────────────────┘  └─────────────────────┘          │
│                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐          │
│  │  Large Object       │  │   Non-Moving        │          │
│  │  Space              │  │   Space             │          │
│  │  (> 12KB objects)   │  │   (GC roots, etc)   │          │
│  └─────────────────────┘  └─────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 空间类型

| 空间 | 说明 | GC 行为 |
|-----|------|--------|
| Region Space | 主要对象分配区 | 并发复制 GC |
| Image Space | boot.art 预加载对象 | 不回收 |
| Zygote Space | fork 时继承 | 共享，不回收 |
| Large Object Space | 大对象（>12KB） | 标记清除 |
| Non-Moving Space | 不可移动对象 | 标记清除 |

### 2.3 Region 类型

```c++
// art/runtime/gc/space/region_space.h
enum class RegionType : uint8_t {
    kRegionTypeToSpace,      // GC to-space
    kRegionTypeFromSpace,    // GC from-space
    kRegionTypeUnevacFromSpace, // 不迁移的 from-space
    kRegionTypeFree,         // 空闲
    kRegionTypeNone,         // 未使用
};

// 每个 Region 大小
static constexpr size_t kRegionSize = 256 * KB;
```

---

## 三、对象分配

### 3.1 分配路径

```
对象分配流程：

new Object()
    │
    ▼
┌─────────────────────┐
│ ART Runtime         │
│ AllocObjectWithAllocator()
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 检查 TLAB           │ ← Thread Local Allocation Buffer
│ (线程本地缓冲)      │
└─────────┬───────────┘
          │
    ┌─────┴─────┐
    │           │
 有空间     无空间
    │           │
    ▼           ▼
快速分配    申请新 Region
(无锁)     或触发 GC
```

### 3.2 TLAB (Thread Local Allocation Buffer)

```c++
// art/runtime/thread.h
class Thread {
    // TLAB 指针
    uint8_t* tlsPtr_.thread_local_start;
    uint8_t* tlsPtr_.thread_local_pos;
    uint8_t* tlsPtr_.thread_local_end;
    
    // 快速分配（无锁）
    mirror::Object* AllocFromTLAB(size_t alloc_size) {
        uint8_t* old_pos = tlsPtr_.thread_local_pos;
        uint8_t* new_pos = old_pos + alloc_size;
        
        if (new_pos <= tlsPtr_.thread_local_end) {
            tlsPtr_.thread_local_pos = new_pos;
            return reinterpret_cast<mirror::Object*>(old_pos);
        }
        return nullptr;  // TLAB 空间不足
    }
};
```

### 3.3 大对象分配

```c++
// art/runtime/gc/heap.cc
mirror::Object* Heap::AllocLargeObject(Thread* self, ObjPtr<mirror::Class> klass,
                                       size_t byte_count) {
    // 大对象直接分配到 Large Object Space
    mirror::Object* obj = large_object_space_->Alloc(self, byte_count, ...);
    
    if (obj == nullptr) {
        // 触发 GC
        CollectGarbageInternal(collector::kGcTypeFull, ...);
        obj = large_object_space_->Alloc(self, byte_count, ...);
    }
    
    return obj;
}
```

---

## 四、垃圾回收 (GC)

### 4.1 GC 类型

| 类型 | 说明 | 触发条件 |
|-----|------|---------|
| Concurrent Copying (CC) | 并发复制 GC | 堆使用达阈值 |
| Semi-Space | 半空间复制 | Zygote 进程 |
| Mark-Sweep | 标记清除 | 大对象空间 |
| Mark-Compact | 标记压缩 | 后台压缩 |

### 4.2 Concurrent Copying GC

```
并发复制 GC 流程：

阶段 1: 初始标记（STW）
┌─────────────────────────────────────┐
│ 标记 GC Roots                        │
│ (栈引用、全局引用、JNI 引用等)       │
│ 暂停时间: ~1ms                       │
└─────────────────────────────────────┘
                   │
                   ▼
阶段 2: 并发标记
┌─────────────────────────────────────┐
│ 遍历对象图，标记存活对象             │
│ 应用线程继续运行                     │
│ 使用写屏障跟踪修改                   │
└─────────────────────────────────────┘
                   │
                   ▼
阶段 3: 并发复制
┌─────────────────────────────────────┐
│ 将存活对象从 from-space 复制到      │
│ to-space                            │
│ 应用线程继续运行（读屏障）           │
└─────────────────────────────────────┘
                   │
                   ▼
阶段 4: 最终标记（STW）
┌─────────────────────────────────────┐
│ 处理并发阶段的修改                   │
│ 暂停时间: ~1ms                       │
└─────────────────────────────────────┘
                   │
                   ▼
阶段 5: 清理
┌─────────────────────────────────────┐
│ 回收 from-space                     │
│ 交换 from-space 和 to-space         │
└─────────────────────────────────────┘
```

### 4.3 GC 实现

```c++
// art/runtime/gc/collector/concurrent_copying.cc
void ConcurrentCopying::RunPhases() {
    // 阶段 1: 初始标记
    {
        ScopedPause pause(this);  // STW
        MarkRoots();
    }
    
    // 阶段 2: 并发标记
    {
        MarkingPhase();
    }
    
    // 阶段 3: 并发复制
    {
        CopyingPhase();
    }
    
    // 阶段 4: 最终标记
    {
        ScopedPause pause(this);  // STW
        FinishMarking();
    }
    
    // 阶段 5: 清理
    {
        ReclaimPhase();
    }
}

// 读屏障（访问对象时检查是否需要复制）
mirror::Object* ConcurrentCopying::Mark(mirror::Object* from_ref) {
    if (IsInFromSpace(from_ref)) {
        // 对象在 from-space，需要复制
        mirror::Object* to_ref = Copy(from_ref);
        return to_ref;
    }
    return from_ref;
}
```

### 4.4 GC 触发条件

```c++
// art/runtime/gc/heap.cc
void Heap::AllocateInternal(...) {
    // 检查是否需要 GC
    if (concurrent_gc_pending_) {
        // 已有 GC 挂起
    } else if (bytes_allocated > concurrent_start_bytes_) {
        // 超过阈值，触发并发 GC
        RequestConcurrentGC(self, kGcCauseForAlloc);
    }
}

// GC 原因
enum GcCause {
    kGcCauseForAlloc,        // 分配触发
    kGcCauseBackground,      // 后台 GC
    kGcCauseExplicit,        // 显式调用
    kGcCauseForNativeAlloc,  // Native 分配触发
    kGcCauseCollectorTransition, // 收集器切换
    // ...
};
```

---

## 五、JNI 内存管理

### 5.1 JNI 引用类型

```c++
// Local Reference: 自动释放（方法返回时）
jobject obj = env->NewObject(clazz, methodID);
// 方法返回后自动释放

// Global Reference: 手动管理
jobject globalRef = env->NewGlobalRef(obj);
// 必须手动释放
env->DeleteGlobalRef(globalRef);

// Weak Global Reference: 可被 GC
jweak weakRef = env->NewWeakGlobalRef(obj);
// 使用前检查是否被回收
if (!env->IsSameObject(weakRef, NULL)) {
    // 对象仍然存活
}
env->DeleteWeakGlobalRef(weakRef);
```

### 5.2 JNI 内存泄漏

```c++
// 常见泄漏模式 1: 忘记释放 Global Reference
JNIEXPORT void JNICALL Java_Example_leak(JNIEnv *env, jobject thiz) {
    jobject globalRef = env->NewGlobalRef(someObject);
    // 忘记调用 DeleteGlobalRef
    // 内存泄漏！
}

// 常见泄漏模式 2: 循环中创建 Local Reference
JNIEXPORT void JNICALL Java_Example_loop(JNIEnv *env, jobject thiz) {
    for (int i = 0; i < 10000; i++) {
        jstring str = env->NewStringUTF("test");
        // Local Reference 默认限制 512 个
        // 循环中可能溢出
    }
}

// 正确做法: 使用 PushLocalFrame/PopLocalFrame
JNIEXPORT void JNICALL Java_Example_loopCorrect(JNIEnv *env, jobject thiz) {
    for (int i = 0; i < 10000; i++) {
        env->PushLocalFrame(16);
        jstring str = env->NewStringUTF("test");
        // 使用 str
        env->PopLocalFrame(NULL);  // 自动释放所有 Local Reference
    }
}
```

### 5.3 Native 内存分配

```c++
// 使用 Android 的 Native 内存跟踪
#include <android/trace.h>

void* my_alloc(size_t size) {
    void* ptr = malloc(size);
    // 记录分配
    ATrace_beginSection("my_alloc");
    // ...
    ATrace_endSection();
    return ptr;
}

// 使用 malloc_debug 检测泄漏
// $ adb shell setprop libc.debug.malloc.options backtrace
```

---

## 六、内存优化

### 6.1 堆配置

```bash
# 系统属性（/system/build.prop 或设备属性）
dalvik.vm.heapsize=512m           # 最大堆大小
dalvik.vm.heapgrowthlimit=256m    # 单应用堆限制
dalvik.vm.heapstartsize=8m        # 初始堆大小
dalvik.vm.heaptargetutilization=0.75  # 目标利用率
dalvik.vm.heapminfree=512k        # 最小空闲
dalvik.vm.heapmaxfree=8m          # 最大空闲

# 运行时配置
Runtime.getRuntime().maxMemory()      # 获取最大堆
Runtime.getRuntime().totalMemory()    # 获取当前堆大小
Runtime.getRuntime().freeMemory()     # 获取空闲内存
```

### 6.2 GC 日志分析

```bash
# 启用 GC 日志
$ adb shell setprop dalvik.vm.dex2oat-Xms64m
$ adb logcat -s art:D

# GC 日志示例
I/art: Explicit concurrent copying GC freed 123456(12MB) 
       AllocSpace objects, 12(1MB) LOS objects, 
       40% free, 100MB/167MB, paused 1.2ms total 234ms

# 字段说明:
# - Explicit: GC 原因（Explicit/Alloc/Background）
# - concurrent copying: GC 类型
# - freed: 释放的对象数和大小
# - free: 当前空闲百分比
# - paused: STW 时间
# - total: GC 总时间
```

### 6.3 最佳实践

```java
// 1. 避免频繁创建对象
// Bad
for (int i = 0; i < 1000; i++) {
    String s = new String("test");
}

// Good
String s = "test";
for (int i = 0; i < 1000; i++) {
    // 使用 s
}

// 2. 使用对象池
ObjectPool<MyObject> pool = new ObjectPool<>(10);
MyObject obj = pool.acquire();
// 使用 obj
pool.release(obj);

// 3. 及时释放资源
Bitmap bitmap = BitmapFactory.decodeFile(path);
// 使用 bitmap
bitmap.recycle();

// 4. 使用 SparseArray 代替 HashMap<Integer, Object>
SparseArray<Object> sparseArray = new SparseArray<>();

// 5. 避免内存泄漏
// - 避免非静态内部类持有 Activity 引用
// - 及时取消注册监听器
// - 使用 WeakReference
```

---

## 总结

### ART 堆结构

| 空间 | 用途 | 特点 |
|-----|------|-----|
| Region Space | 常规对象 | 并发复制 GC |
| Large Object Space | 大对象 | 标记清除 |
| Image Space | 系统类 | 共享，不 GC |
| Zygote Space | fork 继承 | 共享 |

### GC 特点

- **并发**：大部分工作与应用并发
- **低暂停**：STW 时间约 1-2ms
- **分代**：不同空间不同策略
- **压缩**：复制 GC 自动压缩

### JNI 注意事项

1. 及时释放 Global Reference
2. 循环中使用 PushLocalFrame/PopLocalFrame
3. 注意 Native 内存泄漏

### 后续学习

- [内存与其他子系统交互](19-内存与其他子系统交互.md) - 了解系统交互
- [内存性能分析与问题排查](20-内存性能分析与问题排查.md) - 内存问题诊断

## 参考资源

- ART 源码：`art/runtime/gc/`
- Android 文档：[管理应用内存](https://developer.android.com/topic/performance/memory)

## 更新记录

- 2026-01-28：初始创建，包含 ART 虚拟机内存管理详解
