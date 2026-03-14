# kworker 与 Workqueue 机制深度解析

## 📋 文档概述

在 Android 系统性能分析（如 Systrace/Perfetto）或 `top` 命令中，我们经常看到以 `kworker` 开头的进程。它们无处不在，有时会占用大量 CPU，有时会陷入 D 状态导致系统卡顿。本文档将深入解析 `kworker` 的本质——Linux 内核的 **Workqueue（工作队列）** 机制，从原理、命名规则、源码实现到问题定位，帮助开发者彻底掌握这一核心机制。

## 🎯 学习目标

- 理解 `kworker` 的本质和设计目的
- 掌握 Workqueue 的核心概念（Bound/Unbound, CMWQ）
- 能够解读 `kworker` 的进程命名（如 `kworker/u16:0`）
- 掌握定位 `kworker` 高 CPU 或 D 状态根因的方法
- 理解 Workqueue 在系统稳定性中的角色

---

## 第一章：kworker 是什么？

### 1.1 定义与本质

**kworker**（Kernel Worker）是 Linux 内核中用于执行 **Workqueue（工作队列）** 中排队任务的内核线程。

- **本质**：它是内核线程（Kernel Thread），运行在内核态。
- **职责**：处理内核中那些需要**异步执行**、**允许睡眠**的任务。
- **与 Softirq/Tasklet 的区别**：
    - Softirq/Tasklet 运行在中断上下文，**不能睡眠**，必须快速执行。
    - Workqueue（即 kworker）运行在进程上下文，**可以睡眠**（例如进行 IO 操作、获取互斥锁）。

### 1.2 为什么需要 kworker？

内核中有大量任务不需要立即在中断处理函数中完成，或者需要进行可能引起睡眠的操作（如读写磁盘）。这些任务被设计为"下半部"（Bottom Half）机制。

- **Workqueue 优势**：
    - 提供统一的异步执行框架，驱动开发者无需自己创建和管理内核线程。
    - 自动管理并发，优化 CPU 利用率（通过 CMWQ 机制）。

---

## 第二章：Workqueue 核心架构 (CMWQ)

Linux 2.6.36 引入了 **CMWQ (Concurrency Managed Workqueue)**，彻底改变了工作队列的实现。

### 2.1 核心概念

1.  **Work Item (工作项)**
    - 具体的任务结构体 `struct work_struct`，包含要执行的函数指针。

2.  **Workqueue (工作队列)**
    - 任务的容器。驱动程序把 Work Item 提交到 Workqueue 中。
    - API: `alloc_workqueue()`, `schedule_work()`.

3.  **Worker Pool (工作者池)**
    - 管理一组 `kworker` 线程的池子。
    - CMWQ 根据 CPU 和 NUMA 节点自动维护 Worker Pool。

4.  **Worker (工作者)**
    - 即 `kworker` 线程，负责从池子中取出 Work Item 并执行。

### 2.2 Bound vs Unbound (关键区分)

这是理解 kworker 行为的关键：

| 特性 | Bound Workqueue (绑定) | Unbound Workqueue (非绑定) |
| :--- | :--- | :--- |
| **CPU 亲和性** | **严格绑定到特定 CPU**。在 CPU 0 上提交的任务，必须在 CPU 0 的 kworker 上执行。 | **不绑定特定 CPU**。可以在任何空闲 CPU 上执行，调度器自动负载均衡。 |
| **用途** | 利用 CPU 缓存热度（Cache Locality），适合轻量级任务。 | 适合 CPU 密集型或长时间阻塞的任务，避免阻塞特定 CPU 的其他任务。 |
| **kworker 命名** | `kworker/X:Y` (X 是 CPU 号) | `kworker/uX:Y` (u 代表 Unbound, X 是 Pool ID) |
| **典型标志** | 无 `WQ_UNBOUND` 标志 | 带有 `WQ_UNBOUND` 标志 |

---

## 第三章：解读 kworker 命名规则

在 `ps` 或 `top` 中看到的 kworker 名称包含重要信息，必须学会解读。

### 3.1 命名格式

格式通常为：`kworker/[u]PoolID:WorkerID[H]`

*   **u**: 表示 **Unbound** (非绑定)。如果没有 u，表示 **Bound** (绑定到特定 CPU)。
*   **PoolID**:
    *   对于 Bound 类型：直接代表 **CPU ID**。
    *   对于 Unbound 类型：代表 **Worker Pool ID**。
