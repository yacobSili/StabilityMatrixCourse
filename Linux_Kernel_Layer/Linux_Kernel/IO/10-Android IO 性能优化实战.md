# Android IO 性能优化实战

## 学习目标

- 掌握 Android 系统级 IO 优化方法
- 学会应用层 IO 优化技巧
- 理解存储设备优化策略
- 掌握 IO 监控与告警方法
- 能够独立进行 IO 性能优化

## 背景介绍

IO 性能优化是 Android 系统性能优化的重要组成部分。通过系统级的配置优化、应用层的代码优化、存储设备的参数调优，可以显著提升系统响应速度、应用启动速度和整体用户体验。本文提供 Android IO 性能优化的实战方法和最佳实践，帮助在实际项目中应用这些优化技巧。

## Android 系统级 IO 优化

### 1. 关键进程的 IO 优先级设置

#### SystemServer 进程优化

**SystemServer 是 Android 系统的核心服务进程**，其 IO 性能直接影响系统响应速度。

**优化方法**：
```bash
# 方法 1：使用 ionice 设置（临时）
adb shell su -c "ionice -c 1 -n 4 -p $(pidof system_server)"

# 方法 2：在 init 脚本中设置（永久）
# 在 /system/etc/init/ 目录下创建脚本
# 文件：system_server_io_priority.rc
service set_system_server_io_priority /system/bin/sh /system/etc/init/set_io_priority.sh
    class main
    user root
    oneshot

# 脚本内容：/system/etc/init/set_io_priority.sh
#!/system/bin/sh
pid=$(pidof system_server)
if [ -n "$pid" ]; then
    ionice -c 1 -n 4 -p $pid
fi
```

**方法 3：在 Framework 层设置**（需要修改源码）：
```java
// 在 SystemServer.java 的 main() 方法中
// 注意：Android Framework 中没有 Process.setIOPriority() 方法
// 需要通过其他方式设置 IO 优先级

public static void main(String[] args) {
    // 方法 1：通过 Runtime.exec() 调用 ionice
    try {
        int pid = Process.myPid();
        Process process = Runtime.getRuntime().exec(
            new String[]{"su", "-c", "ionice -c 1 -n 4 -p " + pid});
        process.waitFor();
    } catch (Exception e) {
        // 处理异常
    }
    
    // 方法 2：如果实现了 JNI 函数（需要先添加）
    // int ioprio = (1 << 13) | 4;  // RT 类别，级别 4
    // Process.setIOPriority(Process.myPid(), ioprio);
    
    // ... 其他初始化代码
}
```

#### Zygote 进程优化

**Zygote 是应用进程的父进程**，优化其 IO 性能可以加快应用启动速度。

**优化方法**：
```bash
# 在 init 脚本中设置
# 文件：zygote_io_priority.rc
service set_zygote_io_priority /system/bin/sh /system/etc/init/set_zygote_io_priority.sh
    class main
    user root
    oneshot

# 脚本内容
#!/system/bin/sh
pid=$(pidof zygote)
if [ -n "$pid" ]; then
    ionice -c 2 -n 0 -p $pid  # BE 类别，最高级别
fi
```

#### 前台应用优化

**为前台应用设置较高的 IO 优先级**，确保用户体验。

**优化方法**：
```java
// 在 ActivityManagerService.java 中
// 当应用切换到前台时
// 注意：Android Framework 中没有 Process.setIOPriority() 方法
private void setForegroundAppIOPriority(ProcessRecord app) {
    if (app.pid > 0) {
        // 方法 1：通过 Runtime.exec() 调用 ionice（推荐）
        try {
            Runtime.getRuntime().exec(
                new String[]{"su", "-c", "ionice -c 2 -n 0 -p " + app.pid}).waitFor();
        } catch (Exception e) {
            // 处理异常
        }
        
        // 方法 2：如果实现了 JNI 函数（需要先添加）
        // int ioprio = (2 << 13) | 0;  // BE 类别，级别 0
        // Process.setIOPriority(app.pid, ioprio);
    }
}
```

### 2. 系统服务的 IO 调度器配置

#### 选择适合的调度器

**根据设备类型选择调度器**：

