# Perfetto SQL 查询 Block 层 Trace Events 指南

## 学习目标

- 理解 Perfetto 数据库的表结构
- 掌握 Block 层 trace events 的 SQL 查询方法
- 学会分析 IO 延迟、吞吐量、进程 IO 行为
- 能够编写自定义查询进行性能分析

## 概述

Perfetto 是 Android 和 Linux 系统的现代化追踪工具，它将 trace 数据存储为 SQLite 数据库格式。通过 SQL 查询，我们可以灵活地分析 Block 层的 IO 行为。本文详细介绍如何使用 SQL 查询 block trace events。

---

## 一、Perfetto 数据库表结构

### 1.1 核心表关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Perfetto ftrace 相关表结构                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐         ┌──────────────────┐                     │
│  │   ftrace_event   │         │      args        │                     │
│  ├──────────────────┤         ├──────────────────┤                     │
│  │ id               │────────►│ arg_set_id       │                     │
│  │ ts (时间戳)       │         │ key (参数名)     │                     │
│  │ name (事件名)     │         │ display_value    │                     │
│  │ cpu              │         │ int_value        │                     │
│  │ utid             │         │ string_value     │                     │
│  │ arg_set_id ──────┼────────►│ real_value       │                     │
│  └────────┬─────────┘         └──────────────────┘                     │
│           │                                                             │
│           │ utid                                                        │
│           ▼                                                             │
│  ┌──────────────────┐         ┌──────────────────┐                     │
│  │     thread       │         │     process      │                     │
│  ├──────────────────┤         ├──────────────────┤                     │
│  │ utid             │         │ upid             │                     │
│  │ tid              │────────►│ pid              │                     │
│  │ name             │         │ name             │                     │
│  │ upid ────────────┼────────►│                  │                     │
│  └──────────────────┘         └──────────────────┘                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 表字段说明

#### ftrace_event 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER | 事件唯一标识 |
| `ts` | INTEGER | 时间戳（纳秒） |
| `name` | TEXT | 事件名称（如 `block_rq_issue`） |
| `cpu` | INTEGER | 发生事件的 CPU 编号 |
| `utid` | INTEGER | 唯一线程 ID（关联 thread 表） |
| `arg_set_id` | INTEGER | 参数集 ID（关联 args 表） |

#### args 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `arg_set_id` | INTEGER | 参数集 ID |
| `key` | TEXT | 参数名称 |
| `display_value` | TEXT | 显示值（字符串形式） |
| `int_value` | INTEGER | 整数值 |
| `string_value` | TEXT | 字符串值 |
| `real_value` | REAL | 浮点值 |

### 1.3 Block 事件的参数字段

```sql
-- 查看所有 block 事件的参数字段
SELECT DISTINCT 
  fe.name as event_name,
  args.key as field_name
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name LIKE 'block%'
ORDER BY fe.name, args.key;
```

**常见参数字段**：

| 参数名 | 说明 | 示例值 |
|--------|------|--------|
| `dev` | 设备号（major,minor 编码） | `8388624` |
| `sector` | 起始扇区号 | `12345678` |
| `nr_sector` | 扇区数量 | `8` |
| `bytes` | 字节数 | `4096` |
| `rwbs` | 操作类型标志 | `WS`、`R`、`RA` |
| `comm` | 进程名 | `dd`、`kworker/0:1` |
| `ioprio` | IO 优先级 | `16388` |
| `error` | 错误码（仅 complete） | `0` |

---

## 二、基础查询

### 2.1 查看所有 Block 事件

```sql
-- 查看抓取到的 block 事件类型和数量
SELECT 
  name,
  COUNT(*) as count
FROM ftrace_event
WHERE name LIKE 'block%'
GROUP BY name
ORDER BY count DESC;
```

**输出示例**：
```
name                  count
--------------------  -----
block_rq_complete     1234
block_rq_issue        1234
block_rq_insert       1200
block_bio_queue       1500
block_bio_complete    1500
```

### 2.2 查看事件时间范围

```sql
-- 查看 trace 的时间范围
SELECT 
  MIN(ts) / 1000000000.0 as start_sec,
  MAX(ts) / 1000000000.0 as end_sec,
  (MAX(ts) - MIN(ts)) / 1000000000.0 as duration_sec
FROM ftrace_event
WHERE name LIKE 'block%';
```

### 2.3 查看单个事件的完整信息

