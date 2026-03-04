1、dm指的是什么能否详细介绍一下。
2、我是一名稳定性系统工程师，我负责看护整机稳定性和性能，所以我需要监控版本升级后IO是否存在劣化，所以从上述日志中我应该通过批量解析日志来分析。我想定制一个IO检查的SOP，也就是从设置压测脚本，到分析日志到生成结论，能够让两个版本通过跑IO模型来找到是否IO存在劣化。
3、说回日志解析本身，page cache命中的话，没什么好分析的不涉及IO操作。有block_rq_insert说明存在IO交互。我在统计时间时肯定需要要从block_bio_queue到block_bio_complete。但是block_bio_queue存在合并，并且从日志来看block_bio_queue和block_rq_insert不知道怎么关联。所以我该如何设定一个通用的IO请求日志序列来分析。因为是海量日志的自动化分析，所以要提前考虑到所有的边界情况的，以及所有的可能性。

```
	行    8029:  ssion.deskclock-2269    (   2269) [000] .....  1989.101585: block_bio_queue: 252,0 RS 1332960 + 8 [ssion.deskclock]
	行    8033:  ssion.deskclock-2269    (   2269) [000] .....  1989.101618: block_bio_complete: 252,0 RS 1332960 + 8 [0]
	行    8078:     grpc-timer-0-4931    (   4465) [001] .....  1989.102267: block_bio_queue: 252,0 RS 2645384 + 8 [grpc-timer-0]
	行    8080:     grpc-timer-0-4931    (   4465) [001] .....  1989.102302: block_bio_complete: 252,0 RS 2645384 + 8 [0]
	行    8098:    binder:4821_8-14304   (   4821) [000] .....  1989.102468: block_bio_queue: 252,0 RS 10900152 + 8 [binder:4821_8]
	行    8103:    binder:4821_8-14304   (   4821) [000] .....  1989.102487: block_bio_complete: 252,0 RS 10900152 + 8 [0]
	行    8112:    binder:4821_8-14304   (   4821) [000] .....  1989.102655: block_bio_queue: 252,0 RS 3165840 + 8 [binder:4821_8]
	行    8114:    binder:4821_8-14304   (   4821) [000] .....  1989.102668: block_bio_complete: 252,0 RS 3165840 + 8 [0]
	行    8116:    binder:4821_8-14304   (   4821) [000] .....  1989.102693: block_bio_queue: 252,0 RS 10106688 + 8 [binder:4821_8]
	行    8120:    binder:4821_8-14304   (   4821) [000] .....  1989.102703: block_bio_complete: 252,0 RS 10106688 + 8 [0]
	行    8122:    binder:4821_8-14304   (   4821) [000] .....  1989.102716: block_bio_queue: 252,0 RS 10106696 + 8 [binder:4821_8]
	行    8123:    binder:4821_8-14304   (   4821) [000] .....  1989.102724: block_bio_complete: 252,0 RS 10106696 + 8 [0]
	行    8263:    binder:2566_5-2699    (   2566) [004] .....  1989.103774: block_bio_queue: 252,0 RS 9850248 + 8 [binder:2566_5]
	行    8269:    binder:2566_5-2699    (   2566) [004] .....  1989.103804: block_bio_complete: 252,0 RS 9850248 + 8 [0]
	行    8625:    binder:2566_5-2699    (   2566) [004] .....  1989.106952: block_bio_queue: 252,0 RS 7621416 + 8 [binder:2566_5]
	行    8628:    binder:2566_5-2699    (   2566) [004] .....  1989.106982: block_bio_complete: 252,0 RS 7621416 + 8 [0]
	行    8629:    binder:2566_5-2699    (   2566) [004] .....  1989.107018: block_bio_queue: 252,0 RS 9850240 + 8 [binder:2566_5]
	行    8630:    binder:2566_5-2699    (   2566) [004] .....  1989.107032: block_bio_complete: 252,0 RS 9850240 + 8 [0]
	行    8854:     binder:971_4-2293    (    971) [006] .....  1989.109635: block_bio_queue: 252,0 RS 1720008 + 8 [binder:971_4]
	行    8858:     binder:971_4-2293    (    971) [006] .....  1989.109649: block_bio_complete: 252,0 RS 1720008 + 8 [0]
	行    8987:  om.mediatek.ims-2566    (   2566) [003] .....  1989.110801: block_bio_queue: 254,16 RA 3767264 + 592 [om.mediatek.ims]
	行    8992:  om.mediatek.ims-2566    (   2566) [003] .....  1989.110864: block_bio_queue: 254,4 RA 3767264 + 592 [om.mediatek.ims]
	行    8994:  om.mediatek.ims-2566    (   2566) [003] .....  1989.110910: block_bio_queue: 8,32 RA 14334944 + 592 [om.mediatek.ims]
	行    8999:     grpc-timer-0-4931    (   4465) [001] .....  1989.110933: block_bio_queue: 254,16 R 5262664 + 8 [grpc-timer-0]
	行    9001:  om.mediatek.ims-2566    (   2566) [003] ...1.  1989.110946: block_rq_insert: 8,32 RA 303104 () 14334944 + 592 be,0,4 [om.mediatek.ims]
	行    9010:     grpc-timer-0-4931    (   4465) [001] .N...  1989.110973: block_bio_queue: 254,4 R 5262664 + 8 [grpc-timer-0]
	行    9012:  om.mediatek.ims-2566    (   2566) [003] .....  1989.110984: block_rq_issue: 8,32 RA 303104 () 14334944 + 592 be,0,4 [om.mediatek.ims]
	行    9037:     kworker/3:0H-14330   (  14330) [003] .....  1989.111094: block_bio_queue: 254,4 R 5295040 + 8 [kworker/3:0H]
	行    9038:     kworker/3:0H-14330   (  14330) [003] .....  1989.111107: block_bio_queue: 8,32 R 15862720 + 8 [kworker/3:0H]
	行    9042:     kworker/3:0H-14330   (  14330) [003] .....  1989.111139: block_bio_queue: 254,4 R 5295048 + 8 [kworker/3:0H]
	行    9043:     kworker/3:0H-14330   (  14330) [003] .....  1989.111142: block_bio_queue: 8,32 R 15862728 + 8 [kworker/3:0H]
	行    9047:     kworker/3:0H-14330   (  14330) [003] ...1.  1989.111153: block_rq_insert: 8,32 R 8192 () 15862720 + 16 be,0,4 [kworker/3:0H]
	行    9049:     kworker/3:0H-14330   (  14330) [003] .....  1989.111178: block_rq_issue: 8,32 R 8192 () 15862720 + 16 be,0,4 [kworker/3:0H]
	行    9051:     grpc-timer-0-4931    (   4465) [001] .....  1989.111197: block_bio_queue: 8,32 R 15830344 + 8 [grpc-timer-0]
	行    9054:     grpc-timer-0-4931    (   4465) [001] ...1.  1989.111208: block_rq_insert: 8,32 R 4096 () 15830344 + 8 be,0,4 [grpc-timer-0]
	行    9072:     kworker/1:0H-14452   (  14452) [001] .....  1989.111337: block_rq_issue: 8,32 R 4096 () 15830344 + 8 be,0,4 [kworker/1:0H]
	行    9314:  om.mediatek.ims-2566    (   2566) [003] ..s..  1989.113632: block_rq_complete: 8,32 R () 15862720 + 16 be,0,4 [0]
	行    9320:  om.mediatek.ims-2566    (   2566) [003] ..s..  1989.113647: block_bio_complete: 254,4 R 5295040 + 8 [0]
	行    9322:  om.mediatek.ims-2566    (   2566) [003] ..s..  1989.113653: block_bio_complete: 254,4 R 5295048 + 8 [0]
	行    9385:  om.mediatek.ims-2566    (   2566) [003] ..s..  1989.114200: block_rq_complete: 8,32 RA () 14334944 + 592 be,0,4 [0]
	行    9387:  om.mediatek.ims-2566    (   2566) [003] ..s..  1989.114212: block_bio_complete: 254,4 RA 3767264 + 592 [0]
	行    9413:    binder:4821_8-14304   (   4821) [001] ..s..  1989.114404: block_rq_complete: 8,32 R () 15830344 + 8 be,0,4 [0]
	行    9414:    binder:4821_8-14304   (   4821) [001] ..s..  1989.114415: block_bio_complete: 254,4 R 5262664 + 8 [0]
	行    9425:     kworker/1:0H-14452   (  14452) [001] .....  1989.114488: block_bio_complete: 254,16 R 5262664 + 8 [0]
	行    9441:     grpc-timer-0-4931    (   4465) [001] .....  1989.114681: block_bio_queue: 254,16 RA 1186728 + 104 [grpc-timer-0]
	行    9446:     grpc-timer-0-4931    (   4465) [001] .N...  1989.114709: block_bio_queue: 254,4 RA 1186728 + 104 [grpc-timer-0]
	行    9453:     kworker/1:0H-14452   (  14452) [001] .....  1989.114759: block_bio_queue: 254,4 R 5274880 + 8 [kworker/1:0H]
	行    9455:     kworker/1:0H-14452   (  14452) [001] .....  1989.114773: block_bio_queue: 8,32 R 15842560 + 8 [kworker/1:0H]
	行    9458:     kworker/1:0H-14452   (  14452) [001] .....  1989.114809: block_bio_queue: 254,4 R 5274888 + 8 [kworker/1:0H]
	行    9459:     kworker/1:0H-14452   (  14452) [001] .....  1989.114812: block_bio_queue: 8,32 R 15842568 + 8 [kworker/1:0H]
```
非常专业的问题！作为稳定性工程师，你的需求非常明确。让我逐一详细回答。

