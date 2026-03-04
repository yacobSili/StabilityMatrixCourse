# LMKD 与 OOM 机制详解

## 学习目标

- 理解 Android LMKD 的工作原理
- 掌握 oom_adj 和进程优先级机制
- 了解 Kernel OOM Killer 的触发和选择算法
- 理解 PSI 在内存压力监控中的应用

## 一、Android 内存管理架构

### 1.1 整体架构

```
                    Framework 层
┌─────────────────────────────────────────────────────────────┐
│   ActivityManagerService (AMS)                               │
│   - ProcessList: 进程优先级管理                              │
│   - OomAdjuster: oom_adj 计算                               │
│   - ProcessRecord: 进程信息                                  │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ 设置 oom_adj
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      LMKD                                    │
│   - 监控内存压力（PSI）                                      │
│   - 根据 oom_adj 杀死进程                                    │
│   - 与 kernel 协作                                           │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ 杀进程 / 监控 PSI
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Kernel                                   │
│   - PSI (Pressure Stall Information)                        │
│   - OOM Killer（最后手段）                                   │
│   - /proc/pid/oom_score_adj                                 │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 两层防护

| 层 | 组件 | 触发条件 | 特点 |
|---|------|---------|------|
| 用户空间 | LMKD | PSI 内存压力 | 主动、精细控制 |
| 内核空间 | OOM Killer | 内存分配失败 | 被动、最后手段 |

---

## 二、oom_adj 进程优先级

### 2.1 oom_adj 值定义

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

// 系统进程（不会被杀）
static final int NATIVE_ADJ = -1000;
static final int SYSTEM_ADJ = -900;
static final int PERSISTENT_PROC_ADJ = -800;
static final int PERSISTENT_SERVICE_ADJ = -700;

// 前台进程
static final int FOREGROUND_APP_ADJ = 0;
static final int VISIBLE_APP_ADJ = 100;
static final int PERCEPTIBLE_APP_ADJ = 200;
static final int PERCEPTIBLE_LOW_APP_ADJ = 250;

// 后台进程
static final int BACKUP_APP_ADJ = 300;
static final int HEAVY_WEIGHT_APP_ADJ = 400;
static final int SERVICE_ADJ = 500;
static final int HOME_APP_ADJ = 600;
static final int PREVIOUS_APP_ADJ = 700;
static final int SERVICE_B_ADJ = 800;
static final int CACHED_APP_MIN_ADJ = 900;
static final int CACHED_APP_MAX_ADJ = 999;

// 值越小优先级越高，越不容易被杀
```

### 2.2 oom_adj 计算

```java
// OomAdjuster.java
private final boolean computeOomAdjLSP(ProcessRecord app, ...) {
    // 1. 前台 Activity
    if (app.hasActivities()) {
        if (app == mTopApp) {
            adj = ProcessList.FOREGROUND_APP_ADJ;
        } else if (app.hasVisibleActivities()) {
            adj = ProcessList.VISIBLE_APP_ADJ;
        }
    }
    
    // 2. 前台服务
    if (app.hasForegroundServices()) {
        adj = ProcessList.PERCEPTIBLE_APP_ADJ;
    }
    
    // 3. Provider 客户端
    if (app.hasClientProvider()) {
        // 根据客户端调整
    }
    
    // 4. Service 绑定
    if (app.hasServiceConnections()) {
        // 根据绑定进程调整
    }
    
    // 5. 缓存进程
    if (adj >= ProcessList.CACHED_APP_MIN_ADJ) {
        // 根据 LRU 位置细分
    }
    
    // 写入 /proc/pid/oom_score_adj
    ProcessList.setOomAdj(app.pid, app.uid, adj);
}
```

### 2.3 查看进程 oom_adj

```bash
# 查看进程 oom_score_adj
$ cat /proc/<pid>/oom_score_adj
0

# 查看所有进程
$ for pid in /proc/[0-9]*; do
    echo "$(cat $pid/comm 2>/dev/null): $(cat $pid/oom_score_adj 2>/dev/null)"
done | sort -t: -k2 -n

# 使用 dumpsys
$ adb shell dumpsys activity processes | grep -A5 "oom adj"
```

---

## 三、LMKD 实现

### 3.1 LMKD 架构

```
┌─────────────────────────────────────────────────────────────┐
│                          LMKD                                │
│                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │   Socket    │   │ PSI Monitor │   │ Kill Logic  │       │
│  │  Handler    │   │             │   │             │       │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘       │
│         │                 │                 │               │
│         │                 │                 │               │
│    ┌────▼─────────────────▼─────────────────▼────┐         │
│    │              Main Event Loop                 │         │
│    └───────────────────┬─────────────────────────┘         │
│                        │                                    │
└────────────────────────┼────────────────────────────────────┘
                         │
           ┌─────────────┼─────────────┐
           │             │             │
           ▼             ▼             ▼
       AMS 通信       PSI 事件      杀进程
```

### 3.2 PSI 监控

