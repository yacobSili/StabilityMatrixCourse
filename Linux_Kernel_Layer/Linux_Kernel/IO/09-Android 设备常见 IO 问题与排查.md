# Android 设备常见 IO 问题与排查

## 学习目标

- 掌握 Android 设备常见的 IO 问题类型
- 学会分析每种问题的根本原因
- 掌握问题排查的具体方法和工具
- 学会制定针对性的优化方案
- 能够独立解决 Android 设备的 IO 性能问题

## 背景介绍

Android 设备在实际使用中会遇到各种 IO 性能问题，如应用启动慢、系统卡顿、IO 吞吐量下降等。这些问题可能由多种原因引起，包括 IO 调度器配置不当、后台任务占用 IO、Kernel 升级导致的参数变化等。本文系统性地介绍 Android 设备常见的 IO 问题，提供详细的排查方法和解决方案。

## 问题 1：应用启动慢

### 问题描述

**现象**：
- 应用冷启动时间明显增加
- 启动过程中界面卡顿
- 用户感知启动速度变慢

**影响**：
- 用户体验差
- 应用评分下降
- 用户流失

### 原因分析

#### 1. 大量小文件 IO

**原因**：
- 应用启动时需要加载大量配置文件、资源文件
- 每个文件都是独立的 IO 请求
- 小文件 IO 效率低，延迟高

**验证方法**：
```bash
# 使用 strace 追踪文件操作
adb shell strace -p <pid> -e trace=open,openat,read 2>&1 | grep -E "\.(so|dex|xml|json)"

# 统计文件打开次数
adb shell strace -p <pid> -e trace=open,openat 2>&1 | wc -l
```

#### 2. Page Cache 未命中

**原因**：
- 应用启动时需要的文件不在 Page Cache 中
- 需要从存储设备读取，延迟高
- 冷启动时 Page Cache 命中率低

**验证方法**：
```bash
# 查看 Page Cache 使用情况
adb shell cat /proc/meminfo | grep -i cache

# 使用 vmtouch 查看文件缓存状态
adb shell vmtouch -v /data/app/com.example.app/base.apk
```

#### 3. IO 优先级不足

**原因**：
- 应用启动时的 IO 请求优先级低
- 被后台任务的 IO 请求阻塞
- 无法及时获得 IO 资源

**验证方法**：
```bash
# 查看应用的 IO 优先级
adb shell ionice -p <pid>

# 查看是否有其他高 IO 占用的进程
adb shell iotop -o -d 1
```

### 排查方法

#### 步骤 1：分析启动过程的 IO 模式

```bash
# 使用 systrace/perfetto 追踪启动过程
python systrace.py -t 10 -o start_trace.html -a com.example.app sched freq idle am wm gfx view binder_driver hal dalvik camera input res

# 分析 trace 文件中的 IO 事件
# 查找 block_rq_issue 和 block_rq_complete 事件
# 分析 IO 延迟分布
```

#### 步骤 2：检查文件访问模式

```bash
# 使用 strace 追踪文件操作
adb shell strace -f -e trace=open,openat,read,pread64 -p <pid> 2>&1 | tee file_trace.log

# 分析文件访问模式
# - 哪些文件被频繁访问
# - 文件访问的顺序
# - 是否有重复访问
```

#### 步骤 3：检查 IO 调度器配置

```bash
# 查看当前调度器
adb shell cat /sys/block/sda/queue/scheduler

# 查看调度器参数
adb shell cat /sys/block/sda/queue/iosched/*

# 检查是否有优化空间
```

### 优化方案

#### 方案 1：提升 IO 优先级

```bash
# 为应用进程设置较高的 IO 优先级
adb shell su -c "ionice -c 2 -n 0 -p <pid>"

# 或者在应用启动时设置
# 需要在 Framework 层或应用层实现
```

#### 方案 2：预加载关键文件

```bash
# 使用 vmtouch 预加载文件到 Page Cache
adb shell vmtouch -t /data/app/com.example.app/base.apk

# 或者在系统启动时预加载
# 需要在 init 脚本或系统服务中实现
```

#### 方案 3：优化 IO 调度器参数