## 1. Device Mapper (DM) 详解

### 什么是 Device Mapper

**Device Mapper** 是 Linux 内核的一个框架，用于创建虚拟块设备，提供逻辑设备到物理设备的映射。

```
┌─────────────────────────────────────────────────────┐
│  Device Mapper Framework                            │
│                                                      │
│  作用：在物理设备之上创建虚拟块设备                    │
│  位置：内核空间                                       │
│  设备号：254,x (major=254)                           │
└─────────────────────────────────────────────────────┘
```

### DM 的主要类型

#### 1. dm-crypt（加密）

```bash
# 最常见的 Android 用途：全盘加密

# 查看加密设备
dmsetup table | grep crypt

# 输出示例：
# userdata: 0 20971520 crypt aes-xts-plain64 \
#   :64:logon:ext4-18a7c...:0 0 259:3 0 1 allow_discards

结构：
┌──────────────┐
│   /data      │  文件系统层
└──────┬───────┘
       ↓
┌──────────────┐
│   dm-crypt   │  加密层 (dm-4)
│ AES-XTS-256  │  数据加密/解密
└──────┬───────┘
       ↓
┌──────────────┐
│  mmcblk0p31  │  物理分区
└──────────────┘

I/O 流程：
  写入: 明文 → 加密 → 写入物理设备
  读取: 读取物理设备 → 解密 → 明文
```

#### 2. dm-verity（验证）

```bash
# Android Verified Boot：确保系统分区未被篡改

# 查看 verity 设备
dmsetup table | grep verity

# 输出示例：
# system: 0 4194304 verity 1 259:2 259:2 4096 4096 \
#   524288 1 sha256 4a7c8b...

结构：
┌──────────────┐
│   /system    │  只读系统分区
└──────┬───────┘
       ↓
┌──────────────┐
│  dm-verity   │  验证层 (dm-0)
│  校验哈希树   │  每个块都有哈希值
└──────┬───────┘
       ↓
┌──────────────┐
│  mmcblk0p25  │  物理分区
└──────────────┘

读取流程：
  1. 读取数据块
  2. 计算哈希值
  3. 与哈希树对比
  4. 验证通过 → 返回数据
  5. 验证失败 → I/O 错误
```

#### 3. dm-linear（线性映射）

```bash
# 将多个设备合并成一个逻辑设备

# 查看 linear 映射
dmsetup table | grep linear

# 输出示例：
# vendor: 0 1048576 linear 259:4 2048

结构：
┌──────────────────────────────┐
│   逻辑设备 (dm-linear)        │
│   大小: 20 GB                 │
└───────┬──────────────────────┘
        ↓
   ┌────┴────┐
   ↓         ↓
┌──────┐  ┌──────┐
│ sda1 │  │ sda2 │  物理设备/分区
│ 10GB │  │ 10GB │
└──────┘  └──────┘

扇区映射：
  逻辑扇区 0-20480    → sda1 扇区 0-20480
  逻辑扇区 20481-40960 → sda2 扇区 0-20480
```

