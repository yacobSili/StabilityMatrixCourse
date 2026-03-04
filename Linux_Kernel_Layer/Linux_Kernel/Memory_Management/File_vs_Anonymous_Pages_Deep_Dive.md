# 文件页与匿名页深度解析

## 📋 文档概述

本文档系统深入地解析 Linux 内核中的文件页（File-backed Pages）和匿名页（Anonymous Pages），包括两者的概念、区别、内存映射机制、回收策略、swap 机制以及与内存管理其他组件的关系。理解文件页和匿名页是深入理解 Linux 内存管理的关键。

## 🎯 学习目标

- 深入理解文件页和匿名页的概念和区别
- 掌握两者的内存映射机制
- 理解文件页和匿名页的回收策略差异
- 掌握 swap 机制的工作原理
- 理解 LRU 列表如何管理这两种页面
- 能够分析文件页和匿名页相关的内存问题
- 掌握相关的监控和诊断方法

---

## 第一章：基础概念

### 1.1 什么是文件页（File-backed Pages）？

**文件页（File-backed Pages）** 是指内容可以从文件系统恢复的页面。这些页面是文件在内存中的缓存，当需要时可以重新从磁盘读取。

**特点**：

1. **有后备存储**
   - 内容存储在磁盘文件中
   - 可以从文件系统重新加载

2. **可丢弃**
   - 可以直接释放，无需保存
   - 需要时从磁盘重新读取

3. **共享性**
   - 多个进程可以共享同一个文件页
   - 节省内存

4. **写回机制**
   - 修改后需要写回磁盘
   - 通过 page cache 机制管理

**典型场景**：

- 可执行文件代码段
- 共享库（.so 文件）
- 数据文件（通过 mmap 映射）
- 文件系统缓存

### 1.2 什么是匿名页（Anonymous Pages）？

**匿名页（Anonymous Pages）** 是指没有文件系统后备存储的页面。这些页面的内容只存在于内存中，没有对应的磁盘文件。

**特点**：

1. **无后备存储**
   - 内容只存在于内存
   - 没有对应的磁盘文件

2. **不可直接丢弃**
   - 不能直接释放
   - 必须先交换到 swap 分区

3. **进程私有**
   - 通常属于特定进程
   - 进程退出后可以释放

4. **需要 swap**
   - 内存不足时必须交换到 swap
   - 有 I/O 开销

**典型场景**：

- 进程堆（heap）
- 进程栈（stack）
- 匿名 mmap 映射
- 动态分配的内存

### 1.3 两者的核心区别

**对比表**：

| 特性 | 文件页（File-backed） | 匿名页（Anonymous） |
|------|---------------------|-------------------|
| 后备存储 | 有（磁盘文件） | 无 |
| 可丢弃性 | 可以直接丢弃 | 必须先交换 |
| 共享性 | 可以共享 | 通常私有 |
| 回收方式 | 直接释放 | 交换到 swap |
| 恢复方式 | 从文件读取 | 从 swap 读取 |
| 典型用途 | 文件缓存、代码段 | 堆、栈、匿名映射 |
| I/O 开销 | 读取时 | 交换时 |

**内存回收优先级**：

```
1. 文件页（inactive_file） ← 最容易回收
2. 匿名页（inactive_anon） ← 需要 swap
3. 活跃页面（active） ← 最后选择
```

---

## 第二章：内存映射机制

### 2.1 文件页的内存映射

**mmap 系统调用**：

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

**文件映射类型**：

1. **私有映射（MAP_PRIVATE）**
   ```c
   mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE, fd, 0);
   ```
   - 修改不会写回文件
   - 使用 Copy-on-Write（COW）机制
   - 适合加载可执行文件

2. **共享映射（MAP_SHARED）**
   ```c
   mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
   ```
   - 修改会写回文件
   - 多个进程可以共享
   - 适合进程间通信

**文件页的生命周期**：

```
打开文件
    ↓
mmap 映射
    ↓
页面错误（page fault）
    ↓
从磁盘读取到内存（文件页）
    ↓
访问页面
    ↓
可能被回收（直接释放）
    ↓
再次访问时从磁盘重新读取
```

### 2.2 匿名页的内存映射

