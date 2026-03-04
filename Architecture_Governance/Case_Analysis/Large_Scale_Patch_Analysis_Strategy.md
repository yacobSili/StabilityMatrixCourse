# 大规模 Patch 排查策略指南

## 📋 文档概述

本文档提供系统化的方法，用于高效排查大规模代码变更（2000+ 文件）中可能导致 IO 性能劣化的关键 patch。适用于内核版本升级、大型功能合并等场景。

## 🎯 适用场景

- 内核版本升级（如 android13-5.15-2024-05_r2 → 2024-11_r3）
- 大型功能合并
- 大规模代码重构
- 需要快速定位性能劣化根因的场景

---

## 第一章：排查策略总览

### 1.1 核心原则

**分层筛选，重点突破**

1. **按模块分类**：优先排查 IO 核心模块
2. **按影响排序**：优先排查高风险变更
3. **系统化记录**：建立排查清单，避免遗漏
4. **工具辅助**：使用脚本和工具提高效率

### 1.2 排查流程

```
大规模 Patch (2000+ 文件)
    ↓
第一步：按模块分类筛选
    ↓
第二步：建立优先级清单
    ↓
第三步：逐个分析关键文件
    ↓
第四步：风险评估和验证
    ↓
第五步：记录排查结论
```

---

## 第二章：模块分类与优先级

### 2.1 优先级 1：IO 核心模块（必须重点分析）⭐⭐⭐⭐⭐

这些模块直接影响 IO 性能，必须优先排查。

#### 2.1.1 文件系统层 (`fs/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `fs/f2fs/*.c` | ⭐⭐⭐⭐⭐ | F2FS 是 Android 常用文件系统，直接影响文件 IO |
| `fs/ext4/*.c` | ⭐⭐⭐⭐ | ext4 文件系统，可能影响系统分区 IO |
| `fs/read_write.c` | ⭐⭐⭐⭐ | 文件读写核心函数 |
| `fs/file_table.c` | ⭐⭐⭐ | 文件表管理 |
| `fs/buffer.c` | ⭐⭐⭐ | 缓冲区管理 |

**排查重点**：
- 写回策略变更
- 元数据管理变更
- 文件操作路径变更
- 同步/异步 IO 处理变更

#### 2.1.2 Page Cache 层 (`mm/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `mm/filemap.c` | ⭐⭐⭐⭐⭐ | Page Cache 核心，直接影响文件 IO 性能 |
| `mm/page-writeback.c` | ⭐⭐⭐⭐⭐ | 脏页写回机制，直接影响 IO 延迟 |
| `mm/readahead.c` | ⭐⭐⭐⭐ | 预读机制，影响顺序 IO 性能 |
| `mm/truncate.c` | ⭐⭐⭐ | 文件截断，可能影响 IO |

**排查重点**：
- Page Cache 管理逻辑变更
- 写回策略和阈值变更
- 预读算法变更
- 缓存淘汰策略变更

#### 2.1.3 内存管理 (`mm/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `mm/vmscan.c` | ⭐⭐⭐⭐⭐ | 内存回收（kswapd），影响 swap IO |
| `mm/swap*.c` | ⭐⭐⭐⭐ | Swap 机制，直接影响 swap IO |
| `mm/page_alloc.c` | ⭐⭐⭐ | 页面分配，可能影响 Page Cache |
| `mm/shmem.c` | ⭐⭐⭐ | 共享内存，可能影响文件映射 |

**排查重点**：
- 内存回收策略变更
- Swap 策略变更
- 内存分配策略变更
- 可能影响 Page Cache 的变更

#### 2.1.4 Block I/O 层 (`block/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `block/blk-core.c` | ⭐⭐⭐⭐⭐ | Block 层核心，IO 请求处理 |
| `block/blk-mq*.c` | ⭐⭐⭐⭐⭐ | 多队列机制，现代 IO 核心 |
| `block/elevator.c` | ⭐⭐⭐⭐ | IO 调度器 |
| `block/blk-merge.c` | ⭐⭐⭐⭐ | IO 请求合并 |
| `block/blk-settings.c` | ⭐⭐⭐ | IO 参数设置 |

**排查重点**：
- IO 调度策略变更
- 请求合并逻辑变更
- 队列深度限制变更
- 超时处理变更

#### 2.1.5 ARM64 内存映射 (`arch/arm64/mm/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `arch/arm64/mm/mmu.c` | ⭐⭐⭐⭐⭐ | 内存映射，影响 Page Cache 映射 |
| `arch/arm64/mm/fault.c` | ⭐⭐⭐⭐ | 页面错误处理 |
| `arch/arm64/mm/pageattr.c` | ⭐⭐⭐ | 页面属性管理 |