*   **WorkerID**: 该池子中的第几个 Worker 线程。
*   **H**: 表示 **High Priority** (高优先级)。

### 3.2 实例解析

| 进程名 | 含义解析 | 关键推断 |
| :--- | :--- | :--- |
| `kworker/0:1` | **CPU 0** 上的普通优先级 Worker | 该线程只在 CPU 0 运行。如果它占 100%，CPU 0 会卡死。 |
| `kworker/0:1H` | **CPU 0** 上的**高优先级** Worker | 处理高优先级任务（如 `WQ_HIGHPRI`）。 |
| `kworker/u16:0`| **Unbound** 池子 16 中的 Worker | 不绑定 CPU。可能在任何 CPU 运行。通常用于文件系统、Crypto 等。 |
| `kworker/u16:0H`| **Unbound** 高优先级 Worker | 同上，但优先级更高。 |

**注意**：`u16` 并不代表 CPU 16，它只是一个 Pool ID。要查看 Pool ID 对应的属性，需要查看 `/sys/devices/virtual/workqueue/`（部分信息）或通过 trace。

---

## 第四章：kworker 问题分析与定位

kworker 问题通常表现为两类：**CPU 占用过高** 和 **D 状态（IO 阻塞）**。

### 4.1 场景一：kworker CPU 占用过高

**现象**：系统卡顿，`top` 显示某个 `kworker` 占用大量 CPU。
**难点**：`kworker` 只是一个执行器（壳），我们需要知道**它里面正在执行哪个函数**。

#### 定位方法 1：cat /proc/PID/stack (静态堆栈)

这是最快的方法，查看当前时刻调用栈。

```bash
cat /proc/<kworker_pid>/stack
```

**输出示例**：
```text
[<0>] process_one_work+0x1a8/0x3e0
[<0>] worker_thread+0x50/0x3c8
[<0>] kthread+0x108/0x138
...
```
*分析*：这通常只能看到框架代码。如果运气好，能看到具体的执行函数；如果 kworker 正在通过函数指针执行任务，可能需要结合符号表。

#### 定位方法 2：Workqueue Tracepoints (动态追踪) 🔥推荐

使用 `ftrace` 抓取 `workqueue_execute_start` 事件，这是最精准的方法。

**步骤**：
1.  开启 trace：
    ```bash
    echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_execute_start/enable
    echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_queue_work/enable
    echo 1 > /sys/kernel/debug/tracing/tracing_on
    ```
2.  重现问题（等待几秒）。
3.  关闭 trace 并查看：
    ```bash
    cat /sys/kernel/debug/tracing/trace_pipe
    ```

**Trace 输出解读**：

```text
kworker/u16:1-1234 [001] ... workqueue_execute_start: work struct 0xffff888123456780: function my_driver_work_func
```
*   **function my_driver_work_func**: **这就是真凶！**
*   这行 log 明确告诉你，`kworker/u16:1` 正在执行 `my_driver_work_func` 这个函数。
*   接下来去代码中 grep 这个函数名，找到是哪个驱动提交的。

### 4.2 场景二：kworker 处于 D 状态 (Uninterruptible Sleep)

**现象**：`Load Average` 飙高，系统响应慢，`top` 显示 kworker 状态为 `D`。
**原因**：Worker 线程正在等待 IO（磁盘读写）或 锁。

#### 定位方法：SysRq-w (Blocked Tasks)

如果系统还能响应 shell：

```bash
echo w > /proc/sysrq-trigger
dmesg
```

或者查看 `stack`：

```bash
cat /proc/<kworker_pid>/stack
```

**典型案例分析**：
如果堆栈显示：
```c
io_schedule
blk_mq_get_tag
...
ext4_writepages
process_one_work
```
*分析*：
1.  该 kworker 正在执行文件系统写回（ext4_writepages）。
2.  卡在 `io_schedule`，说明在等待磁盘 IO 完成。
3.  **结论**：这是 IO 瓶颈，不是 CPU 问题。检查磁盘性能或 IO 压力（参考 [IO_Pressure_PSI_Deep_Dive.md](../03_IO_Blocking/IO_Pressure_PSI_Deep_Dive.md)）。

---

## 第五章：系统稳定性中的 kworker

### 5.1 Watchdog (Soft Lockup)

