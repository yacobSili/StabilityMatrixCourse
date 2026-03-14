# LMKD 进阶

## 📋 学习目标

深入理解 LMKD 的源码实现和高级机制，掌握 PSI 监控、高级配置和调优方法。

## 📚 学习内容

### 第一章：LMKD 源码深度解析

#### 1.1 LMKD 源码结构

**源码位置**：`system/core/lmkd/`

**主要文件**：

```
lmkd/
├── lmkd.c              # 主程序入口
├── lmkd.h              # 头文件
├── lmkd_main.c         # 主循环和事件处理
├── lmkd_psi.c          # PSI 监控实现
├── lmkd_vmpressure.c   # vmpressure 监控实现（已废弃）
├── lmkd_utils.c        # 工具函数
└── Android.bp          # 构建文件
```

**关键数据结构**：

```c
// 进程信息结构
struct proc {
    pid_t pid;                    // 进程 ID
    uid_t uid;                    // 用户 ID
    char *name;                   // 进程名称
    int oom_score_adj;            // OOM 调整值
    struct proc *next;            // 链表指针
};

// 内存压力事件
struct psi_event {
    enum psi_stall_type stall_type;  // 压力类型（some/full）
    uint64_t threshold_us;           // 阈值（微秒）
    uint64_t threshold_ms;            // 阈值（毫秒）
};
```

#### 1.2 LMKD 主循环

**主循环流程**（简化版）：

```c
int main(int argc, char *argv[]) {
    // 1. 初始化
    init_monitors();           // 初始化监控器
    init_proc_pidfd();         // 初始化进程文件描述符
    init_mp_common();          // 初始化内存压力通用模块
    
    // 2. 设置调度策略
    set_scheduler();           // 设置为 SCHED_FIFO
    
    // 3. 进入事件循环
    while (1) {
        // 等待内存压力事件
        wait_for_event();
        
        // 处理事件
        handle_memory_pressure();
    }
    
    return 0;
}
```

**关键函数**：

1. **`init_monitors()`**：初始化 PSI 或 vmpressure 监控
2. **`wait_for_event()`**：等待内存压力事件
3. **`handle_memory_pressure()`**：处理内存压力事件

#### 1.3 PSI 监控实现

**PSI 监控初始化**：

```c
// lmkd_psi.c
int init_psi_monitor(void) {
    int fd;
    
    // 打开 PSI 接口
    fd = open("/proc/pressure/memory", O_RDONLY);
    if (fd < 0) {
        return -1;
    }
    
    // 设置压力阈值
    set_psi_thresholds(fd);
    
    return fd;
}
```

**PSI 事件监听**：

```c
// 监听 PSI 事件
int wait_for_psi_event(int fd) {
    struct pollfd pfd;
    char buf[256];
    ssize_t len;
    
    pfd.fd = fd;
    pfd.events = POLLPRI;  // 高优先级事件
    
    // 等待事件
    if (poll(&pfd, 1, -1) < 0) {
        return -1;
    }
    
    // 读取事件数据
    len = read(fd, buf, sizeof(buf) - 1);
    if (len < 0) {
        return -1;
    }
    
    buf[len] = '\0';
    
    // 解析事件
    return parse_psi_event(buf);
}
```

**PSI 阈值设置**：

```c
// 设置 PSI 阈值
int set_psi_thresholds(int fd) {
    char buf[64];
    int ret;
    
    // 设置 some 压力阈值（10秒窗口，5%阈值）
    snprintf(buf, sizeof(buf), "some 10000 5\n");
    ret = write(fd, buf, strlen(buf));
    
    // 设置 full 压力阈值（60秒窗口，10%阈值）
    snprintf(buf, sizeof(buf), "full 60000 10\n");
    ret = write(fd, buf, strlen(buf));
    
    return ret;
}
```

#### 1.4 进程选择算法

**进程选择流程**：

```c
// 选择要终止的进程
struct proc *select_target_process(int min_score_adj, 
                                   long min_size) {
    struct proc *proc;
    struct proc *selected = NULL;
    long max_size = 0;
    
    // 遍历所有进程
    for (proc = proc_list; proc != NULL; proc = proc->next) {
        // 检查 oom_score_adj
        if (proc->oom_score_adj < min_score_adj) {
            continue;
        }
        
        // 获取进程内存使用
        long size = get_proc_size(proc->pid);
        if (size < min_size) {
            continue;
        }
        
        // 选择内存使用最大的进程
        if (ro.lmk.kill_heaviest_task) {
            if (size > max_size) {
                max_size = size;
                selected = proc;
            }
        } else {
            // 选择第一个符合条件的进程
            selected = proc;
            break;
        }
    }
    
    return selected;
}
```