```bash
# 如果使用 mq-deadline，降低读延迟
adb shell su -c "echo 200 > /sys/block/sda/queue/iosched/read_expire"

# 如果使用 kyber，设置更严格的目标延迟
adb shell su -c "echo 1000000 > /sys/block/sda/queue/iosched/read_lat_nsec"
```

#### 方案 4：应用层优化

- 减少启动时的文件访问
- 合并小文件读取
- 使用异步 IO
- 延迟加载非关键资源

## 问题 2：系统卡顿（IO 等待）

### 问题描述

**现象**：
- 系统整体响应变慢
- 界面操作有延迟
- 应用切换卡顿
- 输入响应延迟

**影响**：
- 用户体验严重下降
- 系统流畅度降低
- 可能影响系统稳定性

### 原因分析

#### 1. 后台写回占用 IO

**原因**：
- 系统定期将脏页写回存储设备
- 写回操作占用大量 IO 资源
- 前台应用的 IO 请求被阻塞

**验证方法**：
```bash
# 查看写回相关的进程
adb shell ps aux | grep -E "(kworker|flush)"

# 使用 iotop 查看写 IO
adb shell su -c "iotop -o -d 1" | grep -E "(kworker|flush)"

# 查看脏页数量
adb shell cat /proc/vmstat | grep nr_dirty
```

#### 2. IO 调度器配置不当

**原因**：
- 写请求截止时间过长
- 读请求被写请求阻塞
- 调度器参数不适合当前负载

**验证方法**：
```bash
# 查看调度器配置
adb shell cat /sys/block/sda/queue/scheduler
adb shell cat /sys/block/sda/queue/iosched/*

# 使用 iostat 查看 IO 等待
adb shell su -c "iostat -x 1" | grep await
```

#### 3. 多个进程竞争 IO 资源

**原因**：
- 多个应用同时进行 IO 操作
- IO 资源竞争激烈
- 没有有效的资源隔离

**验证方法**：
```bash
# 使用 iotop 查看所有 IO 占用进程
adb shell su -c "iotop -o -d 1"

# 使用 pidstat 统计各进程 IO
adb shell su -c "pidstat -d 1 10"
```

### 排查方法

#### 步骤 1：确认 IO 压力

```bash
# 使用 iostat 查看 IO 压力
adb shell su -c "iostat -x 1 10"

# 关键指标：
# - %util > 90%：设备饱和
# - await > 50ms：延迟高
# - w_await > r_await：写 IO 延迟高
```

#### 步骤 2：定位 IO 占用进程

```bash
# 使用 iotop 找出 IO 占用最高的进程
adb shell su -c "iotop -o -d 1 -P"

# 重点关注：
# - kworker 进程（写回）
# - 后台应用进程
# - 系统服务进程
```

#### 步骤 3：分析 IO 模式

```bash
# 使用 blktrace 追踪 IO 路径
adb shell su -c "blktrace -d /dev/block/sda -o trace -a write"
# 运行一段时间后停止
adb shell su -c "blkparse -i trace -d trace.bin"

# 分析写 IO 的来源和模式
```

### 优化方案

#### 方案 1：调整写回策略

```bash
# 减少脏页阈值，更频繁但更轻量的写回
adb shell su -c "echo 10 > /proc/sys/vm/dirty_ratio"
adb shell su -c "echo 5 > /proc/sys/vm/dirty_background_ratio"

# 减少脏页写回间隔
adb shell su -c "echo 500 > /proc/sys/vm/dirty_expire_centisecs"
```

#### 方案 2：设置 IO 优先级

```bash
# 为前台应用设置高 IO 优先级
adb shell su -c "ionice -c 2 -n 0 -p <foreground_pid>"

# 为后台写回设置低 IO 优先级
adb shell su -c "ionice -c 3 -p <kworker_pid>"
```

#### 方案 3：优化 IO 调度器参数

```bash
# 如果使用 mq-deadline
# 减少写请求截止时间
adb shell su -c "echo 2000 > /sys/block/sda/queue/iosched/write_expire"

# 增加写请求饥饿阈值，优先处理读请求
adb shell su -c "echo 1 > /sys/block/sda/queue/iosched/writes_starved"
```

#### 方案 4：使用 QoS 控制

