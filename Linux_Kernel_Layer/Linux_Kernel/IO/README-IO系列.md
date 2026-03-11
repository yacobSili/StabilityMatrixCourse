# 面向稳定性的 Linux 内核 IO 深度解析系列

## 为什么要写这个系列

**Linux IO 子系统是 Android 存储与卡顿的底层支柱。** 主线程的一次 `SharedPreferences.commit()`、一次 SQLite 写、一次文件 `fsync`，都会从用户态一路穿到 VFS、Page Cache、块层、设备驱动；任何一层变慢，都会直接放大为 ANR、卡顿或 iowait 飙升。

对于稳定性架构师来说，IO 子系统的重要性在于：

- **ANR 的常见根因**：主线程同步 IO（commit、DB、文件关闭触发的 fsync）导致进程进入 D 状态，无法及时处理 Input/广播，触发 Input ANR 或 Broadcast ANR。
- **卡顿与 iowait**：iowait 高表示 CPU 在等 IO；大量进程处于 D 状态会拖慢整机响应；后台进程 IO 未限速会抢占前台，导致前台 IO 延迟大。
- **IO 优先级与进程状态**：谁先被块层调度（ionice、cgroup blkio）、进程等 IO 时的 D 状态与唤醒、与 CPU 调度的衔接，都直接影响"谁先被服务"。
- **存储满与只读**：空间不足、inode 耗尽、remount ro 会导致应用无法安装/升级、日志无法写入，进而引发各类异常。

本系列的目标：**让你理解从系统调用到块设备的完整 IO 路径，理解 IO 优先级、调度与进程状态的关系，能从 iostat/ftrace/ANR trace 中定位 IO 相关根因，并建立 IO 监控与治理体系。**

## 系列设计思路

```
Linux IO 子系统是什么？为什么 IO 延迟会直接导致 Android ANR/卡顿？（定位）
    ↓
从 VFS 到块设备，IO 路径经过哪些层？与 Page Cache、调度器如何协作？（边界与交互）
    ↓
请求如何下发？合并与调度如何做？IO 优先级如何影响调度？进程 D 状态与唤醒如何与 IO 衔接？
    Page Fault 与回写如何影响延迟？（核心机制）
    ↓
IO 瓶颈、iowait 高、主线程被 IO 阻塞、后台 IO 抢占前台、存储满各是怎么发生的？（风险地图）
    ↓
怎么用 iostat/ftrace/blk tracer 排查？怎么建 IO 监控与治理？（诊断与治理）
```

## 篇章划分与递进关系（共 9 篇 + 1 延伸专题）

| 篇章 | 篇数 | 递进逻辑 |
| :--- | :--- | :--- |
| **第一篇章：建立全局观** | 2 篇 | 01 回答「是什么、为什么重要、五层架构、一次读/写概要」；02 回答「IO 在系统中的边界与交互（优先级、进程状态、Binder/ART/LMK/StrictMode）」 |
| **第二篇章：核心机制深潜** | 5 篇 | 按 IO 数据流**自顶向下**逐层：03 VFS → 04 Page Cache → 05 块层 → 06 设备层（f2fs、eMMC/UFS）→ 07 进程与 IO 的衔接（横切主题） |
| **第三篇章：诊断实战与治理** | 2 篇 | 08 风险地图（会出什么问题）；09 诊断工具与治理（怎么查、怎么防） |

---

## 第一篇章：建立全局观（2 篇）

> 核心问题：Linux IO 是什么？一次读/写请求经历了哪些层？IO 优先级与进程状态在整体中的位置？

### [01-IO 子系统总览：从系统调用到块设备](01-IO子系统总览.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. Linux IO 是什么 | 从 read/write 到块设备完成请求的完整路径；用户态、VFS、Page Cache、块层、设备驱动各层职责 | `fs/`、`block/`、`mm/` | 任何一层变慢都会放大为 ANR/卡顿 |
| 2. 为什么 Android 稳定性必须懂 IO | 主线程同步 IO → 阻塞 → Input/Broadcast ANR；iowait 高 → 系统卡顿；存储满 → 安装/升级失败 | — | 建立「IO 即稳定性」的视角 |
| 3. 五层架构全景图 | Application → VFS → Page Cache/文件系统 → Block Layer → Device Driver；各层职责与边界 | `fs/read_write.c`、`block/blk-mq.c` | 出问题时定位在哪一层 |
| 4. 一次读请求与一次写请求的概要旅程 | 端到端路径（不深入每层细节）；关键节点与阻塞点 | `mm/filemap.c`、`fs/fs-writeback.c` | 建立心智模型 |
| 5. 核心源码目录导航 | fs/、block/、mm/、f2fs、ext4、mmc/ufs | 多个目录 | 排查问题时的导航地图 |

