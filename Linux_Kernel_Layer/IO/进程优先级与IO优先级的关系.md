# 进程优先级与 I/O 优先级的关系：Android 稳定性视角

> 在 Android 系统中，确保 UI 流畅度（60fps 或 120fps）的核心挑战之一，就是防止后台繁重的 I/O 操作（如日志写入、数据库同步）阻塞前台的关键读取（如加载资源、启动 App）。本文从进程优先级与 I/O 优先级的联动出发，深入解析 Linux 块层、调度器、BIO 拆分与合并，以及 Android 的应对策略。

**快速定位**：进程优先级架构 §1 → 联动机制与源码 §2（含 blkio cgroup 约束机制 §2.4）→ §3–4 → 优先级反转与 eMMC §5–7 → BIO 拆分/合并 §8–11 → 宏观流程 §12 → 总结与调优 §13–15 → io_uring §16 → 排查工具 §17

---

## 1. Android 进程优先级的多层架构

在 Android 中，进程优先级并不是一个单一的数值，它由三个维度共同决定：

| 维度 | 含义 | 作用范围 |
| :--- | :--- | :--- |
| **OOM Adjustment (ADJ)** | 由 ActivityManagerService 动态维护 | 决定进程在内存不足时是否被杀掉 |
| **Thread Priority (Nice 值)** | 传统的 Linux 进程优先级 | 决定 CPU 时间片的分配 |
| **Cgroups (Control Groups)** | Android 特有的调度方式 | 将进程划分为「前台」「后台」「顶层（Top-app）」等组，直接限制 CPU 周期和 I/O 带宽 |

三者共同作用：OOM_ADJ 决定「谁先死」，Nice 决定「谁先跑」，Cgroup 决定「谁能用多少资源」。

---

## 2. 进程优先级与 I/O 优先级的联动机制

### 2.1 原生 Linux 与 Android 的差异

在原生 Linux 中，I/O 优先级通常默认跟随 Nice 值。但在 Android 中，这种关系是通过 **libutils**、**SetTaskProfiles** 以及内核的 Nice 映射共同实现的。

### 2.2 核心联动逻辑与源码佐证

当 Android 应用通过 `Process.setThreadPriority(int priority)` 设置优先级时，实际有两条路径影响 I/O 优先级：

#### 路径 A：setpriority → Nice → 内核映射到 I/O 优先级

```
Process.setThreadPriority(pid, pri)
  → JNI: android_os_Process_setThreadPriority()     // frameworks/base/core/jni/android_util_Process.cpp
  → androidSetThreadPriority(pid, pri)               // system/core/libutils/Threads.cpp
  → setpriority(PRIO_PROCESS, tid, pri)             // 仅修改 Nice 值，不直接调用 ioprio_set
```

**源码**（`system/core/libutils/Threads.cpp`）：

```c
int androidSetThreadPriority(pid_t tid, int pri)
{
    int rc = 0;
    if (setpriority(PRIO_PROCESS, tid, pri) < 0) {
        rc = INVALID_OPERATION;
    } else {
        errno = 0;
    }
    return rc;
}
```

I/O 优先级的生效发生在**块层取优先级时**：若进程未显式设置 I/O 优先级，内核根据 Nice 推导。`block/ioprio.c` 中 `__get_task_ioprio()`：

```c
#ifdef CONFIG_BLOCK
/*
 * If the task has set an I/O priority, use that. Otherwise, return
 * the default I/O priority.
 *
 * Expected to be called for current task or with task_lock() held to keep
 * io_context stable.
 */
int __get_task_ioprio(struct task_struct *p)
{
    struct io_context *ioc = p->io_context;
    int prio;
    if (ioc)
        prio = ioc->ioprio;
    else
        prio = IOPRIO_DEFAULT;
    // 关键：若未显式设置，则从 Nice 推导
    if (IOPRIO_PRIO_CLASS(prio) == IOPRIO_CLASS_NONE)
        prio = IOPRIO_PRIO_VALUE(task_nice_ioclass(p), task_nice_ioprio(p));
    return prio;
}
```

因此：**setpriority 修改 Nice → 块层在取优先级时用 `task_nice_ioprio()` 映射为 I/O 优先级**。

#### 路径 B：SetTaskProfiles → cgroup blkio（进程/线程组变化时）

当进程或线程**调度组**变化时（如 App 切后台、`setThreadGroup`），系统通过 Task Profiles 应用包含 I/O 策略的聚合配置。

**调用链：**