#### 4. dm-bow (Block on Write)

```bash
# Android 特有：用于 A/B 无缝更新

结构：
┌──────────────┐
│  Slot A      │  当前运行的系统
└──────────────┘
┌──────────────┐
│   dm-bow     │  写时复制层
└──────┬───────┘
       ↓
┌──────────────┐
│  Slot B      │  更新目标
└──────────────┘

作用：
  - 更新时不影响 Slot A
  - 写入数据暂存在 dm-bow
  - 更新完成后切换启动槽位
```

### Android 中的典型 DM 层次

```bash
# 查看所有 dm 设备
adb shell ls -l /dev/block/dm-*

# 查看映射关系
adb shell dmsetup ls --tree

# 典型输出：
dm-16 (userdata)
  └─dm-4 (userdata-real)
      └─mmcblk0p31 (259:3)

dm-0 (system)
  └─mmcblk0p25 (259:2)

dm-1 (vendor)
  └─mmcblk0p26 (259:4)
```

### DM 性能影响

```bash
# 加密层的性能开销

未加密设备:
  顺序读: 300 MB/s
  随机读: 15000 IOPS

dm-crypt 加密:
  顺序读: 280 MB/s  (-7%)    ← 加解密 CPU 开销
  随机读: 14000 IOPS (-7%)

dm-verity 验证:
  顺序读: 295 MB/s  (-2%)    ← 哈希验证开销
  随机读: 14500 IOPS (-3%)

dm-crypt + dm-verity:
  顺序读: 270 MB/s  (-10%)
  随机读: 13000 IOPS (-13%)
```

---

## 2. I/O 劣化检测 SOP

### 完整的 I/O 性能回归测试流程

```
┌────────────────────────────────────────────────────┐
│  I/O Performance Regression Testing SOP            │
└────────────────────────────────────────────────────┘

Phase 1: 环境准备
  ├─ 1.1 设备准备
  ├─ 1.2 版本刷写
  ├─ 1.3 基线设置
  └─ 1.4 监控工具配置

Phase 2: 测试执行
  ├─ 2.1 预热阶段
  ├─ 2.2 基准测试
  ├─ 2.3 压力测试
  └─ 2.4 日志收集

Phase 3: 数据分析
  ├─ 3.1 日志解析
  ├─ 3.2 指标计算
  ├─ 3.3 对比分析
  └─ 3.4 异常检测

Phase 4: 报告生成
  ├─ 4.1 结论生成
  ├─ 4.2 可视化
  └─ 4.3 归档
```

### Phase 1: 环境准备

#### 1.1 设备准备脚本

```bash
#!/bin/bash
# prepare_device.sh

DEVICE_SERIAL="your_device_serial"

# 1. 连接设备
adb -s $DEVICE_SERIAL wait-for-device

# 2. 获取设备信息
echo "=== Device Info ==="
adb shell getprop ro.build.version.release  # Android 版本
adb shell getprop ro.build.id               # Build ID
adb shell getprop ro.product.model          # 型号

# 3. 获取存储信息
echo "=== Storage Info ==="
adb shell cat /proc/partitions
adb shell df -h

# 4. 检查文件系统类型
for partition in system vendor data; do
    echo "$partition: $(adb shell mount | grep $partition | awk '{print $5}')"
done

# 5. 获取调度器信息
echo "=== I/O Scheduler ==="
for dev in mmcblk0 sda dm-0 dm-4; do
    if adb shell "[ -f /sys/block/$dev/queue/scheduler ]"; then
        echo "$dev: $(adb shell cat /sys/block/$dev/queue/scheduler)"
    fi
done

# 6. 清空日志
adb logcat -c
adb shell dmesg -c > /dev/null

# 7. 禁用不必要的服务（减少干扰）
adb shell stop thermal-engine
adb shell stop perfd
```

#### 1.2 基线配置

```bash
#!/bin/bash
# set_baseline.sh

# 1. 固定 CPU 频率（避免 DVFS 干扰）
adb shell "echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor"

# 2. 禁用 zRAM（避免内存压缩干扰）
adb shell swapoff /dev/block/zram0

# 3. 清空页缓存
adb shell "echo 3 > /proc/sys/vm/drop_caches"

# 4. 设置 I/O 调度器（统一测试环境）
SCHEDULER="mq-deadline"  # 或 none
adb shell "echo $SCHEDULER > /sys/block/mmcblk0/queue/scheduler"

# 5. 调整队列深度（如果需要）
# adb shell "echo 64 > /sys/block/mmcblk0/queue/nr_requests"
```

### Phase 2: 测试执行

#### 2.1 I/O 压测脚本