**匿名映射**：

```c
// 创建匿名映射
void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE, 
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```

**匿名页的来源**：

1. **进程堆（Heap）**
   ```c
   // malloc 内部可能使用 mmap
   void *ptr = malloc(size);
   ```

2. **进程栈（Stack）**
   - 自动分配
   - 栈增长时创建匿名页

3. **匿名 mmap**
   ```c
   mmap(NULL, size, PROT_READ|PROT_WRITE, 
        MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
   ```

**匿名页的生命周期**：

```
分配内存（malloc/mmap）
    ↓
页面错误（page fault）
    ↓
分配匿名页（初始化为 0）
    ↓
使用页面
    ↓
内存压力时交换到 swap
    ↓
再次访问时从 swap 读回
```

### 2.3 Page Cache 机制

**什么是 Page Cache？**

Page Cache 是内核用于缓存文件页的机制，提高文件访问性能。

**Page Cache 的作用**：

1. **减少磁盘 I/O**
   - 最近访问的文件内容缓存在内存
   - 再次访问时直接从内存读取

2. **预读（Read-ahead）**
   - 预测性读取后续内容
   - 提高顺序访问性能

3. **写回（Write-back）**
   - 延迟写回磁盘
   - 批量写入提高效率

**Page Cache 的管理**：

```c
// 文件页存储在 address_space 中
struct address_space {
    struct inode *host;  // 对应的 inode
    struct radix_tree_root page_tree;  // 页面树
    // ...
};
```

**查看 Page Cache**：

```bash
# 查看文件缓存大小
cat /proc/meminfo | grep -E "Cached|Buffers"

# 查看详细的缓存统计
cat /proc/vmstat | grep -E "nr_file_pages|nr_dirty"
```

---

## 第三章：LRU 列表管理

### 3.1 LRU 列表的分类

Linux 内核使用 LRU（Least Recently Used）算法管理页面，将页面分为多个列表：

**四种 LRU 列表**：

1. **active_anon**：活跃的匿名页面
2. **inactive_anon**：非活跃的匿名页面
3. **active_file**：活跃的文件页面
4. **inactive_file**：非活跃的文件页面

**列表关系**：

```
所有页面
    ├── 匿名页（Anonymous）
    │   ├── active_anon（活跃）
    │   └── inactive_anon（非活跃）← 优先回收
    └── 文件页（File-backed）
        ├── active_file（活跃）
        └── inactive_file（非活跃）← 最优先回收
```

### 3.2 页面的老化过程

**老化（Aging）机制**：

页面在 active 和 inactive 列表之间移动：

```
active_anon
    ↓ (老化，长时间未访问)
inactive_anon
    ↓ (回收)
swap 分区

active_file
    ↓ (老化)
inactive_file
    ↓ (回收)
直接释放
```

**老化算法**：

```c
// 简化逻辑
if (page_referenced(page)) {
    // 页面被访问，保持或提升到 active
    activate_page(page);
} else {
    // 页面未被访问，老化到 inactive
    deactivate_page(page);
}
```

**查看 LRU 列表**：

```bash
# 查看各 LRU 列表的页面数
cat /proc/meminfo | grep -E "Active|Inactive"

# 输出示例：
# Active(anon):     123456 kB
# Inactive(anon):    45678 kB
# Active(file):     234567 kB
# Inactive(file):    56789 kB
```

### 3.3 回收优先级

**回收顺序**：

kswapd 和 direct reclaim 按照以下顺序回收：

```
1. inactive_file  ← 最优先，直接释放
2. inactive_anon  ← 交换到 swap
3. active_file    ← 先老化到 inactive
4. active_anon    ← 先老化到 inactive
```

**原因**：

1. **文件页可以直接释放**
   - 有后备存储
   - 无数据丢失风险
   - 无 I/O 开销

2. **匿名页需要交换**
   - 无后备存储
   - 必须先保存到 swap
   - 有 I/O 开销

3. **活跃页面最后回收**
   - 正在使用中
   - 回收后可能立即需要
   - 效率低

---

## 第四章：Swap 机制

### 4.1 Swap 的作用

**为什么需要 Swap？**

