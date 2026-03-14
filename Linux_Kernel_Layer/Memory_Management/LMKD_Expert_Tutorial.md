# LMKD 专家

## 📋 学习目标

精通 LMKD 性能分析和问题诊断，能够处理复杂的内存问题、版本劣化问题，进行深度性能调优。

## 📚 学习内容

### 第一章：复杂问题诊断

#### 1.1 高 iowait 与 kswapd 异常诊断

**问题场景**：

在版本劣化分析中，如果发现：
- `iowait` 持续高于 20%
- `kswapd` 频繁唤醒
- PSI memory 压力不高但系统卡顿
- ANR 大面积增加

**诊断流程**：

```
1. 确认问题现象
   ├─ iowait 高
   ├─ kswapd 活跃
   └─ PSI memory 压力不高
   
2. 分析根本原因
   ├─ 检查内核内存管理变更
   ├─ 检查回收策略变化
   └─ 检查页面缓存行为
   
3. 定位问题代码
   ├─ 对比内核版本差异
   ├─ 分析 mm/vmscan.c 变更
   └─ 分析 arch/arm64/mm/mmu.c 变更
   
4. 验证和修复
   ├─ 回退可疑变更
   ├─ 调整回收参数
   └─ 验证修复效果
```

**关键检查点**：

1. **检查 kswapd 活动**
   ```bash
   # 查看 kswapd 唤醒次数
   adb shell cat /proc/vmstat | grep kswapd
   
   # 查看 kswapd CPU 使用
   adb shell top -H | grep kswapd
   
   # 查看 kswapd 堆栈
   adb shell cat /proc/$(pidof kswapd0)/stack
   ```

2. **检查内存回收效率**
   ```bash
   # 查看内存回收统计
   adb shell cat /proc/vmstat | grep -E "pgscan|pgsteal|pgmajfault"
   
   # 计算回收效率
   # 效率 = pgsteal / pgscan
   # 如果效率低，说明回收效果差
   ```

3. **检查页面缓存行为**
   ```bash
   # 查看页面缓存大小
   adb shell cat /proc/meminfo | grep -E "Cached|Buffers"
   
   # 查看页面缓存命中率
   adb shell cat /proc/vmstat | grep -E "pgpgin|pgpgout"
   ```

**可能原因分析**：

1. **内存回收过于激进**
   - 内核 `mm/vmscan.c` 变更导致回收策略变化
   - 频繁回收页面缓存，导致大量 IO
   - 解决方案：调整回收参数或回退变更

2. **内存碎片严重**
   - 内存压缩（compaction）被限制
   - 导致无法分配连续内存，触发更多回收
   - 解决方案：检查 `gfp_compaction_allowed()` 相关变更

3. **页面缓存映射问题**
   - `arch/arm64/mm/mmu.c` 变更影响页面缓存映射
   - 导致页面缓存失效，需要从磁盘重新读取
   - 解决方案：检查虚拟地址范围检查逻辑

#### 1.2 PSI 压力不高但系统卡顿

**问题现象**：
- PSI memory some < 5%
- 但系统明显卡顿
- 应用响应慢

**可能原因**：

1. **IO 压力而非内存压力**
   ```bash
   # 检查 IO 压力
   adb shell cat /proc/pressure/io
   
   # 如果 IO some 很高，说明是 IO 问题
   ```

2. **CPU 调度问题**
   ```bash
   # 检查 CPU 压力
   adb shell cat /proc/pressure/cpu
   
   # 检查 CPU 使用率
   adb shell top
   ```

3. **内存分配延迟**
   - 虽然 PSI 压力不高，但内存分配可能很慢
   - 检查 direct reclaim 频率
   ```bash
   # 查看 direct reclaim 统计
   adb shell cat /proc/vmstat | grep pgmajfault
   ```

**诊断方法**：

```bash
# 综合压力监控脚本
#!/bin/bash
while true; do
    clear
    echo "=== PSI Pressure ==="
    echo "Memory: $(adb shell cat /proc/pressure/memory)"
    echo "IO: $(adb shell cat /proc/pressure/io)"
    echo "CPU: $(adb shell cat /proc/pressure/cpu)"
    echo ""
    echo "=== Memory Stats ==="
    adb shell cat /proc/meminfo | head -10
    echo ""
    echo "=== kswapd Activity ==="
    adb shell cat /proc/vmstat | grep kswapd
    sleep 2
done
```

#### 1.3 LMKD 杀进程但内存不释放

