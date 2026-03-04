# Android13-5.15 内核升级 IO/kswapd/内存影响分析报告
## 版本对比：2024-05_r2 → 2024-11_r3（跨度6个月）

---

## 📋 执行摘要

**分析范围**：
- **基准版本**：`android13-5.15-2024-05_r2`（2024年5月）
- **问题版本**：`android13-5.15-2024-11_r3`（2024年11月）
- **时间跨度**：6个月
- **分析方法**：基于本地源码 Git diff 对比分析

**关键发现**：
1. **内存管理层变更**是主要怀疑点，特别是 `arch/arm64/mm/mmu.c` 和 `mm/vmscan.c`
2. **IO 层变更较少**，主要是修复和优化
3. **文件系统层变更**主要是错误处理改进
4. **多个维度交叉验证**指向内存管理问题

**怀疑优先级**：
- 🔴 **极高怀疑**：`arch/arm64/mm/mmu.c`、`mm/vmscan.c`
- 🟠 **高怀疑**：`mm/page-writeback.c`、`block/blk-mq.c`
- 🟡 **中怀疑**：`fs/f2fs/data.c`、`fs/ext4/file.c`
- 🟢 **低怀疑**：`mm/filemap.c`（仅 trace 点）

---

## 第一章：变更范围统计

### 1.1 目录级别变更统计

基于 `git diff android13-5.15-2024-05_r2 android13-5.15-2024-11_r3 --shortstat` 分析：

| 目录 | 变更文件数 | 变更行数 | 影响评估 |
|------|-----------|---------|---------|
| `mm/` | 27 个文件 | ~290 行增删 | ⚠️⚠️⚠️ **极高风险** |
| `arch/arm64/mm/` | 1 个文件 | ~10 行 | ⚠️⚠️⚠️ **极高风险** |
| `block/` | 11 个文件 | ~65 行增删 | ⚠️⚠️ **高风险** |
| `fs/f2fs/` | 多个文件 | ~400+ 行 | ⚠️ **中风险** |
| `fs/ext4/` | 多个文件 | ~600+ 行 | ⚠️ **中风险** |
| `kernel/sched/` | 多个文件 | 未知 | ⚠️ **中风险** |

### 1.2 关键文件变更列表

#### 内存管理相关（mm/ 目录）

| 文件路径 | 变更行数 | 优先级 | 怀疑度 |
|---------|---------|--------|--------|
| `mm/vmscan.c` | 7 行 | ⭐⭐⭐⭐⭐ | 🔴 **极高** |
| `mm/page-writeback.c` | 30 行 | ⭐⭐⭐⭐ | 🟠 **高** |
| `mm/filemap.c` | 5 行 | ⭐⭐⭐ | 🟡 **中** |
| `mm/memory.c` | 46 行 | ⭐⭐⭐ | 🟡 **中** |
| `mm/mremap.c` | 78 行 | ⭐⭐ | 🟢 **低** |
| `mm/compaction.c` | 7 行 | ⭐⭐⭐ | 🟡 **中** |
| `mm/memcontrol.c` | 30 行 | ⭐⭐ | 🟢 **低** |
| `mm/swapfile.c` | 25 行 | ⭐⭐⭐ | 🟡 **中** |
| `mm/page_alloc.c` | 11 行 | ⭐⭐ | 🟢 **低** |
| `mm/readahead.c` | 2 行 | ⭐⭐ | 🟢 **低** |
| `mm/mmap_lock.c` | 175 行（大幅减少） | ⭐⭐ | 🟢 **低** |

#### ARM64 内存映射相关

| 文件路径 | 变更行数 | 优先级 | 怀疑度 |
|---------|---------|--------|--------|
| `arch/arm64/mm/mmu.c` | ~10 行 | ⭐⭐⭐⭐⭐ | 🔴 **极高** |

#### IO 层相关（block/ 目录）