```
Process.setThreadGroup(tid, SP_BACKGROUND)   // Java API
  → JNI: android_os_Process_setThreadGroup()  // frameworks/base/core/jni/android_util_Process.cpp
  → SetTaskProfiles(tid, {"SCHED_SP_BACKGROUND"}, true)
  → TaskProfiles::SetTaskProfiles(tid, profiles)  // system/core/libprocessgroup/task_profiles.cpp
  → profile->ExecuteForTask(tid)  // 执行 SCHED_SP_BACKGROUND 聚合配置
  → ApplyProfileAction 展开为 [HighEnergySaving, LowIoPriority, TimerSlackHigh]
  → LowIoPriority 的 JoinCgroup 解析为 SetCgroupAction(blkio, "background")
  → SetCgroupAction::ExecuteForTask(tid)
  → write(tid) 到 /sys/fs/cgroup/blkio/background/tasks  // 将 tid 写入 cgroup
```

**源码 1**：`frameworks/base/core/jni/android_util_Process.cpp` — JNI 入口

```c
void android_os_Process_setThreadGroup(JNIEnv* env, jobject clazz, int tid, jint grp)
{
    if (!verifyGroup(env, grp)) return;
    int res = SetTaskProfiles(tid, {get_sched_policy_profile_name((SchedPolicy)grp)}, true) ? 0 : -1;
    if (res != NO_ERROR) {
        signalExceptionForGroupError(env, -res, tid);
    }
}
```

**源码 2**：`system/core/libprocessgroup/sched_policy.cpp` — 策略名到 Profile 的映射

```c
case SP_BACKGROUND:
    return SetTaskProfiles(tid, {"SCHED_SP_BACKGROUND"}, true) ? 0 : -1;
```

**源码 3**：`system/core/libprocessgroup/task_profiles.cpp` — SetTaskProfiles 与 JoinCgroup 解析

```c
// SetTaskProfiles：遍历 profiles，执行每个 profile 的 ExecuteForTask
bool TaskProfiles::SetTaskProfiles(pid_t tid, std::span<const std::string> profiles, ...) {
    for (const auto& name : profiles) {
        TaskProfile* profile = GetProfile(name);  // 如 "SCHED_SP_BACKGROUND"
        if (profile != nullptr) {
            if (!profile->ExecuteForTask(tid)) { ... }
        }
    }
}

// 解析 task_profiles.json 时，JoinCgroup 创建 SetCgroupAction
if (action_name == "JoinCgroup") {
    std::string controller_name = params_val["Controller"].asString();  // "blkio"
    std::string path = params_val["Path"].asString();                    // "background"
    auto controller = cg_map.FindController(controller_name);
    if (controller.HasValue() && controller.version() == 1) {
        profile->Add(std::make_unique<SetCgroupAction>(controller, path));
    }
}

// SetCgroupAction::ExecuteForTask：将 tid 写入 cgroup 的 tasks 文件
bool SetCgroupAction::AddTidToCgroup(pid_t tid, int fd, ResourceCacheType cache_type) const {
    std::string value = std::to_string(tid);
    if (TEMP_FAILURE_RETRY(write(fd, value.c_str(), value.length())) == value.length()) {
        return true;  // 成功将 tid 写入 blkio/background/tasks
    }
    ...
}
```

**task_profiles.json 片段**（`system/core/libprocessgroup/profiles/task_profiles.json`）：

```json
{"Name": "LowIoPriority", "Actions": [{"Name": "JoinCgroup", "Params": {"Controller": "blkio", "Path": "background"}}]},
{"Name": "SCHED_SP_BACKGROUND", "Profiles": ["HighEnergySaving", "LowIoPriority", "TimerSlackHigh"]},
{"Name": "SCHED_SP_TOP_APP", "Profiles": ["MaxPerformance", "MaxIoPriority", "TimerSlackNormal"]}
```

**最终效果**：在 **cgroup v1** 设备上，tid 被写入 `/sys/fs/cgroup/blkio/background/tasks`，该线程的 I/O 请求受 blkio cgroup 的 `background` 子组策略约束（权重、限速等）。

**为何在设备上找不到该节点？** 许多新机型已迁移到 **cgroup v2**。在 `task_profiles.cpp` 中，当 blkio 控制器为 cgroup v2 时，JoinCgroup 会被**显式忽略**：

```c
if (controller.version() == 1) {
    profile->Add(std::make_unique<SetCgroupAction>(controller, path));
} else {
    LOG(WARNING) << "A JoinCgroup action in the " << profile_name
        << " profile is used for controller " << controller_name
        << " in the cgroup v2 hierarchy and will be ignored";
}
```

**如何确认设备使用的 cgroup 版本？**

- **cgroup v1**：`ls /sys/fs/cgroup/blkio/` 可见 `background` 等子目录，`/sys/fs/cgroup/blkio/background/tasks` 存在
- **cgroup v2**：通常为单一挂载点 `/sys/fs/cgroup`，无独立 `blkio` 目录；io 控制器通过 `io.stat`、`io.max`、`io.weight` 等接口提供（路径如 `/sys/fs/cgroup/.../io.max`，具体取决于 Android 的 cgroup 布局）