1. **扩展虚拟内存**
   - 物理内存不足时，使用磁盘作为扩展
   - 允许系统运行更多进程

2. **保存匿名页**
   - 匿名页没有文件后备
   - 必须交换到 swap 才能释放内存

3. **系统休眠**
   - 休眠时需要保存内存内容
   - 写入 swap 分区

### 4.2 Swap 分区和文件

**Swap 分区**：

```bash
# 创建 swap 分区（假设 /dev/sdb2）
mkswap /dev/sdb2

# 启用 swap
swapon /dev/sdb2

# 查看 swap
swapon -s
```

**Swap 文件**：

```bash
# 创建 swap 文件（1GB）
dd if=/dev/zero of=/swapfile bs=1M count=1024
chmod 600 /swapfile
mkswap /swapfile

# 启用 swap 文件
swapon /swapfile
```

**查看 Swap 使用**：

```bash
# 查看 swap 信息
free -h
swapon -s

# 查看详细的 swap 统计
cat /proc/swaps
cat /proc/meminfo | grep -i swap
```

### 4.3 Swap 的工作原理

**交换出（Swap Out）**：

```
内存压力
    ↓
选择 inactive_anon 页面
    ↓
分配 swap 槽（swap slot）
    ↓
写入 swap 分区/文件
    ↓
更新页表（标记为不在内存）
    ↓
释放物理页面
```

**交换入（Swap In）**：

```
访问已交换的页面
    ↓
页面错误（page fault）
    ↓
检测到页面在 swap
    ↓
从 swap 读取内容
    ↓
分配新的物理页面
    ↓
更新页表
```

**Swap 槽管理**：

```c
// 每个交换的页面有一个 swap 槽号
// 存储在页表项中
struct page {
    // ...
    swp_entry_t swap;  // swap 槽信息
    // ...
};
```

### 4.4 Swap 的性能影响

**Swap 的性能开销**：

1. **I/O 延迟**
   - 磁盘 I/O 比内存访问慢几个数量级
   - 机械硬盘：毫秒级
   - 内存：纳秒级

2. **系统响应**
   - 频繁 swap 会导致系统卡顿
   - 进程可能因为等待 swap 而阻塞

3. **磁盘磨损**
   - SSD 有写入寿命
   - 频繁 swap 会加速磨损

**优化建议**：

1. **使用 SSD 作为 swap**
   - 比机械硬盘快得多
   - 但仍比内存慢

2. **增加物理内存**
   - 减少 swap 使用
   - 最佳方案

3. **调整 swappiness**
   ```bash
   # 降低 swap 使用倾向
   echo 10 > /proc/sys/vm/swappiness
   ```

---

## 第五章：回收策略详解

### 5.1 文件页的回收

**回收特点**：

- ✅ 可以直接释放
- ✅ 无数据丢失
- ✅ 无 I/O 开销（释放时）
- ✅ 可以重新从文件读取

**回收流程**：

```
选择 inactive_file 页面
    ↓
检查是否为脏页（dirty）
    ↓
如果是脏页：写回磁盘
    ↓
清除页表项
    ↓
释放物理页面
    ↓
从 LRU 列表移除
```

**脏页处理**：

```c
// 检查页面是否为脏页
if (PageDirty(page)) {
    // 写回磁盘
    writepage(page, ...);
    // 等待写回完成
    wait_on_page_writeback(page);
}

// 然后可以安全释放
free_page(page);
```

**查看文件页回收统计**：

```bash
# 查看文件页回收统计
cat /proc/vmstat | grep -E "pgsteal_file|pgscan_file"

# pgsteal_file: 回收的文件页数量
# pgscan_file: 扫描的文件页数量
```

### 5.2 匿名页的回收

**回收特点**：

- ⚠️ 必须先交换到 swap
- ⚠️ 有 I/O 开销
- ⚠️ 需要 swap 空间
- ⚠️ 恢复时需要从 swap 读取

**回收流程**：

```
选择 inactive_anon 页面
    ↓
分配 swap 槽
    ↓
写入 swap 分区/文件
    ↓
更新页表（标记为在 swap）
    ↓
释放物理页面
    ↓
从 LRU 列表移除
```

**Swap 分配**：