**内存使用计算**：

```c
// 获取进程内存使用（PSS）
long get_proc_size(pid_t pid) {
    char path[64];
    FILE *fp;
    long pss = 0;
    
    // 读取 /proc/<pid>/smaps_rollup
    snprintf(path, sizeof(path), "/proc/%d/smaps_rollup", pid);
    fp = fopen(path, "r");
    if (!fp) {
        return 0;
    }
    
    // 解析 PSS 值
    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "Pss:")) {
            sscanf(line, "Pss: %ld", &pss);
            break;
        }
    }
    
    fclose(fp);
    return pss;
}
```

#### 1.5 进程终止实现

**发送 SIGKILL**：

```c
// 终止进程
int kill_target_process(struct proc *proc) {
    int ret;
    
    // 记录日志
    ALOGI("Killing '%s' (pid %d), adj %d, "
          "to free %ldKB, reason: %s",
          proc->name, proc->pid, proc->oom_score_adj,
          get_proc_size(proc->pid), kill_reason);
    
    // 发送 SIGKILL
    ret = kill(proc->pid, SIGKILL);
    if (ret < 0) {
        ALOGE("kill(%d): %s", proc->pid, strerror(errno));
        return -1;
    }
    
    // 等待进程终止
    wait_for_process_exit(proc->pid);
    
    return 0;
}
```

---

### 第二章：PSI (Pressure Stall Information) 深度解析

#### 2.1 PSI 基本概念

**什么是 PSI？**

PSI (Pressure Stall Information) 是 Linux 内核从 4.20 版本开始引入的资源压力监控机制，用于量化系统资源（CPU、内存、IO）压力对任务执行的影响。

**PSI 的核心思想**：
- 测量**压力停滞时间**（Pressure Stall Time）
- 即由于资源不足导致任务无法执行的时间
- 提供可操作的性能指标

**PSI 的优势**：

1. **量化影响**：直接测量任务被阻塞的时间
2. **预测性**：可以预测即将到来的资源压力
3. **统一接口**：CPU、内存、IO 使用统一的接口

#### 2.2 PSI 内存压力类型

**两种压力类型**：

1. **some 压力（部分压力）**
   - 至少有一个任务因为内存不足而被阻塞
   - 表示系统正在经历内存压力
   - 但可能还有部分任务可以运行

2. **full 压力（完全压力）**
   - 所有任务都因为内存不足而被阻塞
   - 表示系统严重内存不足
   - 所有任务都无法执行

**PSI 输出示例**：

