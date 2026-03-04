# Android IO 优先级机制与设置方法

## 学习目标

- 理解 IO 优先级的概念和作用
- 掌握 IO 优先级的分类和级别
- 学会使用系统调用和工具设置 IO 优先级
- 理解 cgroup v2 IO 优先级策略
- 掌握 Android 系统中 IO 优先级的应用场景

## 背景介绍

在 Android 系统中，多个进程和线程会同时进行 IO 操作。当系统 IO 压力大时，如何保证关键进程（如 SystemServer、前台应用）的 IO 请求能够优先处理，避免被后台任务阻塞，是提升系统响应性的关键。IO 优先级机制允许我们为不同的进程设置不同的 IO 优先级，确保重要任务的 IO 请求能够优先得到处理。

## IO 优先级的概念

### 为什么需要 IO 优先级

**问题场景**：
- 后台应用进行大量文件写入，占用 IO 资源
- 前台应用的用户交互需要读取文件，但被阻塞
- 系统服务（如 SystemServer）的关键 IO 操作被延迟

**解决方案**：
- 为前台应用和系统服务设置更高的 IO 优先级
- 为后台任务设置较低的 IO 优先级
- 确保关键 IO 请求优先处理

### IO 优先级的作用机制

IO 优先级影响 IO 调度器的行为：
- **高优先级**：请求在调度器队列中优先被处理
- **低优先级**：请求在调度器队列中延后处理
- **IDLE 优先级**：只有在没有其他 IO 请求时才处理

## IO 优先级分类

### 优先级类别（Class）

IO 优先级由两部分组成：**类别（Class）** 和 **级别（Level）**。

**类别定义**（`include/uapi/linux/ioprio.h`）：
```c
enum {
    IOPRIO_CLASS_NONE = 0,    // 无优先级（默认）
    IOPRIO_CLASS_RT = 1,      // 实时优先级（Real-Time）
    IOPRIO_CLASS_BE = 2,      // 尽力而为（Best Effort，默认）
    IOPRIO_CLASS_IDLE = 3,    // 空闲优先级（最低）
};
```

**优先级编码**：
```c
#define IOPRIO_CLASS_SHIFT  13
#define IOPRIO_PRIO_VALUE(class, data) \
    (((class) & IOPRIO_CLASS_MASK) << IOPRIO_CLASS_SHIFT) | \
    ((data) & IOPRIO_PRIO_MASK)
```

### 各优先级类别详解

#### 1. IOPRIO_CLASS_RT（实时优先级）

**特点**：
- 最高优先级，优先于所有其他类别的请求
- 需要 `CAP_SYS_ADMIN` 或 `CAP_SYS_NICE` 权限
- 有 8 个级别（0-7），0 最高，7 最低
- 使用不当可能导致其他进程 IO 饥饿

**使用场景**：
- 系统关键服务（如 SystemServer）
- 实时性要求极高的应用

**注意事项**：
- 谨慎使用，避免导致系统其他部分 IO 饥饿
- 通常只给少量关键进程使用

#### 2. IOPRIO_CLASS_BE（尽力而为）

**特点**：
- 默认优先级类别
- 有 8 个级别（0-7），0 最高，7 最低
- 级别映射：`io_nice = (cpu_nice + 20) / 5`
- 适合大多数应用场景

**使用场景**：
- 前台应用：使用较高级别（0-3）
- 后台应用：使用较低级别（4-7）

#### 3. IOPRIO_CLASS_IDLE（空闲优先级）

**特点**：
- 最低优先级
- 只有在没有其他 IO 请求时才处理
- 没有级别区分

**使用场景**：
- 后台维护任务
- 低优先级的数据同步

## 进程 IO 优先级设置方法

### 1. 使用系统调用

**系统调用接口**（`block/ioprio.c`）：
```c
SYSCALL_DEFINE3(ioprio_set, int, which, int, who, int, ioprio)
```

**参数说明**：
- `which`：设置范围
  - `IOPRIO_WHO_PROCESS`：单个进程
  - `IOPRIO_WHO_PGRP`：进程组
  - `IOPRIO_WHO_USER`：用户的所有进程
- `who`：进程 ID、进程组 ID 或用户 ID（0 表示当前）
- `ioprio`：优先级值（类别 + 级别）