如果一个 Bound kworker（如 `kworker/0:1`）正在执行一个死循环任务，且关闭了抢占，会导致 CPU 0 无法调度其他任务，触发 **Soft Lockup**。

**日志特征**：
```text
watchdog: BUG: soft lockup - CPU#0 stuck for 22s! [kworker/0:1:1234]
RIP: 0010:bad_driver_loop_function+0x20/0x40
```
*   直接定位 `bad_driver_loop_function`。

### 5.2 ANR 分析中的 kworker：实战案例解析

以下是一个真实的 SystemUI ANR 案例，我们通过分析其中的 kworker 来理解系统状态。

#### 案例背景

**ANR 日志关键信息**：
- **IO PSI**: some 78.08%, full 52.78%（极高）
- **iowait**: 21%（极高）
- **Load Average**: 29.23 / 21.68 / 16.36（极高）
- **Memory PSI**: some 6.33%（中等）
- **kswapd CPU**: 42%（活跃）

#### kworker 详细分析

**1. kworker/u16:4+events_unbound (51% CPU) - 最大嫌疑**

```text
51% 25495/kworker/u16:4+events_unbound: 0% user + 51% kernel
```

**解读**：
- **`u16`**: Unbound Workqueue，Pool ID 为 16
- **`events_unbound`**: 这是内核的通用事件处理 Workqueue
- **51% CPU**: 占用极高，说明有大量事件需要处理

**可能在做的事情**：
- 处理文件系统事件（如 inotify 事件）
- 处理设备热插拔事件
- 处理内核模块加载/卸载事件
- 处理其他子系统提交的异步任务

**在这个 ANR 中的角色**：
- 结合 IO PSI 78% 和大量 major faults（41261），很可能是：
  1. **处理文件系统相关事件**：大量文件操作触发事件通知
  2. **处理内存回收相关事件**：kswapd 活跃时可能触发事件
  3. **处理 IO 完成事件**：大量 IO 操作完成后需要处理回调

**定位方法**：
```bash
# 查看该 kworker 的堆栈
cat /proc/25495/stack

# 使用 ftrace 追踪具体执行函数
echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_execute_start/enable
# 等待几秒后查看
cat /sys/kernel/debug/tracing/trace_pipe | grep "kworker/u16:4"
```

**2. kworker/u17:x-blk_crypto_wq (总计约 35% CPU) - IO 加密解密**

```text
6.5% 6453/kworker/u17:4-blk_crypto_wq
7.1% 20248/kworker/u17:5-blk_crypto_wq
7.5% 20723/kworker/u17:6-blk_crypto_wq
7.6% 21759/kworker/u17:7-blk_crypto_wq
4.4% 10768/kworker/u17:1-blk_crypto_wq
2.3% 20246/kworker/u17:0-blk_crypto_wq
```

**解读**：
- **`u17`**: Unbound Workqueue，Pool ID 为 17
- **`blk_crypto_wq`**: **块设备加密/解密 Workqueue**
- **多个 worker**: 说明加密任务积压，需要多个线程并行处理

**在做的事情**：
- **加密写入**：应用写入数据时，如果启用了文件级加密（FBE），需要加密后再写入存储
- **解密读取**：应用读取数据时，需要解密后再返回
- **密钥管理**：处理加密密钥相关的操作

**在这个 ANR 中的角色**：
- **IO 压力的放大器**：
  1. 每个 IO 操作都需要加密/解密
  2. 加密解密是 CPU 密集型操作
  3. 大量 IO → 大量加密任务 → 多个 kworker 并行处理 → CPU 占用高
  4. 加密解密可能阻塞 IO 完成，进一步加剧 IO 压力

**为什么有这么多 blk_crypto_wq？**
- Android 设备通常启用 FBE（File-Based Encryption）
- 大量文件操作（读取、写入）都需要加密/解密
- 任务积压导致内核创建多个 worker 并行处理

**3. kworker/2:3-events (2.3% CPU) - Bound 事件处理**

```text
2.3% 20130/kworker/2:3-events: 0% user + 2.3% kernel
```

**解读**：
- **`2`**: 绑定到 **CPU 2**
- **`events`**: Bound 版本的 events Workqueue
- **3**: 该 CPU 上的第 3 个 worker

**在做的事情**：
- 处理绑定到 CPU 2 的事件任务
- 与 `events_unbound` 类似，但任务必须在该 CPU 上执行