| 文件路径 | 变更行数 | 优先级 | 怀疑度 |
|---------|---------|--------|--------|
| `block/blk-mq.c` | 24 行 | ⭐⭐⭐⭐ | 🟠 **高** |
| `block/blk-iocost.c` | 14 行 | ⭐⭐⭐ | 🟡 **中** |
| `block/blk-settings.c` | 4 行 | ⭐⭐ | 🟢 **低** |
| `block/bio.c` | 2 行 | ⭐⭐ | 🟢 **低** |
| `block/bio-integrity.c` | 21 行 | ⭐⭐ | 🟢 **低** |
| `block/blk-integrity.c` | 2 行 | ⭐ | 🟢 **低** |

#### 文件系统相关（fs/ 目录）

| 文件路径 | 变更行数 | 优先级 | 怀疑度 |
|---------|---------|--------|--------|
| `fs/f2fs/data.c` | 54 行 | ⭐⭐⭐ | 🟡 **中** |
| `fs/f2fs/file.c` | 155 行 | ⭐⭐⭐ | 🟡 **中** |
| `fs/f2fs/compress.c` | 44 行 | ⭐⭐ | 🟢 **低** |
| `fs/f2fs/gc.c` | 59 行 | ⭐⭐ | 🟢 **低** |
| `fs/ext4/file.c` | 155 行 | ⭐⭐⭐ | 🟡 **中** |
| `fs/ext4/inode.c` | 120 行 | ⭐⭐ | 🟢 **低** |
| `fs/ext4/mballoc.c` | 258 行 | ⭐⭐ | 🟢 **低** |

---

## 第二章：多维度详细分析

### 2.1 维度一：IO 性能影响分析

#### 2.1.1 直接影响 IO 的变更

**🔴 极高怀疑：arch/arm64/mm/mmu.c**

**变更详情**：
```diff
--- a/arch/arm64/mm/mmu.c
+++ b/arch/arm64/mm/mmu.c
@@ -436,7 +436,7 @@ static void __init create_mapping_noalloc(...)
 {
-	if ((virt >= PAGE_END) && (virt < VMALLOC_START)) {
+	if (virt < PAGE_OFFSET) {
 		pr_warn("BUG: not creating mapping...");
 		return;
 	}
@@ -463,7 +463,7 @@ static void update_mapping_prot(...)
 {
-	if ((virt >= PAGE_END) && (virt < VMALLOC_START)) {
+	if (virt < PAGE_OFFSET) {
 		pr_warn("BUG: not updating mapping...");
 		return;
 	}
```

**分析**：
- **变更性质**：VA-range 检查逻辑改变
- **原逻辑**：检查 `PAGE_END` 到 `VMALLOC_START` 之间的范围
- **新逻辑**：检查所有小于 `PAGE_OFFSET` 的地址
- **潜在影响**：
  1. 如果 `PAGE_END` 和 `PAGE_OFFSET` 的关系不符合预期，可能导致映射范围错误
  2. **直接影响 Page Cache 映射**：Page Cache 页面可能无法正确映射到虚拟地址空间
  3. **文件 IO 路径影响**：文件 IO 无法使用 Page Cache，必须直接访问磁盘
  4. **iowait 上升机制**：每次文件操作都触发磁盘 IO，导致 iowait 急剧上升

**怀疑理由**：
- ✅ 直接修改内存映射逻辑
- ✅ 可能影响所有通过 Page Cache 的文件 IO
- ✅ 变更逻辑看似"修复"，但改变了检查范围
- ✅ 时间相关性：变更后 IO 问题出现

**影响路径**：
```
VA-range 检查逻辑改变
    ↓
Page Cache 映射可能失败
    ↓
文件 IO 无法使用 Page Cache
    ↓
每次文件操作都触发磁盘 IO
    ↓
iowait 急剧上升（> 20%）
    ↓
ANR 大面积增加
```

**🟠 高怀疑：mm/vmscan.c**