```c
// 分配 swap 槽
swp_entry_t entry = get_swap_page();
if (!entry.val) {
    // swap 空间不足，无法回收
    return -ENOMEM;
}

// 写入 swap
swap_writepage(page, entry, ...);
```

**查看匿名页回收统计**：

```bash
# 查看匿名页回收统计
cat /proc/vmstat | grep -E "pgsteal_anon|pgscan_anon"

# 查看 swap 活动
cat /proc/vmstat | grep -E "pswpin|pswpout"
# pswpin: swap in（从 swap 读入）
# pswpout: swap out（写入 swap）
```

### 5.3 swappiness 参数

**swappiness 的作用**：

控制匿名页和文件页的回收比例。

**参数范围**：0-100

- **0**：只回收文件页，不交换匿名页
- **100**：匿名页和文件页同等对待
- **默认**：60

**计算公式**（简化）：

```c
// 伪代码
anon_cost = 200 + (swappiness * (reclaim_cost + swap_cost)) / 100;
file_cost = 200 - (swappiness * reclaim_cost) / 100;

// 如果 anon_cost < file_cost，优先回收匿名页
// 否则优先回收文件页
```

**查看和设置**：

```bash
# 查看当前值
cat /proc/sys/vm/swappiness

# 设置值
echo 10 > /proc/sys/vm/swappiness

# 永久设置
echo "vm.swappiness = 10" >> /etc/sysctl.conf
sysctl -p
```

**调优建议**：

- **服务器**：10-30（减少 swap 使用）
- **桌面系统**：60（平衡）
- **数据库**：1-10（避免 swap）
- **内存充足**：可以设置为 0（完全禁用 swap）

---

## 第六章：监控与诊断

### 6.1 查看页面类型分布

**使用 /proc/meminfo**：

```bash
cat /proc/meminfo | grep -E "Active|Inactive|Anon|File"

# 输出示例：
# Active(anon):     123456 kB    ← 活跃匿名页
# Inactive(anon):    45678 kB     ← 非活跃匿名页
# Active(file):     234567 kB     ← 活跃文件页
# Inactive(file):    56789 kB     ← 非活跃文件页
```

**计算总匿名页和文件页**：

```bash
# 匿名页总数
anon_total = Active(anon) + Inactive(anon)

# 文件页总数
file_total = Active(file) + Inactive(file)

# 总内存使用
total_used = anon_total + file_total
```

**监控脚本**：

```bash
#!/bin/bash
# monitor_page_types.sh

while true; do
    clear
    echo "=== Page Type Distribution ==="
    date
    echo ""
    
    # 读取 meminfo
    active_anon=$(grep "Active(anon)" /proc/meminfo | awk '{print $2}')
    inactive_anon=$(grep "Inactive(anon)" /proc/meminfo | awk '{print $2}')
    active_file=$(grep "Active(file)" /proc/meminfo | awk '{print $2}')
    inactive_file=$(grep "Inactive(file)" /proc/meminfo | awk '{print $2}')
    
    anon_total=$((active_anon + inactive_anon))
    file_total=$((active_file + inactive_file))
    total=$((anon_total + file_total))
    
    echo "Anonymous Pages:"
    echo "  Active:   $(printf "%8d" $active_anon) KB"
    echo "  Inactive: $(printf "%8d" $inactive_anon) KB"
    echo "  Total:    $(printf "%8d" $anon_total) KB ($(($anon_total * 100 / $total))%)"
    echo ""
    echo "File Pages:"
    echo "  Active:   $(printf "%8d" $active_file) KB"
    echo "  Inactive: $(printf "%8d" $inactive_file) KB"
    echo "  Total:    $(printf "%8d" $file_total) KB ($(($file_total * 100 / $total))%)"
    
    sleep 1
done
```

### 6.2 监控回收活动

**使用 /proc/vmstat**：

```bash
# 查看文件页回收
cat /proc/vmstat | grep -E "pgscan_file|pgsteal_file|pgmajfault"

# 查看匿名页回收
cat /proc/vmstat | grep -E "pgscan_anon|pgsteal_anon"

# 查看 swap 活动
cat /proc/vmstat | grep -E "pswpin|pswpout"
```

**关键指标**：