在 cgroup v2 设备上，LowIoPriority 的 JoinCgroup 当前不生效，I/O 优先级主要依赖**路径 A**（Nice → `task_nice_ioprio`）以及 CPU/cpuset 等 cgroup 的间接影响。

**优先级映射策略：**

- **高优先级（如 Top-App）**：MaxIoPriority → blkio 根或高权重组；Nice 小 → `task_nice_ioprio` 映射为 BE 0–2
- **低优先级（如 Background）**：LowIoPriority → blkio/background；Nice 大 → 映射为 BE 7 或 Idle
- **Real-Time (RT)**：极少使用，慎用，可能饿死其他进程

### 2.3 Task Profiles 与 cgroup

**Task Profiles** 是 Android 框架中的配置系统，用于将进程状态（如 foreground、background、top-app）映射到具体的资源策略。它基于 cgroup，在进程进入不同状态时，自动应用预定义的 CPU 调度类、Nice 值、I/O 优先级等。配置文件通常位于 `system/core/libprocessgroup/profiles/task_profiles.json`（设备上为 `/system/etc/task_profiles.json`），由 `ProcessList` 和 `ActivityManagerService` 在进程状态变化时触发应用。

Android 通过 Task Profiles 集中管理进程优先级与 I/O 优先级的联动。当一个 App 从前台切到后台，系统会自动降低该进程所有线程的 I/O 优先级，以确保前台应用拥有「磁盘优先通行权」。

### 2.4 blkio/io cgroup 对 I/O 的约束机制：内核源码视角

当线程被加入 blkio cgroup 的 `background` 子组后，其 I/O 请求如何受「权重、限速」约束？内核在块层通过 **blkcg（Block cgroup）** 子系统实现。

#### 2.4.1 数据结构与关联

| 结构体 | 含义 |
| :--- | :--- |
| **blkcg** | 块层 cgroup，对应 blkio 控制器下的一个 cgroup |
| **blkcg_gq (blkg)** | blkcg 与 request_queue 的关联，每个 (blkcg, queue) 一对 |
| **blkcg_policy** | 策略（如 throttle、ioprio）的 per-blkg 数据 |

进程在 `submit_bio` 时，内核通过 `blkcg_css()` 获取当前任务的 cgroup 归属：

```c
// block/blk-cgroup.c
static struct cgroup_subsys_state *blkcg_css(void)
{
    struct cgroup_subsys_state *css;
    css = kthread_blkcg();
    if (css)
        return css;
    return task_css(current, io_cgrp_id);  // 当前任务所属 blkio cgroup
}
```

#### 2.4.2 权重 / 限速策略的生效

**权重（blkio.weight）**：由 I/O 调度器（如 BFQ、CFQ）使用。调度器根据每个 blkg 的 `blkg->pd[policy]->weight` 决定请求的调度顺序。background 子组通常配置较低 weight，在带宽分配时获得更少份额。

**限速（blk-throttle）**：由 `block/blk-throttle.c` 实现。BIO 在进入 `blk_mq_submit_bio` 前，会经过 `blk_throtl_bio()` 检查：

- 若该 blkg 的 BPS/IOPS 已超限，BIO 被挂入 `throtl` 队列，进程可能进入等待
- 限速通过 `blkio.throttle.read_bps_device`、`blkio.throttle.write_bps_device` 等配置

**关键路径**：`submit_bio` → `submit_bio_noacct` → `blk_throtl_bio()` 检查限速 → `submit_bio_noacct_nocheck` → `blkcg_bio_issue_init`（取 blkg）→ `__submit_bio` → `blk_mq_submit_bio` → 调度器按 blkg 权重排序

**源码 1**：`block/blk-core.c` — submit_bio 入口与限速检查

```c
void submit_bio(struct bio *bio)
{
    if (blkcg_punt_bio_submit(bio))
        return;
    if (bio_op(bio) == REQ_OP_READ) {
        task_io_account_read(bio->bi_iter.bi_size);
        count_vm_events(PGPGIN, bio_sectors(bio));
    } else if (bio_op(bio) == REQ_OP_WRITE) {
        count_vm_events(PGPGOUT, bio_sectors(bio));
    }
    bio_set_ioprio(bio);      // 设置 bio->bi_ioprio（含 Nice 映射）
    submit_bio_noacct(bio);
}

void submit_bio_noacct(struct bio *bio)
{
    // ... 各种校验 ...
    if (blk_throtl_bio(bio))   // 限速检查：超限则 BIO 入 throtl 队列，直接 return
        return;
    submit_bio_noacct_nocheck(bio);
}

static void __submit_bio(struct bio *bio)
{
    // ...
    if (!disk->fops->submit_bio) {
        blk_mq_submit_bio(bio);   // 多队列设备走此路径
    } else {
        disk->fops->submit_bio(bio);
    }
}
```

**源码 2**：`block/blk-throttle.c` — __blk_throtl_bio 限速逻辑