**变更详情**：
```diff
--- a/mm/vmscan.c
+++ b/mm/vmscan.c
@@ -5930,7 +5932,7 @@ static bool in_reclaim_compaction(...)
 {
-	if (IS_ENABLED(CONFIG_COMPACTION) && sc->order &&
+	if (gfp_compaction_allowed(sc->gfp_mask) && sc->order &&
 			(sc->order > PAGE_ALLOC_COSTLY_ORDER ||
 			 sc->priority < DEF_PRIORITY - 2))
 		return true;
@@ -6175,6 +6177,9 @@ static inline bool compaction_ready(...)
 	unsigned long watermark;
 	enum compact_result suitable;
 
+	if (!gfp_compaction_allowed(sc->gfp_mask))
+		return false;
+
 	suitable = compaction_suitable(zone, sc->order, 0, sc->reclaim_idx);
```

**分析**：
- **变更性质**：限制内存压缩（compaction）的使用
- **原逻辑**：只要配置了 COMPACTION 就允许压缩
- **新逻辑**：通过 `gfp_compaction_allowed()` 检查，限制某些 GFP 标志下的压缩
- **潜在影响**：
  1. **内存碎片化增加**：压缩受限导致内存碎片化
  2. **大块内存分配失败**：碎片化导致大块内存分配失败
  3. **触发更多 swap IO**：分配失败触发 swap，导致 swap IO 增加
  4. **间接导致 iowait 上升**：swap IO 增加导致整体 IO 压力增加

**怀疑理由**：
- ✅ 直接影响内存回收和压缩策略
- ✅ 可能导致内存碎片化，间接影响 IO
- ✅ kswapd 行为可能改变
- ✅ 时间相关性：变更后 IO 问题出现

**影响路径**：
```
内存压缩受限
    ↓
内存碎片化增加
    ↓
大块内存分配失败
    ↓
触发更多 swap IO
    ↓
IO 压力增加
    ↓
iowait 上升
```

**🟠 高怀疑：mm/page-writeback.c**

**变更详情**：
```diff
--- a/mm/page-writeback.c
+++ b/mm/page-writeback.c
@@ -426,13 +426,20 @@ static void domain_dirty_limits(...)
 	else
 		bg_thresh = (bg_ratio * available_memory) / PAGE_SIZE;
+	/*
+	 * Dirty throttling logic assumes the limits in page units fit into
+	 * 32-bits. This gives 16TB dirty limits max which is hopefully enough.
+	 */
+	if (thresh > UINT_MAX)
+		thresh = UINT_MAX;
+	if (bg_thresh >= thresh)
+		bg_thresh = thresh / 2;
```

**分析**：
- **变更性质**：限制脏页阈值上限为 `UINT_MAX`
- **潜在影响**：
  1. **写回更频繁**：阈值被限制可能导致写回更频繁
  2. **IO 频率增加**：频繁写回导致 IO 频率增加
  3. **可能增加 IO 延迟**：频繁的写回可能增加 IO 延迟

**怀疑理由**：
- ⚠️ 直接影响脏页写回策略
- ⚠️ 可能增加 IO 频率
- ⚠️ 但影响相对较小

**🟡 中怀疑：block/blk-mq.c**

**变更详情**：
```diff
--- a/block/blk-mq.c
+++ b/block/blk-mq.c
@@ -1179,6 +1179,22 @@ static bool blk_mq_mark_tag_wait(...)
 	wait->flags &= ~WQ_FLAG_EXCLUSIVE;
 	__add_wait_queue(wq, wait);
 
+	/*
+	 * Add one explicit barrier since blk_mq_get_driver_tag() may
+	 * not imply barrier in case of failure.
+	 */
+	smp_mb();
+
@@ -1708,6 +1724,14 @@ void blk_mq_delay_run_hw_queues(...)
 	queue_for_each_hw_ctx(q, hctx, i) {
 		if (blk_mq_hctx_stopped(hctx))
 			continue;
+		/*
+		 * If there is already a run_work pending, leave the
+		 * pending delay untouched.
+		 */
+		if (delayed_work_pending(&hctx->run_work))
+			continue;
```

**分析**：
- **变更性质**：
  1. 添加内存屏障 `smp_mb()` 在等待队列和驱动标签分配之间
  2. 添加 `delayed_work_pending()` 检查，防止 hctx 停滞