```bash
#!/bin/bash
# io_benchmark.sh

TEST_DIR="/data/local/tmp/io_test"
DEVICE="mmcblk0"  # 或主存储设备

# 创建测试目录
adb shell "mkdir -p $TEST_DIR"
adb shell "cd $TEST_DIR && rm -f *"

# ═══════════════════════════════════════════════════
# 测试 1: 顺序读
# ═══════════════════════════════════════════════════
test_sequential_read() {
    echo "=== Sequential Read Test ==="
    
    # 启动 trace
    adb shell "echo 1 > /sys/kernel/debug/tracing/events/block/enable"
    adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"
    
    # 清空缓存
    adb shell "echo 3 > /proc/sys/vm/drop_caches"
    
    # 执行测试
    adb shell "dd if=$TEST_DIR/testfile of=/dev/null bs=1M count=1000"
    
    # 停止 trace
    adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"
    
    # 收集日志
    adb shell cat /sys/kernel/debug/tracing/trace > logs/seq_read_trace.txt
    
    # 清空 trace buffer
    adb shell "echo > /sys/kernel/debug/tracing/trace"
}

# ═══════════════════════════════════════════════════
# 测试 2: 随机读
# ═══════════════════════════════════════════════════
test_random_read() {
    echo "=== Random Read Test ==="
    
    adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"
    adb shell "echo 3 > /proc/sys/vm/drop_caches"
    
    # 使用 fio（如果可用）
    if adb shell "command -v fio" > /dev/null; then
        adb shell "fio --name=randread --rw=randread --bs=4k \
                       --size=100M --numjobs=4 --direct=1 \
                       --filename=$TEST_DIR/testfile"
    else
        # 备用方案：自定义随机读脚本
        adb push random_read.sh $TEST_DIR/
        adb shell "sh $TEST_DIR/random_read.sh"
    fi
    
    adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"
    adb shell cat /sys/kernel/debug/tracing/trace > logs/rand_read_trace.txt
    adb shell "echo > /sys/kernel/debug/tracing/trace"
}

# ═══════════════════════════════════════════════════
# 测试 3: 顺序写
# ═══════════════════════════════════════════════════
test_sequential_write() {
    echo "=== Sequential Write Test ==="
    
    adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"
    
    adb shell "dd if=/dev/zero of=$TEST_DIR/testfile bs=1M count=1000 oflag=direct"
    
    # 强制刷盘
    adb shell sync
    
    adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"
    adb shell cat /sys/kernel/debug/tracing/trace > logs/seq_write_trace.txt
    adb shell "echo > /sys/kernel/debug/tracing/trace"
}

# ═══════════════════════════════════════════════════
# 测试 4: 混合负载
# ═══════════════════════════════════════════════════
test_mixed_workload() {
    echo "=== Mixed Workload Test ==="
    
    adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"
    
    # 同时进行读写
    adb shell "
    dd if=/dev/zero of=$TEST_DIR/write_test bs=1M count=500 oflag=direct &
    dd if=$TEST_DIR/testfile of=/dev/null bs=1M count=500 &
    wait
    "
    
    adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"
    adb shell cat /sys/kernel/debug/tracing/trace > logs/mixed_trace.txt
}

# ═══════════════════════════════════════════════════
# 测试 5: 真实应用模拟
# ═══════════════════════════════════════════════════
test_app_simulation() {
    echo "=== App Launch Simulation ==="
    
    adb shell "echo 1 > /sys/kernel/debug/tracing/tracing_on"
    adb shell "echo 3 > /proc/sys/vm/drop_caches"
    
    # 启动多个应用
    for app in com.android.chrome com.android.camera2 com.android.gallery3d; do
        adb shell am start -n $app/.MainActivity
        sleep 2
    done
    
    sleep 5  # 等待应用完全加载
    
    adb shell "echo 0 > /sys/kernel/debug/tracing/tracing_on"
    adb shell cat /sys/kernel/debug/tracing/trace > logs/app_launch_trace.txt
}

# 执行所有测试
mkdir -p logs
test_sequential_read
test_random_read
test_sequential_write
test_mixed_workload
test_app_simulation
```

### Phase 3: 日志解析和分析

#### 3.1 完整的日志解析脚本