**示例代码**：
```c
#include <sys/syscall.h>
#include <linux/ioprio.h>

// ARM64 系统调用号（通用架构为 30 和 31）
#define __NR_ioprio_set 30
#define __NR_ioprio_get 31

// 设置当前进程为 BE 类别，级别 0（最高 BE 优先级）
int ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE, 0);
syscall(__NR_ioprio_set, IOPRIO_WHO_PROCESS, 0, ioprio);

// 设置进程 ID 为 1234 的进程为 RT 类别，级别 4
ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_RT, 4);
syscall(__NR_ioprio_set, IOPRIO_WHO_PROCESS, 1234, ioprio);
```

**注意**：
- 不同架构的系统调用号可能不同，ARM64 使用通用系统调用号（30 和 31）
- 也可以使用 glibc 的包装函数（如果可用）：`ioprio_set()` 和 `ioprio_get()`

### 2. 使用 ionice 工具

**基本用法**：
```bash
# 查看当前进程的 IO 优先级
ionice -p $$

# 设置当前进程为 BE 类别，级别 0
ionice -c 2 -n 0 -p $$

# 设置进程 ID 为 1234 的进程为 RT 类别，级别 4
ionice -c 1 -n 4 -p 1234

# 设置进程组的所有进程为 IDLE 优先级
ionice -c 3 -p -1234

# 启动新进程并设置 IO 优先级
ionice -c 2 -n 0 /path/to/program
```

**参数说明**：
- `-c`：类别（1=RT, 2=BE, 3=IDLE）
- `-n`：级别（0-7，仅对 RT 和 BE 有效）
- `-p`：进程 ID（负数表示进程组 ID）

### 3. Android Framework 中的设置

**重要说明**：Android Framework 中**没有**直接的 Java API 来设置 IO 优先级（如 `Process.setIOPriority()`）。如果需要从 Java 层设置 IO 优先级，需要自己实现 JNI 函数。

**如果需要实现，可以参考以下示例**：

**JNI 实现示例**（需要添加到 Android Framework 源码中）：
```cpp
// 文件：frameworks/base/core/jni/android_os_Process.cpp
#include <sys/syscall.h>
#include <linux/ioprio.h>

// ARM64 系统调用号
#define __NR_ioprio_set 30
#define __NR_ioprio_get 31

// JNI 函数实现（示例，实际不存在于 Android Framework 中）
static jint android_os_Process_setIOPriority(JNIEnv* env, jobject clazz, jint pid, jint ioprio)
{
    if (pid <= 0) {
        pid = getpid();  // 如果 pid <= 0，使用当前进程
    }
    return syscall(__NR_ioprio_set, IOPRIO_WHO_PROCESS, pid, ioprio);
}

// 需要在 JNI 注册表中注册此函数
// static const JNINativeMethod gMethods[] = {
//     { "setIOPriority", "(II)I", (void*)android_os_Process_setIOPriority },
// };
```

**Java 层接口**（如果实现了 JNI）：
```java
// 文件：frameworks/base/core/java/android/os/Process.java
// 注意：这是示例代码，Android Framework 中实际不存在此方法
public static final int setIOPriority(int pid, int ioprio) {
    return nativeSetIOPriority(pid, ioprio);
}

private static native int nativeSetIOPriority(int pid, int ioprio);
```

**实际应用**：
- **Android Framework 现状**：Android Framework 中**没有**提供设置 IO 优先级的 Java API
- **替代方案**：
  1. 使用 `ionice` 工具（通过 `Runtime.exec()` 或 init 脚本）
  2. 使用 cgroup IO 优先级策略（通过 sysfs）
  3. 在 Native 层（C/C++）直接调用系统调用
  4. 通过 init 脚本在系统启动时设置
- **系统级管理**：Android 系统可能通过 cgroup 机制管理 IO 优先级，而不是通过 Framework API

## cgroup v2 IO 优先级策略

### cgroup IO 优先级控制器

cgroup v2 提供了更灵活的 IO 优先级控制机制，可以基于 cgroup 设置 IO 优先级策略。

**位置**：`block/blk-ioprio.c`

**策略类型**（`enum prio_policy`）：
```c
enum prio_policy {
    POLICY_NO_CHANGE = 0,        // 不修改 IO 优先级类别
    POLICY_PROMOTE_TO_RT = 1,     // 提升到 RT 类别
    POLICY_RESTRICT_TO_BE = 2,    // 限制为 BE 类别
    POLICY_ALL_TO_IDLE = 3,       // 全部改为 IDLE 类别
    POLICY_NONE_TO_RT = 4,        // 无优先级改为 RT（已废弃）
};
```

### 设置方法