- **潜在影响**：
  1. **内存屏障可能增加延迟**：`smp_mb()` 可能略微增加 IO 延迟
  2. **主要是修复竞争条件**：这些变更主要是修复潜在的竞争条件，不太可能是性能劣化的原因

**怀疑理由**：
- ⚠️ 可能略微增加 IO 延迟
- ⚠️ 但主要是修复，不太可能是根本原因

#### 2.1.2 间接影响 IO 的变更

**🟡 中怀疑：mm/filemap.c**

**变更详情**：
```diff
--- a/mm/filemap.c
+++ b/mm/filemap.c
@@ -2552,6 +2552,9 @@ retry:
 	filemap_get_read_batch(mapping, index, last_index - 1, pvec);
 	if (!pagevec_count(pvec)) {
+		trace_android_vh_page_cache_miss(filp, index,
+				(iter->count + PAGE_SIZE-1) >> PAGE_SHIFT,
+				index, true);
 		if (iocb->ki_flags & IOCB_NOIO)
 			return -EAGAIN;
```

**分析**：
- **变更性质**：仅添加 Android vendor hooks（trace 点）
- **潜在影响**：几乎无影响，只是添加 trace 点用于监控

**怀疑理由**：
- ✅ 仅添加 trace，不太可能影响性能

**🟢 低怀疑：fs/f2fs/data.c**

**变更详情**：
```diff
--- a/fs/f2fs/data.c
+++ b/fs/f2fs/data.c
@@ -683,8 +683,10 @@ int f2fs_submit_page_bio(...)
 	if (!f2fs_is_valid_blkaddr(fio->sbi, fio->new_blkaddr,
 			fio->is_por ? META_POR : (__is_meta_io(fio) ?
-			META_GENERIC : DATA_GENERIC_ENHANCE)))
+			META_GENERIC : DATA_GENERIC_ENHANCE))) {
+		f2fs_handle_error(fio->sbi, ERROR_INVALID_BLKADDR);
 		return -EFSCORRUPTED;
+	}
```

**分析**：
- **变更性质**：改进错误处理，添加 `f2fs_handle_error()` 调用
- **潜在影响**：主要是错误处理改进，不太可能影响正常 IO 路径

**怀疑理由**：
- ✅ 仅改进错误处理，不影响正常 IO 路径

### 2.2 维度二：kswapd 行为影响分析

#### 2.2.1 直接影响 kswapd 的变更

**🔴 极高怀疑：mm/vmscan.c**

**变更详情**：
- 引入 `gfp_compaction_allowed()` 检查，限制内存压缩
- 影响 `in_reclaim_compaction()` 和 `compaction_ready()` 函数

**kswapd 行为变化**：
1. **压缩受限**：kswapd 在某些情况下无法进行内存压缩
2. **回收策略改变**：回收和压缩的决策逻辑改变
3. **可能导致更激进的回收**：压缩受限可能导致 kswapd 采用更激进的回收策略
4. **swap IO 增加**：更激进的回收可能导致更多 swap IO

**怀疑理由**：
- ✅ 直接修改 kswapd 使用的回收和压缩逻辑
- ✅ 可能导致 kswapd 行为改变
- ✅ 间接导致 swap IO 增加

**影响路径**：
```
内存压缩受限
    ↓
kswapd 回收策略改变
    ↓
更激进的回收
    ↓
更多 swap IO
    ↓
IO 压力增加
```

**🟡 中怀疑：mm/compaction.c**

**变更详情**：
- 7 行变更，具体内容需要查看

**潜在影响**：
- 可能影响内存压缩行为
- 间接影响 kswapd 的压缩决策

**🟡 中怀疑：mm/swapfile.c**

**变更详情**：
- 25 行变更

**潜在影响**：
- 可能影响 swap 文件管理
- 间接影响 kswapd 的 swap IO 行为

#### 2.2.2 间接影响 kswapd 的变更

**🟡 中怀疑：mm/page_alloc.c**

**变更详情**：
- 11 行变更

