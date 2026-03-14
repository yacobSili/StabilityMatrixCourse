# CFS 调度器详解

## 学习目标

- 理解 CFS（完全公平调度器）的设计理念
- 掌握虚拟运行时间（vruntime）的计算
- 理解 CFS 运行队列的红黑树结构
- 了解调度决策和时间片计算
- 理解组调度（Group Scheduling）机制

## 概述

CFS（Completely Fair Scheduler，完全公平调度器）是 Linux 2.6.23 引入的默认调度器，设计目标是为所有进程提供公平的 CPU 时间分配。

**核心思想**：
- 跟踪每个进程的虚拟运行时间（vruntime）
- 总是选择 vruntime 最小的进程运行
- 通过权重调整不同优先级进程的 vruntime 增长速度

---

## 一、CFS 设计理念

### 理想的公平调度

在理想情况下，n 个进程应该同时运行，每个进程获得 1/n 的 CPU 时间。但由于 CPU 不能真正并行运行多个进程，CFS 使用虚拟运行时间来模拟这种公平性。

### 虚拟运行时间

```
虚拟运行时间 = 实际运行时间 × (NICE_0_WEIGHT / 进程权重)

其中：
- NICE_0_WEIGHT = 1024（nice 值为 0 的进程权重）
- 进程权重由 nice 值决定
```

**示例**：
- nice=0 的进程：运行 10ms，vruntime 增加 10ms
- nice=-5 的进程（权重更高）：运行 10ms，vruntime 增加 < 10ms
- nice=5 的进程（权重更低）：运行 10ms，vruntime 增加 > 10ms

### 权重表

```c
// kernel/sched/core.c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};

// 每个 nice 值差异大约是 1.25 倍
// 1024 / 820 ≈ 1.25
// 820 / 655 ≈ 1.25
```

---

## 二、CFS 运行队列

### 红黑树结构

CFS 使用红黑树来管理就绪进程，按 vruntime 排序：

```c
// kernel/sched/sched.h
struct cfs_rq {
    // 红黑树根节点（带缓存的最左节点）
    struct rb_root_cached   tasks_timeline;
    
    // 最小 vruntime
    u64                     min_vruntime;
    
    // 当前运行的调度实体
    struct sched_entity     *curr;
    
    // ...
};
```

### 红黑树示意图

```
                    红黑树（按 vruntime 排序）
                              
                           (30)
                          /    \
                       (20)    (40)
                       /  \      \
                    (10)  (25)   (50)
                     ↑
               最左节点（vruntime 最小）
               下一个运行的进程
```

### 入队操作

```c
// kernel/sched/fair.c
static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
    bool curr = cfs_rq->curr == se;
    
    // 1. 如果是新进程，初始化 vruntime
    if (renorm && !curr)
        se->vruntime += cfs_rq->min_vruntime;
    
    // 2. 更新负载统计
    update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH);
    update_cfs_group(se);
    
    // 3. 插入红黑树
    if (!curr)
        __enqueue_entity(cfs_rq, se);
    
    se->on_rq = 1;
    
    // 4. 更新 cfs_rq 统计
    add_nr_running(rq_of(cfs_rq), 1);
}

// 插入红黑树
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_root.rb_node;
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    bool leftmost = true;
    
    // 查找插入位置
    while (*link) {
        parent = *link;
        entry = rb_entry(parent, struct sched_entity, run_node);
        
        if (entity_before(se, entry)) {
            link = &parent->rb_left;
        } else {
            link = &parent->rb_right;
            leftmost = false;
        }
    }
    
    // 插入节点
    rb_link_node(&se->run_node, parent, link);
    rb_insert_color_cached(&se->run_node, &cfs_rq->tasks_timeline, leftmost);
}
```

### 出队操作

```c
// kernel/sched/fair.c
static void dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    // 1. 更新运行时间
    update_curr(cfs_rq);
    
    // 2. 更新负载
    update_load_avg(cfs_rq, se, UPDATE_TG);
    
    // 3. 从红黑树移除
    if (se != cfs_rq->curr)
        __dequeue_entity(cfs_rq, se);
    
    se->on_rq = 0;
    
    // 4. 更新 min_vruntime
    update_min_vruntime(cfs_rq);
    
    // 5. 更新统计
    sub_nr_running(rq_of(cfs_rq), 1);
}
```

---

## 三、vruntime 计算

### update_curr() - 更新当前进程