- **pgscan_file**：扫描的文件页数量
- **pgsteal_file**：回收的文件页数量
- **pgscan_anon**：扫描的匿名页数量
- **pgsteal_anon**：回收的匿名页数量
- **pswpin**：从 swap 读入的页面数
- **pswpout**：写入 swap 的页面数

**回收效率**：

```bash
# 计算回收效率
file_efficiency = pgsteal_file / pgscan_file
anon_efficiency = pgsteal_anon / pgscan_anon

# 效率越高，说明可回收页面越多
```

### 6.3 使用工具分析

**smem**（按页面类型统计进程内存）：

```bash
# 安装
# Ubuntu: sudo apt-get install smem

# 查看进程内存（按页面类型）
smem -t -c "pid user command swap uss pss rss"

# 输出显示：
# swap: 交换到 swap 的内存（匿名页）
# uss: 进程独占内存
# pss: 按比例共享内存
# rss: 物理内存使用
```

**查看进程的内存映射**：

```bash
# 查看进程的内存映射
cat /proc/<pid>/maps

# 输出示例：
# 00400000-00401000 r-xp 00000000 08:01 123456 /bin/ls  ← 文件页（代码段）
# 00600000-00601000 r--p 00000000 08:01 123456 /bin/ls  ← 文件页（数据段）
# 00601000-00602000 rw-p 00001000 08:01 123456 /bin/ls  ← 文件页（可写，私有）
# 7f8a12345000-7f8a12346000 rw-p 00000000 00:00 0       ← 匿名页（堆/栈）
```

**解析 maps 文件**：

```bash
# 脚本：分析进程的页面类型
#!/bin/bash
pid=$1

echo "=== Process $pid Memory Map Analysis ==="

# 统计文件页和匿名页
file_pages=0
anon_pages=0

while read line; do
    # 检查是否有文件路径（文件页）
    if echo "$line" | grep -q " /"; then
        file_pages=$((file_pages + 1))
    else
        # 没有文件路径的是匿名页
        if echo "$line" | grep -q "rw"; then
            anon_pages=$((anon_pages + 1))
        fi
    fi
done < /proc/$pid/maps

echo "File-backed pages: $file_pages"
echo "Anonymous pages: $anon_pages"
```

---

## 第七章：源码分析

### 7.1 页面类型判断

**源码位置**：`include/linux/mm.h`

**判断函数**：

```c
// 判断是否为匿名页
static inline int PageAnon(struct page *page)
{
    return ((unsigned long)page->mapping & PAGE_MAPPING_ANON) != 0;
}

// 判断是否为文件页
static inline int PageFileBacked(struct page *page)
{
    return page->mapping != NULL && 
           !PageAnon(page);
}
```

**mapping 字段**：

```c
struct page {
    // ...
    struct address_space *mapping;  // 文件页：指向 address_space
                                     // 匿名页：低位置位表示匿名
    // ...
};
```

### 7.2 LRU 列表操作

**添加到 LRU**：

```c
// 添加页面到 LRU 列表
void lru_cache_add(struct page *page)
{
    if (PageAnon(page))
        add_page_to_anon_lru(page);  // 添加到匿名页 LRU
    else
        add_page_to_file_lru(page);   // 添加到文件页 LRU
}
```

**从 LRU 移除**：

```c
// 从 LRU 列表移除
void del_page_from_lru_list(struct page *page, enum lru_list lru)
{
    list_del(&page->lru);
    __dec_zone_page_state(page, NR_LRU_BASE + lru);
}
```

### 7.3 回收实现

**shrink_page_list 函数**：

```c
// 回收页面列表
static unsigned long shrink_page_list(struct list_head *page_list,
                                      struct pglist_data *pgdat,
                                      struct scan_control *sc,
                                      enum lru_list lru)
{
    LIST_HEAD(ret_pages);
    unsigned long nr_reclaimed = 0;
    
    while (!list_empty(page_list)) {
        struct page *page = lru_to_page(page_list);
        
        // 检查页面类型
        if (PageAnon(page)) {
            // 匿名页：需要交换
            if (add_to_swap(page)) {
                // 成功交换，可以释放
                list_add(&page->lru, &ret_pages);
                nr_reclaimed++;
            }
        } else {
            // 文件页：可以直接释放
            if (page_mapping(page)) {
                // 如果是脏页，先写回
                if (PageDirty(page)) {
                    writeback_page(page);
                }
            }
            // 可以释放
            list_add(&page->lru, &ret_pages);
            nr_reclaimed++;
        }
    }
    
    return nr_reclaimed;
}
```