```bash
# 检测设备类型
device_type=$(cat /sys/block/sda/queue/rotational)

if [ "$device_type" = "0" ]; then
    # SSD/UFS 设备，使用 none 或 kyber
    echo "kyber" > /sys/block/sda/queue/scheduler
    echo 2000000 > /sys/block/sda/queue/iosched/read_lat_nsec
    echo 10000000 > /sys/block/sda/queue/iosched/write_lat_nsec
else
    # 机械硬盘，使用 mq-deadline
    echo "mq-deadline" > /sys/block/sda/queue/scheduler
    echo 200 > /sys/block/sda/queue/iosched/read_expire
    echo 5000 > /sys/block/sda/queue/iosched/write_expire
fi
```

#### 优化调度器参数

**mq-deadline 参数优化**：
```bash
# 针对 Android 设备优化的参数
echo 200 > /sys/block/sda/queue/iosched/read_expire      # 读延迟：200ms
echo 3000 > /sys/block/sda/queue/iosched/write_expire   # 写延迟：3s
echo 8 > /sys/block/sda/queue/iosched/fifo_batch        # 批处理：8
echo 2 > /sys/block/sda/queue/iosched/writes_starved    # 写饥饿阈值：2
```

**kyber 参数优化**：
```bash
# 针对 Android 设备优化的参数
echo 2000000 > /sys/block/sda/queue/iosched/read_lat_nsec   # 读延迟：2ms
echo 10000000 > /sys/block/sda/queue/iosched/write_lat_nsec  # 写延迟：10ms
```

### 3. cgroup IO 资源隔离

#### 系统服务隔离

**为系统服务创建独立的 cgroup**，避免相互影响。

```bash
# 创建系统服务 cgroup
mkdir -p /sys/fs/cgroup/system_services

# 设置 IO 权重
echo "default 200" > /sys/fs/cgroup/system_services/io.weight

# 将 SystemServer 加入 cgroup
echo $(pidof system_server) > /sys/fs/cgroup/system_services/cgroup.procs
```

#### 后台应用隔离

**限制后台应用的 IO 带宽**，避免影响前台应用。

```bash
# 创建后台应用 cgroup
mkdir -p /sys/fs/cgroup/background_apps

# 限制 IO 带宽（例如：2MB/s 写入）
echo "8:16 wbps=2097152" > /sys/fs/cgroup/background_apps/io.max

# 设置较低的 IO 权重
echo "default 50" > /sys/fs/cgroup/background_apps/io.weight
```

#### 前台应用保护

**为前台应用设置延迟保证**。

```bash
# 创建前台应用 cgroup
mkdir -p /sys/fs/cgroup/foreground_apps

# 设置 IO 延迟目标（10ms）
echo "8:16 target=10000" > /sys/fs/cgroup/foreground_apps/io.latency

# 设置较高的 IO 权重
echo "default 300" > /sys/fs/cgroup/foreground_apps/io.weight
```

## 应用层 IO 优化

### 1. 减少小文件 IO

#### 合并写入

**将多个小文件写入合并为一次大文件写入**。

**优化前**：
```java
// 多次小文件写入
for (int i = 0; i < 100; i++) {
    File file = new File("/data/app/data_" + i + ".txt");
    FileWriter writer = new FileWriter(file);
    writer.write("data " + i);
    writer.close();
}
```

**优化后**：
```java
// 合并为一次写入
FileOutputStream fos = new FileOutputStream("/data/app/data_all.txt");
BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(fos));
for (int i = 0; i < 100; i++) {
    writer.write("data " + i + "\n");
}
writer.close();
```

#### 批量读取

**批量读取文件，减少 IO 次数**。

**优化前**：
```java
// 逐个读取文件
for (String filename : fileList) {
    String content = readFile(filename);
    process(content);
}
```

**优化后**：
```java
// 批量读取
List<String> contents = new ArrayList<>();
for (String filename : fileList) {
    contents.add(readFile(filename));
}
// 批量处理
for (String content : contents) {
    process(content);
}
```

### 2. 使用异步 IO

#### AsyncTask 异步写入

**使用异步任务进行 IO 操作**，避免阻塞主线程。

```java
private class AsyncWriteTask extends AsyncTask<String, Void, Boolean> {
    @Override
    protected Boolean doInBackground(String... params) {
        try {
            FileWriter writer = new FileWriter(params[0]);
            writer.write(params[1]);
            writer.close();
            return true;
        } catch (IOException e) {
            return false;
        }
    }
    
    @Override
    protected void onPostExecute(Boolean success) {
        if (success) {
            // 写入成功
        }
    }
}

// 使用
new AsyncWriteTask().execute(filename, content);
```