```c
// kernel/sched/fair.c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;
    
    if (unlikely(!curr))
        return;
    
    // 计算实际运行时间
    delta_exec = now - curr->exec_start;
    if (unlikely((s64)delta_exec <= 0))
        return;
    
    curr->exec_start = now;
    
    // 更新累计运行时间
    curr->sum_exec_runtime += delta_exec;
    
    // 更新 vruntime
    curr->vruntime += calc_delta_fair(delta_exec, curr);
    
    // 更新 min_vruntime
    update_min_vruntime(cfs_rq);
    
    // 更新调度统计
    account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

### calc_delta_fair() - 计算 vruntime 增量

```c
// kernel/sched/fair.c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
    // 如果权重等于 NICE_0_WEIGHT，直接返回
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    
    return delta;
}

// 通用计算：delta_exec * weight / lw->weight
static u64 __calc_delta(u64 delta_exec, unsigned long weight,
                        struct load_weight *lw)
{
    u64 fact = scale_load_down(weight);
    int shift = WMULT_SHIFT;
    
    // 使用乘法逆元优化除法
    __update_inv_weight(lw);
    
    if (unlikely(fact >> 32)) {
        while (fact >> 32) {
            fact >>= 1;
            shift--;
        }
    }
    
    // delta_exec * weight / lw->weight
    fact = mul_u64_u32_shr(delta_exec, fact, shift);
    fact = mul_u64_u32_shr(fact, lw->inv_weight, WMULT_SHIFT);
    
    return fact;
}
```

### vruntime 计算示例

```
假设：
- 进程 A：nice = 0，权重 = 1024
- 进程 B：nice = 5，权重 = 335

A 运行 10ms：
  vruntime_A += 10ms × (1024/1024) = 10ms

B 运行 10ms：
  vruntime_B += 10ms × (1024/335) ≈ 30.6ms

结果：
- B 的 vruntime 增长更快
- 调度器更倾向于运行 A
- A 获得更多 CPU 时间
```

### update_min_vruntime() - 更新最小 vruntime

```c
// kernel/sched/fair.c
static void update_min_vruntime(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    struct rb_node *leftmost = rb_first_cached(&cfs_rq->tasks_timeline);
    
    u64 vruntime = cfs_rq->min_vruntime;
    
    if (curr) {
        if (curr->on_rq)
            vruntime = curr->vruntime;
        else
            curr = NULL;
    }
    
    if (leftmost) {
        struct sched_entity *se = rb_entry(leftmost, struct sched_entity, run_node);
        
        if (!curr)
            vruntime = se->vruntime;
        else
            vruntime = min_vruntime(vruntime, se->vruntime);
    }
    
    // min_vruntime 只能单调递增
    cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, vruntime);
}
```

---

## 四、调度决策

### pick_next_task_fair() - 选择下一个进程

```c
// kernel/sched/fair.c
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;
    struct task_struct *p;
    
    if (!cfs_rq->nr_running)
        return NULL;
    
    do {
        // 选择 vruntime 最小的调度实体
        se = pick_next_entity(cfs_rq, NULL);
        
        // 设置为当前调度实体
        set_next_entity(cfs_rq, se);
        
        // 处理组调度
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);
    
    p = task_of(se);
    return p;
}
```

### pick_next_entity() - 选择调度实体

```c
// kernel/sched/fair.c
static struct sched_entity *pick_next_entity(struct cfs_rq *cfs_rq,
                                             struct sched_entity *curr)
{
    // 获取红黑树最左节点（vruntime 最小）
    struct sched_entity *left = __pick_first_entity(cfs_rq);
    struct sched_entity *se;
    
    // 如果没有其他实体，返回当前实体
    if (!left || (curr && entity_before(curr, left)))
        left = curr;
    
    se = left;
    
    // 检查是否有 skip、next、last 实体需要处理
    if (cfs_rq->skip == se) {
        struct sched_entity *second;
        
        if (se == curr) {
            second = __pick_first_entity(cfs_rq);
        } else {
            second = __pick_next_entity(se);
            if (!second || (curr && entity_before(curr, second)))
                second = curr;
        }
        
        if (second && wakeup_preempt_entity(second, left) < 1)
            se = second;
    }
    