```bash
# 使用 cgroup IO 控制器限制后台 IO
mkdir -p /sys/fs/cgroup/background
echo <pid> > /sys/fs/cgroup/background/cgroup.procs
echo "8:16 wbps=1048576" > /sys/fs/cgroup/background/io.max  # 限制 1MB/s
```

## 问题 3：IO 吞吐量下降

### 问题描述

**现象**：
- IO 吞吐量明显低于预期
- 文件复制、应用安装等操作变慢
- 系统整体 IO 性能下降

**影响**：
- 用户体验下降
- 系统性能降低
- 可能影响应用功能

### 原因分析

#### 1. Tag 资源耗尽

**原因**：
- 并发 IO 请求过多
- Tag 数量不足，请求等待
- 无法充分利用硬件能力

**验证方法**：
```bash
# 查看 Tag 使用情况
adb shell cat /sys/block/sda/queue/nr_requests      # 总 Tag 数
adb shell cat /sys/block/sda/queue/nr_active       # 已使用 Tag 数

# 计算使用率
usage=$(echo "scale=2; $(cat /sys/block/sda/queue/nr_active) * 100 / $(cat /sys/block/sda/queue/nr_requests)" | bc)
echo "Tag 使用率: ${usage}%"
```

#### 2. 硬件队列深度不足

**原因**：
- 硬件队列深度配置过小
- 无法充分利用设备并发能力
- 设备性能未完全发挥

**验证方法**：
```bash
# 查看硬件队列数量
adb shell cat /sys/block/sda/queue/nr_hw_queues

# 查看队列深度
adb shell cat /sys/block/sda/queue/nr_requests
```

#### 3. IO 调度器参数不当

**原因**：
- 批处理大小过小
- 请求合并不充分
- 调度器参数不适合当前负载

**验证方法**：
```bash
# 查看调度器参数
adb shell cat /sys/block/sda/queue/iosched/*

# 使用 iostat 查看 IO 模式
adb shell su -c "iostat -x 1" | grep -E "(rkB/s|wkB/s)"
```

### 排查方法

#### 步骤 1：检查 Tag 资源

```bash
# 监控 Tag 使用情况
while true; do
    nr_requests=$(cat /sys/block/sda/queue/nr_requests)
    nr_active=$(cat /sys/block/sda/queue/nr_active)
    usage=$(echo "scale=1; $nr_active * 100 / $nr_requests" | bc)
    echo "Tag: $nr_active/$nr_requests (${usage}%)"
    sleep 1
done
```

#### 步骤 2：分析 IO 模式

```bash
# 使用 iostat 分析 IO 模式
adb shell su -c "iostat -x 1 10"

# 关注：
# - IOPS 是否达到设备上限
# - 平均请求大小
# - 读写比例
```

#### 步骤 3：检查硬件配置

```bash
# 查看设备信息
adb shell cat /sys/block/sda/queue/hw_sector_size
adb shell cat /sys/block/sda/queue/max_sectors_kb
adb shell cat /sys/block/sda/queue/nr_hw_queues
```

### 优化方案

#### 方案 1：增加 Tag 数量

```bash
# 如果硬件支持，增加队列深度
adb shell su -c "echo 256 > /sys/block/sda/queue/nr_requests"

# 注意：需要驱动支持，且会增加内存占用
```

#### 方案 2：优化 IO 调度器参数

```bash
# 如果使用 mq-deadline，增加批处理大小
adb shell su -c "echo 32 > /sys/block/sda/queue/iosched/fifo_batch"

# 如果使用 kyber，调整目标延迟
adb shell su -c "echo 5000000 > /sys/block/sda/queue/iosched/write_lat_nsec"
```

#### 方案 3：优化并发 IO 数量

```bash
# 限制单个进程的并发 IO 数量
# 使用 cgroup IO 控制器
mkdir -p /sys/fs/cgroup/io_limit
echo <pid> > /sys/fs/cgroup/io_limit/cgroup.procs
echo "8:16 riops=1000 wiops=1000" > /sys/fs/cgroup/io_limit/io.max
```

## 问题 4：IO 延迟波动大

### 问题描述

**现象**：
- IO 延迟不稳定，波动大
- 有时延迟很低，有时突然升高
- 用户体验不一致

**影响**：
- 用户体验差
- 应用响应不稳定
- 可能影响实时性要求高的应用