**排查重点**：
- 内存映射范围检查变更
- 页面错误处理变更
- 可能影响 Page Cache 映射的变更

### 2.2 优先级 2：调度和性能相关 ⭐⭐⭐

#### 2.2.1 进程调度 (`kernel/sched/`)

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `kernel/sched/core.c` | ⭐⭐⭐ | 调度核心 |
| `kernel/sched/fair.c` | ⭐⭐⭐ | CFS 调度器 |
| `kernel/sched/rt.c` | ⭐⭐ | 实时调度 |

**排查重点**：
- IO 等待进程的调度策略变更
- iowait 计算方式变更

### 2.3 优先级 3：驱动层 ⭐⭐

#### 2.3.1 存储驱动

**关键文件**：

| 文件路径 | 优先级 | 影响说明 |
|---------|--------|---------|
| `drivers/mmc/` | ⭐⭐ | eMMC 驱动 |
| `drivers/block/` | ⭐⭐ | 块设备驱动 |

**排查重点**：
- 驱动性能优化或退化
- 错误处理变更

---

## 第三章：Gerrit 排查方法

### 3.1 在 Gerrit 网页上筛选

#### 方法 1：使用文件列表筛选

1. **打开文件列表**：
   - 在 Gerrit 页面，点击 "Files" 标签
   - 查看所有修改的文件列表

2. **使用搜索框筛选**：
   - 在文件列表上方找到搜索框
   - 输入路径关键词：
     ```
     mm/filemap
     mm/page-writeback
     mm/vmscan
     fs/f2fs
     block/blk
     arch/arm64/mm
     ```

3. **按变更大小排序**：
   - 点击 "Delta" 列标题进行排序
   - 优先查看变更量大的文件
   - 但也要关注小变更但关键的文件

#### 方法 2：逐个查看关键文件

1. **点击文件名**：
   - 在文件列表中，点击关键文件名
   - 查看详细的 diff 视图

2. **查看变更上下文**：
   - 展开查看上下文代码
   - 理解变更的完整逻辑

3. **记录关键变更**：
   - 截图或复制关键代码片段
   - 记录变更摘要

### 3.2 下载完整 Patch（包含源代码变更）

#### ⚠️ 重要说明

**如何获取包含源代码变更的 diff**：

Gerrit 可能提供多种格式的 diff，但只有某些格式包含完整的源代码变更。以下是推荐的方法：

#### 方法 1：下载 ZIP 格式的 Diff（推荐）✅

1. **在 Gerrit 下载菜单中**：
   - 找到 "Patch file" 部分
   - 点击 `854297a.diff.zip`（或类似的 `.diff.zip` 文件）
   - 这会下载包含源代码变更的完整 diff

2. **解压文件**：
   ```bash
   unzip 854297a.diff.zip
   # 或使用 PowerShell
   Expand-Archive -Path 854297a.diff.zip -DestinationPath ./
   ```

3. **验证内容**：
   ```bash
   # 检查是否包含源代码文件
   grep "^diff --git.*\.c$" 854297a.diff | head -10
   grep "^diff --git.*\.h$" 854297a.diff | head -10
   ```

#### 方法 2：使用 Git Format-Patch（最可靠）✅✅

这是**最推荐的方法**，可以确保获取完整的源代码变更：

1. **在 Gerrit 下载菜单中**：
   - 找到 "Format Patch" 命令
   - 复制完整的命令，例如：
     ```bash
     git fetch ssh://jiabo.wang@gerrit-cq.transsion.com:29418/UNISOC/kernel/common refs/changes/40/236740/3 && git format-patch -1 --stdout FETCH_HEAD
     ```

2. **执行命令**：
   ```bash
   # 在本地 git 仓库中执行
   git fetch ssh://jiabo.wang@gerrit-cq.transsion.com:29418/UNISOC/kernel/common refs/changes/40/236740/3
   git format-patch -1 --stdout FETCH_HEAD > kernel_upgrade.diff
   ```

3. **验证**：
   ```bash
   # 检查文件大小（应该很大，包含源代码）
   ls -lh kernel_upgrade.diff
   
   # 检查是否包含源代码文件
   grep "^diff --git.*\.c$" kernel_upgrade.diff | wc -l
   ```

#### 方法 3：使用 Git Checkout + Diff（适合有完整仓库）