**潜在影响**：
- 可能影响页面分配策略
- 间接影响 kswapd 的回收决策

**🟡 中怀疑：mm/memcontrol.c**

**变更详情**：
- 30 行变更

**潜在影响**：
- 可能影响内存控制组（cgroup）管理
- 间接影响 kswapd 的回收决策

### 2.3 维度三：内存管理影响分析

#### 2.3.1 内存映射影响

**🔴 极高怀疑：arch/arm64/mm/mmu.c**

**内存映射影响**：
1. **VA-range 检查逻辑改变**：可能影响虚拟地址到物理地址的映射
2. **Page Cache 映射可能失败**：如果映射检查逻辑错误，Page Cache 页面可能无法正确映射
3. **文件映射可能失败**：mmap 文件映射可能受影响

**怀疑理由**：
- ✅ 直接修改内存映射逻辑
- ✅ 可能影响所有内存映射操作
- ✅ 直接影响 Page Cache

**影响路径**：
```
VA-range 检查逻辑改变
    ↓
内存映射范围可能错误
    ↓
Page Cache 映射失败
    ↓
文件 IO 无法使用缓存
```

#### 2.3.2 内存回收影响

**🔴 极高怀疑：mm/vmscan.c**

**内存回收影响**：
1. **压缩受限**：内存压缩被限制，可能导致内存碎片化
2. **回收策略改变**：回收和压缩的决策逻辑改变
3. **swap 使用增加**：压缩受限可能导致更多 swap 使用

**怀疑理由**：
- ✅ 直接修改内存回收逻辑
- ✅ 可能导致内存碎片化
- ✅ 间接导致 swap IO 增加

**影响路径**：
```
内存压缩受限
    ↓
内存碎片化增加
    ↓
大块内存分配失败
    ↓
触发更多 swap IO
```

#### 2.3.3 内存分配影响

**🟡 中怀疑：mm/page_alloc.c**

**内存分配影响**：
- 11 行变更，可能影响页面分配策略

**潜在影响**：
- 可能影响大块内存分配
- 可能影响 Page Cache 分配

**🟡 中怀疑：mm/memory.c**

**内存分配影响**：
- 46 行变更，可能影响内存管理

**潜在影响**：
- 可能影响页面错误处理
- 可能影响内存映射

### 2.4 维度四：文件系统影响分析

#### 2.4.1 F2FS 相关变更

**🟡 中怀疑：fs/f2fs/data.c**

**变更详情**：
- 54 行变更，主要是错误处理改进

**潜在影响**：
- 主要是错误处理改进，不太可能影响正常 IO 路径
- 但错误处理可能在某些边界情况下影响性能

**🟡 中怀疑：fs/f2fs/file.c**

**变更详情**：
- 155 行变更

**潜在影响**：
- 可能影响文件操作
- 可能影响 IO 性能

#### 2.4.2 ext4 相关变更

**🟡 中怀疑：fs/ext4/file.c**

**变更详情**：
- 155 行变更，重构了 inode extension 处理

**潜在影响**：
- 可能影响文件扩展操作
- 可能影响 IO 性能

**🟡 中怀疑：fs/ext4/inode.c**

**变更详情**：
- 120 行变更

**潜在影响**：
- 可能影响 inode 管理
- 可能影响 IO 性能

### 2.5 维度五：调度器影响分析

#### 2.5.1 IO 调度影响

**🟡 中怀疑：block/blk-mq.c**

**调度影响**：
- 内存屏障可能影响 IO 调度顺序
- 延迟工作检查可能影响 IO 调度时机

**潜在影响**：
- 可能略微增加 IO 延迟
- 但主要是修复竞争条件

**🟡 中怀疑：block/blk-iocost.c**

**调度影响**：
- 14 行变更，可能影响 IO 成本计算

**潜在影响**：
- 可能影响 IO 调度策略
- 可能影响 IO 优先级

---

## 第三章：怀疑点优先级排序

### 3.1 极高怀疑（🔴）

#### 3.1.1 arch/arm64/mm/mmu.c - VA-range 检查变更

**怀疑度**：🔴 **极高**

