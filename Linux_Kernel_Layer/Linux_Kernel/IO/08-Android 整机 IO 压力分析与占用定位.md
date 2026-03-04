# Android 整机 IO 压力分析与占用定位

## 学习目标

- 掌握 IO 压力指标的识别方法
- 学会使用各种工具定位 IO 占用者
- 理解 Android 特有的 IO 监控方法
- 掌握 cgroup IO 统计的解读
- 能够快速定位导致系统卡顿的 IO 问题

## 背景介绍

当 Android 设备出现卡顿、应用启动慢、系统响应延迟等问题时，IO 压力往往是主要原因之一。快速准确地定位 IO 占用者，分析 IO 压力来源，是解决性能问题的关键。本文介绍 Android 设备上 IO 压力分析和占用定位的完整方法，帮助快速找到 IO 性能瓶颈。

## IO 压力指标识别

### 1. 使用 iostat 工具

**iostat 是最常用的 IO 性能分析工具**。

**基本用法**：
```bash
# 每秒更新一次，显示所有设备
iostat -x 1

# 输出示例：
# Device            r/s     w/s     rkB/s    wkB/s    await   svctm   %util
# sda              10.5     5.2     420.0    208.0    15.2    8.5     13.4
```

**关键指标解读**：

| 指标 | 含义 | 正常范围 | 异常判断 |
|------|------|---------|---------|
| `r/s`, `w/s` | 每秒读写次数（IOPS） | < 1000 | > 5000 可能异常 |
| `rkB/s`, `wkB/s` | 每秒读写数据量（KB/s） | 根据设备而定 | 持续接近设备上限 |
| `await` | 平均等待时间（ms） | < 10ms (SSD) | > 50ms 可能异常 |
| `svctm` | 平均服务时间（ms） | < 5ms (SSD) | > 20ms 可能异常 |
| `%util` | 设备利用率 | < 80% | > 90% 表示饱和 |

**Android 设备上使用**：
```bash
# 在 Android 设备上（需要 root）
adb shell
su
iostat -x 1 10  # 每秒更新，共 10 次
```

### 2. 解读 /proc/diskstats

**位置**：`/proc/diskstats`

**格式**（每行 14 个字段）：
```
major minor name    reads    reads_merged  reads_sectors  reads_time
      writes writes_merged writes_sectors writes_time  io_in_progress
      io_time weighted_io_time
```

**关键字段**：
- `reads`, `writes`：完成的读写次数
- `reads_sectors`, `writes_sectors`：读写的扇区数（1 扇区 = 512 字节）
- `reads_time`, `writes_time`：读写总时间（毫秒）
- `io_in_progress`：当前正在进行的 IO 数量
- `io_time`：设备忙碌时间（毫秒）
- `weighted_io_time`：加权 IO 时间（毫秒）

**计算指标**：
```bash
# 读取 /proc/diskstats
cat /proc/diskstats

# 计算 IOPS（需要两次采样，计算差值）
# IOPS = (reads2 - reads1) / time_interval
# 吞吐量 = (reads_sectors2 - reads_sectors1) * 512 / time_interval
```

### 3. 使用 /sys/block/*/stat

**位置**：`/sys/block/sda/stat`

**格式**（与 `/proc/diskstats` 相同）：
```bash
cat /sys/block/sda/stat
# 输出：reads reads_merged reads_sectors reads_time writes writes_merged writes_sectors writes_time io_in_progress io_time weighted_io_time
```

**实时监控脚本**：
```bash
#!/bin/bash
# 监控 IO 统计
while true; do
    clear
    echo "=== IO Statistics ==="
    cat /sys/block/sda/stat | awk '{
        printf "Reads: %d, Writes: %d\n", $1, $5
        printf "Read Sectors: %d, Write Sectors: %d\n", $3, $7
        printf "IO in progress: %d\n", $9
        printf "IO time: %d ms\n", $10
    }'
    sleep 1
done
```