**4. kworker/1:48H-kblockd (2.6% CPU) - 块设备处理**

```text
2.6% 22091/kworker/1:48H-kblockd: 0% user + 2.6% kernel
3.1% 25424/kworker/3:46H-kblockd: 0% user + 3.1% kernel
```

**解读**：
- **`1`、`3`**: 分别绑定到 CPU 1 和 CPU 3
- **`H`**: **高优先级**（High Priority）
- **`kblockd`**: **块设备处理 Workqueue**

**在做的事情**：
- 处理块设备的 IO 请求
- 处理 IO 调度、合并、完成回调
- 处理块设备驱动相关任务

**在这个 ANR 中的角色**：
- 结合 IO PSI 78%，说明块设备 IO 压力极大
- kblockd 正在处理大量 IO 请求
- 高优先级（H）说明这些任务比较紧急

**5. kworker/5:21H-kverityd (3.1% CPU) - 验证设备**

```text
3.1% 24002/kworker/5:21H-kverityd: 0% user + 3.1% kernel
3.2% 25457/kworker/0:40H-kverityd: 0% user + 3.2% kernel
```

**解读**：
- **`5`、`0`**: 分别绑定到 CPU 5 和 CPU 0
- **`H`**: 高优先级
- **`kverityd`**: **dm-verity 验证设备 Workqueue**

**在做的事情**：
- Android 使用 dm-verity 验证系统分区完整性
- 读取系统分区时，需要验证数据完整性
- 验证失败会阻止读取（安全机制）

**在这个 ANR 中的角色**：
- 大量文件读取操作触发验证
- 验证是 CPU 密集型操作
- 虽然占用不高，但也是 IO 路径上的开销

#### F2FS、kworker、kswapd 之间的真实关系（基于事实）

**重要说明**：以下分析基于 Linux 内核的实际机制，避免强行关联。

##### 1. kswapd 与 F2FS 的关系（事实）

**事实 1：kswapd 回收文件页的机制**

```c
// kswapd 回收文件页的流程（源码：mm/vmscan.c）
kswapd 被唤醒
    ↓
扫描 LRU 列表（包括 inactive_file）
    ↓
选择可回收的文件页
    ↓
如果页面是脏页（dirty），需要先写回
    ↓
调用 address_space_operations.writepage()
    ↓
对于 F2FS：调用 f2fs_write_data_page()
    ↓
F2FS 写回数据到存储设备
```

**关键事实**：
- kswapd **直接调用**文件系统的 `writepage` 函数
- 对于 F2FS，会调用 `f2fs_write_data_page()`
- 这是**同步调用**，kswapd 会等待写回完成

**事实 2：kswapd 与 F2FS 的间接关系**

```
内存压力
    ↓
kswapd 被唤醒
    ↓
回收 F2FS 的 Page Cache（文件页）
    ↓
如果是脏页，触发 F2FS 写回
    ↓
F2FS 写回产生 IO
```

**关键点**：
- kswapd **不直接操作** F2FS
- kswapd 只是回收 Page Cache 中的页面
- 如果页面属于 F2FS 文件，写回时会调用 F2FS 的函数
- 这是**间接关系**：kswapd → Page Cache → F2FS

##### 2. kswapd 与 kworker 的关系（事实）

**事实 1：kswapd 本身不是 kworker**

- kswapd 是**独立的内核线程**（kernel thread）
- kswapd **不使用** workqueue 机制
- kswapd 的入口函数是 `kswapd()`，不是通过 workqueue 执行

**事实 2：kswapd 可能间接触发 kworker**

```
kswapd 回收文件页
    ↓
触发文件系统写回（如 F2FS）
    ↓
文件系统写回可能使用 workqueue（需要查证）
    ↓
触发 kworker 执行写回任务
```

**关键点**：
- kswapd **不直接**使用 kworker
- 但 kswapd 触发的写回操作**可能**使用 workqueue
- 这取决于文件系统的实现（需要查证 F2FS 是否使用 workqueue 进行写回）

**需要查证的事实**：
- F2FS 的写回是否使用 workqueue？
- 还是直接同步执行？

##### 3. F2FS 与 kworker 的关系（事实）

**事实 1：F2FS 的 IO 操作经过块设备层**