如果您有完整的 git 仓库，可以使用：

1. **获取变更**：
   ```bash
   # 复制 Gerrit 中的 "Checkout" 命令
   git fetch ssh://jiabo.wang@gerrit-cq.transsion.com:29418/UNISOC/kernel/common refs/changes/40/236740/3
   git checkout FETCH_HEAD
   ```

2. **生成 diff**：
   ```bash
   # 与基准版本对比
   git diff <base_commit> HEAD > kernel_upgrade.diff
   ```

#### 方法 4：直接下载 Base64 格式（需要解码）

1. **下载 base64 文件**：
   - 点击 `854297a.diff.base64`

2. **解码**：
   ```bash
   # Linux/Mac
   base64 -d 854297a.diff.base64 > 854297a.diff
   
   # Windows PowerShell
   $content = Get-Content 854297a.diff.base64 -Raw
   $bytes = [System.Convert]::FromBase64String($content)
   [System.IO.File]::WriteAllBytes("854297a.diff", $bytes)
   ```

3. **验证**：
   ```bash
   # 检查文件大小（应该很大）
   ls -lh 854297a.diff
   ```

#### ⚠️ 注意事项

1. **避免只包含二进制文件的 diff**：
   - 如果 diff 文件很小（< 1MB），可能只包含二进制文件
   - 包含源代码的 diff 文件通常很大（几十 MB 到几百 MB）

2. **验证方法**：
   ```bash
   # 检查是否包含源代码文件
   grep -c "^diff --git.*\.c$" your_diff_file
   grep -c "^diff --git.*\.h$" your_diff_file
   
   # 如果数量为 0，说明没有源代码变更，需要换方法
   ```

3. **推荐顺序**：
   - 首选：方法 2（Git Format-Patch）- 最可靠
   - 次选：方法 1（ZIP 格式）- 简单快速
   - 备选：方法 3（Git Checkout + Diff）- 需要完整仓库
   - 最后：方法 4（Base64）- 需要额外解码步骤

---

## 第四章：本地分析工具

### 4.1 筛选关键文件脚本

创建 `analyze_patch.sh`：

```bash
#!/bin/bash
# analyze_patch.sh - 分析内核升级 patch，筛选关键文件

PATCH_FILE="${1:-kernel_upgrade.diff}"

if [ ! -f "$PATCH_FILE" ]; then
    echo "错误: 文件 $PATCH_FILE 不存在"
    exit 1
fi

echo "=========================================="
echo "内核升级 Patch 分析报告"
echo "=========================================="
echo ""

echo "=== IO 相关文件统计 ==="
echo ""
echo "文件系统相关 (fs/):"
FS_COUNT=$(grep -c "^diff --git.*fs/" "$PATCH_FILE" 2>/dev/null || echo "0")
echo "  文件数: $FS_COUNT"
echo "  关键文件:"
grep "^diff --git.*fs/" "$PATCH_FILE" | grep -E "(f2fs|ext4|read_write|file_table|buffer)" | head -20

echo ""
echo "Block I/O 相关 (block/):"
BLOCK_COUNT=$(grep -c "^diff --git.*block/" "$PATCH_FILE" 2>/dev/null || echo "0")
echo "  文件数: $BLOCK_COUNT"
echo "  关键文件:"
grep "^diff --git.*block/" "$PATCH_FILE" | grep -E "(blk-core|blk-mq|elevator|blk-merge)" | head -20

echo ""
echo "内存管理相关 (mm/):"
MM_COUNT=$(grep -c "^diff --git.*mm/" "$PATCH_FILE" 2>/dev/null || echo "0")
echo "  文件数: $MM_COUNT"
echo "  关键文件:"
grep "^diff --git.*mm/" "$PATCH_FILE" | grep -E "(filemap|page-writeback|vmscan|readahead|swap)" | head -20

echo ""
echo "ARM64 内存管理 (arch/arm64/mm/):"
ARM64_MM_COUNT=$(grep -c "^diff --git.*arch/arm64/mm/" "$PATCH_FILE" 2>/dev/null || echo "0")
echo "  文件数: $ARM64_MM_COUNT"
echo "  关键文件:"
grep "^diff --git.*arch/arm64/mm/" "$PATCH_FILE" | head -20

echo ""
echo "进程调度相关 (kernel/sched/):"
SCHED_COUNT=$(grep -c "^diff --git.*kernel/sched/" "$PATCH_FILE" 2>/dev/null || echo "0")
echo "  文件数: $SCHED_COUNT"

echo ""
echo "=========================================="
echo "=== 最高优先级文件列表 ==="
echo "=========================================="
echo ""
echo "Page Cache 核心:"
grep "^diff --git.*mm/filemap" "$PATCH_FILE"

echo ""
echo "写回机制:"
grep "^diff --git.*mm/page-writeback" "$PATCH_FILE"

echo ""
echo "内存回收:"
grep "^diff --git.*mm/vmscan" "$PATCH_FILE"

echo ""
echo "F2FS 文件系统:"
grep "^diff --git.*fs/f2fs" "$PATCH_FILE" | head -10

echo ""
echo "Block 层核心:"
grep "^diff --git.*block/blk-core" "$PATCH_FILE"
grep "^diff --git.*block/blk-mq" "$PATCH_FILE" | head -5

echo ""
echo "ARM64 内存映射:"
grep "^diff --git.*arch/arm64/mm/mmu" "$PATCH_FILE"
```