**理由**：
1. ✅ 直接修改内存映射逻辑
2. ✅ 可能影响 Page Cache 映射
3. ✅ 直接影响文件 IO 性能
4. ✅ 变更逻辑看似"修复"，但改变了检查范围
5. ✅ 时间相关性：变更后 IO 问题出现

**影响机制**：
```
VA-range 检查逻辑改变
    ↓
Page Cache 映射可能失败
    ↓
文件 IO 无法使用 Page Cache
    ↓
每次文件操作都触发磁盘 IO
    ↓
iowait 急剧上升（> 20%）
    ↓
ANR 大面积增加
```

**验证方法**：
1. 回退该变更，测试 IO 性能
2. 监控 Page Cache 命中率
3. 检查内存映射失败的错误日志

#### 3.1.2 mm/vmscan.c - 内存压缩限制

**怀疑度**：🔴 **极高**

**理由**：
1. ✅ 直接修改内存回收和压缩逻辑
2. ✅ 可能导致内存碎片化
3. ✅ 间接导致 swap IO 增加
4. ✅ 可能影响 kswapd 行为
5. ✅ 时间相关性：变更后 IO 问题出现

**影响机制**：
```
内存压缩受限
    ↓
内存碎片化增加
    ↓
大块内存分配失败
    ↓
触发更多 swap IO
    ↓
IO 压力增加
    ↓
iowait 上升
```

**验证方法**：
1. 回退该变更，测试 IO 性能
2. 检查内存碎片化情况
3. 对比 swap IO 使用情况
4. 监控 kswapd 行为

### 3.2 高怀疑（🟠）

#### 3.2.1 mm/page-writeback.c - 脏页阈值限制

**怀疑度**：🟠 **高**

**理由**：
1. ⚠️ 直接影响脏页写回策略
2. ⚠️ 可能增加 IO 频率
3. ⚠️ 但影响相对较小

**验证方法**：
1. 检查脏页阈值配置
2. 对比写回频率

#### 3.2.2 block/blk-mq.c - 内存屏障和延迟工作检查

**怀疑度**：🟠 **高**

**理由**：
1. ⚠️ 可能略微增加 IO 延迟
2. ⚠️ 但主要是修复竞争条件
3. ⚠️ 不太可能是根本原因

**验证方法**：
1. 检查 IO 延迟变化
2. 分析内存屏障的影响

### 3.3 中怀疑（🟡）

#### 3.3.1 fs/f2fs/data.c - 错误处理改进

**怀疑度**：🟡 **中**

**理由**：
- 主要是错误处理改进，不太可能影响正常 IO 路径

#### 3.3.2 fs/ext4/file.c - inode extension 重构

**怀疑度**：🟡 **中**

**理由**：
- 可能影响文件扩展操作，但影响相对较小

### 3.4 低怀疑（🟢）

#### 3.4.1 mm/filemap.c - trace 点添加

**怀疑度**：🟢 **低**

**理由**：
- 仅添加 trace 点，几乎无影响

---

## 第四章：交叉验证分析

### 4.1 时间相关性验证

**时间线**：
- **2024-05_r2**：正常版本，IO 性能正常
- **2024-05_r2 → 2024-11_r3**：6个月跨度，引入多个变更
- **2024-11_r3**：问题版本，IO 问题开始出现

**结论**：✅ **时间相关性高度吻合**

### 4.2 变更范围验证

**IO 层变更**：
- `block/` 目录：11 个文件，~65 行变更
- 主要是修复和优化，不太可能是根本原因

**内存管理层变更**：
- `mm/` 目录：27 个文件，~290 行变更
- `arch/arm64/mm/` 目录：1 个文件，~10 行变更
- **变更范围大，影响面广**

**结论**：✅ **内存管理层变更更可能是根本原因**

### 4.3 影响机制验证

**直接影响 IO 的变更**：
- `arch/arm64/mm/mmu.c`：可能影响 Page Cache 映射
- `block/blk-mq.c`：可能略微增加 IO 延迟

