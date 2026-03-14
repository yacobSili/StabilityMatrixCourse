# Android 进程内存模型

## 学习目标

- 理解 Android 进程的内存布局
- 掌握 Android 内存指标（PSS、RSS、USS）
- 了解 /proc 接口查看进程内存
- 理解 Java 堆和 Native 堆的关系

## 一、Android 进程内存布局

### 1.1 整体布局

```
Android 进程地址空间布局：

高地址
┌─────────────────────────────────────┐
│           内核空间                   │ ← 用户不可访问
├─────────────────────────────────────┤
│           栈 (Stack)                │ ← 主线程栈
│              ↓                      │
├─────────────────────────────────────┤
│           线程栈                    │ ← 其他线程栈
├─────────────────────────────────────┤
│                                     │
│           mmap 区域                  │
│  ┌─────────────────────────────┐   │
│  │ 共享库 (libc, libart, ...)  │   │
│  ├─────────────────────────────┤   │
│  │ ART 堆 (Region Space)       │   │ ← Java 对象
│  ├─────────────────────────────┤   │
│  │ Binder mmap                 │   │ ← IPC 缓冲区
│  ├─────────────────────────────┤   │
│  │ ashmem 映射                 │   │ ← 共享内存
│  ├─────────────────────────────┤   │
│  │ 文件映射 (APK, DEX, ...)    │   │
│  └─────────────────────────────┘   │
│              ↑                      │
├─────────────────────────────────────┤
│           Native 堆                 │ ← malloc 分配
│              ↑                      │
├─────────────────────────────────────┤
│           数据段                    │
├─────────────────────────────────────┤
│           代码段                    │ ← 可执行文件
├─────────────────────────────────────┤
│           保留区域                   │
低地址
└─────────────────────────────────────┘
```

### 1.2 内存区域类型

| 区域 | 说明 | 特点 |
|-----|------|-----|
| Java 堆 | ART 管理的对象内存 | GC 自动管理 |
| Native 堆 | malloc/free 分配 | 手动管理 |
| 代码映射 | APK/DEX/SO 文件 | 只读，共享 |
| 栈 | 函数调用栈 | 自动管理 |
| Binder | IPC 缓冲区 | 系统管理 |
| ashmem | 进程间共享内存 | 应用管理 |
| 图形缓冲 | GPU 内存 | 系统管理 |

---

## 二、内存指标

### 2.1 常用内存指标

```
VSS (Virtual Set Size)
  - 虚拟内存总量
  - 包括未映射的地址空间
  - 意义不大

RSS (Resident Set Size)
  - 驻留内存大小
  - 实际占用的物理内存
  - 包含共享库的全部大小（不准确）

PSS (Proportional Set Size)
  - 比例共享内存
  - 共享内存按比例计算
  - 最常用的指标

USS (Unique Set Size)
  - 独占内存
  - 不包含任何共享部分
  - 进程退出后释放的内存

关系：USS <= PSS <= RSS <= VSS
```

### 2.2 PSS 计算示例

```
假设：
- 进程 A 和 B 共享 libc.so (4MB)
- 进程 A 独占 2MB 匿名内存
- 进程 B 独占 3MB 匿名内存

进程 A:
  - USS = 2MB（独占部分）
  - PSS = 2MB + 4MB/2 = 4MB
  - RSS = 2MB + 4MB = 6MB

进程 B:
  - USS = 3MB
  - PSS = 3MB + 4MB/2 = 5MB
  - RSS = 3MB + 4MB = 7MB

系统总内存 = A.USS + B.USS + 共享
           = 2MB + 3MB + 4MB = 9MB
           = A.PSS + B.PSS
           = 4MB + 5MB = 9MB ✓
```

### 2.3 Android 特有指标

```java
// android.os.Debug.MemoryInfo

public static class MemoryInfo {
    // Native 堆
    public int dalvikPss;           // Dalvik/ART 堆 PSS
    public int dalvikPrivateDirty;  // Dalvik/ART 私有脏页
    public int dalvikSharedDirty;   // Dalvik/ART 共享脏页
    
    // Native
    public int nativePss;
    public int nativePrivateDirty;
    public int nativeSharedDirty;
    
    // 其他
    public int otherPss;
    public int otherPrivateDirty;
    public int otherSharedDirty;
    
    // 总计
    public int getTotalPss();
    public int getTotalPrivateDirty();
    public int getTotalSharedDirty();
}
```

---

## 三、/proc 内存接口

### 3.1 /proc/pid/maps