### [02-IO 优先级、进程调度与 Android 模块交互](02-IO优先级进程调度与Android模块交互.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. IO 优先级与进程调度的关系（概要） | ionice、cgroup blkio 决定「谁先被下发」；进程发起同步 IO 时进入 D 状态、被唤醒后参与 CPU 调度；三者（IO 优先级、块层调度、进程状态）在 ANR/卡顿中的位置 | `block/ioprio.c`、`kernel/sched/` | 建立整体心智模型 |
| 2. 与 Android 其他模块的交互 | Binder 与 IO、ART（dex2oat）与 IO、LMK/OOM 与回写、StrictMode 主线程 IO 检测 | — | IO 是交叉问题的常见因素 |

---

## 第二篇章：核心机制深潜（5 篇）

> 核心问题：VFS/Page Cache/块层如何工作？IO 优先级如何设置与生效？进程等 IO 时状态如何、如何被唤醒？

### [03-VFS 与文件读写路径](03-VFS与文件读写路径.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. VFS 的定位 | 统一抽象（inode、dentry、file、super_block）；file_operations 与各文件系统实现 | `fs/` | 理解读写入口 |
| 2. read/write 系统调用入口 | ksys_read/vfs_read、ksys_write/vfs_write 的源码路径与参数传递 | `fs/read_write.c` | 用户态到内核的边界 |
| 3. 与 Page Cache 的协作 | generic_file_read_iter、generic_perform_write；缓存命中/未命中分支 | `mm/filemap.c` | 读写的实际落点 |
| 4. fsync/fdatasync | 刷脏页、写元数据、块设备刷缓存；为什么 commit() 会卡主线程数秒 | `fs/sync.c` | 主线程 fsync → ANR |

### [04-Page Cache 与回写](04-PageCache与回写.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. Page Cache 是什么 | 文件数据在内存中的缓存；address_space、struct page、radix tree 索引 | `mm/filemap.c` | 理解缓存层 |
| 2. 读路径 | 缓存命中直接返回；未命中触发 readpage/readahead → 块层 | `mm/filemap.c` | 缺页与 IO 的关系 |
| 3. 写路径与回写时机 | 写回 Page 成脏页；周期回写、内存压力、fsync、sync | `mm/page-writeback.c`、`fs/fs-writeback.c` | 脏页堆积 → 回写风暴 → iowait 飙升 |
| 4. 回写线程与 writeback | flush 线程、wb（per-device）、balance_dirty_pages_ratelimited | `fs/fs-writeback.c` | OOM 与回写抢占 |

### [05-块层与 I/O 调度：优先级、调度器与进程优先级](05-块层与IO调度.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. 块层职责与 bio/request | 将 BIO 请求封装、合并、排序后下发给设备驱动；request、request_queue、bio 三者关系与生命周期 | `block/blk-core.c`、`block/blk-mq.c` | 理解请求生命周期 |
| 2. I/O 优先级 | I/O 调度类（RT/BEST_EFFORT/IDLE）；ionice/ioprio_set；进程 nice 映射；io_context、ioc（blkcg） | `block/ioprio.c` | 谁先被下发 → 前台/后台延迟 |
| 3. cgroup 与 IO 限速/权重 | blkio.weight、blkio.throttle（v1）；io.max、io.weight（v2）；Android 上对应用的隔离 | `block/blk-cgroup.c` 等 | cgroup 未配置 → 某进程占满 IO |
| 4. I/O 调度器 | mq-deadline、Kyber、BFQ；各调度器如何利用进程/IO 优先级排序；对前台/后台延迟的影响 | `block/` 下调度器实现 | 调度器选择不当 → 尾延迟大 |
| 5. 请求合并与 blk_mq | front/back merge；多队列与软件队列 | `block/blk-mq.c` | 请求堆积 → 超时与 ANR |

**延伸专题**：[进程优先级与 I/O 优先级的关系](进程优先级与IO优先级的关系.md) — 从 Android 稳定性视角，深入解析进程优先级与 I/O 优先级的联动，含源码佐证（setpriority、SetTaskProfiles、blk_throtl_bio、BIO 拆分、cgroup v1/v2 等）。

### [06-Android 存储栈：f2fs 与设备层](06-Android存储栈.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. f2fs | 针对 Flash 的布局（segment、block、node）；GC、checkpoint、trim | `fs/f2fs/` | f2fs GC 导致卡顿 |
| 2. eMMC/UFS | mmc_blk、ufs 驱动；命令队列与乱序完成 | `drivers/mmc/`、`drivers/scsi/ufs/` | UFS 队列满 → 高延迟 |
| 3. dm-verity / 加密 | 块层之上的虚拟设备；对 IO 路径与延迟的影响 | `drivers/md/` | 加密导致额外 CPU 与延迟 |