```sql
-- 查看 block_rq_issue 事件的所有参数
SELECT 
  fe.id,
  fe.ts / 1000000.0 as time_ms,
  fe.name,
  fe.cpu,
  args.key,
  args.display_value
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_issue'
ORDER BY fe.ts, args.key
LIMIT 50;
```

---

## 三、展开参数查询（PIVOT 模式）

由于参数存储在 args 表中是行格式，需要转换为列格式便于分析。

### 3.1 block_rq_issue 详情查询

```sql
-- 展开 block_rq_issue 事件的所有参数
SELECT 
  fe.id,
  fe.ts / 1000000.0 as time_ms,
  fe.cpu,
  MAX(CASE WHEN args.key = 'dev' THEN args.display_value END) as dev,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
  MAX(CASE WHEN args.key = 'comm' THEN args.display_value END) as comm,
  MAX(CASE WHEN args.key = 'ioprio' THEN args.int_value END) as ioprio
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_issue'
GROUP BY fe.id
ORDER BY fe.ts
LIMIT 100;
```

### 3.2 block_rq_complete 详情查询

```sql
-- 展开 block_rq_complete 事件
SELECT 
  fe.id,
  fe.ts / 1000000.0 as time_ms,
  fe.cpu,
  MAX(CASE WHEN args.key = 'dev' THEN args.display_value END) as dev,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
  MAX(CASE WHEN args.key = 'error' THEN args.int_value END) as error
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_complete'
GROUP BY fe.id
ORDER BY fe.ts
LIMIT 100;
```

### 3.3 block_bio_queue 详情查询

```sql
-- 展开 block_bio_queue 事件
SELECT 
  fe.id,
  fe.ts / 1000000.0 as time_ms,
  fe.cpu,
  MAX(CASE WHEN args.key = 'dev' THEN args.display_value END) as dev,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
  MAX(CASE WHEN args.key = 'comm' THEN args.display_value END) as comm
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_bio_queue'
GROUP BY fe.id
ORDER BY fe.ts
LIMIT 100;
```

---

## 四、IO 延迟分析

### 4.1 计算单次 IO 延迟（issue → complete）

```sql
-- 创建临时视图：issue 事件
WITH issue_events AS (
  SELECT 
    fe.id,
    fe.ts as issue_ts,
    fe.cpu,
    MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
    MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
),
-- 创建临时视图：complete 事件
complete_events AS (
  SELECT 
    fe.id,
    fe.ts as complete_ts,
    MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
    MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
    MAX(CASE WHEN args.key = 'error' THEN args.int_value END) as error
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_complete'
  GROUP BY fe.id
)
-- 关联计算延迟
SELECT 
  i.sector,
  i.nr_sector,
  i.rwbs,
  i.bytes,
  i.issue_ts / 1000000.0 as issue_ms,
  c.complete_ts / 1000000.0 as complete_ms,
  (c.complete_ts - i.issue_ts) / 1000.0 as latency_us,
  (c.complete_ts - i.issue_ts) / 1000000.0 as latency_ms,
  c.error
FROM issue_events i
JOIN complete_events c 
  ON i.sector = c.sector 
  AND i.nr_sector = c.nr_sector
  AND c.complete_ts > i.issue_ts
  AND c.complete_ts < i.issue_ts + 1000000000  -- 1秒内完成
ORDER BY latency_us DESC
LIMIT 100;
```

### 4.2 IO 延迟统计

```sql
-- 统计 IO 延迟分布
WITH io_latency AS (
  SELECT 
    (c.complete_ts - i.issue_ts) / 1000.0 as latency_us
  FROM (
    SELECT fe.ts as issue_ts,
           MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
           MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector
    FROM ftrace_event fe
    JOIN args ON fe.arg_set_id = args.arg_set_id
    WHERE fe.name = 'block_rq_issue'
    GROUP BY fe.id
  ) i
  JOIN (
    SELECT fe.ts as complete_ts,
           MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
           MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector
    FROM ftrace_event fe
    JOIN args ON fe.arg_set_id = args.arg_set_id
    WHERE fe.name = 'block_rq_complete'
    GROUP BY fe.id
  ) c ON i.sector = c.sector AND i.nr_sector = c.nr_sector
     AND c.complete_ts > i.issue_ts
     AND c.complete_ts < i.issue_ts + 1000000000
)
SELECT 
  COUNT(*) as total_ios,
  MIN(latency_us) as min_latency_us,
  MAX(latency_us) as max_latency_us,
  AVG(latency_us) as avg_latency_us,
  -- 百分位数近似
  COUNT(CASE WHEN latency_us < 100 THEN 1 END) as under_100us,
  COUNT(CASE WHEN latency_us < 1000 THEN 1 END) as under_1ms,
  COUNT(CASE WHEN latency_us < 10000 THEN 1 END) as under_10ms,
  COUNT(CASE WHEN latency_us >= 10000 THEN 1 END) as over_10ms
FROM io_latency;
```