### 原因分析

#### 1. IO 调度器参数不当

**原因**：
- 调度器参数不适合当前负载
- 请求处理顺序不合理
- 批处理导致延迟波动

**验证方法**：
```bash
# 使用 blktrace 分析延迟分布
adb shell su -c "blktrace -d /dev/block/sda -o trace"
# 运行一段时间后停止
adb shell su -c "blkparse -i trace -d trace.bin"
adb shell su -c "btt -i trace.bin -l latency.dat"

# 分析延迟分布
```

#### 2. 后台任务干扰

**原因**：
- 后台任务突然产生大量 IO
- 与前台 IO 竞争资源
- 导致延迟突然升高

**验证方法**：
```bash
# 使用 iotop 监控后台 IO
adb shell su -c "iotop -o -d 1 -P"

# 使用 systrace 分析 IO 事件时间线
```

### 排查方法

#### 步骤 1：分析延迟分布

```bash
# 使用 blktrace 收集数据
adb shell su -c "blktrace -d /dev/block/sda -o trace -a read -a write"

# 使用 btt 分析
adb shell su -c "btt -i trace.bin"

# 查看延迟分布统计
```

#### 步骤 2：识别干扰源

```bash
# 使用 systrace 分析 IO 事件
# 找出延迟突然升高的时间点
# 分析该时间点的 IO 活动
```

### 优化方案

#### 方案 1：调整调度器参数

```bash
# 如果使用 mq-deadline
# 减少批处理大小，提高响应性
adb shell su -c "echo 8 > /sys/block/sda/queue/iosched/fifo_batch"

# 降低截止时间，保证延迟上限
adb shell su -c "echo 200 > /sys/block/sda/queue/iosched/read_expire"
```

#### 方案 2：使用 QoS 控制

```bash
# 使用 cgroup IO 延迟控制
mkdir -p /sys/fs/cgroup/foreground
echo <pid> > /sys/fs/cgroup/foreground/cgroup.procs
echo "8:16 target=10000" > /sys/fs/cgroup/foreground/io.latency  # 10ms 目标延迟
```

#### 方案 3：隔离后台 IO

```bash
# 限制后台任务的 IO 带宽
mkdir -p /sys/fs/cgroup/background
echo <pid> > /sys/fs/cgroup/background/cgroup.procs
echo "8:16 wbps=2097152" > /sys/fs/cgroup/background/io.max  # 限制 2MB/s
```

## 问题 5：Kernel 升级后 IO 性能劣化

### 问题描述

**现象**：
- Kernel 从 r2 升级到 r3 后 IO 性能下降
- 应用启动变慢
- 系统整体响应变慢

**影响**：
- 系统性能下降
- 用户体验变差
- 可能需要回退或优化

### 原因分析

#### 1. 调度器默认参数变化

**原因**：
- 新版本 Kernel 可能修改了调度器默认参数
- 参数变化导致性能下降
- 需要重新调优

**验证方法**：
```bash
# 对比 r2 和 r3 的默认参数
# 在 r2 设备上：
adb shell cat /sys/block/sda/queue/iosched/* > r2_params.txt

# 在 r3 设备上：
adb shell cat /sys/block/sda/queue/iosched/* > r3_params.txt

# 对比差异
diff r2_params.txt r3_params.txt
```

#### 2. 调度器选择变化

**原因**：
- 新版本可能改变了默认调度器
- 不同调度器的性能特征不同
- 需要根据设备类型重新选择

**验证方法**：
```bash
# 查看当前调度器
adb shell cat /sys/block/sda/queue/scheduler

# 查看可用调度器
adb shell cat /sys/block/sda/queue/scheduler
# 输出：[mq-deadline] kyber none
```

#### 3. 其他内核参数变化

**原因**：
- 内存管理参数变化
- 文件系统参数变化
- 其他影响 IO 的内核参数

**验证方法**：
```bash
# 对比关键内核参数
# r2 设备：
adb shell cat /proc/sys/vm/dirty_ratio > r2_vm_params.txt
adb shell cat /proc/sys/vm/dirty_background_ratio >> r2_vm_params.txt

# r3 设备：
adb shell cat /proc/sys/vm/dirty_ratio > r3_vm_params.txt
adb shell cat /proc/sys/vm/dirty_background_ratio >> r3_vm_params.txt

# 对比差异
diff r2_vm_params.txt r3_vm_params.txt
```