## IO 占用者定位方法

### 1. 使用 iotop 工具

**iotop 可以实时显示每个进程的 IO 使用情况**。

**基本用法**：
```bash
# 实时显示所有进程的 IO 使用情况
iotop

# 只显示有 IO 活动的进程
iotop -o

# 按 IO 使用量排序
iotop -o -d 1
```

**输出解读**：
```
Total DISK READ:  10.5 M/s | Total DISK WRITE:  5.2 M/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN      IO    COMMAND
 1234  be/4 root       10.5 M/s    0.0 B/s  0.00 % 85.23 % system_server
 5678  be/4 system     0.0 B/s    5.2 M/s   0.00 % 12.45 % com.example.app
```

**关键列**：
- `DISK READ`：每秒读取数据量
- `DISK WRITE`：每秒写入数据量
- `IO`：IO 等待时间百分比

**Android 设备上使用**：
```bash
# 需要 root 权限
adb shell
su
iotop -o -d 1
```

### 2. 使用 pidstat 工具

**pidstat 可以按进程统计 IO 使用情况**。

**基本用法**：
```bash
# 显示所有进程的 IO 统计
pidstat -d 1

# 显示指定进程的 IO 统计
pidstat -d -p 1234 1

# 显示详细信息
pidstat -d -h 1
```

**输出解读**：
```
Linux 5.15.0 (hostname)    01/27/26    _x86_64_    (8 CPU)

Time      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
10:00:01     0      1234    420.00    208.00      0.00  system_server
10:00:01  1000      5678      0.00    520.00      0.00  com.example.app
```

**关键指标**：
- `kB_rd/s`：每秒读取数据量（KB）
- `kB_wr/s`：每秒写入数据量（KB）
- `kB_ccwr/s`：每秒取消的写入数据量（KB）

### 3. 使用 blktrace + blkparse 追踪 IO 路径

**blktrace 可以追踪 IO 请求的完整路径**。

**基本用法**：
```bash
# 开始追踪
blktrace -d /dev/sda -o trace

# 在另一个终端运行测试
# ...

# 停止追踪（Ctrl+C），然后解析
blkparse -i trace -d trace.bin

# 使用 btt 分析
btt -i trace.bin
```

**输出解读**：
```
  8,0    3        1     0.000000000  1234  Q   R 123456 + 8 [system_server]
  8,0    3        2     0.000001234  1234  G   R 123456 + 8 [system_server]
  8,0    3        3     0.000002468  1234  I   R 123456 + 8 [system_server]
  8,0    3        4     0.000003702  1234  D   R 123456 + 8 [system_server]
  8,0    3        5     0.000012345  1234  C   R 123456 + 8 [0]
```

**事件类型**：
- `Q`：请求入队
- `G`：获取请求
- `I`：插入请求
- `D`：发送到驱动
- `C`：完成

### 4. 使用 ftrace 追踪 IO 相关函数

**ftrace 可以追踪内核函数的调用**。

**基本用法**：
```bash
# 启用 ftrace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 设置追踪函数
echo blk_mq_submit_bio > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# 查看追踪结果
cat /sys/kernel/debug/tracing/trace

# 停止追踪
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**追踪 IO 相关函数**：
```bash
# 追踪 bio 提交
echo blk_mq_submit_bio > /sys/kernel/debug/tracing/set_ftrace_filter

# 追踪请求完成
echo blk_mq_end_request > /sys/kernel/debug/tracing/set_ftrace_filter

# 追踪调度器
echo dd_dispatch_request > /sys/kernel/debug/tracing/set_ftrace_filter
```

## Android 特有的 IO 监控

### 1. dumpsys diskstats 命令

**dumpsys 是 Android 系统提供的调试工具**。

**基本用法**：
```bash
# 显示磁盘统计信息
adb shell dumpsys diskstats