**问题现象**：
- LMKD 频繁杀进程
- 但系统内存仍然紧张
- 内存释放效果不明显

**诊断步骤**：

1. **检查被杀进程的内存**
   ```bash
   # 查看被杀进程的内存使用
   adb logcat | grep "Killing" | tail -10
   
   # 检查这些进程实际释放了多少内存
   adb shell dumpsys meminfo | grep -A 5 "Total PSS"
   ```

2. **检查内存泄漏**
   ```bash
   # 查看系统内存趋势
   adb shell dumpsys meminfo | grep "Total RAM"
   
   # 如果持续增长，可能有内存泄漏
   ```

3. **检查内核内存使用**
   ```bash
   # 查看内核内存
   adb shell cat /proc/meminfo | grep -E "Slab|KernelStack"
   
   # 如果内核内存很大，可能是内核泄漏
   ```

**可能原因**：

1. **被杀进程内存未完全释放**
   - 进程被杀后，内存可能被其他进程共享
   - 实际释放的内存少于预期

2. **内核内存泄漏**
   - 内核模块或驱动泄漏内存
   - 需要检查内核日志和内存统计

3. **内存碎片**
   - 内存碎片导致无法有效利用
   - 需要检查内存碎片情况

---

### 第二章：版本劣化问题分析

#### 2.1 内核版本升级导致的问题

**问题场景**：

从 `android13-5.15-2024-05_r2` 升级到 `android13-5.15-2024-11_r3` 后：
- IO 性能严重劣化
- `iowait` 持续高于 20%
- ANR 大面积增加
- PSI memory 压力不高但 `kswapd` 异常活跃

**分析方法**：

1. **对比内核版本差异**
   ```bash
   # 在本地仓库中对比
   cd D:\ACK\common
   git diff android13-5.15-2024-05_r2 android13-5.15-2024-11_r3 -- mm/vmscan.c
   git diff android13-5.15-2024-05_r2 android13-5.15-2024-11_r3 -- arch/arm64/mm/mmu.c
   ```

2. **识别可疑变更**
   - 关注内存回收逻辑变更
   - 关注页面缓存处理变更
   - 关注内存分配策略变更

3. **分析影响机制**
   - 变更如何影响 IO 性能
   - 变更如何影响内存回收
   - 变更如何影响页面缓存

#### 2.2 关键变更分析案例

**案例 1：mm/vmscan.c 的 compaction 限制**

**变更内容**：
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
```

**影响分析**：
- **问题**：限制了内存压缩（compaction）的使用
- **后果**：内存碎片增加，无法分配连续内存
- **连锁反应**：
  1. 内存分配失败增加
  2. 触发更多 direct reclaim
  3. 频繁回收页面缓存
  4. 大量 IO 操作
  5. `iowait` 升高

**验证方法**：
```bash
# 检查内存碎片
adb shell cat /proc/buddyinfo

# 检查 compaction 活动
adb shell cat /proc/vmstat | grep compact
```

**案例 2：arch/arm64/mm/mmu.c 的虚拟地址检查**

**变更内容**：
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
```

**影响分析**：
- **问题**：虚拟地址范围检查逻辑变更
- **后果**：可能影响页面缓存映射
- **连锁反应**：
  1. 页面缓存映射可能失效
  2. 需要从磁盘重新读取
  3. 增加 IO 操作
  4. `iowait` 升高

**验证方法**：
```bash
# 检查页面缓存命中率
adb shell cat /proc/vmstat | grep -E "pgmajfault|pgpgin"

# 检查页面缓存大小变化
adb shell watch -n 1 'cat /proc/meminfo | grep Cached'
```

#### 2.3 后续版本修复验证

**验证策略**：

1. **检查后续版本是否有修复**
   ```bash
   # 检查 r3 之后的版本
   cd D:\ACK\common
   
   # 检查 r4 到 r14
   for tag in android13-5.15-2024-11_r{4..14}; do
       echo "Checking $tag..."
       git diff $tag android13-5.15-2024-11_r3 -- mm/vmscan.c
   done
   
   # 检查 2025-01 版本
   for tag in android13-5.15-2025-01_r{1..6}; do
       echo "Checking $tag..."
       git diff $tag android13-5.15-2024-11_r3 -- mm/vmscan.c
   done
   ```

2. **查找相关修复提交**
   ```bash
   # 查找 revert 提交
   git log --grep="revert" --oneline android13-5.15-2024-11_r3..HEAD
   
   # 查找相关修复
   git log --grep="vmscan\|compaction\|mmu" --oneline android13-5.15-2024-11_r3..HEAD
   ```