```c
bool __blk_throtl_bio(struct bio *bio)
{
    struct request_queue *q = bdev_get_queue(bio->bi_bdev);
    struct blkcg_gq *blkg = bio->bi_blkg;           // 当前 BIO 所属 blkg（来自 cgroup）
    struct throtl_grp *tg = blkg_to_tg(blkg);      // blkg 对应的 throttle 组
    struct throtl_service_queue *sq = &tg->service_queue;
    bool rw = bio_data_dir(bio);
    bool throttled = false;

    spin_lock_irq(&q->queue_lock);
    // ...
    while (true) {
        /* if above limits, break to queue */
        if (!tg_may_dispatch(tg, bio, NULL)) {      // 检查 BPS/IOPS 是否超限
            tg->last_low_overflow_time[rw] = jiffies;
            break;
        }
        /* within limits, let's charge and dispatch directly */
        throtl_charge_bio(tg, bio);                 // 未超限：扣减配额，直接放行
        throtl_trim_slice(tg, rw);
        sq = sq->parent_sq;
        tg = sq_to_tg(sq);
        if (!tg) {
            goto out_unlock;                        // 到达根，直接下发
        }
    }
    /* out-of-limit, queue to @tg */
    throtl_add_bio_tg(bio, qn, tg);                // 超限：BIO 入 throtl 队列等待
    throttled = true;
out_unlock:
    spin_unlock_irq(&q->queue_lock);
    return throttled;
}
```

**源码 3**：`block/blk-mq.c` — blk_mq_submit_bio 与调度器插入

```c
void blk_mq_submit_bio(struct bio *bio)
{
    struct request_queue *q = bdev_get_queue(bio->bi_bdev);
    struct blk_plug *plug = blk_mq_plug(bio);
    // ...
    rq = blk_mq_get_new_requests(q, plug, bio, nr_segs);  // 创建 request
    // ...
    blk_mq_bio_to_request(rq, bio, nr_segs);       // bio 转 request，继承 blkg
    if (plug)
        blk_add_rq_to_plug(plug, rq);
    else if ((rq->rq_flags & RQF_ELV) || ...)
        blk_mq_sched_insert_request(rq, false, true, true);  // 入调度器，按 blkg 权重排序
    else
        blk_mq_try_issue_directly(rq->mq_hctx, rq);
}
```

#### 2.4.3 cgroup v1 与 v2 的差异

| 类型 | 路径 | 接口 |
| :--- | :--- | :--- |
| **cgroup v1 blkio** | `/sys/fs/cgroup/blkio/background/` | `blkio.weight`、`blkio.throttle.write_bps_device` 等 |
| **cgroup v2 io** | `/sys/fs/cgroup/.../io.max`、`io.weight` | `io.max`（按设备限速）、`io.weight`（权重） |

cgroup v2 的 io 控制器统一在单一层级下，接口更简洁，但 Android 的 task_profiles 当前对 blkio JoinCgroup 在 v2 下不生效，需等待后续适配。

---

## 3. I/O 优先级在调度器中是如何起作用的？

I/O 优先级在 Linux **块设备层（Block Layer）** 发挥作用，具体行为取决于当前内核使用的 I/O 调度算法（I/O Scheduler）。

### 3.1 BFQ (Budget Fair Queuing) — 现代 Android 的主流选择

BFQ 是目前高端 Android 手机（如 Pixel 系列）常用的调度器，它对 I/O 优先级非常敏感：

- **分配「预算」**：BFQ 为每个进程分配磁盘扇区的读取「预算」（Budget）。优先级越高，单次分配的预算越大，且获取磁盘控制权的频率越高。
- **低延迟保证**：如果检测到是交互式应用（前台），BFQ 会给予抢占特权，哪怕后台正在进行大文件的顺序写入。

### 3.2 CFQ (Completely Fair Queuing) — 旧版设备

- **时间片调度**：每个 I/O 优先级对应不同的时间片。
- **优先级反转预防**：如果一个高优先级进程在等待一个低优先级进程释放文件锁，CFQ 会临时提升后者的 I/O 表现。

### 3.3 Kyber / MQ-Deadline — 闪存优化调度器

在采用 UFS 3.0+ 高速闪存的设备上，Android 倾向于使用多队列（blk-mq）调度器：

- **限流机制**：Kyber 通过监控 I/O 延迟来工作。当后台 I/O 导致延迟增加时，它会严格限制 Idle 类别进程的并发请求数（Queue Depth），给 Best-Effort 请求让路。

---

## 4. 进程优先级与 I/O 优先级在 Android 中的深度联动

### 4.1 静态映射：从 Nice 到 I/O Priority

在 Android 的 libutils 和 Process.java 中，当设置进程的 `setThreadPriority` 时，内核底层遵循 **Best-Effort (BE)** 调度类。

**计算公式**：内核将 Nice 值（-20 到 19）映射为 8 个 I/O 优先级（0 到 7），近似为 `ioprio = (nice + 20) / 5`（Nice -20 → 0，Nice 19 → 7）。