### 4.3 按 IO 类型统计延迟

```sql
-- 分别统计读/写延迟
WITH io_latency AS (
  SELECT 
    i.rwbs,
    (c.complete_ts - i.issue_ts) / 1000.0 as latency_us
  FROM (
    SELECT fe.ts as issue_ts,
           MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
           MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
           MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
    FROM ftrace_event fe
    JOIN args ON fe.arg_set_id = args.arg_set_id
    WHERE fe.name = 'block_rq_issue'
    GROUP BY fe.id
  ) i
  JOIN (
    SELECT fe.ts as complete_ts,
           MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
           MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector
    FROM ftrace_event fe
    JOIN args ON fe.arg_set_id = args.arg_set_id
    WHERE fe.name = 'block_rq_complete'
    GROUP BY fe.id
  ) c ON i.sector = c.sector AND i.nr_sector = c.nr_sector
     AND c.complete_ts > i.issue_ts
     AND c.complete_ts < i.issue_ts + 1000000000
)
SELECT 
  rwbs,
  COUNT(*) as io_count,
  AVG(latency_us) as avg_latency_us,
  MIN(latency_us) as min_latency_us,
  MAX(latency_us) as max_latency_us
FROM io_latency
GROUP BY rwbs
ORDER BY io_count DESC;
```

---

## 五、IO 吞吐量分析

### 5.1 总体吞吐量

```sql
-- 计算总体 IO 吞吐量
WITH io_data AS (
  SELECT 
    fe.ts,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
),
time_range AS (
  SELECT 
    MIN(ts) as start_ts,
    MAX(ts) as end_ts,
    (MAX(ts) - MIN(ts)) / 1000000000.0 as duration_sec
  FROM io_data
)
SELECT 
  COUNT(*) as total_ios,
  SUM(bytes) as total_bytes,
  SUM(bytes) / 1024.0 / 1024.0 as total_mb,
  (SELECT duration_sec FROM time_range) as duration_sec,
  SUM(bytes) / 1024.0 / 1024.0 / (SELECT duration_sec FROM time_range) as throughput_mbps,
  COUNT(*) / (SELECT duration_sec FROM time_range) as iops
FROM io_data;
```

### 5.2 按秒统计 IOPS

```sql
-- 每秒 IOPS 统计
WITH io_data AS (
  SELECT 
    fe.ts / 1000000000 as second,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
)
SELECT 
  second - (SELECT MIN(second) FROM io_data) as relative_sec,
  COUNT(*) as iops,
  SUM(bytes) / 1024.0 / 1024.0 as mb_per_sec,
  COUNT(CASE WHEN rwbs LIKE '%R%' THEN 1 END) as read_iops,
  COUNT(CASE WHEN rwbs LIKE '%W%' THEN 1 END) as write_iops
FROM io_data
GROUP BY second
ORDER BY second;
```

### 5.3 按 IO 大小分布

```sql
-- IO 大小分布统计
WITH io_sizes AS (
  SELECT 
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
)
SELECT 
  CASE 
    WHEN bytes <= 4096 THEN '0-4KB'
    WHEN bytes <= 16384 THEN '4-16KB'
    WHEN bytes <= 65536 THEN '16-64KB'
    WHEN bytes <= 131072 THEN '64-128KB'
    WHEN bytes <= 524288 THEN '128-512KB'
    ELSE '>512KB'
  END as size_bucket,
  COUNT(*) as count,
  SUM(bytes) / 1024.0 / 1024.0 as total_mb
FROM io_sizes
GROUP BY size_bucket
ORDER BY MIN(bytes);
```

---

## 六、进程 IO 行为分析

### 6.1 按进程统计 IO