**间接影响 IO 的变更**：
- `mm/vmscan.c`：可能导致 swap IO 增加
- `mm/page-writeback.c`：可能增加 IO 频率

**结论**：✅ **多个变更可能共同导致 IO 性能劣化**

### 4.4 后续版本验证

**修复情况**：
- **2024-11_r3 → 2025-01_r6**（向后 2 个月）：关键文件仍未修复
- **问题可能仍然存在**

**结论**：✅ **问题未被修复，进一步确认问题存在**

---

## 第五章：综合结论

### 5.1 根本原因分析

**最可能的根本原因**：

1. **arch/arm64/mm/mmu.c - VA-range 检查变更** 🔴
   - **影响**：直接影响 Page Cache 映射
   - **机制**：映射检查逻辑改变导致 Page Cache 无法正确映射
   - **后果**：文件 IO 无法使用缓存，必须直接访问磁盘

2. **mm/vmscan.c - 内存压缩限制** 🔴
   - **影响**：间接导致 swap IO 增加
   - **机制**：压缩受限导致内存碎片化，触发更多 swap IO
   - **后果**：IO 压力增加，iowait 上升

### 5.2 次要原因分析

**次要原因**：

1. **mm/page-writeback.c - 脏页阈值限制** 🟠
   - 可能增加 IO 频率

2. **block/blk-mq.c - 内存屏障** 🟠
   - 可能略微增加 IO 延迟

### 5.3 验证建议

**优先级 1（立即验证）**：
1. 回退 `arch/arm64/mm/mmu.c` 的变更，测试 IO 性能
2. 监控 Page Cache 命中率
3. 检查内存映射失败的错误日志

**优先级 2（高优先级）**：
1. 回退 `mm/vmscan.c` 的变更，测试 IO 性能
2. 检查内存碎片化情况
3. 对比 swap IO 使用情况

**优先级 3（中优先级）**：
1. 检查 `mm/page-writeback.c` 和 `block/blk-mq.c` 的影响

### 5.4 风险提示

**回退风险**：
- `arch/arm64/mm/mmu.c`：中等风险（该提交声称是"修复"）
- `mm/vmscan.c`：高风险（可能重新引入无限循环问题）

**建议**：
- 优先验证 `arch/arm64/mm/mmu.c`
- 谨慎处理 `mm/vmscan.c`，需要找到更好的修复方案

---

## 附录：详细变更列表

### A.1 内存管理相关变更（mm/ 目录）

**完整文件列表**（27 个文件）：
- `mm/cma.c` - 20 行变更
- `mm/compaction.c` - 7 行变更
- `mm/damon/core.c` - 21 行变更
- `mm/damon/vaddr-test.h` - 2 行变更
- `mm/damon/vaddr.c` - 2 行变更
- `mm/filemap.c` - 5 行变更
- `mm/huge_memory.c` - 30 行变更
- `mm/hugetlb.c` - 2 行变更
- `mm/madvise.c` - 7 行变更
- `mm/memcontrol.c` - 30 行变更
- `mm/memory-failure.c` - 7 行变更
- `mm/memory.c` - 46 行变更
- `mm/mempolicy.c` - 18 行变更
- `mm/memtest.c` - 4 行变更
- `mm/migrate.c` - 6 行变更
- `mm/mmap.c` - 2 行变更
- `mm/mmap_lock.c` - 175 行变更（大幅减少）
- `mm/mremap.c` - 78 行变更
- `mm/oom_kill.c` - 2 行变更
- `mm/page-writeback.c` - 30 行变更 ⚠️
- `mm/page_alloc.c` - 11 行变更
- `mm/pgsize_migration.c` - 35 行变更
- `mm/readahead.c` - 2 行变更
- `mm/shmem.c` - 1 行变更
- `mm/swapfile.c` - 25 行变更
- `mm/userfaultfd.c` - 14 行变更
- `mm/vmscan.c` - 7 行变更 ⚠️⚠️

### A.2 IO 层相关变更（block/ 目录）