| Android 进程类型 | Nice 值 | I/O Priority |
| :--- | :--- | :--- |
| Foreground/Visible | 0 或更小 | 4 或更高 |
| Background | 约 10 | 6 |
| Cached | 19 | 7 |

### 4.2 Android 特有的调度器约束：CFS 与 BFQ/Kyber

I/O 优先级能否发挥作用，取决于底层 I/O Scheduler 的实现。目前主流机型（如 UFS 3.0+ 设备）通常使用 **mq-deadline** 或 **BFQ**。

| 调度器 | 特点 |
| :--- | :--- |
| **BFQ** | 参考进程 Nice 值分配「I/O 预算」。若 SystemServer 因日志写入请求磁盘，而某后台 App 正在大文件下载，BFQ 会根据 I/O 优先级强制剥夺后台 App 的时隙，确保 SystemServer 不被 I/O Wait 阻塞导致 Watchdog 重启 |
| **mq-deadline** | 在低端机或特定闪存上使用，对 I/O 优先级的感知较弱，主要通过读写队列分离来保证读取（UI 渲染通常是读取）优先 |

### 4.3 I/O 优先级在调度中的三大作用机制

| 机制 | 说明 |
| :--- | :--- |
| **A. 权重分配（Weight Distribution）** | 在 Best-Effort 类中，优先级 0 的进程获得的磁盘时间片权重远高于优先级 7。权重与 I/O 优先级成反比：`Weight ∝ 1/(io_priority + 1)`。这意味着当多个进程竞争闪存带宽时，高 I/O 优先级的进程能获得更多 IOPS，从而降低 I/O Latency |
| **B. 交互式启发（Interactive Hinting）** | 调度器（如 BFQ）具有软实时特性。若进程被标识为 Top-App，调度器会通过观察其 CPU 活跃度，自动将其识别为「交互式」，并临时提升其 I/O 优先级，防止应用启动时加载资源过慢 |
| **C. 避免优先级反转（Priority Inversion）** | 当 OOM_ADJ 极高的前台进程等待 Background 进程释放文件锁时：CPU 维度通过 libperfmgr 提升频率；I/O 维度通过 Writeback 节流限制低优先级进程的写入量，防止其占满硬件队列（Queue Depth），为高优先级请求留出通道 |

**总结：两者的协作效应** — 进程优先级（OOM_ADJ、Nice、Cgroup）与 I/O 优先级并非独立运作，而是形成闭环：Cgroup 决定进程归属与资源上限，Nice 与 ioprio_set 联动决定 CPU 与 I/O 的权重，调度器（BFQ/Kyber/mq-deadline）在块层将 I/O 优先级转化为实际的请求排序与限流，最终共同保障前台应用的「磁盘优先通行权」。

### 4.4 稳定性专家的避坑指南

**核心风险点：I/O 优先级不是万能药**

| 风险 | 说明 |
| :--- | :--- |
| **硬件队列瓶颈** | 在 UFS 设备上，若硬件命令队列（Command Queue）被填满，OS 层面的优先级可能失效 |
| **Cgroup 隔离** | 即使提升了进程的 I/O 优先级，若该进程被限制在 background cgroup 中，其最大带宽 (Throttle) 依然会被锁定 |
| **同步 vs 异步** | **Buffered Writes（异步）**：进程写到 Page Cache 就返回，此时 I/O 优先级影响的是内核 kworker 刷盘的优先级；**Direct/Sync I/O（同步）**：优先级立即生效，这也是为什么 SQLite 操作对 I/O 优先级极其敏感 |

**结论**：在 Android 稳定性治理中，CPU 优先级决定了进程「想写」的快慢，而 I/O 优先级决定了内核「允许它写」的快慢。对于 ANR 治理，应检查是否存在低优先级进程占满 IOPS 导致高优先级进程 D 状态（Uninterruptible Sleep）过长。对于启动优化，通过 `ionice` 手动将 dex2oat 或资源解压进程设为 Best-Effort 0 或 Real-Time 往往能获得显著收益，例如：`ionice -c 2 -n 0 -p <pid>`（BE 最高优先级）或 `ionice -c 3 -p <pid>`（IDLE，用于后台任务降权）。

---

## 5. 什么是优先级反转？eMMC 单队列的困境

**优先级反转（Priority Inversion）** 是指：高优先级任务因等待低优先级任务持有的资源（如锁、I/O 完成）而被阻塞，导致高优任务的实际执行顺序反而落后于低优任务。在 I/O 场景下，典型表现为：SystemServer 的紧急请求必须等后台进程的大块写入完成才能得到服务。

**问题**：低端机用 eMMC 存储，I/O 硬件队列只有一个。若后台正在做大量 I/O，此时 SystemServer 来了一个很紧急的 I/O 请求，会发生什么？一个队列一次是否只能处理一个 I/O 请求？