```bash
$ cat /proc/pressure/memory
some avg10=5.23 avg60=2.45 avg300=1.23 total=12345678
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**输出解读**：
- `some avg10=5.23`：过去 10 秒平均 5.23% 的时间有任务被阻塞
- `avg60/300`：过去 60/300 秒的平均值
- `total`：总阻塞时间（微秒）

#### 2.3 PSI 触发机制

**PSI 在以下情况记录内存压力**：

1. **Direct Reclaim**
   ```c
   // mm/page_alloc.c
   if (direct_reclaim) {
       psi_mem_stall_start(...);  // 开始记录压力
       // ... 执行回收
       psi_mem_stall_end(...);    // 结束记录
   }
   ```

2. **kswapd 唤醒**
   - kswapd 被唤醒表示内存压力
   - 记录唤醒到完成的时间

3. **OOM 相关**
   - OOM Killer 触发前的压力积累
   - 内存分配失败导致的阻塞

**PSI 阈值触发**：

```c
// 设置阈值后，当压力超过阈值时触发事件
echo "some 10000 5" > /proc/pressure/memory
// 含义：10秒窗口内，如果 some 压力超过 5%，触发事件
```

#### 2.4 PSI 与 vmpressure 的对比

**vmpressure（已废弃）**：

- 基于内存回收效率（scan/reclaim 比例）
- 触发条件：`scan/reclaim > threshold`
- 接口：`/dev/memcg`（需要 cgroup）
- 精度：较低，基于回收统计

**PSI（现代方案）**：

- 基于任务阻塞时间
- 触发条件：任务阻塞时间超过阈值
- 接口：`/proc/pressure/memory`
- 精度：高，直接测量影响

**迁移原因**：
- PSI 更准确地反映用户体验影响
- PSI 不需要 cgroup 支持
- PSI 提供更细粒度的监控

---

### 第三章：高级配置和调优

#### 3.1 PSI 阈值调优

**关键配置属性**：

| 属性 | 说明 | 默认值（Go） | 默认值（Non-Go） | 调优建议 |
|------|------|------------|----------------|---------|
| `ro.lmk.psi_partial_stall_ms` | PSI some 阈值 | `70` | `70` | 提高可减少杀进程频率 |
| `ro.lmk.psi_complete_stall_ms` | PSI full 阈值 | `500` | `150` | 降低可更快响应严重压力 |
| `ro.lmk.thrashing_limit` | 页面缓存抖动限制 | `30` | `100` | 降低可更早检测抖动 |
| `ro.lmk.thrashing_limit_decay` | 抖动阈值衰减 | `50` | `10` | 提高可更快恢复 |

**调优策略**：

1. **减少杀进程频率**
   ```bash
   # 提高 PSI some 阈值，给系统更多缓冲时间
   setprop ro.lmk.psi_partial_stall_ms 100
   ```

2. **更快响应严重压力**
   ```bash
   # 降低 PSI full 阈值，更快触发严重压力处理
   setprop ro.lmk.psi_complete_stall_ms 300
   ```

3. **更早检测页面缓存抖动**
   ```bash
   # 降低抖动限制，更早检测和响应抖动
   setprop ro.lmk.thrashing_limit 20
   ```

#### 3.2 杀进程策略调优

**kill_heaviest_task 策略**：

```bash
# 启用：优先杀内存占用最大的进程（更有效）
setprop ro.lmk.kill_heaviest_task true

# 禁用：快速杀第一个符合条件的进程（更快响应）
setprop ro.lmk.kill_heaviest_task false
```

**策略对比**：

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **kill_heaviest_task=true** | 释放更多内存，更有效 | 选择时间稍长 | 内存压力持续 |
| **kill_heaviest_task=false** | 响应更快 | 可能释放内存较少 | 需要快速响应 |

#### 3.3 进程优先级调整

**调整进程的 oom_score_adj**：

```bash
# 查看进程的 oom_score_adj
adb shell cat /proc/<pid>/oom_score_adj

# 修改进程的 oom_score_adj（需要 root）
adb root
echo 100 > /proc/<pid>/oom_score_adj

# 通过 ActivityManager 调整（应用层）
# 使用 setProcessImportance() 或 setOomAdj()
```

**优先级调整场景**：

1. **保护关键进程**
   ```bash
   # 将关键进程设置为 -1000（永不杀死）
   echo -1000 > /proc/<critical_pid>/oom_score_adj
   ```

2. **优先清理特定应用**
   ```bash
   # 将应用设置为高优先级（更容易被杀死）
   echo 1000 > /proc/<app_pid>/oom_score_adj
   ```

#### 3.4 与内核 kswapd 的协同

**kswapd 与 LMKD 的关系**：

```
内存压力
    ↓
kswapd 尝试异步回收
    ↓
回收速度 < 内存消耗速度？
    ├─ 是 → PSI 压力上升 → LMKD 介入
    └─ 否 → 压力缓解 → 继续监控
```

**调优建议**：

1. **给 kswapd 更多时间**
   - 提高 PSI 阈值，避免过早触发 LMKD
   - 让 kswapd 有更多机会回收内存

2. **监控 kswapd 活动**
   ```bash
   # 查看 kswapd 活动
   adb shell cat /proc/vmstat | grep kswapd
   
   # 查看 kswapd CPU 使用
   adb shell top -H | grep kswapd
   ```

3. **平衡回收策略**
   - 如果 kswapd 频繁唤醒，考虑调整 `swappiness`
   - 如果 kswapd 无法缓解压力，LMKD 需要更积极

---

### 第四章：内存压力评估和响应

#### 4.1 内存压力等级评估

**压力等级判断逻辑**：

```c
// 简化版压力等级判断
enum pressure_level {
    PRESSURE_LOW,
    PRESSURE_MEDIUM,
    PRESSURE_CRITICAL
};