**使用方法**：

```bash
chmod +x analyze_patch.sh
./analyze_patch.sh kernel_upgrade.diff
```

### 4.2 提取单个文件变更

创建 `extract_file_diff.sh`：

```bash
#!/bin/bash
# extract_file_diff.sh - 提取指定文件的 diff

PATCH_FILE="${1:-kernel_upgrade.diff}"
TARGET_FILE="${2}"

if [ -z "$TARGET_FILE" ]; then
    echo "用法: $0 <patch_file> <target_file_path>"
    echo "示例: $0 kernel_upgrade.diff mm/filemap.c"
    exit 1
fi

# 提取指定文件的 diff
awk "/^diff --git.*${TARGET_FILE//\//\\/}/,/^diff --git|^-- $/" "$PATCH_FILE" | head -n -1 > "${TARGET_FILE##*/}.diff"

echo "已提取 ${TARGET_FILE} 的变更到 ${TARGET_FILE##*/}.diff"
```

**使用方法**：

```bash
chmod +x extract_file_diff.sh
./extract_file_diff.sh kernel_upgrade.diff mm/filemap.c
```

### 4.3 统计变更量

创建 `stat_changes.sh`：

```bash
#!/bin/bash
# stat_changes.sh - 统计各模块的变更量

PATCH_FILE="${1:-kernel_upgrade.diff}"

echo "=========================================="
echo "各模块变更统计"
echo "=========================================="
echo ""

echo "文件系统 (fs/):"
grep "^diff --git.*fs/" "$PATCH_FILE" | wc -l

echo "Block I/O (block/):"
grep "^diff --git.*block/" "$PATCH_FILE" | wc -l

echo "内存管理 (mm/):"
grep "^diff --git.*mm/" "$PATCH_FILE" | wc -l

echo "ARM64 内存管理 (arch/arm64/mm/):"
grep "^diff --git.*arch/arm64/mm/" "$PATCH_FILE" | wc -l

echo "进程调度 (kernel/sched/):"
grep "^diff --git.*kernel/sched/" "$PATCH_FILE" | wc -l

echo "驱动层 (drivers/):"
grep "^diff --git.*drivers/" "$PATCH_FILE" | wc -l

echo ""
echo "总文件数:"
grep -c "^diff --git" "$PATCH_FILE"
```

---

## 第五章：排查清单模板

### 5.1 文件排查记录表

| # | 文件路径 | 优先级 | 变更行数 | 关键变更 | 风险评估 | 状态 | 备注 |
|---|---------|--------|---------|---------|---------|------|------|
| 1 | `mm/filemap.c` | ⭐⭐⭐⭐⭐ | +XX -XX | 待分析 | 待评估 | 待排查 | |
| 2 | `mm/page-writeback.c` | ⭐⭐⭐⭐⭐ | +XX -XX | 待分析 | 待评估 | 待排查 | |
| 3 | `mm/vmscan.c` | ⭐⭐⭐⭐⭐ | +XX -XX | 待分析 | 待评估 | 待排查 | |
| 4 | `fs/f2fs/` | ⭐⭐⭐⭐ | +XX -XX | 待分析 | 待评估 | 待排查 | |
| 5 | `block/blk-core.c` | ⭐⭐⭐⭐ | +XX -XX | 待分析 | 待评估 | 待排查 | |
| ... | ... | ... | ... | ... | ... | ... | ... |

### 5.2 关键变更分析模板

对于每个关键文件，记录以下信息：