### 7.4 Swap 实现

**交换出（swap_out）**：

```c
// 将页面交换到 swap
int add_to_swap(struct page *page)
{
    swp_entry_t entry;
    
    // 分配 swap 槽
    entry = get_swap_page();
    if (!entry.val)
        return 0;  // swap 空间不足
    
    // 写入 swap
    if (swap_writepage(page, entry, GFP_KERNEL)) {
        // 写入失败
        swap_free(entry);
        return 0;
    }
    
    // 更新页表
    SetPageSwapCache(page);
    set_page_private(page, entry.val);
    
    return 1;
}
```

**交换入（swap_in）**：

```c
// 从 swap 读回页面
struct page *swapin_readahead(swp_entry_t entry, gfp_t gfp_mask,
                              struct vm_area_struct *vma,
                              unsigned long addr)
{
    struct page *page;
    
    // 从 swap 读取
    page = read_swap_cache_async(entry, gfp_mask, vma, addr);
    
    return page;
}
```

---

## 第八章：实际应用场景

### 场景 1：文件缓存 vs 匿名内存

**问题**：

系统内存紧张，需要判断是文件缓存占用多还是匿名内存占用多。

**分析**：

```bash
# 1. 查看页面类型分布
cat /proc/meminfo | grep -E "Active|Inactive"

# 2. 计算比例
anon_total = Active(anon) + Inactive(anon)
file_total = Active(file) + Inactive(file)

# 3. 判断
if (anon_total > file_total) {
    // 匿名内存占用多，可能需要增加 swap
    // 或优化应用程序内存使用
} else {
    // 文件缓存占用多，可以调整 vfs_cache_pressure
    // 或减少文件缓存
}
```

**优化策略**：

- **匿名内存多**：增加 swap、优化应用、增加物理内存
- **文件缓存多**：调整 `vfs_cache_pressure`、减少缓存

### 场景 2：Swap 使用过高

**问题**：

系统 swap 使用率高，性能下降。

**诊断**：

```bash
# 1. 查看 swap 使用
free -h
swapon -s

# 2. 查看 swap 活动
cat /proc/vmstat | grep -E "pswpin|pswpout"
vmstat 1

# 3. 查看匿名页分布
cat /proc/meminfo | grep -E "Anon"
```

**可能原因**：

1. **物理内存不足**
   - 匿名页无法全部保留在内存
   - 必须交换到 swap

2. **swappiness 设置过高**
   - 过早开始交换匿名页
   - 应该降低 swappiness

3. **应用程序内存使用模式**
   - 大量匿名内存分配
   - 内存泄漏

**解决方案**：

```bash
# 1. 降低 swappiness
echo 10 > /proc/sys/vm/swappiness

# 2. 增加物理内存（如果可能）

# 3. 排查内存泄漏
# 使用 valgrind 或 AddressSanitizer
```

### 场景 3：文件缓存占用过多

**问题**：

文件缓存占用大量内存，影响应用程序。

**诊断**：

```bash
# 查看文件缓存
cat /proc/meminfo | grep -E "Cached|Buffers|File"

# 查看回收统计
cat /proc/vmstat | grep -E "pgsteal_file"
```

**调整策略**：

```bash
# 增加文件缓存回收倾向
echo 200 > /proc/sys/vm/vfs_cache_pressure
# 值越大，越倾向于回收文件缓存
# 默认 100，200 表示两倍倾向
```

**手动清理缓存**（谨慎使用）：

```bash
# 清理页面缓存（不影响数据）
echo 1 > /proc/sys/vm/drop_caches

# 清理目录项和 inode 缓存
echo 2 > /proc/sys/vm/drop_caches

# 清理所有缓存
echo 3 > /proc/sys/vm/drop_caches
```

**⚠️ 警告**：`drop_caches` 会立即释放缓存，可能导致性能下降。仅用于测试或紧急情况。