```python
#!/usr/bin/env python3
# io_trace_analyzer.py

import re
import sys
from collections import defaultdict, namedtuple
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple
import statistics

@dataclass
class BioEvent:
    """bio 事件"""
    timestamp: float
    device: str
    rwbs: str
    sector: int
    size: int
    process: str
    event_type: str  # 'queue' or 'complete'

@dataclass
class RqEvent:
    """request 事件"""
    timestamp: float
    device: str
    rwbs: str
    tag: int
    sector: int
    size: int
    process: str
    event_type: str  # 'insert', 'issue', or 'complete'

@dataclass
class IORequest:
    """完整的 I/O 请求"""
    device: str
    sector: int
    size: int
    rwbs: str
    process: str
    
    # 时间戳
    bio_queue_time: Optional[float] = None
    bio_complete_time: Optional[float] = None
    
    rq_tag: Optional[int] = None
    rq_insert_time: Optional[float] = None
    rq_issue_time: Optional[float] = None
    rq_complete_time: Optional[float] = None
    
    # 计算的延迟
    total_latency: Optional[float] = None  # bio_complete - bio_queue
    queue_latency: Optional[float] = None  # rq_issue - rq_insert
    device_latency: Optional[float] = None  # rq_complete - rq_issue
    
    # 标记
    is_cache_hit: bool = False
    is_merged: bool = False
    merged_into: Optional[int] = None

class IOTraceParser:
    def __init__(self):
        # 事件存储
        self.bio_events: Dict[Tuple[str, int], List[BioEvent]] = defaultdict(list)
        self.rq_events: Dict[Tuple[str, int], List[RqEvent]] = defaultdict(list)
        self.rq_tag_map: Dict[int, Tuple[str, int]] = {}  # tag -> (device, sector)
        
        # 完整的 I/O 请求
        self.io_requests: List[IORequest] = []
        
        # 统计
        self.stats = {
            'total_ios': 0,
            'cache_hits': 0,
            'real_ios': 0,
            'merged_ios': 0,
        }
    
    def parse_trace_line(self, line: str) -> Optional[dict]:
        """解析 trace 的一行"""
        # 示例行：
        # kworker-123 [001] .... 1989.123456: block_bio_queue: 8,32 R 12345 + 8 [process]
        
        pattern = r'(\S+)-(\d+)\s+\(?\s*\d*\)?\s*\[(\d+)\]\s+[\w.]+\s+(\d+\.\d+):\s+(\w+):\s+([\d,]+)\s+(\w+)\s+(\d+)?\s*\(?\)?\s*(\d+)\s+\+\s+(\d+)(?:\s+\[([^\]]+)\])?'
        
        match = re.search(pattern, line)
        if not match:
            return None
        
        return {
            'process': match.group(1),
            'pid': match.group(2),
            'cpu': match.group(3),
            'timestamp': float(match.group(4)),
            'event': match.group(5),
            'device': match.group(6),
            'rwbs': match.group(7),
            'tag': int(match.group(8)) if match.group(8) else None,
            'sector': int(match.group(9)),
            'size': int(match.group(10)),
            'extra': match.group(11) if match.group(11) else None,
        }
    
    def process_bio_queue(self, event_data: dict):
        """处理 block_bio_queue 事件"""
        bio = BioEvent(
            timestamp=event_data['timestamp'],
            device=event_data['device'],
            rwbs=event_data['rwbs'],
            sector=event_data['sector'],
            size=event_data['size'],
            process=event_data['process'],
            event_type='queue'
        )
        
        key = (bio.device, bio.sector)
        self.bio_events[key].append(bio)
        self.stats['total_ios'] += 1
    
    def process_bio_complete(self, event_data: dict):
        """处理 block_bio_complete 事件"""
        bio = BioEvent(
            timestamp=event_data['timestamp'],
            device=event_data['device'],
            rwbs=event_data['rwbs'],
            sector=event_data['sector'],
            size=event_data['size'],
            process=event_data['extra'] or event_data['process'],
            event_type='complete'
        )
        
        key = (bio.device, bio.sector)
        self.bio_events[key].append(bio)
    
    def process_rq_insert(self, event_data: dict):
        """处理 block_rq_insert 事件"""
        rq = RqEvent(
            timestamp=event_data['timestamp'],
            device=event_data['device'],
            rwbs=event_data['rwbs'],
            tag=event_data['tag'],
            sector=event_data['sector'],
            size=event_data['size'],
            process=event_data['process'],
            event_type='insert'
        )
        
        key = (rq.device, rq.sector)
        self.rq_events[key].append(rq)
        
        # 建立 tag 到 sector 的映射
        if rq.tag:
            self.rq_tag_map[rq.tag] = key
        
        self.stats['real_ios'] += 1
    
    def process_rq_issue(self, event_data: dict):
        """处理 block_rq_issue 事件"""
        rq = RqEvent(
            timestamp=event_data['timestamp'],
            device=event_data['device'],
            rwbs=event_data['rwbs'],
            tag=event_data['tag'],
            sector=event_data['sector'],
            size=event_data['size'],
            process=event_data['process'],
            event_type='issue'
        )
        
        key = (rq.device, rq.sector)
        self.rq_events[key].append(rq)
    
    def process_rq_complete(self, event_data: dict):
        """处理 block_rq_complete 事件"""
        rq = RqEvent(
            timestamp=event_data['timestamp'],
            device=event_data['device'],
            rwbs=event_data['rwbs'],
            tag=event_data['tag'],
            sector=event_data['sector'],
            size=event_data['size'],
            process=event_data['extra'] or event_data['process'],
            event_type='complete'
        )
        
        key = (rq.device, rq.sector)
        self.rq_events[key].append(rq)
    
    def parse_file(self, filename: str):
        """解析 trace 文件"""
        with open(filename, 'r') as f:
            for line in f:
                event_data = self.parse_trace_line(line)
                if not event_data:
                    continue
                
                event = event_data['event']
                
                if event == 'block_bio_queue':
                    self.process_bio_queue(event_data)
                elif event == 'block_bio_complete':
                    self.process_bio_complete(event_data)
                elif event == 'block_rq_insert':
                    self.process_rq_insert(event_data)
                elif event == 'block_rq_issue':
                    self.process_rq_issue(event_data)
                elif event == 'block_rq_complete':
                    self.process_rq_complete(event_data)
    
    def match_events(self):
        """匹配事件，构建完整的 I/O 请求"""
        # 遍历所有 bio_queue 事件
        for key, bio_list in self.bio_events.items():
            device, sector = key
            
            # 找到 queue 和 complete 事件
            queue_events = [b for b in bio_list if b.event_type == 'queue']
            complete_events = [b for b in bio_list if b.event_type == 'complete']
            
            if not queue_events:
                continue
            
            for queue_event in queue_events:
                io_req = IORequest(
                    device=device,
                    sector=sector,
                    size=queue_event.size,
                    rwbs=queue_event.rwbs,
                    process=queue_event.process,
                    bio_queue_time=queue_event.timestamp
                )
                
                # 查找对应的 complete 事件
                matching_complete = None
                for complete_event in complete_events:
                    if (complete_event.timestamp > queue_event.timestamp and
                        complete_event.size == queue_event.size):
                        matching_complete = complete_event
                        break
                
                if matching_complete:
                    io_req.bio_complete_time = matching_complete.timestamp
                    io_req.total_latency = (matching_complete.timestamp - 
                                          queue_event.timestamp) * 1000000  # 转换为微秒
                    
                    # 判断是否缓存命中
                    if io_req.total_latency < 100:  # < 100 微秒
                        io_req.is_cache_hit = True
                        self.stats['cache_hits'] += 1
                
                # 查找对应的 rq 事件
                rq_list = self.rq_events.get(key, [])
                
                insert_events = [r for r in rq_list if r.event_type == 'insert']
                issue_events = [r for r in rq_list if r.event_type == 'issue']
                complete_events_rq = [r for r in rq_list if r.event_type == 'complete']
                
                # 匹配时间最接近的 rq 事件
                for insert in insert_events:
                    if abs(insert.timestamp - queue_event.timestamp) < 0.001:  # 1ms 内
                        io_req.rq_insert_time = insert.timestamp
                        io_req.rq_tag = insert.tag
                        
                        # 查找对应的 issue
                        for issue in issue_events:
                            if issue.tag == insert.tag or abs(issue.timestamp - insert.timestamp) < 0.01:
                                io_req.rq_issue_time = issue.timestamp
                                
                                # 查找对应的 complete
                                for complete_rq in complete_events_rq:
                                    if (complete_rq.tag == insert.tag or 
                                        abs(complete_rq.timestamp - issue.timestamp) < 0.1):
                                        io_req.rq_complete_time = complete_rq.timestamp
                                        break
                                break
                        break
                
                # 计算各阶段延迟
                if io_req.rq_insert_time and io_req.rq_issue_time:
                    io_req.queue_latency = (io_req.rq_issue_time - 
                                           io_req.rq_insert_time) * 1000000
                
                if io_req.rq_issue_time and io_req.rq_complete_time:
                    io_req.device_latency = (io_req.rq_complete_time - 
                                            io_req.rq_issue_time) * 1000000
                
                self.io_requests.append(io_req)
    
    def generate_statistics(self) -> dict:
        """生成统计数据"""
        cache_hits = [req for req in self.io_requests if req.is_cache_hit]
        real_ios = [req for req in self.io_requests if not req.is_cache_hit and req.device_latency]
        
        reads = [req for req in real_ios if 'R' in req.rwbs and 'W' not in req.rwbs]
        writes = [req for req in real_ios if 'W' in req.rwbs]
        
        stats = {
            'summary': {
                'total_ios': len(self.io_requests),
                'cache_hits': len(cache_hits),
                'real_ios': len(real_ios),
                'cache_hit_rate': len(cache_hits) / len(self.io_requests) * 100 if self.io_requests else 0,
            },
            'latency': {},
            'throughput': {},
            'by_operation': {
                'read': {'count': len(reads)},
                'write': {'count': len(writes)},
            }
        }
        
        # 延迟统计
        if real_ios:
            total_latencies = [req.total_latency for req in real_ios if req.total_latency]
            device_latencies = [req.device_latency for req in real_ios if req.device_latency]
            queue_latencies = [req.queue_latency for req in real_ios if req.queue_latency]
            
            if total_latencies:
                stats['latency']['total'] = {
                    'mean': statistics.mean(total_latencies),
                    'median': statistics.median(total_latencies),
                    'p95': sorted(total_latencies)[int(len(total_latencies) * 0.95)],
                    'p99': sorted(total_latencies)[int(len(total_latencies) * 0.99)],
                    'max': max(total_latencies),
                }
            
            if device_latencies:
                stats['latency']['device'] = {
                    'mean': statistics.mean(device_latencies),
                    'median': statistics.median(device_latencies),
                    'p95': sorted(device_latencies)[int(len(device_latencies) * 0.95)],
                    'p99': sorted(device_latencies)[int(len(device_latencies) * 0.99)],
                }
            
            if queue_latencies:
                stats['latency']['queue'] = {
                    'mean': statistics.mean(queue_latencies),
                    'median': statistics.median(queue_latencies),
                }
        
        # 读写分别统计
        if reads:
            read_latencies = [req.device_latency for req in reads if req.device_latency]
            if read_latencies:
                stats['by_operation']['read']['latency'] = {
                    'mean': statistics.mean(read_latencies),
                    'p95': sorted(read_latencies)[int(len(read_latencies) * 0.95)],
                }
        
        if writes:
            write_latencies = [req.device_latency for req in writes if req.device_latency]
            if write_latencies:
                stats['by_operation']['write']['latency'] = {
                    'mean': statistics.mean(write_latencies),
                    'p95': sorted(write_latencies)[int(len(write_latencies) * 0.95)],
                }
        
        return stats
    
    def export_csv(self, filename: str):
        """导出为 CSV"""
        import csv
        
        with open(filename, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow([
                'Device', 'Sector', 'Size', 'RWBS', 'Process',
                'Bio Queue Time', 'Bio Complete Time', 'Total Latency (us)',
                'Rq Tag', 'Rq Insert Time', 'Rq Issue Time', 'Rq Complete Time',
                'Queue Latency (us)', 'Device Latency (us)',
                'Is Cache Hit', 'Is Merged'
            ])
            
            for req in self.io_requests:
                writer.writerow([
                    req.device, req.sector, req.size, req.rwbs, req.process,
                    req.bio_queue_time, req.bio_complete_time, req.total_latency,
                    req.rq_tag, req.rq_insert_time, req.rq_issue_time, req.rq_complete_time,
                    req.queue_latency, req.device_latency,
                    req.is_cache_hit, req.is_merged
                ])

def main():
    if len(sys.argv) < 2:
        print("Usage: python io_trace_analyzer.py <trace_file>")
        sys.exit(1)
    
    parser = IOTraceParser()
    
    print(f"Parsing {sys.argv[1]}...")
    parser.parse_file(sys.argv[1])
    
    print("Matching events...")
    parser.match_events()
    
    print("\n=== Statistics ===")
    stats = parser.generate_statistics()
    
    import json
    print(json.dumps(stats, indent=2))
    
    # 导出 CSV
    csv_file = sys.argv[1].replace('.txt', '_parsed.csv')
    parser.export_csv(csv_file)
    print(f"\nExported to {csv_file}")

if __name__ == '__main__':
    main()
```