enum pressure_level evaluate_pressure(uint64_t psi_some,
                                      uint64_t psi_full) {
    // 严重压力：full 压力超过阈值
    if (psi_full > critical_threshold) {
        return PRESSURE_CRITICAL;
    }
    
    // 中等压力：some 压力超过中等阈值
    if (psi_some > medium_threshold) {
        return PRESSURE_MEDIUM;
    }
    
    // 低压力：some 压力超过低阈值
    if (psi_some > low_threshold) {
        return PRESSURE_LOW;
    }
    
    return PRESSURE_NONE;
}
```

**响应策略**：

| 压力等级 | 响应策略 | 目标进程 | 预期释放内存 |
|---------|---------|---------|------------|
| **Low** | 轻度清理 | adj ≥ 900 | 10-20MB |
| **Medium** | 中度清理 | adj ≥ 800 | 50-100MB |
| **Critical** | 积极清理 | adj ≥ 700 | 200MB+ |

#### 4.2 页面缓存抖动检测

**什么是页面缓存抖动？**

页面缓存抖动（Thrashing）是指系统频繁地从存储设备读取页面到内存，然后又写回存储设备，导致大量 IO 操作。

**抖动检测机制**：

```c
// 检测页面缓存抖动
bool detect_thrashing(void) {
    long refault_count = get_workingset_refaults();
    long total_cache = get_file_cache_size();
    long threshold = (total_cache * thrashing_limit) / 100;
    
    // 如果 refault 数量超过阈值，认为发生抖动
    if (refault_count > threshold) {
        return true;
    }
    
    return false;
}
```

**抖动响应**：

- 当检测到抖动时，LMKD 会更积极地清理进程
- 抖动阈值会衰减，允许系统恢复
- 衰减速度由 `ro.lmk.thrashing_limit_decay` 控制

#### 4.3 内存压力预测

**基于 PSI 趋势预测**：

```c
// 预测内存压力趋势
bool predict_pressure_increase(uint64_t psi_avg10,
                                uint64_t psi_avg60) {
    // 如果短期压力远高于长期压力，可能继续上升
    if (psi_avg10 > psi_avg60 * 1.5) {
        return true;
    }
    
    return false;
}
```

**预防性清理**：

- 当预测到压力会上升时，可以提前清理一些进程
- 避免等到严重压力时才采取行动
- 提供更平滑的用户体验

---

### 第五章：高级调试技巧

#### 5.1 启用详细日志

**LMKD 日志级别**：

```bash
# 查看当前日志级别
adb logcat -s lmkd

# 启用详细日志（需要修改源码或使用调试版本）
# 在 lmkd.c 中设置日志级别
```

**关键日志点**：

1. **PSI 事件接收**
   ```
   lmkd: PSI event: some avg10=5.23 threshold=5.00
   ```

2. **进程选择**
   ```
   lmkd: Selected process 'com.example.app' (pid 1234, adj 900, size 50MB)
   ```

3. **进程终止**
   ```
   lmkd: Killing 'com.example.app' (pid 1234), adj 900, to free 50MB
   ```

#### 5.2 性能分析

**LMKD 响应时间分析**：

```bash
# 监控从 PSI 事件到进程终止的时间
# 需要在源码中添加时间戳记录
```

**内存释放效果分析**：

```bash
# 记录杀进程前后的内存使用
adb shell dumpsys meminfo > before.txt
# ... 等待 LMKD 杀进程 ...
adb shell dumpsys meminfo > after.txt
diff before.txt after.txt
```

#### 5.3 压力测试

**模拟内存压力**：

```bash
# 使用 stress-ng 模拟内存压力
adb shell stress-ng --vm 4 --vm-bytes 2G --timeout 60s

# 观察 LMKD 响应
adb logcat | grep lmkd
```

**监控 PSI 变化**：

```bash
# 实时监控 PSI
watch -n 1 'adb shell cat /proc/pressure/memory'
```

---

## 📝 学习检查点

完成进阶学习后，您应该能够：

- [ ] 理解 LMKD 源码结构和实现
- [ ] 掌握 PSI 监控机制和实现
- [ ] 能够进行高级配置和调优
- [ ] 理解与内核 kswapd 的协同机制
- [ ] 能够评估内存压力等级
- [ ] 能够检测页面缓存抖动
- [ ] 能够进行高级调试和性能分析

---

**下一步**：完成进阶学习后，可以进入 `03_Expert` 学习专家级内容，掌握复杂问题诊断和深度性能优化。