### 排查方法

#### 步骤 1：对比版本差异

```bash
# 使用 Git 对比 Kernel 源码差异
cd /path/to/kernel
git diff android13-5.15-2024-05_r2 android13-5.15-2024-11_r3 -- block/

# 重点关注：
# - 调度器参数默认值变化
# - 调度器算法变化
# - IO 相关内核参数变化
```

#### 步骤 2：分析性能指标变化

```bash
# 在相同负载下测试性能
# 使用 iostat 收集数据
adb shell su -c "iostat -x 1 60 > iostat_r3.log"

# 对比 r2 和 r3 的性能指标
# - IOPS
# - 吞吐量
# - 延迟
# - 设备利用率
```

#### 步骤 3：定位具体变化

```bash
# 使用 perf 或 ftrace 分析 IO 路径
adb shell su -c "echo function > /sys/kernel/debug/tracing/current_tracer"
adb shell su -c "echo blk_mq_submit_bio > /sys/kernel/debug/tracing/set_ftrace_filter"
adb shell su -c "echo 1 > /sys/kernel/debug/tracing/tracing_on"
# 运行测试
adb shell su -c "echo 0 > /sys/kernel/debug/tracing/tracing_on"
adb shell su -c "cat /sys/kernel/debug/tracing/trace > ftrace.log"
```

### 优化方案

#### 方案 1：恢复 r2 的参数配置

```bash
# 如果确认是参数变化导致的问题
# 恢复 r2 的参数配置

# 调度器参数
adb shell su -c "echo 500 > /sys/block/sda/queue/iosched/read_expire"
adb shell su -c "echo 5000 > /sys/block/sda/queue/iosched/write_expire"
adb shell su -c "echo 16 > /sys/block/sda/queue/iosched/fifo_batch"

# 内核参数
adb shell su -c "echo 20 > /proc/sys/vm/dirty_ratio"
adb shell su -c "echo 10 > /proc/sys/vm/dirty_background_ratio"
```

#### 方案 2：根据设备类型重新调优

```bash
# 根据设备类型选择最优配置
# 如果是 UFS 3.0+，使用 none 或 kyber
adb shell su -c "echo kyber > /sys/block/sda/queue/scheduler"
adb shell su -c "echo 2000000 > /sys/block/sda/queue/iosched/read_lat_nsec"

# 如果是 eMMC，使用 mq-deadline
adb shell su -c "echo mq-deadline > /sys/block/sda/queue/scheduler"
adb shell su -c "echo 200 > /sys/block/sda/queue/iosched/read_expire"
```

#### 方案 3：应用层优化

- 减少 IO 操作
- 优化 IO 模式
- 使用缓存减少 IO

## 总结

### 核心要点

1. **问题分类**：
   - 应用启动慢：小文件 IO、Page Cache 未命中
   - 系统卡顿：后台写回、调度器配置不当
   - 吞吐量下降：Tag 资源耗尽、硬件配置不足
   - 延迟波动：调度器参数不当、后台干扰
   - Kernel 升级问题：参数变化、调度器变化

2. **排查方法**：
   - 使用 `iostat`、`iotop` 等工具定位问题
   - 使用 `blktrace`、`systrace` 分析 IO 路径
   - 对比版本差异找出变化

3. **优化方案**：
   - 调整 IO 优先级
   - 优化调度器参数
   - 使用 QoS 控制
   - 应用层优化

### 关键概念

- **IO 压力**：设备 IO 资源的使用程度
- **IO 延迟**：IO 请求的等待时间
- **IO 吞吐量**：单位时间的 IO 数据量
- **Tag 资源**：并发 IO 请求的数量限制

### 下一步学习

- [10-Android IO 性能优化实战](10-Android IO 性能优化实战.md) - 学习 IO 性能优化的实践方法

## 参考资料

- Linux 内核源码：`block/` - IO 子系统实现
- Android 性能分析工具文档
- iostat、iotop、blktrace 工具文档

## 更新记录

- 2026-01-27：初始创建，包含 Android 设备常见 IO 问题与排查
