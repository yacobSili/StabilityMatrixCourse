# Kernel r2到r3 IO劣化根因分析

## 问题背景

- **升级路径**: android13-5.15-2024-05_r2 → 2024-11_r3
- **现象**: ANR激增20倍，每次ANR堆栈都显示PSI的IO压力异常高（94.55%）
- **关键发现**: IO劣化导致swap风暴，形成恶性循环

## 问题机制（已验证）

根据你的分析，问题的根本机制是：

```
Kernel tag更新导致IO性能劣化
    ↓
kswapd0的swap操作变慢（IO被阻塞）
    ↓
内存回收不及时
    ↓
内存压力增大，需要swap更多页面
    ↓
产生大量major faults（30,000+）
    ↓
更多IO操作，IO压力进一步增大
    ↓
恶性循环...
```

**关键证据**：
- IO压力先升高（530949: 48% → 531072: 63%），而非内存压力先升高
- kswapd0的CPU使用率与iowait呈反比（IO高时CPU下降）
- IO压力（94.55%）远高于内存压力（58.78%）

## r2到r3可能引入的IO劣化原因

### 1. blk_mq dispatch budget机制的变化

#### 代码位置
- `block/blk-mq-sched.c`: `__blk_mq_do_dispatch_sched()`
- `block/blk-mq.c`: `blk_mq_prep_dispatch_rq()`
- `block/blk-mq.h`: `blk_mq_get_dispatch_budget()`

#### 关键机制
```c
// 在dispatch请求前需要获取budget
budget_token = blk_mq_get_dispatch_budget(q);
if (budget_token < 0)
    break;  // 无法获取budget，停止dispatch

// 只有SCSI实现了get_budget/put_budget
// 注释说明：Only SCSI implements .get_budget and .put_budget
```

#### 可能的问题
1. **budget获取失败导致IO请求被阻塞**
   - 如果r2到r3之间引入了更严格的budget检查
   - swap IO请求可能因为无法获取budget而被阻塞
   - 导致IO请求堆积，IO压力升高

2. **budget释放时机不当**
   - 如果budget在IO完成前就被释放，可能导致资源竞争
   - 或者budget持有时间过长，阻塞其他请求

3. **nr_budgets参数的变化**
   ```c
   bool blk_mq_dispatch_rq_list(struct blk_mq_hw_ctx *hctx, 
                                struct list_head *list,
                                unsigned int nr_budgets)
   ```
   - 如果`nr_budgets`的处理逻辑发生变化
   - 可能影响批量IO请求的dispatch效率

### 2. swap IO的优先级处理

#### 代码位置
- `mm/page_io.c`: `__swap_writepage()`, `swap_readpage()`

#### 关键代码
```c
// swap写操作
bio->bi_opf = REQ_OP_WRITE | REQ_SWAP | wbc_to_write_flags(wbc);

// swap读操作（同步时）
if (synchronous) {
    bio->bi_opf |= REQ_HIPRI;  // 高优先级
    get_task_struct(current);
    bio->bi_private = current;
}
```

#### 可能的问题
1. **REQ_SWAP标志的处理变化**
   - 如果IO调度器对REQ_SWAP的处理逻辑改变
   - 可能导致swap IO的优先级降低或被延迟

2. **REQ_HIPRI的使用范围**
   - 只有同步swap读才使用REQ_HIPRI
   - 如果异步swap IO的处理逻辑变化，可能导致延迟增加

### 3. PSI IO压力计算的准确性

#### 代码位置
- `kernel/sched/psi.c`: `psi_group_change()`, `test_state()`

#### 关键逻辑
```c
case PSI_IO_SOME:
    return unlikely(tasks[NR_IOWAIT]);
case PSI_IO_FULL:
    return unlikely(tasks[NR_IOWAIT] && !tasks[NR_RUNNING]);
```