3. **验证修复效果**
   - 如果找到修复，应用修复并测试
   - 对比修复前后的性能指标
   - 确认问题是否解决

---

### 第三章：深度性能调优

#### 3.1 LMKD 响应时间优化

**优化目标**：
- 减少从 PSI 事件到进程终止的时间
- 提高内存释放效率
- 减少系统卡顿

**优化方法**：

1. **优化进程选择算法**
   ```c
   // 使用更高效的数据结构
   // 例如：使用红黑树按 oom_score_adj 排序
   // 快速找到目标进程
   ```

2. **并行处理**
   ```c
   // 如果内存压力严重，可以并行终止多个进程
   // 但需要注意避免过度杀死
   ```

3. **预计算进程信息**
   ```c
   // 定期更新进程信息，避免在压力时计算
   // 减少响应时间
   ```

#### 3.2 内存压力预测和预防

**预测模型**：

```c
// 基于 PSI 趋势预测
bool predict_critical_pressure(void) {
    uint64_t psi_avg10 = get_psi_avg10();
    uint64_t psi_avg60 = get_psi_avg60();
    uint64_t psi_avg300 = get_psi_avg300();
    
    // 如果短期压力快速上升
    if (psi_avg10 > psi_avg60 * 1.5) {
        return true;
    }
    
    // 如果压力持续上升
    if (psi_avg60 > psi_avg300 * 1.2) {
        return true;
    }
    
    return false;
}
```

**预防性清理**：

- 当预测到压力会上升时，提前清理一些进程
- 避免等到严重压力时才采取行动
- 提供更平滑的用户体验

#### 3.3 与内核内存管理的协同优化

**优化策略**：

1. **给 kswapd 更多时间**
   ```bash
   # 提高 PSI 阈值，避免过早触发 LMKD
   setprop ro.lmk.psi_partial_stall_ms 100
   ```

2. **优化回收策略**
   ```bash
   # 调整 swappiness，平衡匿名页和文件页回收
   echo 60 > /proc/sys/vm/swappiness
   ```

3. **优化内存分配**
   ```bash
   # 调整内存分配参数
   echo 1024 > /proc/sys/vm/min_free_kbytes
   ```

**协同监控**：

```bash
# 综合监控脚本
#!/bin/bash
while true; do
    clear
    echo "=== Memory Pressure ==="
    adb shell cat /proc/pressure/memory
    echo ""
    echo "=== kswapd Activity ==="
    adb shell cat /proc/vmstat | grep kswapd
    echo ""
    echo "=== LMKD Activity ==="
    adb logcat -d | grep lmkd | tail -5
    echo ""
    echo "=== Memory Usage ==="
    adb shell dumpsys meminfo | head -15
    sleep 2
done
```

#### 3.4 透明大页 (THP) 优化

**THP 的影响**：

- 透明大页可以减少 TLB Miss
- 但在内存碎片严重时，合并大页会导致卡顿
- 需要根据实际情况调整

**优化方法**：

```bash
# 检查 THP 状态
adb shell cat /sys/kernel/mm/transparent_hugepage/enabled

# 调整 THP 策略
# always: 总是使用大页
# madvise: 仅在明确请求时使用
# never: 不使用大页
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

**在内存压力下的建议**：

- 如果内存碎片严重，考虑禁用 THP
- 如果内存充足，可以启用 THP 提升性能
- 需要根据实际测试结果调整

---

### 第四章：高级工具和脚本

#### 4.1 综合诊断脚本

**完整诊断脚本**：

```bash
#!/bin/bash
# lmkd_diagnosis.sh

echo "=== LMKD Diagnosis Report ==="
echo "Time: $(date)"
echo ""

echo "=== 1. LMKD Process Status ==="
adb shell ps -A | grep lmkd
echo ""

echo "=== 2. PSI Pressure ==="
adb shell cat /proc/pressure/memory
echo ""

echo "=== 3. Memory Stats ==="
adb shell cat /proc/meminfo | head -20
echo ""

echo "=== 4. kswapd Activity ==="
adb shell cat /proc/vmstat | grep kswapd
echo ""

echo "=== 5. LMKD Recent Kills ==="
adb logcat -d | grep "Killing" | tail -10
echo ""

echo "=== 6. Top Memory Consumers ==="
adb shell dumpsys meminfo | grep -A 1 "Total PSS" | head -20
echo ""