#### 使用线程池

**使用线程池管理异步 IO 任务**。

```java
private ExecutorService ioExecutor = Executors.newFixedThreadPool(4);

public void writeFileAsync(String filename, String content) {
    ioExecutor.submit(() -> {
        try {
            FileWriter writer = new FileWriter(filename);
            writer.write(content);
            writer.close();
        } catch (IOException e) {
            Log.e(TAG, "Write failed", e);
        }
    });
}
```

### 3. 合理使用 Page Cache

#### 预加载关键文件

**在应用启动时预加载关键文件到 Page Cache**。

```java
public void preloadFiles() {
    String[] criticalFiles = {
        "/data/app/base.apk",
        "/data/app/split_config.armeabi_v7a.apk",
        "/data/app/resources.arsc"
    };
    
    for (String file : criticalFiles) {
        new Thread(() -> {
            try {
                // 读取文件到 Page Cache
                FileInputStream fis = new FileInputStream(file);
                byte[] buffer = new byte[8192];
                while (fis.read(buffer) > 0) {
                    // 只是读取，不处理
                }
                fis.close();
            } catch (IOException e) {
                Log.e(TAG, "Preload failed: " + file, e);
            }
        }).start();
    }
}
```

#### 使用内存映射

**使用内存映射文件减少 IO 开销**。

```java
// 使用内存映射读取文件
FileChannel channel = new RandomAccessFile(file, "r").getChannel();
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());

// 直接访问内存，无需系统调用
byte[] data = new byte[buffer.remaining()];
buffer.get(data);
```

## 存储设备优化

### 1. UFS/eMMC 参数调优

#### UFS 设备优化

**优化 UFS 设备的队列深度和参数**。

```bash
# 查看 UFS 设备信息
adb shell cat /sys/class/ufs/*/speed_attr

# 优化队列深度（如果支持）
adb shell su -c "echo 256 > /sys/block/sda/queue/nr_requests"

# 优化调度器
adb shell su -c "echo kyber > /sys/block/sda/queue/scheduler"
```

#### eMMC 设备优化

**优化 eMMC 设备的参数**。

```bash
# 使用 mq-deadline 调度器
adb shell su -c "echo mq-deadline > /sys/block/sda/queue/scheduler"

# 优化参数
adb shell su -c "echo 200 > /sys/block/sda/queue/iosched/read_expire"
adb shell su -c "echo 5000 > /sys/block/sda/queue/iosched/write_expire"
```

### 2. 文件系统选择

#### ext4 vs f2fs

**根据使用场景选择文件系统**。

| 特性 | ext4 | f2fs |
|------|------|------|
| 随机写入性能 | 一般 | 优秀 |
| 顺序写入性能 | 优秀 | 一般 |
| 随机读取性能 | 优秀 | 优秀 |
| 适合场景 | 系统分区、只读分区 | 数据分区、频繁写入 |

**Android 设备常见配置**：
- **系统分区（system）**：ext4（只读，顺序读取多）
- **数据分区（data）**：f2fs（频繁写入，随机写入多）
- **缓存分区（cache）**：f2fs（频繁写入）

### 3. TRIM 操作优化

#### 定期 TRIM

**定期执行 TRIM 操作，保持设备性能**。

```bash
# 手动执行 TRIM
adb shell su -c "fstrim -v /data"

# 在 init 脚本中定期执行
# 文件：trim.rc
service trim_data /system/bin/fstrim -v /data
    class main
    oneshot
    disabled

# 在合适的时间触发（例如：夜间）
on property:sys.boot_completed=1
    # 延迟执行，避免影响启动
    exec_background -s 3600 /system/bin/fstrim -v /data
```

## IO 监控与告警

### 1. 建立 IO 性能基线

#### 收集基线数据

**在正常负载下收集 IO 性能数据**。