```bash
# 查看进程内存映射
$ cat /proc/<pid>/maps

# 格式：地址范围 权限 偏移 设备 inode 路径
12c00000-12e00000 rw-p 00000000 00:00 0          [anon:dalvik-main space]
70000000-70c00000 rw-p 00000000 00:00 0          [anon:dalvik-region space]
70c00000-74c00000 ---p 00000000 00:00 0          [anon:dalvik-region space]
7f1234000000-7f1234100000 rw-p 00000000 00:00 0  [stack:12345]
7f1234500000-7f1234600000 r--p 00000000 fd:01 123456 /system/lib64/libc.so
7f1234600000-7f1234700000 r-xp 00100000 fd:01 123456 /system/lib64/libc.so
7f1234700000-7f1234800000 r--p 00200000 fd:01 123456 /system/lib64/libc.so
7f1234800000-7f1234900000 rw-p 00300000 fd:01 123456 /system/lib64/libc.so
```

### 3.2 /proc/pid/smaps

```bash
# 详细内存信息
$ cat /proc/<pid>/smaps

12c00000-12e00000 rw-p 00000000 00:00 0          [anon:dalvik-main space]
Size:               2048 kB     # VMA 大小
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                1536 kB     # 驻留内存
Pss:                1536 kB     # 比例共享内存
Shared_Clean:          0 kB     # 共享干净页
Shared_Dirty:          0 kB     # 共享脏页
Private_Clean:         0 kB     # 私有干净页
Private_Dirty:      1536 kB     # 私有脏页
Referenced:         1536 kB     # 被引用页
Anonymous:          1536 kB     # 匿名页
...
```

### 3.3 /proc/pid/status

```bash
$ cat /proc/<pid>/status | grep -E "Vm|Rss|Threads"
VmPeak:    1234567 kB    # 虚拟内存峰值
VmSize:    1234567 kB    # 当前虚拟内存
VmLck:          0 kB     # 锁定内存
VmPin:          0 kB     # 固定内存
VmHWM:     123456 kB     # RSS 峰值
VmRSS:     123456 kB     # 当前 RSS
RssAnon:   100000 kB     # 匿名 RSS
RssFile:    23456 kB     # 文件 RSS
RssShmem:       0 kB     # 共享内存 RSS
VmData:    500000 kB     # 数据段大小
VmStk:        136 kB     # 栈大小
VmExe:         16 kB     # 代码段大小
VmLib:      50000 kB     # 共享库大小
VmPTE:       1000 kB     # 页表大小
VmSwap:         0 kB     # Swap 使用
Threads:       45        # 线程数
```

### 3.4 dumpsys meminfo

```bash
# Android 系统内存信息
$ adb shell dumpsys meminfo <package_name>

Applications Memory Usage (in Kilobytes):
Uptime: 12345678 Realtime: 12345678

** MEMINFO in pid 12345 [com.example.app] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap    12345    12000      345        0    32768    30000     2768
  Dalvik Heap     8765     8000      765        0    16384    12000     4384
 Dalvik Other     1234     1000      234        0
        Stack       64       64        0        0
       Ashmem        0        0        0        0
    Other dev        4        0        4        0
     .so mmap     5678      200     4000        0
    .jar mmap     1234        0     1234        0
    .apk mmap      567        0      567        0
    .dex mmap     2345        0     2345        0
    .oat mmap      123        0      123        0
    .art mmap     3456     2000      456        0
   Other mmap      234       34      200        0
   EGL mtrack     4567     4567        0        0
    GL mtrack     8901     8901        0        0
      Unknown     1234     1234        0        0
        TOTAL    51101    38000     9273        0    49152    42000     7152

 App Summary
                       Pss(KB)
                        ------
           Java Heap:    10456
         Native Heap:    12000
                Code:     8269
               Stack:       64
            Graphics:    13468
       Private Other:     2268
              System:     4576
 
               TOTAL:    51101       TOTAL SWAP PSS:        0
```

---

## 四、Java 堆内存

### 4.1 ART 堆结构

```
ART 堆内存布局（Android 10+）：

┌─────────────────────────────────────────────────────────────┐
│                     ART Heap                                 │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Region Space (主要区域)                    │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │   │
│  │  │Region│ │Region│ │Region│ │Region│ │Region│ ...   │   │
│  │  │ 256KB│ │ 256KB│ │ 256KB│ │ 256KB│ │ 256KB│      │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐         │
│  │   Large Object      │  │   Image Space       │         │
│  │   Space             │  │   (boot.art)        │         │
│  └─────────────────────┘  └─────────────────────┘         │
│                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐         │
│  │   Non-Moving        │  │   Zygote Space      │         │
│  │   Space             │  │   (共享)            │         │
│  └─────────────────────┘  └─────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 堆大小限制

```java
// 获取堆限制
Runtime runtime = Runtime.getRuntime();
long maxMemory = runtime.maxMemory();        // 最大堆大小
long totalMemory = runtime.totalMemory();    // 当前堆大小
long freeMemory = runtime.freeMemory();      // 当前空闲

// 系统属性
// dalvik.vm.heapsize        - 最大堆大小（如 512m）
// dalvik.vm.heapgrowthlimit - 单应用堆限制（如 256m）
// dalvik.vm.heapstartsize   - 初始堆大小（如 8m）
// dalvik.vm.heaptargetutilization - 目标利用率（如 0.75）