**回答**：在硬件物理层面，你的理解是正确的。但在内核调度层面，Linux 为了应对这种「独木桥」困境，设计了一套复杂的「交通管制」机制。作为 Android 稳定性专家，处理 eMMC 设备上的 I/O 阻塞（通常表现为 SystemServer 的 D 状态和随后的 Watchdog 重启）时，理解以下三个层面的交互至关重要：

1. **物理现实**：eMMC 单队列的硬件限制（见第 6 节）
2. **内核补救**：I/O 调度器的软件队列与「插队」策略（见第 6 节）
3. **稳定性风险**：线头阻塞、文件系统锁等仍可能导致卡顿的原因（见第 7 节）

---

## 6. 物理现实：单队列（Single Queue）的局限

### 6.1 eMMC 与 UFS 的差异

eMMC 协议（如 eMMC 5.1）**不支持** NCQ (Native Command Queuing) 或 UFS 那样的多队列并行。

| 物理瓶颈 | 说明 |
| :--- | :--- |
| **单队列** | 在任何给定时钟周期内，总线上只能有一个数据块在传输 |
| **不可撤回性** | 一旦一个巨大的写入请求（如后台 App 正在 fsync 一个 100MB 的文件）已经下发到 eMMC 控制器，内核无法中途撤回。SystemServer 的紧急请求必须在硬件队列外「排队」 |

### 6.2 内核补救：I/O 调度器的「插队」艺术

虽然硬件一次只能处理一个，但内核在将请求交给硬件之前，维护着一个**软件队列**。I/O 调度器（如 mq-deadline、CFQ）在此发挥作用：

1. **请求合并与排序**：若 SystemServer 发出一个读取请求（通常是同步的），调度器会识别出其 High Priority，并将其插入到软件队列的最前端，紧跟在当前正在执行的硬件请求之后。
2. **读优先于写（Read over Write）**：在 eMMC 环境下，读取通常比写入更紧急。mq-deadline 会优先处理读取请求，防止 UI 线程因等待配置文件读取而卡死。
3. **打断大块 I/O**：内核会将大的 I/O 分解成多个较小的 bio，这样高优先级的 SystemServer 请求就有机会在两个大块数据之间「加塞」。

**专家总结**：在 eMMC 设备上，硬件确实一次只能处理一个请求。但通过 I/O 优先级映射，内核调度器确保了：在软件队列中，SystemServer 永远排在最前面；通过限制后台进程的并发数，给硬件留出「喘息」机会。

---

## 7. 稳定性风险：为什么还是会卡顿？

### 7.1 硬件层面的「线头阻塞」（Head-of-Line Blocking）

若后台进程下发了一个巨大的「同步写入」（Synchronous Write），且 eMMC 控制器正在处理这个请求。即使 SystemServer 的请求在软件层面排到了第一位，它也必须等待当前硬件操作完成。在低端 eMMC 上，这个延迟可能高达**数百毫秒**，足以触发 Android 的 ANR。

### 7.2 文件系统锁（FS Locks）的连锁反应

这是最隐蔽的联系：

1. 后台进程正在对某分区进行大量写入，可能持有文件系统的 **Journal Lock**（日志锁）。
2. 后台进程由于 I/O 优先级低，写盘慢，持有锁的时间变长。
3. SystemServer 想要进行一个很小的 I/O 操作，但需要获取同一个日志锁。
4. **结果**：SystemServer 在 CPU 上被挂起，等待一个低优先级的 I/O 完成。这就是典型的**跨维度优先级反转**。

**排查建议**：若发现 iowait 异常升高，或 SystemServer 阻塞在 vfs_write 相关的内核调用栈上，可针对具体调用栈分析是否是文件系统锁导致的次生灾害。

### 7.3 Android 的应对策略：Writeback Throttling (WBT)

为解决低端机单队列问题，Android 内核引入了 WBT 机制：

- 内核会监控磁盘的延迟。当发现磁盘响应变慢（表明硬件队列拥塞）时，它会主动**惩罚（Throttling）**那些低优先级的异步写入进程。
- 强制让后台写入进程进入 sleep，减少其下发 I/O 的频率。
- 相当于在「单行道」上人为制造空隙，确保 SystemServer 这种高优先级车辆随时有空位钻过去。

---

## 8. 为什么必须拆分 BIO？

### 8.1 硬件限制

在 Linux 内核中，文件系统发出的 I/O 请求被称为 **BIO**。但硬件（如 eMMC 芯片）对单次传输的大小有物理限制：

| 参数 | 含义 |
| :--- | :--- |
| **max_sectors_kb** | 硬件单次能处理的最大千字节数（eMMC 通常是 128 KB 或 512 KB） |
| **max_segments** | 硬件单次能处理的最大内存分段数 |