### 3.2 版本对比脚本

```python
#!/usr/bin/env python3
# compare_versions.py

import json
import sys

def load_stats(filename):
    """加载统计数据"""
    with open(filename, 'r') as f:
        return json.load(f)

def compare_stats(baseline, current):
    """对比两个版本的统计数据"""
    report = {
        'summary': {},
        'latency_regression': {},
        'throughput_regression': {},
        'verdict': 'PASS'
    }
    
    # 缓存命中率对比
    baseline_cache_hit = baseline['summary']['cache_hit_rate']
    current_cache_hit = current['summary']['cache_hit_rate']
    cache_hit_change = current_cache_hit - baseline_cache_hit
    
    report['summary']['cache_hit_rate'] = {
        'baseline': baseline_cache_hit,
        'current': current_cache_hit,
        'change': cache_hit_change,
        'regression': cache_hit_change < -5  # 下降超过 5% 视为劣化
    }
    
    # 延迟对比
    for latency_type in ['total', 'device', 'queue']:
        if latency_type in baseline['latency'] and latency_type in current['latency']:
            baseline_lat = baseline['latency'][latency_type]
            current_lat = current['latency'][latency_type]
            
            for metric in ['mean', 'median', 'p95', 'p99']:
                if metric in baseline_lat and metric in current_lat:
                    b_val = baseline_lat[metric]
                    c_val = current_lat[metric]
                    change_pct = (c_val - b_val) / b_val * 100 if b_val > 0 else 0
                    
                    key = f"{latency_type}_{metric}"
                    report['latency_regression'][key] = {
                        'baseline': b_val,
                        'current': c_val,
                        'change_pct': change_pct,
                        'regression': change_pct > 10  # 增加超过 10% 视为劣化
                    }
                    
                    if change_pct > 10:
                        report['verdict'] = 'FAIL'
    
    # 读写操作对比
    for op in ['read', 'write']:
        if op in baseline['by_operation'] and op in current['by_operation']:
            if 'latency' in baseline['by_operation'][op] and 'latency' in current['by_operation'][op]:
                b_lat = baseline['by_operation'][op]['latency']['mean']
                c_lat = current['by_operation'][op]['latency']['mean']
                change_pct = (c_lat - b_lat) / b_lat * 100 if b_lat > 0 else 0
                
                report[f'{op}_latency'] = {
                    'baseline': b_lat,
                    'current': c_lat,
                    'change_pct': change_pct,
                    'regression': change_pct > 15
                }
                
                if change_pct > 15:
                    report['verdict'] = 'FAIL'
    
    return report

def generate_html_report(report, output_file):
    """生成 HTML 报告"""
    html = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>I/O Performance Regression Report</title>
        <style>
            body {{ font-family: Arial, sans-serif; margin: 20px; }}
            .pass {{ color: green; font-weight: bold; }}
            .fail {{ color: red; font-weight: bold; }}
            table {{ border-collapse: collapse; width: 100%; margin: 20px 0; }}
            th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
            th {{ background-color: #4CAF50; color: white; }}
            .regression {{ background-color: #ffcccc; }}
        </style>
    </head>
    <body>
        <h1>I/O Performance Regression Test Report</h1>
        <h2>Verdict: <span class="{report['verdict'].lower()}">{report['verdict']}</span></h2>
        
        <h3>Summary</h3>
        <table>
            <tr><th>Metric</th><th>Baseline</th><th>Current</th><th>Change</th><th>Status</th></tr>
    """
    
    # 添加缓存命中率
    cache = report['summary']['cache_hit_rate']
    status_class = 'regression' if cache['regression'] else ''
    html += f"""
            <tr class="{status_class}">
                <td>Cache Hit Rate</td>
                <td>{cache['baseline']:.2f}%</td>
                <td>{cache['current']:.2f}%</td>
                <td>{cache['change']:+.2f}%</td>
                <td>{'REGRESSION' if cache['regression'] else 'OK'}</td>
            </tr>
    """
    
    html += "</table><h3>Latency Comparison</h3><table>"
    html += "<tr><th>Metric</th><th>Baseline (μs)</th><th>Current (μs)</th><th>Change %</th><th>Status</th></tr>"
    
    for key, data in report['latency_regression'].items():
        status_class = 'regression' if data['regression'] else ''
        html += f"""
            <tr class="{status_class}">
                <td>{key}</td>
                <td>{data['baseline']:.2f}</td>
                <td>{data['current']:.2f}</td>
                <td>{data['change_pct']:+.2f}%</td>
                <td>{'REGRESSION' if data['regression'] else 'OK'}</td>
            </tr>
        """
    
    html += "</table></body></html>"
    
    with open(output_file, 'w') as f:
        f.write(html)

def main():
    if len(sys.argv) < 3:
        print("Usage: python compare_versions.py <baseline_stats.json> <current_stats.json>")
        sys.exit(1)
    
    baseline = load_stats(sys.argv[1])
    current = load_stats(sys.argv[2])
    
    report = compare_stats(baseline, current)
    
    print(json.dumps(report, indent=2))
    
    # 生成 HTML 报告
    generate_html_report(report, 'io_regression_report.html')
    print("\nHTML report generated: io_regression_report.html")
    
    # 返回退出码
    sys.exit(0 if report['verdict'] == 'PASS' else 1)

if __name__ == '__main__':
    main()
```