```sql
-- 统计各进程的 IO 量
WITH process_io AS (
  SELECT 
    MAX(CASE WHEN args.key = 'comm' THEN args.display_value END) as comm,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
)
SELECT 
  comm,
  COUNT(*) as io_count,
  SUM(bytes) / 1024.0 / 1024.0 as total_mb,
  COUNT(CASE WHEN rwbs LIKE '%R%' THEN 1 END) as read_count,
  COUNT(CASE WHEN rwbs LIKE '%W%' THEN 1 END) as write_count
FROM process_io
GROUP BY comm
ORDER BY io_count DESC
LIMIT 20;
```

### 6.2 关联 thread/process 表获取更多信息

```sql
-- 结合 thread/process 表查询
SELECT 
  p.name as process_name,
  p.pid,
  t.name as thread_name,
  t.tid,
  fe.name as event_name,
  COUNT(*) as event_count
FROM ftrace_event fe
LEFT JOIN thread t ON fe.utid = t.utid
LEFT JOIN process p ON t.upid = p.upid
WHERE fe.name LIKE 'block%'
GROUP BY p.pid, t.tid, fe.name
ORDER BY event_count DESC
LIMIT 50;
```

### 6.3 追踪特定进程的 IO

```sql
-- 追踪特定进程（如 dd）的 IO
WITH target_io AS (
  SELECT 
    fe.id,
    fe.ts / 1000000.0 as time_ms,
    fe.name as event,
    MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
    MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
    MAX(CASE WHEN args.key = 'comm' THEN args.display_value END) as comm
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name LIKE 'block%'
  GROUP BY fe.id
)
SELECT *
FROM target_io
WHERE comm = 'dd'  -- 修改为你要追踪的进程名
ORDER BY time_ms
LIMIT 200;
```

---

## 七、IO 合并分析

### 7.1 统计合并率

```sql
-- 计算 IO 合并率
SELECT 
  'block_bio_queue' as stage,
  COUNT(*) as count
FROM ftrace_event
WHERE name = 'block_bio_queue'

UNION ALL

SELECT 
  'block_bio_backmerge' as stage,
  COUNT(*) as count
FROM ftrace_event
WHERE name = 'block_bio_backmerge'

UNION ALL

SELECT 
  'block_bio_frontmerge' as stage,
  COUNT(*) as count
FROM ftrace_event
WHERE name = 'block_bio_frontmerge'

UNION ALL

SELECT 
  'block_rq_issue' as stage,
  COUNT(*) as count
FROM ftrace_event
WHERE name = 'block_rq_issue';
```

### 7.2 合并率计算

```sql
-- 计算合并比例
WITH merge_stats AS (
  SELECT 
    SUM(CASE WHEN name = 'block_bio_queue' THEN 1 ELSE 0 END) as bio_queued,
    SUM(CASE WHEN name = 'block_bio_backmerge' THEN 1 ELSE 0 END) as back_merged,
    SUM(CASE WHEN name = 'block_bio_frontmerge' THEN 1 ELSE 0 END) as front_merged,
    SUM(CASE WHEN name = 'block_rq_issue' THEN 1 ELSE 0 END) as rq_issued
  FROM ftrace_event
  WHERE name IN ('block_bio_queue', 'block_bio_backmerge', 
                 'block_bio_frontmerge', 'block_rq_issue')
)
SELECT 
  bio_queued,
  back_merged,
  front_merged,
  back_merged + front_merged as total_merged,
  rq_issued,
  ROUND((back_merged + front_merged) * 100.0 / bio_queued, 2) as merge_rate_percent,
  ROUND(bio_queued * 1.0 / rq_issued, 2) as bio_per_request
FROM merge_stats;
```

---

## 八、错误和异常分析

### 8.1 查看 IO 错误

```sql
-- 查找所有 IO 错误
SELECT 
  fe.ts / 1000000.0 as time_ms,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
  MAX(CASE WHEN args.key = 'error' THEN args.int_value END) as error
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_complete'
GROUP BY fe.id
HAVING MAX(CASE WHEN args.key = 'error' THEN args.int_value END) != 0
ORDER BY fe.ts;
```

### 8.2 查看 requeue 事件

```sql
-- 查看请求重入队情况
SELECT 
  fe.ts / 1000000.0 as time_ms,
  fe.cpu,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_requeue'
GROUP BY fe.id
ORDER BY fe.ts;
```

---

## 九、高级查询技巧

### 9.1 创建视图简化查询