```bash
#!/bin/bash
# 收集 IO 性能基线数据
# 文件：collect_io_baseline.sh

OUTPUT_DIR="/data/io_baseline"
mkdir -p $OUTPUT_DIR

# 收集 iostat 数据（1 分钟）
iostat -x 1 60 > $OUTPUT_DIR/iostat_baseline.log

# 收集 iotop 数据
iotop -o -d 1 -t -P > $OUTPUT_DIR/iotop_baseline.log &

# 收集调度器参数
cat /sys/block/sda/queue/scheduler > $OUTPUT_DIR/scheduler_baseline.txt
cat /sys/block/sda/queue/iosched/* > $OUTPUT_DIR/scheduler_params_baseline.txt

# 收集 Tag 使用情况
cat /sys/block/sda/queue/nr_requests > $OUTPUT_DIR/nr_requests_baseline.txt
cat /sys/block/sda/queue/nr_active > $OUTPUT_DIR/nr_active_baseline.txt

sleep 60
killall iotop
```

#### 分析基线数据

**分析基线数据，确定正常范围**。

```bash
#!/bin/bash
# 分析 IO 性能基线
# 文件：analyze_io_baseline.sh

BASELINE_FILE="/data/io_baseline/iostat_baseline.log"

# 提取关键指标
awk '/sda/ {
    if (NR > 3) {  # 跳过标题行
        sum_util += $NF
        sum_await += $(NF-4)
        count++
    }
}
END {
    avg_util = sum_util / count
    avg_await = sum_await / count
    print "Average %util: " avg_util
    print "Average await: " avg_await " ms"
    print "Baseline range:"
    print "  %util: 0 - " (avg_util * 1.5)
    print "  await: 0 - " (avg_await * 2) " ms"
}' $BASELINE_FILE
```

### 2. 设置 IO 延迟告警

#### 监控脚本

**监控 IO 延迟，超过阈值时告警**。

```bash
#!/bin/bash
# IO 延迟监控脚本
# 文件：monitor_io_latency.sh

THRESHOLD_AWAIT=50  # 告警阈值：50ms
THRESHOLD_UTIL=90   # 告警阈值：90%

while true; do
    # 获取当前指标
    iostat_output=$(iostat -x 1 1 | grep sda | tail -1)
    
    if [ -n "$iostat_output" ]; then
        await=$(echo $iostat_output | awk '{print $(NF-4)}')
        util=$(echo $iostat_output | awk '{print $NF}')
        
        # 检查是否超过阈值
        if (( $(echo "$await > $THRESHOLD_AWAIT" | bc -l) )); then
            echo "$(date): IO await too high: ${await}ms" >> /data/io_alerts.log
            # 发送告警（例如：通过 logcat）
            log -p e -t IOAlert "IO await too high: ${await}ms"
        fi
        
        if (( $(echo "$util > $THRESHOLD_UTIL" | bc -l) )); then
            echo "$(date): IO util too high: ${util}%" >> /data/io_alerts.log
            log -p e -t IOAlert "IO util too high: ${util}%"
        fi
    fi
    
    sleep 5
done
```

### 3. 定期 IO 性能分析

#### 定期收集数据

**定期收集 IO 性能数据，用于趋势分析**。

```bash
#!/bin/bash
# 定期 IO 性能分析
# 文件：periodic_io_analysis.sh

OUTPUT_DIR="/data/io_analysis/$(date +%Y%m%d)"
mkdir -p $OUTPUT_DIR

# 收集 5 分钟的 IO 数据
iostat -x 1 300 > $OUTPUT_DIR/iostat_$(date +%H%M).log &

# 收集进程 IO 数据
iotop -o -d 1 -t -P > $OUTPUT_DIR/iotop_$(date +%H%M).log &

sleep 300
killall iostat iotop

# 生成报告
echo "=== IO Performance Report ===" > $OUTPUT_DIR/report.txt
echo "Date: $(date)" >> $OUTPUT_DIR/report.txt
echo "" >> $OUTPUT_DIR/report.txt

# 分析 iostat 数据
echo "=== Device Statistics ===" >> $OUTPUT_DIR/report.txt
tail -1 $OUTPUT_DIR/iostat_*.log | awk '{print "Avg await: " $(NF-4) "ms, %util: " $NF}' >> $OUTPUT_DIR/report.txt

# 分析 iotop 数据
echo "" >> $OUTPUT_DIR/report.txt
echo "=== Top IO Processes ===" >> $OUTPUT_DIR/report.txt
grep -E "^[0-9]" $OUTPUT_DIR/iotop_*.log | sort -k6 -rn | head -10 >> $OUTPUT_DIR/report.txt
```

## 实战案例

### 案例 1：Android 设备冷启动优化

**目标**：优化设备冷启动时间，减少 IO 延迟。