// 大堆应用（android:largeHeap="true"）
// 可以使用 heapsize 而不是 heapgrowthlimit
```

### 4.3 GC 类型

```
ART GC 类型：

1. Concurrent Mark-Sweep (CMS)
   - 并发执行，停顿时间短
   - 用于大多数 GC 场景

2. Concurrent Copying (CC)
   - Android 10+ 默认
   - 使用 Region Space
   - 复制存活对象，压缩堆

3. Semi-Space
   - 用于 Zygote 进程
   - 紧凑堆布局

GC 触发条件：
- 分配失败
- 堆使用达到阈值
- 显式调用 System.gc()
- Native 分配触发
```

---

## 五、Native 堆内存

### 5.1 Native 内存分配

```c
// 标准分配
void *ptr = malloc(size);
free(ptr);

// Android 特有
// Scudo 分配器（Android 11+）
// 安全性增强的分配器

// jemalloc（旧版本）
// 高性能分配器
```

### 5.2 Native 内存监控

```bash
# 使用 adb shell
$ adb shell dumpsys meminfo --checkin <pid> | grep native

# 使用 malloc_debug
$ adb shell setprop libc.debug.malloc.options backtrace
$ adb shell kill -9 <pid>
# 重启应用后查看 /data/local/tmp/<pid>.txt

# 使用 heapprofd（Android 10+）
$ adb shell perfetto --out /data/misc/perfetto-traces/trace \
    --txt --config - <<EOF
data_sources {
  config {
    name: "android.heapprofd"
    heapprofd_config {
      process_cmdline: "com.example.app"
      sampling_interval_bytes: 4096
    }
  }
}
EOF
```

### 5.3 Bitmap 内存

```java
// Android 8.0+ Bitmap 存储在 Native 堆
// 之前版本存储在 Java 堆

Bitmap bitmap = Bitmap.createBitmap(1920, 1080, Bitmap.Config.ARGB_8888);
// 内存 = 1920 * 1080 * 4 = 8.3MB（Native 堆）

// 查看 Bitmap 内存
int allocationByteCount = bitmap.getAllocationByteCount();

// 回收
bitmap.recycle();
bitmap = null;
```

---

## 六、内存优化建议

### 6.1 监控和分析

```java
// 1. 使用 ActivityManager 获取内存信息
ActivityManager am = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
ActivityManager.MemoryInfo memInfo = new ActivityManager.MemoryInfo();
am.getMemoryInfo(memInfo);

// 2. 使用 Debug 类
Debug.MemoryInfo[] memInfos = am.getProcessMemoryInfo(new int[]{pid});
int totalPss = memInfos[0].getTotalPss();

// 3. 注册内存回调
ComponentCallbacks2 callback = new ComponentCallbacks2() {
    @Override
    public void onTrimMemory(int level) {
        switch (level) {
            case TRIM_MEMORY_RUNNING_MODERATE:
            case TRIM_MEMORY_RUNNING_LOW:
            case TRIM_MEMORY_RUNNING_CRITICAL:
                // 前台运行时内存不足
                break;
            case TRIM_MEMORY_UI_HIDDEN:
                // UI 隐藏，释放 UI 资源
                break;
            case TRIM_MEMORY_BACKGROUND:
            case TRIM_MEMORY_MODERATE:
            case TRIM_MEMORY_COMPLETE:
                // 后台，可能被杀
                break;
        }
    }
};
registerComponentCallbacks(callback);
```

### 6.2 最佳实践

```
1. 避免内存泄漏
   - 注意 Activity 引用
   - 及时取消注册监听器
   - 使用 LeakCanary 检测

2. 优化 Bitmap
   - 按需加载合适尺寸
   - 使用 inBitmap 复用
   - 及时 recycle()

3. 使用缓存
   - LruCache 限制大小
   - 软引用/弱引用

4. Native 内存
   - 避免 JNI 内存泄漏
   - 使用 Perfetto/heapprofd 分析
```

---

## 总结

### 核心概念

1. **内存指标**：PSS 是最有意义的指标
2. **Java 堆**：ART 管理，有大小限制
3. **Native 堆**：手动管理，无硬限制
4. **内存映射**：共享库和文件

### 关键工具

| 工具 | 用途 |
|-----|------|
| /proc/pid/maps | 查看内存映射 |
| /proc/pid/smaps | 详细内存信息 |
| dumpsys meminfo | Android 内存报告 |
| Perfetto/heapprofd | Native 内存分析 |
| Android Studio Profiler | 图形化分析 |

### 后续学习

- [LMKD与OOM机制详解](17-LMKD与OOM机制详解.md) - 了解内存不足处理
- [ART虚拟机内存管理](18-ART虚拟机内存管理.md) - 深入 ART 内存

## 参考资源

- Android 文档：[管理应用内存](https://developer.android.com/topic/performance/memory)
- 内核文档：`Documentation/filesystems/proc.rst`

## 更新记录

- 2026-01-28：初始创建，包含 Android 进程内存模型详解