```c
// system/memory/lmkd/lmkd.cpp

// 初始化 PSI 监控
static bool init_psi_monitor() {
    // 打开 PSI 接口
    int fd = open("/proc/pressure/memory", O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        return false;
    }
    
    // 配置触发条件
    // 格式：<stall_type> <threshold_us> <window_us>
    // 例如：some 70000 1000000 表示 1 秒内有 70ms 的 some 压力
    char buf[64];
    snprintf(buf, sizeof(buf), "some %d %d",
             PSI_SOME_THRESHOLD_US, PSI_WINDOW_US);
    
    if (write(fd, buf, strlen(buf)) < 0) {
        close(fd);
        return false;
    }
    
    psi_fd = fd;
    return true;
}

// 监控循环
static void mainloop() {
    struct pollfd fds[MAX_POLL_FDS];
    
    // 添加 PSI fd
    fds[PSI_POLL_IDX].fd = psi_fd;
    fds[PSI_POLL_IDX].events = POLLPRI;
    
    // 添加 socket fd（与 AMS 通信）
    fds[CTRL_POLL_IDX].fd = ctrl_sock;
    fds[CTRL_POLL_IDX].events = POLLIN;
    
    while (true) {
        int nevents = poll(fds, nfds, -1);
        
        // PSI 事件
        if (fds[PSI_POLL_IDX].revents & POLLPRI) {
            handle_psi_event();
        }
        
        // AMS 请求
        if (fds[CTRL_POLL_IDX].revents & POLLIN) {
            handle_ctrl_socket();
        }
    }
}
```

### 3.3 杀进程逻辑

```c
// system/memory/lmkd/lmkd.cpp

static void handle_psi_event() {
    // 读取当前内存压力级别
    enum psi_level level = get_psi_level();
    
    // 根据压力级别决定杀进程策略
    int min_score_adj;
    switch (level) {
        case PSI_LOW:
            min_score_adj = 900;  // 只杀 cached 进程
            break;
        case PSI_MEDIUM:
            min_score_adj = 700;  // 杀 previous 和 cached
            break;
        case PSI_CRITICAL:
            min_score_adj = 200;  // 杀更多进程
            break;
    }
    
    // 查找并杀死进程
    find_and_kill_process(min_score_adj, level);
}

static int find_and_kill_process(int min_score_adj, enum psi_level level) {
    struct proc *procp;
    int killed_pid = -1;
    
    // 遍历进程列表，从高 oom_adj 开始
    for (int adj = OOM_SCORE_ADJ_MAX; adj >= min_score_adj; adj--) {
        procp = proc_adj_lru[adj];
        while (procp) {
            // 选择内存占用最大的进程
            if (procp->tasksize > max_tasksize) {
                max_tasksize = procp->tasksize;
                selected_procp = procp;
            }
            procp = procp->adj_next;
        }
        
        if (selected_procp) {
            // 杀死进程
            kill(selected_procp->pid, SIGKILL);
            killed_pid = selected_procp->pid;
            break;
        }
    }
    
    return killed_pid;
}
```

### 3.4 与 AMS 通信

```c
// LMKD 端
// Socket 命令处理
static void handle_ctrl_socket() {
    struct lmk_packet pkt;
    
    read(ctrl_sock, &pkt, sizeof(pkt));
    
    switch (pkt.cmd) {
        case LMK_TARGET:
            // 设置内存阈值和对应的 oom_adj
            cmd_target(&pkt);
            break;
        case LMK_PROCPRIO:
            // 更新进程优先级
            cmd_procprio(&pkt);
            break;
        case LMK_PROCREMOVE:
            // 进程退出
            cmd_procremove(&pkt);
            break;
    }
}

// AMS 端
// frameworks/base/services/core/jni/com_android_server_am_LmkdConnection.cpp
static jboolean android_server_am_LmkdConnection_setOomAdj(
        JNIEnv* env, jobject clazz, jint pid, jint uid, jint oomAdj) {
    
    struct lmk_procprio pkt;
    pkt.hdr.type = LMK_PROCPRIO;
    pkt.pid = pid;
    pkt.uid = uid;
    pkt.oomadj = oomAdj;
    
    return lmkd_write(&pkt) == 0;
}
```

---

## 四、Kernel OOM Killer

### 4.1 OOM Killer 触发

```c
// mm/oom_kill.c
// 当内存分配失败且无法回收时触发

// 慢速路径中调用
static inline struct page *__alloc_pages_may_oom(...)
{
    if (oom_killer_disabled)
        return NULL;
    
    // 检查是否应该触发 OOM
    if (!should_force_charge || !mem_cgroup_oom_try_charge(...)) {
        // 选择并杀死进程
        if (out_of_memory(&oc)) {
            // OOM 处理成功
        }
    }
    
    return NULL;
}
```

### 4.2 OOM 选择算法

