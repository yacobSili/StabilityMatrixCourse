# Android 进程优先级与 LMK

## 学习目标

- 理解 Android 进程优先级体系
- 掌握 OOM_ADJ 机制
- 理解 ProcessList 与进程状态管理
- 了解 Low Memory Killer（LMK）机制
- 理解 Cgroups 在 Android 中的应用

## 概述

Android 使用多层次的进程优先级管理机制：
- OOM_ADJ：影响内存不足时的杀进程顺序
- Process State：影响进程的调度优先级
- Cgroups：控制 CPU、内存等资源分配

---

## 一、Android 进程优先级体系

### 进程优先级分层

```
优先级（从高到低）：

┌─────────────────────────────────────────────────────────────┐
│  SYSTEM_ADJ = -900                                          │
│  系统进程（system_server）                                  │
├─────────────────────────────────────────────────────────────┤
│  PERSISTENT_PROC_ADJ = -800                                 │
│  常驻进程（电话、短信等）                                   │
├─────────────────────────────────────────────────────────────┤
│  FOREGROUND_APP_ADJ = 0                                     │
│  前台应用（用户正在交互）                                   │
├─────────────────────────────────────────────────────────────┤
│  VISIBLE_APP_ADJ = 100                                      │
│  可见应用（部分可见但不在前台）                             │
├─────────────────────────────────────────────────────────────┤
│  PERCEPTIBLE_APP_ADJ = 200                                  │
│  可感知应用（播放音乐、下载等）                             │
├─────────────────────────────────────────────────────────────┤
│  SERVICE_ADJ = 500                                          │
│  服务进程                                                   │
├─────────────────────────────────────────────────────────────┤
│  HOME_APP_ADJ = 600                                         │
│  Launcher                                                   │
├─────────────────────────────────────────────────────────────┤
│  PREVIOUS_APP_ADJ = 700                                     │
│  上一个应用                                                 │
├─────────────────────────────────────────────────────────────┤
│  CACHED_APP_MIN_ADJ = 900                                   │
│  缓存进程（可随时被杀死）                                   │
│  CACHED_APP_MAX_ADJ = 999                                   │
└─────────────────────────────────────────────────────────────┘
```

### ProcessList 常量定义

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java
public final class ProcessList {
    // 进程 ADJ 值
    public static final int INVALID_ADJ = -10000;
    public static final int UNKNOWN_ADJ = 1001;
    public static final int CACHED_APP_MAX_ADJ = 999;
    public static final int CACHED_APP_MIN_ADJ = 900;
    public static final int CACHED_APP_LMK_FIRST_ADJ = 950;
    public static final int SERVICE_B_ADJ = 800;
    public static final int PREVIOUS_APP_ADJ = 700;
    public static final int HOME_APP_ADJ = 600;
    public static final int SERVICE_ADJ = 500;
    public static final int HEAVY_WEIGHT_APP_ADJ = 400;
    public static final int BACKUP_APP_ADJ = 300;
    public static final int PERCEPTIBLE_LOW_APP_ADJ = 250;
    public static final int PERCEPTIBLE_APP_ADJ = 200;
    public static final int VISIBLE_APP_ADJ = 100;
    public static final int PERCEPTIBLE_RECENT_FOREGROUND_APP_ADJ = 50;
    public static final int FOREGROUND_APP_ADJ = 0;
    public static final int PERSISTENT_SERVICE_ADJ = -700;
    public static final int PERSISTENT_PROC_ADJ = -800;
    public static final int SYSTEM_ADJ = -900;
    public static final int NATIVE_ADJ = -1000;
}
```

---

## 二、OOM_ADJ 机制

### OOM_ADJ 概念

OOM_ADJ 值写入 `/proc/<pid>/oom_score_adj`，内核根据此值决定内存不足时杀死哪个进程。

```bash
# 查看进程的 OOM_ADJ
cat /proc/<pid>/oom_score_adj

# 查看进程的 OOM 分数（综合考虑 ADJ 和内存使用）
cat /proc/<pid>/oom_score
```

### 设置 OOM_ADJ

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java
public static void setOomAdj(int pid, int uid, int amt) {
    // 写入 /proc/pid/oom_score_adj
    writeProcFile("/proc/" + pid + "/oom_score_adj", Integer.toString(amt));
}

// 通过 lmkd 设置
private boolean updateOomAdjLSP(ProcessRecord app, boolean isOomAdjProcessing) {
    // ...
    if (app.curAdj != app.setAdj) {
        ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
        app.setAdj = app.curAdj;
    }
    // ...
}
```

### OomAdjuster