#### 可能的问题
1. **IO等待时间的计算变化**
   - 如果IO等待时间的统计方式改变
   - 可能导致PSI IO压力值异常高
   - 但这更可能是结果而非原因

2. **任务状态转换的时机**
   - 如果任务进入/退出IO等待状态的时机变化
   - 可能影响PSI计算的准确性

### 4. blk_mq tag分配机制

#### 代码位置
- `block/blk-mq.c`: `blk_mq_get_driver_tag()`
- `block/blk-mq-tag.c`: tag分配逻辑

#### 可能的问题
1. **tag分配策略变化**
   - 如果tag分配变得更保守（更早返回失败）
   - 可能导致IO请求无法及时获得tag，被阻塞

2. **tag等待机制**
   ```c
   if (!blk_mq_mark_tag_wait(hctx, rq)) {
       // tag获取失败，需要等待
   }
   ```
   - 如果tag等待机制变化，可能导致等待时间增加

### 5. IO调度器的变化

#### 可能的问题
1. **mq-deadline调度器的参数变化**
   - 如果deadline时间或批处理大小改变
   - 可能影响swap IO的响应时间

2. **kyber调度器的tunable变化**
   - kyber有多个tunable参数
   - 如果默认值或计算逻辑变化，可能影响IO性能

## 重点分析方向

### 1. dispatch budget机制（最可疑）

**原因**：
- 注释明确说明"Only SCSI implements .get_budget and .put_budget"
- 如果Android设备使用SCSI或类似驱动，budget机制直接影响IO dispatch
- budget获取失败会导致IO请求被阻塞，正好符合观察到的现象

**验证方法**：
1. 检查r2到r3之间`blk_mq_get_dispatch_budget()`的调用逻辑变化
2. 检查`nr_budgets`参数的处理变化
3. 检查budget获取失败后的处理逻辑

### 2. swap IO的REQ_SWAP标志处理

**原因**：
- swap IO使用特殊的REQ_SWAP标志
- 如果IO调度器或底层驱动对REQ_SWAP的处理变化
- 可能导致swap IO优先级降低或被延迟

**验证方法**：
1. 检查IO调度器对REQ_SWAP的处理逻辑
2. 检查底层驱动对REQ_SWAP的响应
3. 对比r2和r3中swap IO的优先级设置

### 3. blk_mq tag分配策略

**原因**：
- tag是IO请求dispatch的必要资源
- 如果tag分配策略变得更保守
- 可能导致swap IO请求无法及时获得tag

**验证方法**：
1. 检查`blk_mq_get_driver_tag()`的实现变化
2. 检查tag等待机制的变化
3. 检查tag分配失败后的处理逻辑

## 建议的排查步骤

### 1. 代码对比分析
```bash
# 对比r2和r3之间的关键文件变化
git diff android13-5.15-2024-05_r2..android13-5.15-2024-11_r3 -- \
    block/blk-mq*.c \
    block/blk-mq*.h \
    mm/page_io.c \
    kernel/sched/psi.c
```

### 2. 运行时验证
- 监控budget获取失败的情况
- 监控tag分配失败的情况
- 监控swap IO的延迟分布
- 对比r2和r3的IO性能指标

### 3. 回退测试
- 回退到r2版本，验证问题消失
- 逐步应用r2到r3的补丁，定位具体引入问题的补丁

### 4. 性能分析
- 使用ftrace跟踪swap IO的完整路径
- 分析IO请求在各个环节的等待时间
- 对比r2和r3的IO路径差异

## 结论

根据代码分析和问题现象，**最可能的原因是blk_mq dispatch budget机制的变化**，导致swap IO请求无法及时dispatch，进而引发恶性循环。

建议优先排查：
1. **dispatch budget机制的实现变化**
2. **swap IO的优先级处理变化**
3. **blk_mq tag分配策略的变化**

这些变化可能导致swap IO请求被阻塞，IO压力升高，kswapd0被阻塞，内存回收变慢，最终形成swap风暴。