```sql
-- 创建 issue 事件视图（在 trace_processor shell 中）
CREATE VIEW block_issue AS
SELECT 
  fe.id,
  fe.ts,
  fe.cpu,
  MAX(CASE WHEN args.key = 'dev' THEN args.display_value END) as dev,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'nr_sector' THEN args.int_value END) as nr_sector,
  MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs,
  MAX(CASE WHEN args.key = 'comm' THEN args.display_value END) as comm
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'block_rq_issue'
GROUP BY fe.id;

-- 然后可以简单查询
SELECT * FROM block_issue ORDER BY ts LIMIT 100;
```

### 9.2 时间窗口分析

```sql
-- 分析特定时间窗口内的 IO
WITH time_bounds AS (
  SELECT 
    MIN(ts) as trace_start,
    MAX(ts) as trace_end
  FROM ftrace_event
  WHERE name LIKE 'block%'
),
io_events AS (
  SELECT 
    fe.ts,
    MAX(CASE WHEN args.key = 'bytes' THEN args.int_value END) as bytes,
    MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
  FROM ftrace_event fe
  JOIN args ON fe.arg_set_id = args.arg_set_id
  WHERE fe.name = 'block_rq_issue'
  GROUP BY fe.id
)
-- 分析前 5 秒
SELECT 
  COUNT(*) as io_count,
  SUM(bytes) / 1024.0 / 1024.0 as total_mb
FROM io_events, time_bounds
WHERE ts < trace_start + 5000000000;  -- 5秒 = 5 * 10^9 纳秒
```

### 9.3 导出为 CSV

```bash
# 使用 trace_processor 导出
./trace_processor trace.perfetto-trace --query "
SELECT 
  fe.ts / 1000000.0 as time_ms,
  fe.name,
  MAX(CASE WHEN args.key = 'sector' THEN args.int_value END) as sector,
  MAX(CASE WHEN args.key = 'rwbs' THEN args.display_value END) as rwbs
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name LIKE 'block%'
GROUP BY fe.id
ORDER BY fe.ts
" > block_events.csv
```

---

## 十、常用查询速查表

### 10.1 基础查询

| 目的 | SQL 片段 |
|------|---------|
| 查看事件类型 | `SELECT DISTINCT name FROM ftrace_event WHERE name LIKE 'block%'` |
| 查看事件数量 | `SELECT name, COUNT(*) FROM ftrace_event WHERE name LIKE 'block%' GROUP BY name` |
| 查看参数字段 | `SELECT DISTINCT key FROM args WHERE arg_set_id IN (SELECT arg_set_id FROM ftrace_event WHERE name LIKE 'block%')` |

### 10.2 性能分析

| 目的 | 关键事件 |
|------|---------|
| IO 延迟 | `block_rq_issue` + `block_rq_complete` |
| 吞吐量 | `block_rq_issue` (bytes 字段) |
| IOPS | `block_rq_issue` COUNT |
| 合并率 | `block_bio_queue` vs `block_bio_backmerge` |
| 错误 | `block_rq_complete` (error 字段) |

### 10.3 时间转换

```sql
-- 纳秒转毫秒
ts / 1000000.0 as time_ms

-- 纳秒转秒
ts / 1000000000.0 as time_sec

-- 相对时间（从 trace 开始）
(ts - (SELECT MIN(ts) FROM ftrace_event)) / 1000000.0 as relative_ms
```

---

## 十一、总结

### 核心查询模式

```sql
-- 标准 block 事件查询模板
SELECT 
  fe.id,
  fe.ts / 1000000.0 as time_ms,
  fe.cpu,
  MAX(CASE WHEN args.key = 'KEY_NAME' THEN args.display_value END) as ALIAS
FROM ftrace_event fe
JOIN args ON fe.arg_set_id = args.arg_set_id
WHERE fe.name = 'EVENT_NAME'
GROUP BY fe.id
ORDER BY fe.ts;
```

### 分析流程

```
1. 先查看有哪些事件：SELECT DISTINCT name FROM ftrace_event
2. 查看事件参数：查询 args 表
3. 展开参数为列格式：使用 CASE WHEN + GROUP BY
4. 关联分析：JOIN issue 和 complete 计算延迟
5. 聚合统计：GROUP BY + 聚合函数
```

### 性能建议

- 使用 `LIMIT` 限制返回行数
- 先过滤再 JOIN（WHERE 条件尽早使用）
- 复杂查询可以创建临时视图
- 大数据量时考虑导出后离线分析