**步骤**：

1. **分析启动过程的 IO 模式**：
```bash
# 使用 systrace 追踪启动过程
python systrace.py -t 30 -o boot_trace.html sched freq idle am wm gfx view binder_driver hal dalvik camera input res
```

2. **优化关键进程的 IO 优先级**：
```bash
# 在 init 脚本中设置
service set_boot_io_priority /system/bin/sh /system/etc/init/set_boot_io_priority.sh
    class main
    user root
    oneshot
```

3. **优化调度器参数**：
```bash
# 针对启动优化
echo 100 > /sys/block/sda/queue/iosched/read_expire  # 降低读延迟
echo 4 > /sys/block/sda/queue/iosched/fifo_batch      # 减少批处理
```

4. **预加载关键文件**：
```java
// 在 SystemServer 启动时预加载
public void preloadCriticalFiles() {
    String[] files = {
        "/system/framework/framework.jar",
        "/system/framework/services.jar"
    };
    // 预加载逻辑
}
```

### 案例 2：后台应用 IO 限制配置

**目标**：限制后台应用的 IO 带宽，避免影响前台应用。

**步骤**：

1. **创建后台应用 cgroup**：
```bash
mkdir -p /sys/fs/cgroup/background_apps
echo "8:16 wbps=2097152 riops=1000 wiops=500" > /sys/fs/cgroup/background_apps/io.max
echo "default 50" > /sys/fs/cgroup/background_apps/io.weight
```

2. **在 ActivityManagerService 中管理**：
```java
// 当应用切换到后台时
private void moveToBackgroundCgroup(ProcessRecord app) {
    if (app.pid > 0) {
        try {
            String cgroupPath = "/sys/fs/cgroup/background_apps/cgroup.procs";
            Files.write(Paths.get(cgroupPath), 
                String.valueOf(app.pid).getBytes());
        } catch (IOException e) {
            Log.e(TAG, "Failed to move to background cgroup", e);
        }
    }
}
```

### 案例 3：系统更新时的 IO 优化

**目标**：优化系统更新过程的 IO 性能。

**步骤**：

1. **设置更新进程的 IO 优先级**：
```bash
# 为更新进程设置较高的 IO 优先级
ionice -c 2 -n 0 -p $(pidof update_engine)
```

2. **限制其他进程的 IO**：
```bash
# 限制非关键进程的 IO 带宽
mkdir -p /sys/fs/cgroup/update_restriction
echo "8:16 wbps=524288" > /sys/fs/cgroup/update_restriction/io.max
# 将非关键进程加入该 cgroup
```

3. **优化调度器参数**：
```bash
# 临时调整调度器参数，优先处理更新 IO
echo 5000 > /sys/block/sda/queue/iosched/write_expire
echo 32 > /sys/block/sda/queue/iosched/fifo_batch
```

## 总结

### 核心要点

1. **系统级优化**：
   - 关键进程的 IO 优先级设置
   - 调度器选择和参数优化
   - cgroup IO 资源隔离

2. **应用层优化**：
   - 减少小文件 IO
   - 使用异步 IO
   - 合理使用 Page Cache

3. **设备优化**：
   - UFS/eMMC 参数调优
   - 文件系统选择
   - TRIM 操作优化

4. **监控与告警**：
   - 建立性能基线
   - 设置延迟告警
   - 定期性能分析

### 最佳实践

1. **根据设备类型选择配置**：
   - UFS 3.0+：使用 `kyber` 或 `none`
   - eMMC：使用 `mq-deadline`

2. **分层优化**：
   - 系统级：关键进程优先级
   - 应用级：代码优化
   - 设备级：参数调优

3. **持续监控**：
   - 建立基线
   - 定期分析
   - 及时告警

### 关键概念

- **IO 优先级**：控制 IO 请求的处理顺序
- **IO 调度器**：对 IO 请求进行排序和调度
- **cgroup IO 控制**：基于 cgroup 的 IO 资源管理
- **Page Cache**：内核的文件缓存机制

## 参考资料

- Linux 内核源码：`block/` - IO 子系统实现
- Android 源码：`frameworks/base/services/core/java/com/android/server/am/`
- iostat、iotop 工具文档
- cgroup v2 文档：`Documentation/admin-guide/cgroup-v2.rst`

## 更新记录

- 2026-01-27：初始创建，包含 Android IO 性能优化实战
