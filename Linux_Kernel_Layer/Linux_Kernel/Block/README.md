# IO 流程关键结构体详解系列

本系列文章按照**真实 IO 请求的生命周期**为主线，逐阶段介绍 Block 层涉及的关键结构体及其成员变量。

## 系列文章列表

### [23-IO流程结构体总览与Bio层](./23-IO流程结构体总览与Bio层.md)

**涵盖内容**：
- 完整 IO 流程概览（时序图 + UML类图）
- 结构体关系总览
- **阶段1：VFS 到 Bio 创建**
  - `struct bio` - Block IO 的基本单位
  - `struct bio_vec` - 内存段描述
  - `struct bvec_iter` - Bio 迭代器

**适合读者**：理解 IO 起点，bio 如何封装数据

---

### [24-Request分配与软件队列](./24-Request分配与软件队列.md)

**涵盖内容**：
- **阶段2：Bio 提交与合并**
  - `struct request_queue` - 请求队列（合并相关字段）
- **阶段3：Request 分配与 Internal Tag**
  - `struct blk_mq_tags` - Tag 管理
  - `struct sbitmap_queue` - 位图队列
  - `struct request` - IO 请求（初始化阶段）
- **阶段4：插入软件队列**
  - `struct blk_mq_ctx` - 软件队列（Per-CPU）

**适合读者**：理解 Tag 机制、request 如何分配

---

### [25-IO调度器结构体详解](./25-IO调度器结构体详解.md)

**涵盖内容**：
- **阶段5：调度器处理**
  - `struct elevator_queue` - IO 调度器队列
  - `struct deadline_data` - mq-deadline 全局数据
  - `struct dd_per_prio` - 每个优先级的数据
  - 调度器插入与选择逻辑

**适合读者**：理解 IO 优先级、调度器如何排序请求

---

### [26-硬件队列与Tag派发](./26-硬件队列与Tag派发.md)

**涵盖内容**：
- **阶段6：触发派发与硬件队列**
  - `struct blk_mq_hw_ctx` - 硬件队列（详解所有字段）
  - Internal Tag vs Driver Tag
- **阶段7：Driver Tag 分配与派发**
  - Driver Tag 分配流程
  - request 状态更新

**适合读者**：理解派发机制、两种 Tag 的区别

---

### [27-驱动处理与IO完成](./27-驱动处理与IO完成.md)

**涵盖内容**：
- **阶段8：驱动处理**
  - 驱动视角的 request 字段
  - 驱动处理示例
- **阶段9：IO 完成与清理**
  - request 完成流程
  - bio 完成流程
- **附录**
  - 结构体大小与内存布局
  - 结构体关系速查表
  - 常用访问宏
  - 调试技巧（crash/bpftrace）

**适合读者**：理解 IO 完成、调试分析

---

## 阅读建议

### 快速入门路径
1. 先看第 23 篇的流程图和类图，建立整体概念
2. 重点看各篇的 **⭐⭐⭐ 核心字段**
3. 遇到问题时查阅对应结构体的详细说明

### 深入学习路径
1. 按顺序阅读全部 5 篇
2. 结合源码验证每个字段的使用
3. 使用附录的调试技巧实际观察

### 问题排查路径
1. 先查第 27 篇附录的速查表
2. 定位到具体结构体所在的文章
3. 查看核心字段说明和代码示例

---

## 成员变量讲解策略

本系列对结构体成员变量采用三级分类：

| 级别 | 标记 | 讲解深度 | 适用字段 |
|------|------|---------|---------|
| 核心字段 | ⭐⭐⭐ | 详细讲解 | IO主路径使用、影响性能、调试常用 |
| 重要字段 | ⭐⭐ | 简要说明 | 辅助功能、统计信息 |
| 次要字段 | ⭐ | 分组概述 | 配置参数、内部管理、特殊场景 |

**示例**：
- `bio->bi_opf` ⭐⭐⭐ - 详细解释操作类型和标志位的组成
- `bio->bi_ioprio` ⭐⭐ - 简要说明用于调度器优先级
- `bio->bi_write_hint` ⭐ - 归入"特殊功能"组概述

---

## 关键源码文件索引

| 结构体类别 | 主要文件 |
|-----------|---------|
| bio 相关 | `include/linux/blk_types.h` |
| request 相关 | `include/linux/blkdev.h` |
| blk-mq 核心 | `include/linux/blk-mq.h`, `block/blk-mq.h` |
| Tag 管理 | `block/blk-mq-tag.h`, `include/linux/sbitmap.h` |
| 调度器 | `include/linux/elevator.h`, `block/mq-deadline.c` |

---

## 相关文章推荐

阅读本系列前建议先看：
- [09-blk_mq基础架构与核心概念](../09-blk_mq基础架构与核心概念.md)
- [16-Tag机制深入解析](../16-Tag机制深入解析.md)
- [22-IO调度器工作流程与代码实现详解](../22-IO调度器工作流程与代码实现详解.md)

阅读本系列后可以看：
- [21-IO完整流程与Tracepoint位置详解](../21-IO完整流程与Tracepoint位置详解.md)
- [18-Block层Tracepoint详解与使用指南](../18-Block层Tracepoint详解与使用指南.md)