```
F2FS 文件操作
    ↓
VFS 层
    ↓
F2FS 文件系统层
    ↓
块设备层（Block I/O）
    ↓
存储设备驱动
```

**事实 2：块设备层的某些操作使用 workqueue**

- **blk_crypto_wq**：块设备加密/解密（如果启用 FBE）
- **kblockd**：块设备 IO 处理
- 这些是**块设备层**的机制，不是 F2FS 直接创建的

**关键点**：
- F2FS **不直接**创建这些 kworker
- F2FS 的 IO 操作**经过**块设备层
- 块设备层的操作**可能**使用 workqueue
- 这是**间接关系**：F2FS → 块设备层 → workqueue

**事实 3：F2FS 本身可能使用 workqueue（需要查证）**

- F2FS 的某些操作（如 GC、检查点更新）可能使用 workqueue
- 但这需要查看 F2FS 源码确认

##### 4. 在这个 ANR 案例中的真实关系

**基于日志的事实**：

1. **kswapd CPU 42%**：
   - kswapd 正在活跃工作
   - 说明内存回收正在进行

2. **大量 major faults（41261）**：
   - 说明有大量文件读取操作
   - 这些读取可能来自 F2FS 分区

3. **IO PSI 78%**：
   - 极高的 IO 压力
   - 可能来自多个源头（F2FS 写回、swap IO、其他 IO）

4. **多个 blk_crypto_wq worker**：
   - 说明有大量加密任务
   - 这些任务来自块设备层，不是 F2FS 直接创建

**可能的关联（推测，需要验证）**：

```
场景 A：kswapd 触发 F2FS 写回
内存压力
    ↓
kswapd 回收 F2FS 的脏页
    ↓
触发 F2FS 写回
    ↓
写回经过块设备层
    ↓
触发 blk_crypto_wq（加密）
    ↓
触发 kblockd（IO 处理）
    ↓
产生大量 IO

场景 B：应用直接触发 F2FS IO
应用大量文件操作
    ↓
F2FS 处理文件操作
    ↓
产生脏页（Page Cache）
    ↓
内存压力增加
    ↓
触发 kswapd
    ↓
同时，F2FS 写回也在进行
    ↓
两者叠加，IO 压力极高
```

**关键点**：
- **不能确定** kswapd 是这些 kworker 活动的直接原因
- **不能确定** F2FS 是这些 kworker 活动的直接原因
- 它们可能是**并行发生**，而不是因果关系

##### 5. 如何验证真实关系？

**方法 1：查看 kswapd 的堆栈**
```bash
cat /proc/103/stack  # kswapd 的 PID 是 103
# 查看 kswapd 是否正在调用 F2FS 的函数
```

**方法 2：使用 ftrace 追踪**
```bash
# 追踪 kswapd 的函数调用
echo kswapd > /sys/kernel/debug/tracing/set_ftrace_pid
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# 等待几秒
cat /sys/kernel/debug/tracing/trace | grep -E "kswapd|f2fs"
```

**方法 3：查看 F2FS 写回是否使用 workqueue**
```bash
# 需要查看 F2FS 源码或使用 ftrace
# 追踪 f2fs_write_data_pages 的调用链
```

##### 6. 结论（基于事实）

**确定的事实**：

1. ✅ kswapd 会回收 F2FS 的 Page Cache
2. ✅ 如果页面是脏页，会触发 F2FS 写回
3. ✅ F2FS 的 IO 操作经过块设备层
4. ✅ 块设备层的某些操作使用 workqueue（blk_crypto_wq, kblockd）

**不确定的（需要验证）**：

1. ❓ kswapd 是否直接导致了这些 kworker 的活动？
2. ❓ F2FS 的写回是否使用 workqueue？
3. ❓ events_unbound 的具体任务是什么？

**建议**：
- 不要强行关联这些机制
- 需要更多信息（堆栈、trace）才能确定真实关系
- 在获得更多信息之前，保持开放态度

#### 综合分析与结论

**kworker 活动总结**：

| kworker 类型 | CPU 占用 | 主要职责 | 与 ANR 的关系 |
|:------------|:--------|:---------|:-------------|
| `events_unbound` | 51% | 通用事件处理 | **最大嫌疑**，可能处理文件系统/IO 事件 |
| `blk_crypto_wq` | ~35% | 加密/解密 | **IO 压力放大器**，每个 IO 都需要加密 |
| `kblockd` | ~6% | 块设备 IO | 处理大量 IO 请求 |
| `kverityd` | ~6% | 分区验证 | IO 路径上的开销 |
| `events` (bound) | ~2% | 绑定事件处理 | 辅助处理 |