---

## 第九章：性能优化

### 9.1 减少匿名页交换

**策略 1：增加物理内存**

- 最直接有效的方法
- 减少对 swap 的依赖

**策略 2：降低 swappiness**

```bash
# 服务器环境
echo 10 > /proc/sys/vm/swappiness

# 数据库服务器
echo 1 > /proc/sys/vm/swappiness
```

**策略 3：优化应用程序**

- 减少内存分配
- 及时释放内存
- 使用内存池

### 9.2 优化文件缓存

**策略 1：调整 vfs_cache_pressure**

```bash
# 默认值
cat /proc/sys/vm/vfs_cache_pressure  # 100

# 增加回收倾向（如果文件缓存占用过多）
echo 200 > /proc/sys/vm/vfs_cache_pressure

# 减少回收倾向（如果需要更多文件缓存）
echo 50 > /proc/sys/vm/vfs_cache_pressure
```

**策略 2：调整 dirty 页面参数**

```bash
# 脏页比例阈值
cat /proc/sys/vm/dirty_ratio          # 默认 20%
cat /proc/sys/vm/dirty_background_ratio  # 默认 10%

# 如果文件写入频繁，可以调整
echo 15 > /proc/sys/vm/dirty_ratio
echo 5 > /proc/sys/vm/dirty_background_ratio
```

### 9.3 内存分配优化

**使用大页（Huge Pages）**：

```bash
# 查看大页配置
cat /proc/meminfo | grep -i huge

# 配置大页（需要 root）
echo 1024 > /proc/sys/vm/nr_hugepages
```

**使用透明大页（Transparent Huge Pages）**：

```bash
# 查看 THP 状态
cat /sys/kernel/mm/transparent_hugepage/enabled

# 禁用 THP（某些场景）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 第十章：问题诊断案例

### 案例 1：匿名页无法交换

**症状**：

- 内存紧张
- swap 空间充足
- 但匿名页无法交换
- 最终触发 OOM

**可能原因**：

1. **页面被锁定（mlock）**
   ```bash
   # 查看锁定的内存
   cat /proc/meminfo | grep -i locked
   
   # 查看进程的锁定内存
   cat /proc/<pid>/status | grep VmLck
   ```

2. **swap 设备故障**
   ```bash
   # 检查 swap 设备
   swapon -s
   dmesg | grep -i swap
   ```

3. **swap 空间碎片化**
   - swap 空间虽然充足，但无法分配连续空间

**解决方案**：

```bash
# 1. 检查 mlock
# 如果进程使用了 mlock，需要解锁或增加 swap

# 2. 检查 swap 设备
# 确保 swap 设备正常

# 3. 重新创建 swap（如果必要）
swapoff -a
mkswap /dev/sdb2
swapon /dev/sdb2
```

### 案例 2：文件页回收效率低

**症状**：

- 文件页很多，但回收效率低
- kswapd 持续工作但效果不明显

**诊断**：

```bash
# 1. 查看文件页回收统计
cat /proc/vmstat | grep -E "pgscan_file|pgsteal_file"

# 2. 计算回收效率
efficiency = pgsteal_file / pgscan_file
# 如果效率低（< 0.1），说明大部分文件页都在使用中

# 3. 查看文件页分布
cat /proc/meminfo | grep -E "Active\(file\)|Inactive\(file\)"
# 如果 Active(file) 远大于 Inactive(file)，说明文件页都在活跃使用
```

**可能原因**：

1. **文件页都在活跃使用**
   - 无法回收活跃页面
   - 需要先老化到 inactive

2. **脏页太多**
   - 需要先写回才能回收
   - 写回速度慢

**解决方案**：

```bash
# 1. 增加文件缓存回收倾向
echo 200 > /proc/sys/vm/vfs_cache_pressure

# 2. 加速脏页写回
echo 5 > /proc/sys/vm/dirty_background_ratio
echo 10 > /proc/sys/vm/dirty_ratio
```

### 案例 3：混合工作负载的内存管理

**场景**：

系统同时运行数据库（大量匿名页）和 Web 服务器（大量文件缓存）。

**挑战**：

- 需要平衡匿名页和文件页
- 避免频繁 swap
- 保持文件缓存性能

**优化策略**：

```bash
# 1. 适中的 swappiness
echo 30 > /proc/sys/vm/swappiness