```c
// mm/oom_kill.c
static int oom_evaluate_task(struct task_struct *task, void *arg)
{
    unsigned long points;
    
    // 不杀内核线程
    if (is_global_init(task) || (task->flags & PF_KTHREAD))
        return 0;
    
    // 计算 oom 分数
    points = oom_badness(task, oc->totalpages);
    
    // 选择分数最高的进程
    if (points > oc->chosen_points) {
        oc->chosen = task;
        oc->chosen_points = points;
    }
    
    return 0;
}

unsigned long oom_badness(struct task_struct *p, unsigned long totalpages)
{
    long points;
    long adj;
    
    // 获取 oom_score_adj
    adj = (long)p->signal->oom_score_adj;
    
    // 特殊进程不杀
    if (adj == OOM_SCORE_ADJ_MIN)
        return 0;
    
    // 基于内存使用量计算分数
    points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS);
    
    // 应用 oom_score_adj
    // adj 范围 -1000 ~ 1000
    // points = points * (1000 + adj) / 1000
    adj *= totalpages / 1000;
    points += adj;
    
    return points > 0 ? points : 1;
}
```

### 4.3 OOM 日志

```bash
# dmesg 中的 OOM 日志
$ dmesg | grep -i oom
[12345.678901] Out of memory: Killed process 1234 (com.example.app) 
               total-vm:1234567kB, anon-rss:123456kB, file-rss:12345kB, 
               shmem-rss:0kB, UID:10123 pgtables:1234kB 
               oom_score_adj:900

# 查看 oom 相关统计
$ cat /proc/vmstat | grep oom
oom_kill 5
```

---

## 五、PSI (Pressure Stall Information)

### 5.1 PSI 概念

```
PSI 测量系统资源压力：
- CPU 压力：任务等待 CPU 的时间
- 内存压力：任务等待内存的时间
- I/O 压力：任务等待 I/O 的时间

压力类型：
- some：至少一个任务停顿
- full：所有任务都停顿
```

### 5.2 读取 PSI

```bash
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# 字段说明：
# some/full: 压力类型
# avg10/60/300: 过去 10/60/300 秒的平均压力百分比
# total: 累计停顿时间（微秒）
```

### 5.3 PSI 触发器

```c
// 配置 PSI 触发器
// 写入 /proc/pressure/memory

// 格式：<type> <threshold_us> <window_us>
// 当 window_us 时间窗口内累计停顿超过 threshold_us 时触发

// 示例：1 秒内 some 压力超过 150ms
"some 150000 1000000"

// 示例：1 秒内 full 压力超过 50ms
"full 50000 1000000"

// 使用 poll() 等待触发
struct pollfd fds;
fds.fd = psi_fd;
fds.events = POLLPRI;

while (true) {
    int ret = poll(&fds, 1, -1);
    if (ret > 0 && (fds.revents & POLLPRI)) {
        // PSI 触发，处理内存压力
    }
}
```

---

## 六、配置和调优

### 6.1 LMKD 配置

```bash
# 系统属性
# /system/etc/init/lmkd.rc

# 启用 PSI
ro.lmk.use_psi=true

# PSI 阈值（毫秒）
ro.lmk.psi_partial_stall_ms=70
ro.lmk.psi_complete_stall_ms=700

# 杀进程阈值
ro.lmk.thrashing_limit=100
ro.lmk.thrashing_limit_decay=10

# swap 使用阈值
ro.lmk.swap_free_low_percentage=10
ro.lmk.swap_util_max=100
```

### 6.2 内核参数

```bash
# OOM 相关
/proc/sys/vm/oom_kill_allocating_task  # 优先杀分配进程
/proc/sys/vm/oom_dump_tasks            # OOM 时 dump 任务
/proc/sys/vm/panic_on_oom              # OOM 时 panic

# 进程级别
/proc/<pid>/oom_score_adj              # 进程 OOM 分数调整
/proc/<pid>/oom_score                  # 当前 OOM 分数（只读）
```

### 6.3 调试

```bash
# 触发 OOM（测试用）
$ echo f > /proc/sysrq-trigger

# 查看 LMKD 日志
$ adb logcat -s lmkd:*

# 查看内存压力
$ while true; do cat /proc/pressure/memory; sleep 1; done
```

---

## 总结

### LMKD vs OOM Killer

| 特性 | LMKD | OOM Killer |
|-----|------|------------|
| 位置 | 用户空间 | 内核空间 |
| 触发 | PSI 内存压力 | 分配失败 |
| 策略 | 可配置，精细 | 简单，基于分数 |
| 时机 | 预防性 | 最后手段 |
| Android | 主要机制 | 备用机制 |

### 进程优先级

- oom_adj 值越小越重要
- 系统进程：-1000 到 -700
- 前台进程：0 到 200
- 后台进程：300 到 999

### 关键组件

| 组件 | 作用 |
|-----|------|
| AMS/OomAdjuster | 计算 oom_adj |
| LMKD | 监控压力，杀进程 |
| PSI | 内存压力指标 |
| OOM Killer | 内核级保护 |

### 后续学习

- [ART虚拟机内存管理](18-ART虚拟机内存管理.md) - 了解 Java 内存管理
- [内存性能分析与问题排查](20-内存性能分析与问题排查.md) - 内存问题诊断

## 参考资源

- Android 源码：`system/memory/lmkd/`
- 内核源码：`mm/oom_kill.c`
- 内核文档：`Documentation/accounting/psi.rst`

## 更新记录

- 2026-01-28：初始创建，包含 LMKD 与 OOM 机制详解