```markdown
### 文件：`mm/filemap.c`

**基本信息**：
- 变更行数：+150 -80
- 变更类型：逻辑优化/修复/重构

**关键变更**：
1. 函数 `xxx()` 的逻辑变更
   - 变更前：...
   - 变更后：...
   - 影响：...

2. 参数或阈值调整
   - 变更：...
   - 影响：...

**风险评估**：
- IO 路径影响：高/中/低
- 可能影响：...
- 风险等级：⭐⭐⭐⭐⭐

**验证计划**：
- [ ] 回退测试
- [ ] 性能对比
- [ ] 代码审查

**结论**：
- 是否相关：是/否/待验证
- 说明：...
```

---

## 第六章：排查最佳实践

### 6.1 排查顺序建议

1. **第一阶段：IO 核心模块**（1-2 天）
   - `mm/filemap.c` - Page Cache 核心
   - `mm/page-writeback.c` - 写回机制
   - `mm/vmscan.c` - 内存回收
   - `arch/arm64/mm/mmu.c` - 内存映射

2. **第二阶段：文件系统和 Block 层**（2-3 天）
   - `fs/f2fs/` - F2FS 文件系统
   - `block/blk-*.c` - Block 层核心

3. **第三阶段：其他相关模块**（1-2 天）
   - `kernel/sched/` - 进程调度
   - `drivers/mmc/` - eMMC 驱动

### 6.2 重点关注类型

在分析每个文件时，重点关注以下类型的变更：

1. **写回策略变更**：
   - 脏页写回阈值
   - 写回频率
   - 写回时机

2. **内存回收变更**：
   - kswapd 行为
   - 回收阈值
   - Swap 策略

3. **IO 调度变更**：
   - 调度算法
   - 请求合并逻辑
   - 队列深度

4. **条件判断变更**：
   - 范围检查
   - 阈值判断
   - 可能引入 bug 的逻辑简化

5. **同步机制变更**：
   - 锁的使用
   - 等待机制
   - 可能引入阻塞的变更

### 6.3 避免的陷阱

1. **不要只看变更量大的文件**：
   - 小变更但关键的文件可能影响更大
   - 例如：条件判断的简单修改可能引入严重 bug

2. **不要忽略头文件变更**：
   - 头文件变更可能影响接口
   - 可能影响多个模块

3. **不要孤立分析**：
   - 多个小变更组合可能产生大影响
   - 需要整体分析

4. **不要忽略注释和提交信息**：
   - 提交信息可能说明变更原因
   - 可能提到已知问题或修复

---

## 第七章：工具和资源

### 7.1 推荐工具

1. **代码分析工具**：
   - `git diff` - 查看变更
   - `git log` - 查看提交历史
   - `grep` - 搜索关键词

2. **性能分析工具**：
   - `perf` - 性能分析
   - `ftrace` - 函数追踪
   - `iostat` - IO 统计

3. **文档资源**：
   - Linux 内核文档
   - Google 内核更新日志
   - 相关 bug report

### 7.2 参考文档

- [文件系统架构与 IO 路径深度解析](../../05_Kernel_Stability_Core/04_FileSystem/02_VFS_Core/FileSystem_Architecture_IO_Path_Deep_Dive.md)
- [IO 压力 PSI 深度解析](../../05_Kernel_Stability_Core/03_IO_Blocking/IO_Pressure_PSI_Deep_Dive.md)
- [内存压力 PSI 深度解析](../../05_Kernel_Stability_Core/02_Memory_Management/Memory_Pressure_PSI_Deep_Dive.md)

---

## 附录：快速参考

### A.1 关键文件路径速查

```
优先级 ⭐⭐⭐⭐⭐：
- mm/filemap.c
- mm/page-writeback.c
- mm/vmscan.c
- arch/arm64/mm/mmu.c

优先级 ⭐⭐⭐⭐：
- fs/f2fs/
- block/blk-core.c
- block/blk-mq*.c
- mm/readahead.c
- mm/swap*.c

优先级 ⭐⭐⭐：
- fs/ext4/
- block/elevator.c
- kernel/sched/
```

### A.2 排查检查清单

- [ ] 已筛选 IO 核心模块文件
- [ ] 已分析 Page Cache 相关变更
- [ ] 已分析写回机制变更
- [ ] 已分析内存回收变更
- [ ] 已分析文件系统变更
- [ ] 已分析 Block I/O 层变更
- [ ] 已记录所有关键变更
- [ ] 已完成风险评估
- [ ] 已制定验证计划

---

**文档版本**：v1.0  
**最后更新**：2025-03-12  
**维护者**：待补充