**通过 sysfs 设置**（cgroup v2）：
```bash
# 查看当前策略
cat /sys/fs/cgroup/io.prio.class

# 设置为 promote-to-rt（提升到 RT）
echo "promote-to-rt" > /sys/fs/cgroup/io.prio.class

# 设置为 restrict-to-be（限制为 BE）
echo "restrict-to-be" > /sys/fs/cgroup/io.prio.class

# 设置为 idle（全部改为 IDLE）
echo "idle" > /sys/fs/cgroup/io.prio.class

# 设置为 no-change（不修改）
echo "no-change" > /sys/fs/cgroup/io.prio.class
```

**注意**：
- cgroup v2 中，IO 优先级属性文件名为 `io.prio.class`（根据内核实现，属性名为 `prio.class`，在 IO 控制器下显示为 `io.prio.class`）
- 对于子 cgroup，路径为 `/sys/fs/cgroup/<cgroup_name>/io.prio.class`
- Android 设备上可能使用 cgroup v1，路径为 `/sys/fs/cgroup/blkio/<cgroup_name>/blkio.prio.class`
- 实际路径可能因 Android 版本和配置而异，建议先检查 `/sys/fs/cgroup` 目录结构

### 策略详解

#### 1. promote-to-rt

**行为**：
- 将非 RT 类别的请求提升为 RT 类别
- 同时将优先级级别设置为 4
- 已经是 RT 类别的请求不受影响

**使用场景**：
- 为特定 cgroup 的所有进程提供实时 IO 优先级
- 确保关键服务的 IO 请求优先处理

#### 2. restrict-to-be

**行为**：
- 将无优先级（NONE）和 RT 类别的请求改为 BE 类别
- 同时将优先级级别设置为 0
- IDLE 类别的请求不受影响

**使用场景**：
- 限制某些进程组的 IO 优先级
- 防止低优先级进程获得过高优先级

#### 3. idle

**行为**：
- 将所有请求的 IO 优先级类别改为 IDLE
- 最低优先级，只有在没有其他 IO 时才处理

**使用场景**：
- 后台维护任务
- 低优先级的数据同步

## Android 系统中的 IO 优先级应用

### 关键进程的 IO 优先级

**SystemServer**：
- 系统核心服务，可能需要较高的 IO 优先级
- 可以通过 cgroup 或系统调用设置

**Zygote**：
- 应用进程的父进程
- 可能需要较高的 IO 优先级以加快应用启动

**前台应用**：
- 用户正在交互的应用
- 应该获得比后台应用更高的 IO 优先级

**后台应用**：
- 不在前台的应用
- 可以使用较低的 IO 优先级

### 实战案例：为关键进程设置 IO 优先级

**场景**：SystemServer 进程的 IO 操作被后台应用阻塞，导致系统响应变慢。

**解决方案**：

1. **使用 ionice 设置**：
```bash
# 查找 SystemServer 的进程 ID
pid=$(pgrep -f "system_server")

# 设置为 RT 类别，级别 4
ionice -c 1 -n 4 -p $pid
```

2. **使用 cgroup 设置**：
```bash
# 创建 SystemServer 的 cgroup（cgroup v2）
mkdir -p /sys/fs/cgroup/system_server

# 将 SystemServer 进程加入 cgroup
echo $pid > /sys/fs/cgroup/system_server/cgroup.procs

# 设置 IO 优先级策略为 promote-to-rt
echo "promote-to-rt" > /sys/fs/cgroup/system_server/io.prio.class

# 或者使用 cgroup v1（如果系统使用 v1）
mkdir -p /sys/fs/cgroup/blkio/system_server
echo $pid > /sys/fs/cgroup/blkio/system_server/tasks
echo "promote-to-rt" > /sys/fs/cgroup/blkio/system_server/blkio.prio.class
```

3. **在 Android 源码中设置**（需要修改 Framework 和添加 JNI）：
```java
// 注意：Android Framework 中没有 Process.setIOPriority() 方法
// 如果需要使用，需要先实现 JNI 函数（参考上面的示例）

// IO 优先级常量定义
private static final int IOPRIO_CLASS_RT = 1;
private static final int IOPRIO_CLASS_SHIFT = 13;

// 方法 1：如果实现了 JNI 函数
// RT 类别，级别 4：ioprio = (1 << 13) | 4 = 8196
int ioprio = (IOPRIO_CLASS_RT << IOPRIO_CLASS_SHIFT) | 4;
Process.setIOPriority(Process.myPid(), ioprio);

// 方法 2：通过 Runtime.exec() 调用 ionice（更实际的方法）
try {
    Process process = Runtime.getRuntime().exec(
        new String[]{"su", "-c", "ionice -c 1 -n 4 -p " + Process.myPid()});
    process.waitFor();
} catch (Exception e) {
    // 处理异常
}
```