```java
// frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
final class OomAdjuster {
    
    // 更新进程 ADJ
    boolean updateOomAdjLocked(ProcessRecord app, int cachedAdj,
            ProcessRecord topApp, boolean doingAll, long now) {
        
        // 1. 计算进程优先级
        int adj = computeOomAdjLocked(app, cachedAdj, topApp, doingAll, now);
        
        // 2. 应用优先级
        if (adj != app.curAdj) {
            applyOomAdjLocked(app, adj);
        }
        
        return true;
    }
    
    // 计算 OOM ADJ
    private int computeOomAdjLocked(ProcessRecord app, int cachedAdj,
            ProcessRecord topApp, boolean doingAll, long now) {
        
        // 前台活动
        if (app == topApp) {
            return FOREGROUND_APP_ADJ;
        }
        
        // 可见活动
        if (app.hasVisibleActivities) {
            return VISIBLE_APP_ADJ;
        }
        
        // 前台服务
        if (app.hasForegroundServices()) {
            return PERCEPTIBLE_APP_ADJ;
        }
        
        // 后台服务
        if (app.hasService()) {
            return SERVICE_ADJ;
        }
        
        // 缓存进程
        return cachedAdj;
    }
}
```

---

## 三、Process State

### 进程状态定义

```java
// frameworks/base/core/java/android/app/ActivityManager.java
public static final int PROCESS_STATE_UNKNOWN = -1;
public static final int PROCESS_STATE_PERSISTENT = 0;
public static final int PROCESS_STATE_PERSISTENT_UI = 1;
public static final int PROCESS_STATE_TOP = 2;
public static final int PROCESS_STATE_FOREGROUND_SERVICE = 3;
public static final int PROCESS_STATE_BOUND_TOP = 4;
public static final int PROCESS_STATE_FOREGROUND = 5;
public static final int PROCESS_STATE_BOUND_FOREGROUND_SERVICE = 6;
public static final int PROCESS_STATE_IMPORTANT_FOREGROUND = 7;
public static final int PROCESS_STATE_IMPORTANT_BACKGROUND = 8;
public static final int PROCESS_STATE_TRANSIENT_BACKGROUND = 9;
public static final int PROCESS_STATE_BACKUP = 10;
public static final int PROCESS_STATE_SERVICE = 11;
public static final int PROCESS_STATE_RECEIVER = 12;
public static final int PROCESS_STATE_TOP_SLEEPING = 13;
public static final int PROCESS_STATE_HEAVY_WEIGHT = 14;
public static final int PROCESS_STATE_HOME = 15;
public static final int PROCESS_STATE_LAST_ACTIVITY = 16;
public static final int PROCESS_STATE_CACHED_ACTIVITY = 17;
public static final int PROCESS_STATE_CACHED_ACTIVITY_CLIENT = 18;
public static final int PROCESS_STATE_CACHED_RECENT = 19;
public static final int PROCESS_STATE_CACHED_EMPTY = 20;
public static final int PROCESS_STATE_NONEXISTENT = 21;
```

### 状态与 ADJ 映射

| Process State | OOM ADJ | 说明 |
|--------------|---------|-----|
| TOP | 0 | 前台 |
| FOREGROUND_SERVICE | 200 | 前台服务 |
| FOREGROUND | 0 | 前台 |
| SERVICE | 500 | 服务 |
| HOME | 600 | Launcher |
| CACHED_ACTIVITY | 900-999 | 缓存 |

---

## 四、Low Memory Killer

### LMK 概述

Low Memory Killer（LMK）是 Android 特有的内存管理机制，在内存不足时杀死低优先级进程。

### LMK 发展历史

| 版本 | 机制 | 说明 |
|-----|------|-----|
| Android 4.x 以前 | 内核 LMK 驱动 | 在内核中实现 |
| Android 4.x - 8.x | lowmemorykiller.c | 内核模块 |
| Android 9.0+ | lmkd | 用户空间守护进程 |

### LMKD 守护进程

```cpp
// system/memory/lmkd/lmkd.cpp
int main(int argc, char **argv) {
    // 1. 初始化
    init();
    
    // 2. 创建 socket 接收 AMS 的请求
    lmkd_init_socket();
    
    // 3. 注册内存压力事件
    register_psi_monitor();
    
    // 4. 主循环
    mainloop();
    
    return 0;
}

static void mp_event_psi(int data, uint32_t events) {
    // 内存压力事件处理
    
    // 1. 获取内存信息
    struct meminfo mi;
    meminfo_get(&mi);
    
    // 2. 计算压力级别
    int level = get_pressure_level(&mi);
    
    // 3. 如果需要，杀死进程
    if (level >= LEVEL_MEDIUM) {
        kill_process(level);
    }
}

static int kill_process(int level) {
    // 1. 获取进程列表
    // 2. 按 ADJ 排序
    // 3. 杀死最高 ADJ 的进程
    
    for (int adj = CACHED_APP_MAX_ADJ; adj >= 0; adj--) {
        for (auto& proc : processes) {
            if (proc.adj == adj) {
                kill(proc.pid, SIGKILL);
                return 0;
            }
        }
    }
    
    return -1;
}
```