---

## 3. 日志序列匹配的通用方案

### 关键问题和解决方案

#### 问题 1：bio 和 request 的关联

```python
# 方法 1：基于 (device, sector) 匹配
key = (device, sector)

# 方法 2：基于时间窗口匹配
# bio_queue 和 rq_insert 通常在 1ms 内

# 方法 3：基于 size 匹配
# 如果 size 相同且时间接近

def match_bio_to_rq(bio_event, rq_events, time_window_ms=1.0):
    """匹配 bio 到 rq"""
    candidates = []
    
    for rq in rq_events:
        # 条件 1：设备和扇区相同
        if rq.device == bio_event.device and rq.sector == bio_event.sector:
            # 条件 2：时间接近
            time_diff = abs(rq.timestamp - bio_event.timestamp) * 1000  # ms
            if time_diff < time_window_ms:
                # 条件 3：大小相同或合并
                if rq.size == bio_event.size or rq.size > bio_event.size:
                    candidates.append((time_diff, rq))
    
    # 返回时间最接近的
    if candidates:
        candidates.sort(key=lambda x: x[0])
        return candidates[0][1]
    
    return None
```

#### 问题 2：请求合并的处理

```python
def detect_merge(bio_events, rq_events):
    """检测请求合并"""
    merges = []
    
    # 查找 size 不匹配的情况
    for rq in rq_events:
        # 查找所有可能被合并的 bio
        related_bios = []
        total_bio_size = 0
        
        for bio in bio_events:
            # 扇区范围重叠
            bio_end = bio.sector + bio.size
            rq_end = rq.sector + rq.size
            
            if (bio.sector >= rq.sector and bio.sector < rq_end) or \
               (rq.sector >= bio.sector and rq.sector < bio_end):
                related_bios.append(bio)
                total_bio_size += bio.size
        
        # 如果多个 bio 的总大小等于 rq 的大小
        if len(related_bios) > 1 and total_bio_size == rq.size:
            merges.append({
                'rq': rq,
                'merged_bios': related_bios,
                'merge_count': len(related_bios)
            })
    
    return merges
```

#### 问题 3：多层设备映射

```python
def trace_device_mapping(bio_events):
    """追踪设备映射"""
    dm_mapping = defaultdict(list)
    
    # 按时间排序
    sorted_bios = sorted(bio_events, key=lambda x: x.timestamp)
    
    for i, bio in enumerate(sorted_bios):
        # 查找时间接近的其他 bio
        for j in range(i+1, min(i+10, len(sorted_bios))):
            next_bio = sorted_bios[j]
            
            # 时间差小于 100 微秒
            if (next_bio.timestamp - bio.timestamp) < 0.0001:
                # 扇区相同或接近
                if abs(next_bio.sector - bio.sector) < 1024:  # 512KB 范围内
                    # 不同设备 → 可能是映射
                    if next_bio.device != bio.device:
                        dm_mapping[bio.device].append(next_bio.device)
    
    return dm_mapping
```

### 完整的匹配策略