    if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1)
        se = cfs_rq->last;
    
    if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1)
        se = cfs_rq->next;
    
    return se;
}
```

---

## 五、时间片计算

### 调度周期

```c
// kernel/sched/fair.c
// 调度周期（所有进程运行一遍的时间）
unsigned int sysctl_sched_latency            = 24000000ULL;  // 24ms
unsigned int sysctl_sched_min_granularity    = 3000000ULL;   // 3ms
unsigned int sysctl_sched_wakeup_granularity = 4000000ULL;   // 4ms
```

### sched_slice() - 计算时间片

```c
// kernel/sched/fair.c
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);
    
    // 按权重分配时间片
    for_each_sched_entity(se) {
        struct load_weight *load;
        struct load_weight lw;
        
        cfs_rq = cfs_rq_of(se);
        load = &cfs_rq->load;
        
        if (unlikely(!se->on_rq)) {
            lw = cfs_rq->load;
            update_load_add(&lw, se->load.weight);
            load = &lw;
        }
        
        // 时间片 = 调度周期 × (进程权重 / 总权重)
        slice = __calc_delta(slice, se->load.weight, load);
    }
    
    return slice;
}

// 计算调度周期
static u64 __sched_period(unsigned long nr_running)
{
    if (unlikely(nr_running > sched_nr_latency))
        // 进程太多，使用最小粒度
        return nr_running * sysctl_sched_min_granularity;
    else
        // 使用目标延迟
        return sysctl_sched_latency;
}
```

### 时间片示例

```
假设调度周期 = 24ms，有 3 个进程：
- 进程 A：nice = 0，权重 = 1024
- 进程 B：nice = 0，权重 = 1024
- 进程 C：nice = 5，权重 = 335

总权重 = 1024 + 1024 + 335 = 2383

时间片：
- A = 24ms × (1024/2383) ≈ 10.3ms
- B = 24ms × (1024/2383) ≈ 10.3ms
- C = 24ms × (335/2383) ≈ 3.4ms
```

---

## 六、抢占检查

### check_preempt_wakeup() - 唤醒抢占检查

```c
// kernel/sched/fair.c
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
    struct task_struct *curr = rq->curr;
    struct sched_entity *se = &curr->se, *pse = &p->se;
    struct cfs_rq *cfs_rq = task_cfs_rq(curr);
    
    // 如果当前不是 CFS 任务，不抢占
    if (unlikely(curr->sched_class != &fair_sched_class))
        return;
    
    // 如果唤醒的是实时任务，立即抢占
    if (unlikely(p->sched_class != &fair_sched_class))
        goto preempt;
    
    // 更新当前任务的 vruntime
    update_curr(cfs_rq);
    
    // 比较 vruntime
    if (wakeup_preempt_entity(se, pse) == 1)
        goto preempt;
    
    return;

preempt:
    resched_curr(rq);
}

// 判断是否应该抢占
static int wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se)
{
    s64 gran, vdiff = curr->vruntime - se->vruntime;
    
    // se 的 vruntime 更大，不抢占
    if (vdiff <= 0)
        return -1;
    
    // vruntime 差值必须超过阈值才抢占
    gran = wakeup_gran(se);
    if (vdiff > gran)
        return 1;
    
    return 0;
}

// 唤醒粒度
static unsigned long wakeup_gran(struct sched_entity *se)
{
    // 默认 4ms
    unsigned long gran = sysctl_sched_wakeup_granularity;
    
    // 按权重调整
    return calc_delta_fair(gran, se);
}
```

### task_tick_fair() - 时钟 tick 处理

```c
// kernel/sched/fair.c
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;
    
    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se, queued);
    }
}

static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    // 更新运行时间
    update_curr(cfs_rq);
    
    // 更新负载
    update_load_avg(cfs_rq, curr, UPDATE_TG);
    update_cfs_group(curr);
    
    // 检查是否需要重新调度
    if (cfs_rq->nr_running > 1)
        check_preempt_tick(cfs_rq, curr);
}

static void check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    struct sched_entity *se;
    s64 delta;
    
    // 计算理想运行时间
    ideal_runtime = sched_slice(cfs_rq, curr);
    
    // 实际运行时间
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    
    // 如果超过理想时间，需要重新调度
    if (delta_exec > ideal_runtime) {
        resched_curr(rq_of(cfs_rq));
        return;
    }
    
    // 如果运行时间不足最小粒度，不调度
    if (delta_exec < sysctl_sched_min_granularity)
        return;
    
    // 比较 vruntime
    se = __pick_first_entity(cfs_rq);
    if (!se)
        return;
    
    delta = curr->vruntime - se->vruntime;
    if (delta < 0)
        return;
    
    if (delta > ideal_runtime)
        resched_curr(rq_of(cfs_rq));
}
```

---

## 七、组调度（Group Scheduling）

### 概述

组调度允许将进程分组，组内进程共享 CPU 时间：

```
                    CPU 时间
                       │
           ┌───────────┼───────────┐
           │           │           │
        组 A (50%)   组 B (30%)   组 C (20%)
           │           │           │
       ┌───┼───┐   ┌───┼───┐      │
       │   │   │   │   │   │      │
      P1  P2  P3   P4  P5  P6     P7
      