### LMK 阈值配置

```bash
# 查看 LMK 阈值
cat /sys/module/lowmemorykiller/parameters/minfree
cat /sys/module/lowmemorykiller/parameters/adj

# 设置示例（页面数）
# 对应 ADJ: 0, 100, 200, 300, 900, 906
echo "18432,23040,27648,32256,55296,80640" > /sys/module/lowmemorykiller/parameters/minfree
```

### PSI（Pressure Stall Information）

Android 10+ 使用 PSI 替代 vmpressure：

```bash
# 查看内存压力
cat /proc/pressure/memory

# 输出示例
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

---

## 五、进程保活机制分析

### 常见保活方式

| 方式 | 原理 | 效果 |
|-----|------|-----|
| 前台服务 | 提高 ADJ | 有效 |
| 双进程守护 | 互相拉起 | 被限制 |
| 账号同步 | 系统回调 | 已失效 |
| JobScheduler | 系统调度 | 受限 |
| WorkManager | 兼容方案 | 推荐 |

### 前台服务保活

```java
// 启动前台服务
public class MyService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 创建通知
        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("Service Running")
                .setSmallIcon(R.drawable.ic_service)
                .build();
        
        // 提升为前台服务
        startForeground(1, notification);
        
        return START_STICKY;
    }
}
```

### Android 后台限制

```java
// Android 8.0+ 后台限制
// 不能在后台启动服务

// 正确方式：使用 JobScheduler 或 WorkManager
WorkManager.getInstance(context)
    .enqueue(OneTimeWorkRequest.Builder(MyWorker.class).build());
```

---

## 六、Cgroups 应用

### Android Cgroups 配置

```bash
# Android cgroups 挂载点
/dev/cpuctl/        # CPU 控制
/dev/cpuset/        # CPU 亲和性
/dev/memcg/         # 内存控制
/dev/stune/         # 调度调优
```

### Cpuset 配置

```bash
# 前台进程可以使用所有 CPU
echo 0-7 > /dev/cpuset/foreground/cpus

# 后台进程限制在小核
echo 0-3 > /dev/cpuset/background/cpus

# 系统后台
echo 0-3 > /dev/cpuset/system-background/cpus

# 将进程移入 cpuset
echo <pid> > /dev/cpuset/foreground/tasks
```

### 进程与 Cgroup

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java
public void setProcessGroup(int pid, int group) {
    // 将进程移入对应的 cgroup
    
    switch (group) {
        case SCHED_GROUP_BACKGROUND:
            Process.setProcessGroup(pid, Process.THREAD_GROUP_BG_NONINTERACTIVE);
            break;
        case SCHED_GROUP_TOP_APP:
            Process.setProcessGroup(pid, Process.THREAD_GROUP_TOP_APP);
            break;
        default:
            Process.setProcessGroup(pid, Process.THREAD_GROUP_DEFAULT);
    }
}
```

---

## 七、调试与监控

### 查看进程信息

```bash
# 查看所有进程的 ADJ
dumpsys activity oom

# 查看特定进程
dumpsys activity p <package>

# 查看进程状态
dumpsys activity processes
```

### 内存信息

```bash
# 内存概览
dumpsys meminfo

# 特定进程内存
dumpsys meminfo <package>

# 系统内存
cat /proc/meminfo
```

### LMK 日志

```bash
# 查看 LMK 杀进程日志
dmesg | grep lowmemorykiller

# lmkd 日志
logcat -s lmkd
```

---

## 总结

### 核心要点

1. **OOM_ADJ**：
   - 决定杀进程顺序
   - 写入 `/proc/pid/oom_score_adj`
   - AMS 动态调整

2. **LMK/LMKD**：
   - 内存不足时杀进程
   - 按 ADJ 从高到低杀
   - Android 9+ 使用 lmkd

3. **Cgroups**：
   - cpuset 控制 CPU 亲和性
   - memcg 控制内存
   - 前台/后台差异化调度

4. **进程保活**：
   - 前台服务是正规方式
   - Android 限制越来越严格
   - 推荐使用 WorkManager

### 后续学习

- [Android Binder与进程调试](18-Android%20Binder与进程调试.md) - Binder 和调试

## 参考资源

- Android 源码：
  - `frameworks/base/services/core/java/com/android/server/am/ProcessList.java`
  - `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`
  - `system/memory/lmkd/lmkd.cpp`

## 更新记录

- 2026-01-27：初始创建，包含 Android 进程优先级与 LMK