**根因链条推测**：

```
大量文件操作（读取/写入）
    ↓
触发大量加密/解密任务（blk_crypto_wq）
    ↓
触发大量块设备 IO（kblockd）
    ↓
触发文件系统事件（events_unbound）
    ↓
IO 队列饱和，存储设备无法承受
    ↓
IO PSI 78%，iowait 21%
    ↓
SystemUI 主线程执行 IO 操作被阻塞
    ↓
无法处理输入事件
    ↓
ANR
```

**关键发现**：

1. **`events_unbound` 占用 51% CPU**：
   - 这是最可疑的，需要进一步定位具体执行函数
   - 很可能是处理文件系统或内存回收相关事件

2. **`blk_crypto_wq` 多个 worker 并行**：
   - 说明加密任务积压严重
   - 每个 IO 都需要加密，加剧了 IO 压力

3. **所有 kworker 都是 kernel 占用**：
   - 说明都是内核态操作
   - 没有用户态开销

**下一步定位建议**：

1. **定位 `events_unbound` 的具体函数**：
   ```bash
   # 方法 1：查看堆栈
   cat /proc/25495/stack
   
   # 方法 2：ftrace 追踪
   echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_execute_start/enable
   cat /sys/kernel/debug/tracing/trace_pipe | grep "events_unbound"
   ```

2. **检查加密任务积压**：
   ```bash
   cat /proc/workqueues | grep blk_crypto
   # 查看 inflight 数量，如果很大说明积压
   ```

3. **检查 IO 统计**：
   ```bash
   iostat -x 1
   # 查看 IO 吞吐量、延迟、队列深度
   ```

**结论**：

在这个 ANR 案例中，kworker 不是问题的根源，而是**问题的症状**。真正的问题是：
- **IO 压力极高**（IO PSI 78%）
- **大量加密操作**（每个 IO 都需要加密）
- **存储设备性能瓶颈**（无法承受当前负载）

kworker 只是忠实地执行了这些任务，但系统整体 IO 能力不足，导致任务积压，最终导致 ANR。

### 5.3 Workqueue 拥塞 (Starvation)

如果大量任务被提交到同一个 Workqueue，且处理速度慢，会导致任务积压。

**查看方法**：
```bash
cat /proc/workqueues
```
该节点列出了所有 Workqueue 的运行状态，包括积压（inflight）的任务数量。

---

## 第六章：源码导读

- **核心实现**：`kernel/workqueue.c`
- **头文件**：`include/linux/workqueue.h`

**关键函数**：

1.  `process_one_work()`: Worker 线程执行任务的核心循环。
    *   在这里加入 tracepoint `workqueue_execute_start`。
2.  `worker_thread()`: kworker 线程的入口函数。
3.  `queue_work()` / `schedule_work()`: 提交任务的接口。

```c
// kernel/workqueue.c

static void process_one_work(struct worker *worker, struct work_struct *work)
__releases(&pool->lock)
__acquires(&pool->lock)
{
    // ...
    // 触发 tracepoint，记录开始执行
    trace_workqueue_execute_start(work);

    // 执行具体的回调函数
    worker->current_func(work);

    // 触发 tracepoint，记录执行结束
    trace_workqueue_execute_end(work);
    // ...
}
```

---

## 📝 总结与实战建议

1.  **看到 kworker 先别慌**，它是干活的人，不是捣乱的人。问题通常在于**指派任务的人**（某个驱动或子系统）。
2.  **看名字**：
    - `u` 开头：Unbound，可能跑在任何 CPU。
    - 无 `u`：Bound，绑定特定 CPU。
    - 后缀名：如果有（如 `blk_crypto`），直接暴露用途。
3.  **找真凶**：
    - 静态：`cat /proc/pid/stack`
    - 动态（绝杀）：`ftrace` 抓 `workqueue_execute_start`
4.  **分状态**：
    - **R (Running)**: 查 CPU 热点函数。
    - **D (Disk Sleep)**: 查 IO 瓶颈。

掌握 kworker 分析，意味着你穿透了进程的迷雾，直接看到了内核的工作现场。