echo "=== 7. Process oom_score_adj ==="
adb shell ps -eo pid,comm,oom_score_adj | sort -k3 -n | head -20
```

#### 4.2 性能监控脚本

**实时监控脚本**：

```bash
#!/bin/bash
# lmkd_monitor.sh

while true; do
    clear
    echo "=== LMKD Real-time Monitor ==="
    echo "Time: $(date +%H:%M:%S)"
    echo ""
    
    # PSI Pressure
    echo "PSI Memory:"
    adb shell cat /proc/pressure/memory | sed 's/^/  /'
    echo ""
    
    # Memory Usage
    echo "Memory Usage:"
    adb shell dumpsys meminfo | grep -E "Total RAM|Free RAM|Used RAM" | sed 's/^/  /'
    echo ""
    
    # kswapd Activity
    echo "kswapd Activity:"
    adb shell cat /proc/vmstat | grep kswapd | sed 's/^/  /'
    echo ""
    
    # Recent Kills
    echo "Recent Kills (last 3):"
    adb logcat -d | grep "Killing" | tail -3 | sed 's/^/  /'
    echo ""
    
    sleep 2
done
```

#### 4.3 压力测试脚本

**内存压力测试**：

```bash
#!/bin/bash
# memory_stress_test.sh

echo "Starting memory stress test..."
echo ""

# 启动压力测试
adb shell stress-ng --vm 4 --vm-bytes 2G --timeout 60s &
STRESS_PID=$!

# 监控 LMKD 响应
echo "Monitoring LMKD response..."
adb logcat -c  # 清空日志
adb logcat | grep lmkd &
LOGCAT_PID=$!

# 等待压力测试完成
wait $STRESS_PID

# 停止监控
kill $LOGCAT_PID

echo ""
echo "Stress test completed."
echo "Check logcat output for LMKD activity."
```

---

### 第五章：最佳实践和总结

#### 5.1 LMKD 调优最佳实践

**配置原则**：

1. **平衡响应速度和用户体验**
   - 不要设置过低的 PSI 阈值（会导致频繁杀进程）
   - 不要设置过高的 PSI 阈值（会导致系统卡顿）

2. **根据设备特性调整**
   - 低内存设备：更积极的策略
   - 高内存设备：更保守的策略

3. **监控和验证**
   - 调整后需要监控效果
   - 根据实际效果进一步优化

**调优检查清单**：

- [ ] PSI 阈值设置合理
- [ ] 杀进程策略适合场景
- [ ] 与内核 kswapd 协同良好
- [ ] 监控和日志完善
- [ ] 性能指标符合预期

#### 5.2 问题排查流程

**标准排查流程**：

```
1. 确认问题现象
   ├─ 收集日志和指标
   ├─ 确认问题范围
   └─ 记录问题时间线
   
2. 分析可能原因
   ├─ 检查系统配置
   ├─ 检查内核版本
   └─ 检查相关变更
   
3. 定位根本原因
   ├─ 对比版本差异
   ├─ 分析代码变更
   └─ 验证影响机制
   
4. 制定解决方案
   ├─ 回退变更
   ├─ 调整参数
   └─ 优化策略
   
5. 验证和监控
   ├─ 应用解决方案
   ├─ 验证效果
   └─ 持续监控
```

#### 5.3 经验总结

**常见问题模式**：

1. **版本升级导致的问题**
   - 内核内存管理变更
   - 回收策略变化
   - 需要仔细对比版本差异

2. **配置不当导致的问题**
   - PSI 阈值设置不合理
   - 杀进程策略不当
   - 需要根据实际情况调整

3. **硬件限制导致的问题**
   - 内存不足
   - 存储性能差
   - 需要优化应用或升级硬件

**关键经验**：

1. **理解系统整体**
   - LMKD 不是孤立的
   - 需要理解与内核的协同
   - 需要理解与应用的交互

2. **数据驱动决策**
   - 基于 PSI 数据调整
   - 基于实际效果优化
   - 避免盲目调优

3. **持续监控和优化**
   - 建立监控体系
   - 定期检查和分析
   - 持续优化和改进

---

## 📝 学习检查点

完成专家学习后，您应该能够：

- [ ] 能够诊断复杂的内存问题
- [ ] 能够分析版本劣化问题
- [ ] 能够进行深度性能调优
- [ ] 能够处理高 iowait 和 kswapd 异常
- [ ] 能够使用高级工具和脚本
- [ ] 能够制定最佳实践和解决方案

---

**完成**：恭喜您完成了 LMKD 从基础到专家的完整学习路径！现在您已经具备了深入理解和优化 LMKD 的能力。