```python
class IORequestMatcher:
    """I/O 请求匹配器"""
    
    def __init__(self):
        self.match_strategies = [
            self.exact_match,        # 策略 1：精确匹配
            self.merge_match,        # 策略 2：合并匹配
            self.dm_aware_match,     # 策略 3：设备映射匹配
            self.fuzzy_time_match,   # 策略 4：模糊时间匹配
        ]
    
    def exact_match(self, bio, rq_list):
        """精确匹配：device, sector, size 都相同"""
        for rq in rq_list:
            if (rq.device == bio.device and 
                rq.sector == bio.sector and
                rq.size == bio.size):
                time_diff = abs(rq.timestamp - bio.timestamp)
                if time_diff < 0.001:  # 1ms
                    return rq
        return None
    
    def merge_match(self, bio, rq_list):
        """合并匹配：rq.size >= bio.size"""
        candidates = []
        for rq in rq_list:
            if rq.device == bio.device:
                # 检查扇区范围是否包含
                rq_end = rq.sector + rq.size
                bio_end = bio.sector + bio.size
                
                if (bio.sector >= rq.sector and bio_end <= rq_end):
                    time_diff = abs(rq.timestamp - bio.timestamp)
                    if time_diff < 0.01:  # 10ms
                        candidates.append((time_diff, rq))
        
        if candidates:
            candidates.sort(key=lambda x: x[0])
            return candidates[0][1]
        return None
    
    def dm_aware_match(self, bio, rq_list, dm_map):
        """设备映射感知匹配"""
        # 如果 bio 的设备是 dm 设备
        if bio.device in dm_map:
            physical_devices = dm_map[bio.device]
            
            for rq in rq_list:
                if rq.device in physical_devices:
                    # 时间接近即可
                    time_diff = abs(rq.timestamp - bio.timestamp)
                    if time_diff < 0.0001:  # 100us
                        return rq
        return None
    
    def fuzzy_time_match(self, bio, rq_list):
        """模糊时间匹配：最后的兜底策略"""
        candidates = []
        for rq in rq_list:
            time_diff = abs(rq.timestamp - bio.timestamp)
            if time_diff < 0.1:  # 100ms
                candidates.append((time_diff, rq))
        
        if candidates:
            candidates.sort(key=lambda x: x[0])
            return candidates[0][1]
        return None
    
    def match(self, bio, rq_list, dm_map=None):
        """尝试所有策略"""
        for strategy in self.match_strategies:
            if strategy == self.dm_aware_match and dm_map:
                result = strategy(bio, rq_list, dm_map)
            else:
                result = strategy(bio, rq_list)
            
            if result:
                return result
        
        return None
```

### 边界情况处理

```python
# 边界情况 1：page cache 命中
if bio_complete_time - bio_queue_time < 0.0001:  # < 100us
    io_req.is_cache_hit = True
    # 不需要匹配 rq 事件

# 边界情况 2：请求合并
# 多个 bio → 一个 rq
# 需要建立 many-to-one 映射

# 边界情况 3：请求分裂
# 一个 bio → 多个 rq（大 I/O 被拆分）
# 需要建立 one-to-many 映射

# 边界情况 4：设备映射
# dm-16 → dm-4 → sdc
# 同一个 I/O 有 3 个 bio_queue 事件
# 需要识别映射关系

# 边界情况 5：乱序完成
# issue 顺序: A → B → C
# complete 顺序: B → A → C
# 需要用 tag 或 sector 匹配，不能用顺序

# 边界情况 6：超时/错误
# 有 queue/insert/issue，但没有 complete
# 需要标记为超时

# 边界情况 7：Direct I/O vs Buffered I/O
# Direct I/O: 完整事件序列
# Buffered I/O: 可能只有 bio 事件（写入页缓存）
```

---

## 最终的 SOP 文档

```markdown
# I/O Performance Regression Testing SOP

## 目标
检测版本升级后 I/O 性能劣化

## 环境要求
- 2 台相同硬件的测试设备
- baseline 版本和 current 版本镜像
- adb 工具
- Python 3.8+

## 步骤

### 1. 准备阶段 (30分钟)
1.1 刷写版本
    - 设备 A: baseline 版本
    - 设备 B: current 版本

1.2 环境配置
    ```bash
    ./prepare_device.sh DEVICE_A
    ./prepare_device.sh DEVICE_B
    ./set_baseline.sh DEVICE_A
    ./set_baseline.sh DEVICE_B
    ```

1.3 预热
    ```bash
    # 运行 3 次预热测试
    for i in {1..3}; do
        ./io_benchmark.sh DEVICE_A warmup
        ./io_benchmark.sh DEVICE_B warmup
    done
    ```

### 2. 测试执行 (2小时)
2.1 基准测试 (每个设备 30分钟)
    ```bash
    ./io_benchmark.sh DEVICE_A baseline
    ./io_benchmark.sh DEVICE_B current
    ```

2.2 压力测试 (每个设备 30分钟)
    ```bash
    ./io_stress_test.sh DEVICE_A baseline
    ./io_stress_test.sh DEVICE_B current
    ```

### 3. 数据分析 (30分钟)
3.1 解析日志
    ```bash
    python io_trace_analyzer.py logs/baseline/*.txt > baseline_stats.json
    python io_trace_analyzer.py logs/current/*.txt > current_stats.json
    ```

3.2 生成对比报告
    ```bash
    python compare_versions.py baseline_stats.json current_stats.json
    ```

### 4. 判定标准
| 指标 | 阈值 | 判定 |
|------|------|------|
| Cache Hit Rate 下降 | > 5% | FAIL |
| Mean Latency 增加 | > 10% | FAIL |
| P95 Latency 增加 | > 15% | FAIL |
| Read Latency 增加 | > 15% | FAIL |
| Write Latency 增加 | > 20% | FAIL |

### 5. 结果归档
- 保存所有日志到 `archive/YYYYMMDD_baseline_vs_current/`
- 提交 HTML 报告到缺陷跟踪系统
- 如果 FAIL，通知开发团队

## 自动化脚本
完整自动化：
```bash
./run_io_regression_test.sh baseline_version current_version
```

输出：
- PASS/FAIL 判定
- HTML 报告
- 详细日志
```

---

希望这个完整的 SOP 能帮助你建立 I/O 性能回归测试体系！关键点总结：

1. **DM 层理解**：多层虚拟设备，需要追踪映射关系
2. **测试 SOP**：环境准备→执行→分析→判定的完整流程
3. **日志解析**：多策略匹配，处理合并、映射、缓存等边界情况

如果有具体的技术细节需要深入，随时告诉我！