当一个巨大的 BIO（比如 1 MB）进入通用块层（Generic Block Layer）时，内核会调用 **blk_queue_split** 将其拆分成多个较小的 Requests。

### 8.2 核心洞察：BIO 拆分与 I/O 优先级

**如果后台进程发起了一个 10 MB 的大块 I/O 请求，而系统不进行拆分，那么即便 SystemServer 的请求优先级再高，也得在胡同口等这 10 MB 全部走完。**（典型场景下，1 MB 的 BIO 会被拆成约 8 个 128 KB 的 Request，见第 9 节场景模拟。）

BIO 的拆分实际上是给 I/O 调度器提供了**「微操」的颗粒度**。如果没有拆分，大 I/O 就会形成长达数百毫秒的「不可中断干扰」。核心逻辑在 `blk_queue_split`（`block/blk-mq.c`）。

---

## 9. 拆分如何给了优先级「插队」的机会？

### 9.1 场景模拟

| 步骤 | 描述 |
| :--- | :--- |
| 后台进程 A | 提交了一个 1 MB BIO，被拆成了 Request-A1 到 Request-A8 |
| 执行开始 | 调度器把 A1 发给了 eMMC |
| 高优请求 B | eMMC 正在处理 A1 时，system_server 发起了一个 4 KB 的同步读取（Request-B） |
| 重新排序 | 软件队列（如 BFQ 的红黑树）中剩下的 Request 是：{A2, A3, ..., A8}。调度器看到 Request-B 进来了，其 I/O 优先级是 RT 或高权重的 BE |
| 抢占成功 | 调度器立即把 Request-B 插到了 A2 的前面。当 eMMC 报告 A1 处理完毕时，调度器不发 A2，而是发 B |

**洞察**：BIO 拆分的大小（颗粒度）直接决定了高优 I/O 的最大等待延迟。拆得越细，插队的机会就越多。

### 9.2 调度的机会窗口

- **如果没有拆分**：1 MB 一旦开始传输，总线就被锁死，直到传输完毕。
- **有了拆分**：调度器在发送完第一个 128 KB 后，会回到软件队列里看一眼：「刚才有没有高优先级的任务进来了？」

---

## 10. 「合并」引发的副作用（I/O Merging）

虽然拆分有利于插队，但内核还有一个相反的操作：**合并（Merging）**，包括 front merge（新请求合并到已有 request 前）和 back merge（合并到后）。

- **目的**：为了提升吞吐量，内核会将相邻扇区的多个小 Request 合并成一个大 Request。
- **风险**：如果内核过度合并了后台进程的请求，可能会无意中增加前台请求的等待时间。
- **Android 优化**：现代 Android 内核在处理 I/O Priority Class 为 **IDLE** 的进程时，会限制其合并的深度，确保它们的 Request 不会变得太大而阻塞总线。

---

## 11. eMMC 环境下的极端情况：写放大与弃权

即使有拆分，低端机（eMMC）依然会卡顿，原因往往在于 **GC（Garbage Collection）**。

- 当后台进程进行大量写入时，eMMC 内部的控制器可能会触发固件级的垃圾回收。
- **现象**：此时即使内核调度器想插队，eMMC 硬件也会因为忙于内部数据搬移（搬移过程中总线是锁死的）而停止响应任何命令。
- **结果**：软件层的优先级算法会彻底失效，造成所谓的「硬卡死」。

---

## 12. 宏观流程：从 BIO 到 Request

当 system_server 或一个后台 App 想要写数据时，数据会经历以下形态演变：

| 形态 | 含义 |
| :--- | :--- |
| **BIO (Block I/O)** | 文件系统产生的基本单位，描述「我想把内存里的这些 Page 写入磁盘的这些扇区」。一个 BIO 可能非常大（如 1 MB） |
| **Request** | 块设备驱动能理解的单位，一个 Request 包含一个或多个 BIO |
| **Dispatch Queue** | 调度器准备发给硬件的「待办清单」 |

### 12.1 BIO 拆分的深层逻辑：化整为零

**拆分过程（blk_queue_split）**：若后台进程发起一个 1 MB 的写入，内核发现 eMMC 单次只能吃下 128 KB，blk_queue_split 会将这个 1 MB 的 BIO 砍成 8 个小块，每个小块被包装成一个 Request。

---

## 13. 优先级「拆」「排」「插」的总结

| 阶段 | 含义 |
| :--- | :--- |
| **大变小** | Generic Block Layer 把巨大的 BIO 拆解成符合硬件胃口的 Requests |
| **排好队** | I/O Scheduler 根据进程优先级（Nice 值映射）对这些 Requests 进行排序 |
| **看空挡** | 在每一个小 Request 完成传输的瞬间，调度器都有机会让高优请求「加塞」 |

要深入理解 BIO 拆分与 I/O 优先级在 Android 系统中的协作，需要进入 Linux 内核的「地下室」——块设备层（Block Layer）。在 eMMC 这种单队列硬件上，「拆分」不仅是硬件的物理要求，更是软件层面实现「抢占」的先决条件。