组 A 内的 P1、P2、P3 平分 50% 的 CPU
```

### 任务组结构

```c
// kernel/sched/sched.h
struct task_group {
    // cgroup 相关
    struct cgroup_subsys_state css;
    
#ifdef CONFIG_FAIR_GROUP_SCHED
    // 每 CPU 的调度实体
    struct sched_entity **se;
    // 每 CPU 的 CFS 运行队列
    struct cfs_rq **cfs_rq;
    
    // 权重（份额）
    unsigned long shares;
#endif
    
#ifdef CONFIG_RT_GROUP_SCHED
    struct sched_rt_entity **rt_se;
    struct rt_rq **rt_rq;
    struct rt_bandwidth rt_bandwidth;
#endif
    
    // 父组
    struct task_group *parent;
    struct list_head siblings;
    struct list_head children;
    
    // ...
};
```

### 组调度示意图

```
                    根 CFS 运行队列
                    ┌─────────────┐
                    │  红黑树     │
                    │  ├─ se_A    │ ← 组 A 的调度实体
                    │  ├─ se_B    │ ← 组 B 的调度实体
                    │  └─ task_X  │ ← 普通任务
                    └─────────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
    ┌───────▼───────┐           ┌───────▼───────┐
    │  组 A 的      │           │  组 B 的      │
    │  CFS 运行队列 │           │  CFS 运行队列 │
    │  ├─ task1     │           │  ├─ task3     │
    │  └─ task2     │           │  └─ task4     │
    └───────────────┘           └───────────────┘
```

### 使用 cgroup 配置组调度

```bash
# 创建 cgroup
mkdir /sys/fs/cgroup/cpu/groupA
mkdir /sys/fs/cgroup/cpu/groupB

# 设置 CPU 份额
echo 1024 > /sys/fs/cgroup/cpu/groupA/cpu.shares
echo 512 > /sys/fs/cgroup/cpu/groupB/cpu.shares
# groupA 获得 2/3 CPU，groupB 获得 1/3 CPU

# 将进程加入组
echo <pid> > /sys/fs/cgroup/cpu/groupA/cgroup.procs
```

---

## 八、CFS 带宽控制

### 概念

CFS 带宽控制限制一组进程在给定周期内的最大 CPU 使用量。

```c
// kernel/sched/sched.h
struct cfs_bandwidth {
    raw_spinlock_t lock;
    ktime_t period;              // 周期
    u64 quota;                   // 配额（每周期可使用的时间）
    u64 runtime;                 // 剩余运行时间
    s64 hierarchical_quota;      // 层级配额
    
    u8 idle;
    u8 period_active;
    u8 distribute_running;
    u8 slack_started;
    
    struct hrtimer period_timer;
    struct hrtimer slack_timer;
    // ...
};
```

### 配置带宽控制

```bash
# 设置周期（微秒）
echo 100000 > /sys/fs/cgroup/cpu/groupA/cpu.cfs_period_us  # 100ms

# 设置配额（微秒）
echo 50000 > /sys/fs/cgroup/cpu/groupA/cpu.cfs_quota_us    # 50ms

# groupA 每 100ms 只能使用 50ms CPU（50%）
```

---

## 总结

### 核心要点

1. **CFS 设计理念**：
   - 完全公平：通过 vruntime 模拟公平分配
   - 权重决定 vruntime 增长速度

2. **vruntime 计算**：
   - vruntime = 实际运行时间 × (NICE_0_WEIGHT / 进程权重)
   - 权重高的进程 vruntime 增长慢

3. **红黑树**：
   - 按 vruntime 排序
   - 总是选择最左节点（vruntime 最小）

4. **时间片**：
   - 时间片 = 调度周期 × (进程权重 / 总权重)
   - 权重高的进程获得更多时间

5. **组调度**：
   - 进程分组，组内共享 CPU 时间
   - 支持 cgroup 配置

### 后续学习

- [进程优先级与实时调度](10-进程优先级与实时调度.md) - 理解优先级机制
- [多核调度与负载均衡](11-多核调度与负载均衡.md) - 理解多核调度

## 参考资源

- 内核源码：`kernel/sched/fair.c`
- 内核文档：`Documentation/scheduler/sched-design-CFS.rst`

## 更新记录

- 2026-01-27：初始创建，包含 CFS 调度器详解