### [07-IO 与进程阻塞：进程状态、iowait 与主线程卡顿](07-IO与进程阻塞.md)

| 章节 | 内容 | 核心源码路径 | 稳定性关联 |
| :--- | :--- | :--- | :--- |
| 1. 进程状态与 IO | 发起同步 IO 时进入 D 状态（TASK_UNINTERRUPTIBLE）；D 与 S、R 的区别；为何 D 不能被 kill | `kernel/sched/` | 主线程 D 状态 → Input ANR |
| 2. 等待队列与唤醒 | 进程在何处等待（wait_on_page_bit、io_schedule、blk_mq_get_request）；IO 完成路径如何 wake_up | `include/linux/wait.h`、`block/blk-mq.c` | 理解阻塞与唤醒的衔接 |
| 3. IO 完成与进程调度 | 被唤醒的进程 D → R；CPU 调度器如何选择被 IO 唤醒的进程 | `kernel/sched/` | 大量 D 状态 → 系统「假死」感 |
| 4. iowait 的含义 | CPU 空闲但存在等待 IO 的进程；/proc/stat 中 iowait 的统计方式 | — | iowait 高 → 整机卡顿 |
| 5. 主线程被 IO 阻塞的典型场景 | SharedPreferences.commit、SQLite 写、FileOutputStream.close 触发的 fsync、ContentResolver 跨进程 IO | — | 与 ANR 的对应关系 |
| 6. 从 ANR trace 识别 IO 阻塞 | 主线程栈在 wait_on_page_bit、io_schedule、blk_mq_get_request 等 | — | 实战归因 |

---

## 第三篇章：诊断实战与治理（2 篇）

> 核心问题：IO 出了问题怎么查？怎么建监控？怎么做到「能查→能治→能防」？

### [08-IO 稳定性风险全景：ANR、卡顿与存储满](08-IO稳定性风险全景.md)

| 章节 | 内容 | 稳定性关联 |
| :--- | :--- | :--- |
| 1. IO 相关 ANR 分类 | 主线程同步 IO 阻塞型、iowait 导致调度延迟型、存储满导致安装/升级超时型 | 三种类型的识别与排查方向 |
| 2. IO 优先级与进程状态相关风险 | 后台高 IO 或未限速抢占设备 → 前台 IO 延迟大；大量 D 状态 → 响应变慢；cgroup 未隔离 → 某应用占满 IO | 优先级与状态导致的卡顿/假死 |
| 3. 卡顿与 IO | 单次大 IO、回写风暴、f2fs GC、UFS 队列深度与超时 | 定位卡顿在 IO 的哪一层 |
| 4. 存储满与只读 | 空间不足、inode 耗尽、remount ro 的触发条件；对应用与系统的影响 | 安装失败、日志写不进去等 |
| 5. 日志与特征 | dmesg、/proc/diskstats、ANR trace 栈、StrictMode 日志 | 问题速查表：现象 → 可能层级 → 排查工具 |

### [09-IO 诊断工具与治理体系](09-IO诊断工具与治理体系.md)

| 章节 | 内容 | 稳定性关联 |
| :--- | :--- | :--- |
| 1. iostat / /proc/diskstats | 读写量、IOPS、延迟、队列深度 | 系统层 IO 健康度 |
| 2. ftrace 与 blk tracer | blk_block_rq_issue、blk_block_rq_complete，定位高延迟请求 | 单请求延迟分析 |
| 3. Android 侧 | StrictMode 主线程 IO 检测、dumpsys diskstats、fstab 与挂载选项 | 应用层与系统配置 |
| 4. 治理最佳实践 | 主线程禁止同步 IO；合理使用 fdatasync/fsync；利用 cgroup/ionice 控制后台 IO；监控 iowait、块设备延迟及 D 状态进程数量 | 从「能查」到「能防」的闭环 |
| 5. 监控与告警 | 开发期 StrictMode、线上 iowait 监控、存储空间告警 | 持续治理 |

---

## 阅读建议

**按篇章顺序**（推荐）：01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09。每篇依赖前文建立的心智模型。

**若时间有限，可优先**：01（全局观）→ 07（进程阻塞，与 ANR 直接相关）→ 05（块层，理解「谁先被服务」）→ 08、09（风险与治理）。

**延伸专题**：若需 Android 侧源码佐证（Process.setThreadPriority、SetTaskProfiles、blk_throtl_bio、BIO 拆分等），可配合 05 阅读 [进程优先级与 I/O 优先级的关系](进程优先级与IO优先级的关系.md)。

每篇文章的设计逻辑是：
```
背景与定义（是什么、为什么需要它）
    → 架构与交互（在系统中的位置、上下游关系）
        → 核心机制与源码（关键数据结构、核心流程）
            → 稳定性风险点（会在哪里出问题）
                → 实战案例（线上真实问题的排查过程）
```