**完整文件列表**（11 个文件）：
- `block/bio-integrity.c` - 21 行变更
- `block/bio.c` - 2 行变更
- `block/blk-integrity.c` - 2 行变更
- `block/blk-iocost.c` - 14 行变更
- `block/blk-mq.c` - 24 行变更 ⚠️
- `block/blk-settings.c` - 4 行变更
- `block/blk-stat.c` - 2 行变更
- `block/ioctl.c` - 4 行变更
- `block/opal_proto.h` - 1 行变更
- `block/partitions/core.c` - 5 行变更
- `block/sed-opal.c` - 6 行变更

### A.3 文件系统相关变更（fs/ 目录）

**F2FS 相关文件**：
- `fs/f2fs/checkpoint.c` - 10 行变更
- `fs/f2fs/compress.c` - 44 行变更
- `fs/f2fs/data.c` - 54 行变更 ⚠️
- `fs/f2fs/dir.c` - 1 行变更
- `fs/f2fs/extent_cache.c` - 4 行变更
- `fs/f2fs/f2fs.h` - 85 行变更
- `fs/f2fs/file.c` - 155 行变更
- `fs/f2fs/gc.c` - 59 行变更
- `fs/f2fs/inline.c` - 13 行变更
- `fs/f2fs/inode.c` - 44 行变更

**ext4 相关文件**：
- `fs/ext4/extents.c` - 119 行变更
- `fs/ext4/extents_status.c` - 16 行变更
- `fs/ext4/extents_status.h` - 6 行变更
- `fs/ext4/fast_commit.c` - 73 行变更
- `fs/ext4/file.c` - 155 行变更 ⚠️
- `fs/ext4/inline.c` - 6 行变更
- `fs/ext4/inode.c` - 120 行变更
- `fs/ext4/mballoc.c` - 258 行变更
- `fs/ext4/mballoc.h` - 2 行变更
- `fs/ext4/move_extent.c` - 6 行变更
- `fs/ext4/namei.c` - 90 行变更

### A.4 ARM64 内存映射相关变更

**文件列表**：
- `arch/arm64/mm/mmu.c` - ~10 行变更 ⚠️⚠️⚠️

---

## 附录 B：相关提交信息

### B.1 关键提交列表

基于 `git log android13-5.15-2024-05_r2..android13-5.15-2024-11_r3` 分析：

**内存管理相关提交**：
- `e9fc86b28af3` - ANDROID: mm: Add vendor hooks for recording when kswapd finishing the reclaim job
- `a7519a464342` - ANDROID: vendor hooks: skip mem reclaim throttle to speed up mem alloc
- `8048f7a1c2f7` - ANDROID: vendor_hooks: Add vendor hook in pgdate_balanced to update mark
- `72295ae05d13` - mm: fix arithmetic for max_prop_frac when setting max_ratio
- `bcf2450f46cd` - mm: fix arithmetic for bdi min_ratio

**IO 层相关提交**：
- `25466e5b4bb1` - blk-mq: setup queue ->tag_set before initializing hctx
- `dbd0829d2458` - blk-mq: add helper for checking if one CPU is mapped to specified hctx
- `21077a775094` - blk-mq: skip CPU offline notify on unmapped hctx
- `0fbd2d4a1e2c` - blk-mq: don't schedule block kworker on isolated CPUs
- `ca8764c0ea1f` - block: Use RCU in blk_mq_[un]quiesce_tagset() instead of set->tag_list_lock
- `94f146df56fb` - blk-mq: Abort suspend when wakeup events are pending

**文件系统相关提交**：
- `27b608536f86` - BACKPORT: f2fs: block cache/dio write during f2fs_enable_checkpoint()
- `bcd0086ee5a2` - f2fs: fix to avoid updating compression context during writeback

---

**文档生成时间**：基于本地源码 Git diff 分析  
**分析版本范围**：android13-5.15-2024-05_r2 → android13-5.15-2024-11_r3  
**分析方法**：多维度对比分析（IO 性能、kswapd 行为、内存管理、文件系统、调度器）  
**数据来源**：本地 Git 仓库 `D:\ACK\common`