# 输出示例：
# /proc/diskstats:
#   179       0 sda 12345 0 1234567 123456 5678 0 567890 56789 0 12345 180245
# 
# /sys/block/sda/stat:
#   12345 0 1234567 123456 5678 0 567890 56789 0 12345 180245
```

**解读**：
- 显示 `/proc/diskstats` 和 `/sys/block/*/stat` 的内容
- 可以快速查看设备级别的 IO 统计

### 2. dumpsys iostats 命令

**iostats 提供更详细的 IO 统计**。

**基本用法**：
```bash
# 显示 IO 统计信息
adb shell dumpsys iostats

# 可能需要特定权限或系统版本支持
```

### 3. 使用 systrace/perfetto

**systrace 和 perfetto 是 Android 性能分析工具**。

**基本用法**：
```bash
# 使用 systrace
python systrace.py -t 10 -o trace.html sched freq idle am wm gfx view binder_driver hal dalvik camera input res

# 使用 perfetto（推荐）
# 通过 perfetto UI 或命令行工具
```

**IO 相关事件**：
- `block_rq_issue`：请求发送到驱动
- `block_rq_complete`：请求完成
- `block_bio_*`：bio 相关事件

**分析 IO 延迟**：
1. 打开 trace 文件
2. 搜索 `block_rq_issue` 和 `block_rq_complete`
3. 分析请求的等待时间和处理时间
4. 找出延迟高的请求和对应的进程

## cgroup IO 统计

### 1. /sys/fs/cgroup/io.stat

**cgroup v2 的 IO 统计文件**。

**基本用法**：
```bash
# 查看当前 cgroup 的 IO 统计
cat /sys/fs/cgroup/io.stat

# 输出示例：
# 8:16 rbytes=1234567 wbytes=567890 rios=1234 wios=567
```

**字段解读**：
- `rbytes`：读取字节数
- `wbytes`：写入字节数
- `rios`：读取次数
- `wios`：写入次数
- `dbytes`：丢弃字节数（如果支持）
- `dios`：丢弃次数（如果支持）

**按进程组统计**：
```bash
# 查看所有 cgroup 的 IO 统计
find /sys/fs/cgroup -name "io.stat" -exec echo {} \; -exec cat {} \;
```

### 2. IO 带宽和 IOPS 统计

**使用 cgroup IO 控制器限制和统计**。

**查看 IO 限制**：
```bash
# 查看 IO 带宽限制
cat /sys/fs/cgroup/io.max

# 查看 IO 权重
cat /sys/fs/cgroup/io.weight
```

**监控 IO 使用**：
```bash
# 定期采样，计算 IOPS 和带宽
while true; do
    stat1=$(cat /sys/fs/cgroup/io.stat)
    sleep 1
    stat2=$(cat /sys/fs/cgroup/io.stat)
    # 计算差值，得到每秒的 IO 量
done
```

### 3. IO 延迟统计（io.latency）

**cgroup v2 支持 IO 延迟统计**。

**查看延迟统计**：
```bash
# 查看 IO 延迟配置
cat /sys/fs/cgroup/io.latency

# 查看延迟统计（在 io.stat 中）
cat /sys/fs/cgroup/io.stat
# 可能包含：depth, avg_lat, win 等字段
```

## 实战案例：定位导致系统卡顿的 IO 占用进程

### 案例场景

**问题描述**：
- Android 设备出现明显卡顿
- 应用启动慢，界面响应延迟
- 系统整体性能下降

### 排查步骤

#### 步骤 1：确认 IO 压力

```bash
# 使用 iostat 查看 IO 压力
adb shell su -c "iostat -x 1 5"

# 关键指标：
# - %util > 90%：设备饱和
# - await > 50ms：延迟高
# - 持续高 IOPS：可能有异常进程
```

#### 步骤 2：定位 IO 占用进程

```bash
# 使用 iotop 找出 IO 占用最高的进程
adb shell su -c "iotop -o -d 1"

# 或者使用 pidstat
adb shell su -c "pidstat -d 1 10"
```

#### 步骤 3：分析进程行为

```bash
# 查看进程的详细信息
adb shell su -c "ps aux | grep <pid>"

# 查看进程打开的文件
adb shell su -c "lsof -p <pid>"

# 使用 strace 追踪系统调用
adb shell su -c "strace -p <pid> -e trace=open,read,write"
```

#### 步骤 4：分析 IO 模式

```bash
# 使用 blktrace 追踪 IO 路径
adb shell su -c "blktrace -d /dev/block/sda -o trace"
# 运行一段时间后停止
adb shell su -c "blkparse -i trace -d trace.bin"
```

#### 步骤 5：验证和解决

```bash
# 如果确认是某个进程导致的问题：

# 方案 1：降低进程的 IO 优先级
adb shell su -c "ionice -c 2 -n 7 -p <pid>"

# 方案 2：限制进程的 IO 带宽（使用 cgroup）
# 创建 cgroup 并设置限制
mkdir -p /sys/fs/cgroup/problem_process
echo <pid> > /sys/fs/cgroup/problem_process/cgroup.procs
echo "8:16 wbps=1048576" > /sys/fs/cgroup/problem_process/io.max

# 方案 3：如果是应用问题，联系应用开发者
```

### 常见 IO 占用场景

#### 场景 1：后台应用大量写入

**特征**：
- `iotop` 显示某个应用进程写入量很大
- `iostat` 显示写 IOPS 高

**解决方案**：
- 降低应用的 IO 优先级
- 限制应用的 IO 带宽
- 优化应用的写入策略

#### 场景 2：系统服务频繁读取

**特征**：
- `iotop` 显示系统服务（如 system_server）读取量大
- 可能是配置文件、数据库等频繁访问

**解决方案**：
- 检查是否有配置问题
- 优化数据访问模式
- 使用缓存减少 IO

#### 场景 3：文件系统检查或修复

**特征**：
- `iotop` 显示 `fsck` 或类似进程
- IO 持续高，但进程可能不在 `iotop` 中显示

**解决方案**：
- 等待文件系统检查完成
- 检查文件系统是否有问题
- 考虑在低峰期进行维护

## 总结

### 核心要点

1. **IO 压力识别**：
   - 使用 `iostat` 查看设备级指标
   - 关注 `%util`、`await`、IOPS 等关键指标

2. **占用者定位**：
   - 使用 `iotop` 快速找出 IO 占用高的进程
   - 使用 `pidstat` 进行详细统计
   - 使用 `blktrace` 追踪 IO 路径

3. **Android 特有工具**：
   - `dumpsys diskstats` 查看系统级统计
   - `systrace/perfetto` 分析 IO 延迟

4. **cgroup 统计**：
   - 使用 `io.stat` 查看 cgroup 级别的 IO 统计
   - 结合 cgroup 限制进行问题定位

### 关键概念

- **IOPS**：每秒 IO 操作次数
- **吞吐量**：每秒 IO 数据量
- **延迟**：IO 请求的等待时间
- **设备利用率**：设备忙碌时间的百分比

### 下一步学习

- [09-Android 设备常见 IO 问题与排查](09-Android 设备常见 IO 问题与排查.md) - 学习常见 IO 问题的排查方法
- [10-Android IO 性能优化实战](10-Android IO 性能优化实战.md) - 学习 IO 性能优化实践

## 参考资料

- Linux 内核源码：`block/blk-core.c` - IO 统计实现
- Linux 内核源码：`block/blk-iocost.c` - IO 成本统计
- Linux 内核源码：`block/blk-iolatency.c` - IO 延迟统计
- iostat 工具文档：`man iostat`
- iotop 工具文档：`man iotop`
- blktrace 工具文档：`man blktrace`

## 更新记录

- 2026-01-27：初始创建，包含 Android 整机 IO 压力分析与占用定位