---

## 14. Android 的特殊调优：不仅仅是插队

在低端 eMMC 设备上，仅仅靠插队是不够的，因为 eMMC 还有个致命弱点：**读写互斥**。若在疯狂写入，读取请求即便插队成功，也会被硬件内部的 Flash 转换层（FTL）卡住。因此，Android 内核引入了以下机制：

### 14.1 写入限流 (Writeback Throttling)

- Android 的 cgroup 会监控 I/O 延迟。若 system_server 的读取延迟超过了阈值（如 20 ms），内核会强制挂起后台进程的 BIO 提交。
- 从源头上减少后台 I/O 请求进入队列的数量，防止「堵死」胡同。

### 14.2 优先级继承 (I/O Priority Inheritance)

- 若 system_server 发起的 I/O 实际上是在等待一个后台进程持有的文件锁，Android 会触发**优先级继承**。
- 不仅提升该进程的 CPU 优先级（Nice 值），还会通过 ionice 逻辑提升其 I/O 优先级。
- **目的**：让这个「挡路」的进程赶紧把剩下的 BIO 走完，释放锁。

---

## 15. 堵车时的思考：类比与局限

**类比**：BIO 拆分好比「公共汽车拆成小轿车」：

- **硬件队列**：是一条只能过一辆车的窄单行道。
- **BIO 拆分**：把后台的大卡车拆成一串小轿车，这样每过去一辆小车，交警（I/O 调度器）就有机会拦住后面的小车，让救护车（SystemServer）先过。
- **局限性**：若救护车来的时候，卡车的一半已经进窄路了（In-flight），救护车还是得等（物理延迟）。
- **Android 优化**：交警不仅会拦车，还会给救护车前面的车「开绿灯」（优先级继承），甚至限制卡车上路的总数（限流）。

---

## 16. 进阶思考：io_uring 与 Android

**io_uring** 自 Linux 5.1 引入，已进入较新 Android 内核（如 5.10+、6.x）。与传统 `submit_bio` 不同，io_uring 通过共享 ring buffer 批量提交 I/O，减少系统调用开销。**在 Android 上**：目前主流应用与框架仍以传统路径为主，io_uring 在 Android 用户态的实际采用尚在演进中；若未来普及，需关注 I/O 优先级（io_context、blkcg）在 io_uring 路径上是否正确传递，以保障「磁盘优先通行权」。本节作为前瞻，供关注新内核特性的读者参考。

---

## 17. Android 设备排查工具速查

| 工具 / 路径 | 用途 | 说明 |
| :--- | :--- | :--- |
| **ANR trace** | 识别主线程 I/O 阻塞 | 主线程栈若在 `wait_on_page_bit`、`io_schedule`、`blk_mq_get_request` 等，多为 I/O 阻塞；Android 原生，无需 root |
| **StrictMode** | 开发期检测主线程同步 I/O | `StrictMode.setThreadPolicy` 检测 commit/DB/fsync 等，及早发现卡顿风险 |
| **dumpsys diskstats** | 查看 I/O 统计 | `adb shell dumpsys diskstats`，输出各应用读写量等 |
| **/proc/diskstats** | 设备级 IOPS、延迟 | `adb shell cat /proc/diskstats`，需在 PC 端用脚本或 iostat 解析 |
| **atrace / systrace** | 系统级追踪 | 含 I/O 相关事件，`adb shell atrace` 或 Chrome systrace |
| **ftrace + blk tracer** | 块层请求追踪 | 需 root 或 eng 内核，追踪 `blk_block_rq_issue`、`blk_block_rq_complete` |
| **ionice** | 查看/设置 I/O 优先级 | 设备上通常无此命令，可经 `adb shell` 在含 util-linux 的调试环境中使用；日常优先级由系统 Task Profiles 管理 |

---

## 关键源码路径

| 模块 | 路径 |
| :--- | :--- |
| **Android 进程优先级** | `frameworks/base/core/jni/android_util_Process.cpp`（JNI 入口） |
| **androidSetThreadPriority** | `system/core/libutils/Threads.cpp`（setpriority 调用） |
| **SetTaskProfiles / 调度策略** | `system/core/libprocessgroup/sched_policy.cpp` |
| **Task Profiles 配置** | `system/core/libprocessgroup/profiles/task_profiles.json` |
| **内核 I/O 优先级** | `block/ioprio.c`（`__get_task_ioprio`、`task_nice_ioprio`） |
| **块层 cgroup（blkcg）** | `block/blk-cgroup.c`（blkcg_css、blkg） |
| **I/O 限速（throttle）** | `block/blk-throttle.c`（blk_throtl_bio） |
| **BIO 拆分** | `block/blk-mq.c`（blk_queue_split） |
| **BFQ 调度器** | `block/bfq-*.c` |

---