# 2. 适中的 vfs_cache_pressure
echo 100 > /proc/sys/vm/vfs_cache_pressure

# 3. 监控两种页面的比例
# 根据实际情况调整参数
```

---

## 附录

### A. 相关内核参数

| 参数 | 路径 | 说明 |
|------|------|------|
| swappiness | /proc/sys/vm/swappiness | 控制匿名页和文件页的回收比例 |
| vfs_cache_pressure | /proc/sys/vm/vfs_cache_pressure | 控制文件缓存的回收倾向 |
| dirty_ratio | /proc/sys/vm/dirty_ratio | 脏页比例阈值 |
| dirty_background_ratio | /proc/sys/vm/dirty_background_ratio | 后台写回阈值 |

### B. 关键统计项

**/proc/vmstat 中的相关统计**：

- `nr_file_pages`：文件页总数
- `nr_anon_pages`：匿名页总数
- `nr_dirty`：脏页数量
- `pgscan_file`：扫描的文件页
- `pgsteal_file`：回收的文件页
- `pgscan_anon`：扫描的匿名页
- `pgsteal_anon`：回收的匿名页
- `pswpin`：swap in 次数
- `pswpout`：swap out 次数

### C. 源码文件位置

- `include/linux/mm.h`：页面类型判断
- `include/linux/swap.h`：swap 相关定义
- `mm/swap_state.c`：swap 状态管理
- `mm/swapfile.c`：swap 文件/分区管理
- `mm/vmscan.c`：页面回收（区分文件页和匿名页）
- `mm/filemap.c`：文件页管理（Page Cache）

### D. 诊断脚本集合

**页面类型监控脚本**：

```bash
#!/bin/bash
# page_type_monitor.sh

watch -n 1 '
echo "=== Page Type Distribution ==="
echo ""
cat /proc/meminfo | grep -E "Active|Inactive" | \
awk "{
    if (\$1 ~ /Active.*anon/) anon_active=\$2;
    if (\$1 ~ /Inactive.*anon/) anon_inactive=\$2;
    if (\$1 ~ /Active.*file/) file_active=\$2;
    if (\$1 ~ /Inactive.*file/) file_inactive=\$2;
}
END {
    anon_total = anon_active + anon_inactive;
    file_total = file_active + file_inactive;
    total = anon_total + file_total;
    printf \"Anonymous: %d KB (%.1f%%)\n\", anon_total, (anon_total*100/total);
    printf \"File:      %d KB (%.1f%%)\n\", file_total, (file_total*100/total);
}"
'
```

**Swap 活动监控**：

```bash
#!/bin/bash
# swap_activity_monitor.sh

prev_pswpin=0
prev_pswpout=0

while true; do
    pswpin=$(grep pswpin /proc/vmstat | awk "{print \$2}")
    pswpout=$(grep pswpout /proc/vmstat | awk "{print \$2}")
    
    pswpin_rate=$((pswpin - prev_pswpin))
    pswpout_rate=$((pswpout - prev_pswpout))
    
    echo "Swap In:  $pswpin_rate pages/s"
    echo "Swap Out: $pswpout_rate pages/s"
    
    prev_pswpin=$pswpin
    prev_pswpout=$pswpout
    
    sleep 1
done
```

---

## 📝 学习检查点

完成文件页与匿名页学习后，您应该能够：

- [ ] 理解文件页和匿名页的概念和区别
- [ ] 理解两者的内存映射机制
- [ ] 理解 LRU 列表如何管理这两种页面
- [ ] 理解 swap 机制的工作原理
- [ ] 理解回收策略的差异
- [ ] 能够查看和分析页面类型分布
- [ ] 能够诊断相关内存问题
- [ ] 能够优化系统内存使用
- [ ] 能够阅读相关源码

---

**下一步**：完成理论学习后，可以：
1. 阅读内核源码，深入理解实现细节
2. 在实际系统中观察文件页和匿名页的行为
3. 分析相关的实际问题
4. 结合 kswapd 和 Zone 学习，形成完整的内存管理知识体系