**或者使用 init 脚本**（更简单的方法）：
```bash
# 在 /system/etc/init/ 目录下创建脚本
# 文件：system_server_io_priority.rc
on property:sys.boot_completed=1
    exec_background u:r:system_server:s0 /system/bin/sh -c "ionice -c 1 -n 4 -p \$(pidof system_server)"
```

## 查看 IO 优先级

### 查看进程的 IO 优先级

**使用 ionice**：
```bash
# 查看当前进程
ionice -p $$

# 查看指定进程
ionice -p 1234
```

**使用 /proc 文件系统**：
```bash
# 查看进程的 IO 上下文（需要内核支持）
cat /proc/1234/io
```

**使用系统调用**：
```c
#include <sys/syscall.h>
#include <linux/ioprio.h>

#define __NR_ioprio_get 31

int ioprio = syscall(__NR_ioprio_get, IOPRIO_WHO_PROCESS, 0);
if (ioprio >= 0) {
    int class = IOPRIO_PRIO_CLASS(ioprio);
    int level = IOPRIO_PRIO_DATA(ioprio);
    // 处理优先级值
}
```

### 查看 cgroup 的 IO 优先级策略

```bash
# 查看当前 cgroup 的策略（cgroup v2）
cat /sys/fs/cgroup/io.prio.class

# 查看所有 cgroup 的策略（cgroup v2）
find /sys/fs/cgroup -name "io.prio.class" -exec echo {} \; -exec cat {} \;

# 查看 cgroup v1 的策略
find /sys/fs/cgroup/blkio -name "blkio.prio.class" -exec echo {} \; -exec cat {} \;
```

## 注意事项和最佳实践

### 权限要求

- **RT 类别**：需要 `CAP_SYS_ADMIN` 或 `CAP_SYS_NICE` 权限
- **BE 和 IDLE 类别**：普通用户可以使用，但只能降低自己的优先级

### 最佳实践

1. **谨慎使用 RT 类别**：
   - 只给少量关键进程使用
   - 避免导致其他进程 IO 饥饿

2. **合理设置级别**：
   - 前台应用：BE 类别，级别 0-3
   - 后台应用：BE 类别，级别 4-7
   - 维护任务：IDLE 类别

3. **使用 cgroup 管理**：
   - 对于多个进程，使用 cgroup 统一管理
   - 便于批量设置和监控

4. **监控 IO 性能**：
   - 设置 IO 优先级后，监控系统 IO 性能
   - 确保没有导致其他问题

## 总结

### 核心要点

1. **IO 优先级分类**：
   - RT：实时优先级（最高）
   - BE：尽力而为（默认）
   - IDLE：空闲优先级（最低）

2. **设置方法**：
   - 系统调用：`ioprio_set()`
   - 工具：`ionice`
   - cgroup：`io.prio.class`

3. **Android 应用**：
   - 关键进程（SystemServer）使用较高优先级
   - 前台应用优先于后台应用
   - 使用 cgroup 统一管理

### 关键概念

- **IO 优先级**：控制 IO 请求在调度器中的处理顺序
- **优先级类别**：RT、BE、IDLE
- **优先级级别**：每个类别内的细分级别（0-7）
- **cgroup IO 优先级策略**：基于 cgroup 的批量优先级管理

### 下一步学习

- [07-Android IO 调度算法详解](07-Android IO 调度算法详解.md) - 了解 IO 调度器如何处理不同优先级的请求
- [08-Android 整机 IO 压力分析与占用定位](08-Android 整机 IO 压力分析与占用定位.md) - 学习如何分析 IO 性能问题
- [09-Android 设备常见 IO 问题与排查](09-Android 设备常见 IO 问题与排查.md) - 掌握 IO 问题的排查方法

## 参考资料

- Linux 内核源码：`block/ioprio.c` - IO 优先级系统调用实现
- Linux 内核源码：`block/blk-ioprio.c` - cgroup IO 优先级控制器
- Linux 内核头文件：`include/uapi/linux/ioprio.h` - IO 优先级定义
- Linux 内核文档：`Documentation/block/ioprio.rst` - IO 优先级文档
- cgroup v2 文档：`Documentation/admin-guide/cgroup-v2.rst` - cgroup IO 优先级策略

## 更新记录

- 2026-01-27：初始创建，包含 Android IO 优先级机制与设置方法
