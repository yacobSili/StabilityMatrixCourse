# Page Cache 与内存交互

## 学习目标

- 理解 Page Cache 的作用和原理
- 掌握 address_space 数据结构
- 了解页面缓存的查找、添加、回写机制
- 理解 Page Cache 与内存管理的交互

## 一、Page Cache 概述

### 1.1 什么是 Page Cache

Page Cache（页缓存）是 Linux 内核用于缓存文件数据的内存区域：

```
用户空间          内核空间                   存储设备
┌─────────┐      ┌──────────────┐           ┌─────────┐
│         │ read │              │  读取     │         │
│ 用户    │◄─────│  Page Cache  │◄──────────│  磁盘   │
│ 缓冲区  │      │              │           │         │
│         │ write│              │  回写     │         │
│         │─────►│              │──────────►│         │
└─────────┘      └──────────────┘           └─────────┘

优点：
1. 减少磁盘 I/O
2. 加速文件访问
3. 合并写入操作
4. 支持内存映射
```

### 1.2 Page Cache vs Buffer Cache

在早期 Linux 中：
- **Page Cache**：缓存文件数据
- **Buffer Cache**：缓存块设备数据

现代 Linux（2.4+）已统一：
- 统一使用 Page Cache
- Buffer Cache 作为 Page Cache 的一部分（buffer_head）

### 1.3 Page Cache 的位置

```c
// Page Cache 在内存统计中的表示
// /proc/meminfo
Cached:          2765440 kB    // Page Cache 大小
Buffers:          162652 kB    // Buffer Cache（块设备元数据）
```

---

## 二、核心数据结构

### 2.1 address_space

```c
// include/linux/fs.h
struct address_space {
    struct inode        *host;          // 所属 inode
    struct xarray       i_pages;        // 页面基数树/Xarray
    struct rw_semaphore invalidate_lock;// 无效化锁
    gfp_t               gfp_mask;       // 分配掩码
    atomic_t            i_mmap_writable;// 可写映射计数
    
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
    atomic_t            nr_thps;        // 透明大页数量
#endif
    
    struct rb_root_cached i_mmap;       // VMA 树（映射该文件的 VMA）
    struct rw_semaphore i_mmap_rwsem;   // VMA 树锁
    unsigned long       nrpages;        // 页面总数
    pgoff_t             writeback_index;// 回写起始索引
    const struct address_space_operations *a_ops;  // 操作函数
    unsigned long       flags;          // 标志
    errseq_t            wb_err;         // 回写错误
    spinlock_t          private_lock;   // 私有数据锁
    struct list_head    private_list;   // 私有数据链表
    void                *private_data;  // 私有数据
};
```

### 2.2 address_space 与 inode 的关系

```
文件系统视角：

struct inode
    │
    ├── i_mapping ──────► address_space
    │                         │
    │                         ├── i_pages (Xarray)
    │                         │      │
    │                         │      └── page 0, page 1, page 2, ...
    │                         │
    │                         └── a_ops (操作函数)
    │
    └── i_data (嵌入的 address_space)

// 大多数文件：i_mapping = &inode->i_data
// 块设备：i_mapping = bdev->bd_inode->i_mapping
```

### 2.3 address_space_operations

```c
// include/linux/fs.h
struct address_space_operations {
    // 回写单个页面
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    
    // 读取页面
    int (*readpage)(struct file *, struct page *);
    
    // 批量回写
    int (*writepages)(struct address_space *, struct writeback_control *);
    
    // 设置页面为脏
    int (*set_page_dirty)(struct page *page);
    
    // 批量读取
    int (*readpages)(struct file *filp, struct address_space *mapping,
                     struct list_head *pages, unsigned nr_pages);
    
    // 读取 folio（新接口）
    int (*readahead)(struct readahead_control *);
    
    // 写入开始
    int (*write_begin)(struct file *, struct address_space *mapping,
                       loff_t pos, unsigned len, unsigned flags,
                       struct page **pagep, void **fsdata);
    
    // 写入结束
    int (*write_end)(struct file *, struct address_space *mapping,
                     loff_t pos, unsigned len, unsigned copied,
                     struct page *page, void *fsdata);
    
    // 将页面标记为脏的回调
    int (*launder_page)(struct page *);
    
    // 直接 I/O
    ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *iter);
    
    // 迁移页面
    int (*migratepage)(struct address_space *, struct page *newpage,
                       struct page *page, enum migrate_mode);
    
    // 释放页面
    int (*releasepage)(struct page *, gfp_t);
    void (*freepage)(struct page *);
    
    // 错误处理
    int (*error_remove_page)(struct address_space *, struct page *);
    
    // swap 相关
    int (*swap_activate)(struct swap_info_struct *, struct file *, sector_t *);
    void (*swap_deactivate)(struct file *);
};
```

---

## 三、Page Cache 操作

### 3.1 页面查找

```c
// mm/filemap.c

// 查找页面（不增加引用）
struct page *find_get_page(struct address_space *mapping, pgoff_t index)
{
    return pagecache_get_page(mapping, index, 0, 0);
}

// 查找页面（带引用）
struct page *find_lock_page(struct address_space *mapping, pgoff_t index)
{
    return pagecache_get_page(mapping, index, FGP_LOCK, 0);
}

// 核心查找函数
struct page *pagecache_get_page(struct address_space *mapping, pgoff_t index,
                                int fgp_flags, gfp_t gfp)
{
    struct page *page;
    
repeat:
    // 从 Xarray 查找
    page = find_get_entry(mapping, index);
    if (!page) {
        if (fgp_flags & FGP_CREAT) {
            // 需要创建页面
            page = __page_cache_alloc(gfp);
            if (!page)
                return NULL;
            
            // 添加到缓存
            int err = add_to_page_cache_lru(page, mapping, index, gfp);
            if (err) {
                put_page(page);
                if (err == -EEXIST)
                    goto repeat;
                return NULL;
            }
        }
    }
    
    if (page && (fgp_flags & FGP_LOCK)) {
        lock_page(page);
    }
    
    return page;
}
```

### 3.2 添加页面到缓存

```c
// mm/filemap.c
int add_to_page_cache_lru(struct page *page, struct address_space *mapping,
                          pgoff_t offset, gfp_t gfp_mask)
{
    void *shadow = NULL;
    int ret;
    
    __SetPageLocked(page);
    
    // 添加到 address_space
    ret = __add_to_page_cache_locked(page, mapping, offset, gfp_mask, &shadow);
    if (ret)
        return ret;
    
    // 添加到 LRU 链表
    lru_cache_add(page);
    
    return 0;
}

static int __add_to_page_cache_locked(struct page *page,
                                      struct address_space *mapping,
                                      pgoff_t offset, gfp_t gfp,
                                      void **shadowp)
{
    XA_STATE(xas, &mapping->i_pages, offset);
    void *old;
    
    // 设置页面映射
    page->mapping = mapping;
    page->index = offset;
    
    // 插入 Xarray
    do {
        xas_lock_irq(&xas);
        old = xas_load(&xas);
        if (old && !xa_is_value(old)) {
            // 已存在
            xas_unlock_irq(&xas);
            return -EEXIST;
        }
        xas_store(&xas, page);
        xas_unlock_irq(&xas);
    } while (xas_nomem(&xas, gfp));
    
    mapping->nrpages++;
    
    return 0;
}
```

### 3.3 标记页面为脏

```c
// mm/page-writeback.c
int set_page_dirty(struct page *page)
{
    struct address_space *mapping = page_mapping(page);
    
    if (likely(mapping)) {
        // 调用文件系统的 set_page_dirty
        if (mapping->a_ops->set_page_dirty)
            return mapping->a_ops->set_page_dirty(page);
        return __set_page_dirty_buffers(page);
    }
    
    // 匿名页面
    if (!PageDirty(page)) {
        SetPageDirty(page);
        return 1;
    }
    return 0;
}

// 通用实现
int __set_page_dirty_nobuffers(struct page *page)
{
    struct address_space *mapping = page_mapping(page);
    
    if (mapping) {
        xa_lock_irq(&mapping->i_pages);
        if (page->mapping) {
            // 设置脏标志
            if (!TestSetPageDirty(page)) {
                // 标记 mapping 需要回写
                __xa_set_mark(&mapping->i_pages, page_index(page), PAGECACHE_TAG_DIRTY);
            }
        }
        xa_unlock_irq(&mapping->i_pages);
        
        if (!PageDirty(page))
            return 0;
    }
    
    return 1;
}
```

---

## 四、页面回写

### 4.1 回写触发条件

```c
// 回写触发条件：
// 1. 脏页比例超过阈值（dirty_ratio）
// 2. 脏页老化超时（dirty_expire_centisecs）
// 3. sync/fsync 系统调用
// 4. 内存压力导致页面回收

// 相关参数
/proc/sys/vm/dirty_ratio            // 脏页占内存比例阈值（默认 20%）
/proc/sys/vm/dirty_background_ratio // 后台回写阈值（默认 10%）
/proc/sys/vm/dirty_expire_centisecs // 脏页过期时间（默认 3000 = 30秒）
/proc/sys/vm/dirty_writeback_centisecs // 回写周期（默认 500 = 5秒）
```

### 4.2 回写控制结构

```c
// include/linux/writeback.h
struct writeback_control {
    long nr_to_write;           // 需要写的页数
    long pages_skipped;         // 跳过的页数
    
    loff_t range_start;         // 起始偏移
    loff_t range_end;           // 结束偏移
    
    enum writeback_sync_modes sync_mode;  // 同步模式
    unsigned for_kupdate:1;     // 周期性回写
    unsigned for_background:1;  // 后台回写
    unsigned tagged_writepages:1;// 标记回写
    unsigned for_reclaim:1;     // 为回收而回写
    unsigned range_cyclic:1;    // 循环范围
    unsigned for_sync:1;        // sync 触发
    unsigned unpinned_fscache_wb:1;
    
    #ifdef CONFIG_CGROUP_WRITEBACK
    struct bdi_writeback *wb;   // 回写设备
    struct inode *inode;        // 当前 inode
    int wb_id;                  // 回写 ID
    int wb_lcand_id;
    int wb_tcand_id;
    size_t wb_bytes;
    size_t wb_lcand_bytes;
    size_t wb_tcand_bytes;
    #endif
};

enum writeback_sync_modes {
    WB_SYNC_NONE,   // 不等待
    WB_SYNC_ALL,    // 等待所有完成
};
```

### 4.3 回写流程

```c
// mm/page-writeback.c
// 通用回写入口
int write_cache_pages(struct address_space *mapping,
                      struct writeback_control *wbc,
                      writepage_t writepage, void *data)
{
    struct page *pages[PAGEVEC_SIZE];
    pgoff_t index, end;
    int ret = 0;
    
    // 确定回写范围
    if (wbc->range_cyclic) {
        index = mapping->writeback_index;
        end = -1;
    } else {
        index = wbc->range_start >> PAGE_SHIFT;
        end = wbc->range_end >> PAGE_SHIFT;
    }
    
    // 遍历脏页
    while (index <= end) {
        int nr_pages = find_get_pages_range_tag(mapping, &index, end,
                                                PAGECACHE_TAG_DIRTY, PAGEVEC_SIZE, pages);
        if (nr_pages == 0)
            break;
        
        for (i = 0; i < nr_pages; i++) {
            struct page *page = pages[i];
            
            lock_page(page);
            
            if (page->mapping != mapping || !PageDirty(page)) {
                unlock_page(page);
                continue;
            }
            
            // 写入页面
            ret = (*writepage)(page, wbc);
            if (ret < 0)
                goto out;
            
            if (--wbc->nr_to_write <= 0)
                goto out;
        }
        
        cond_resched();
    }
    
out:
    if (wbc->range_cyclic)
        mapping->writeback_index = index;
    
    return ret;
}
```

### 4.4 单页面回写

```c
// 文件系统实现 writepage
// fs/ext4/inode.c
static int ext4_writepage(struct page *page, struct writeback_control *wbc)
{
    // 获取 buffer_head
    // 提交 bio 到块层
    // ...
    
    return ext4_bio_write_page(page, wbc);
}

// 通用文件系统回写
// mm/page-writeback.c
int __block_write_full_page(struct inode *inode, struct page *page,
                            get_block_t *get_block, struct writeback_control *wbc,
                            bh_end_io_t *handler)
{
    struct buffer_head *bh, *head;
    
    // 遍历页面的所有 buffer_head
    bh = head = page_buffers(page);
    do {
        if (!buffer_mapped(bh)) {
            // 获取块映射
            get_block(inode, block, bh, 0);
        }
        
        if (buffer_dirty(bh)) {
            // 提交写入
            submit_bh(REQ_OP_WRITE, bh);
        }
    } while ((bh = bh->b_this_page) != head);
    
    return 0;
}
```

---

## 五、Page Cache 与内存管理的交互

### 5.1 Page Cache 的内存统计

```c
// 页面类型判断
static inline bool page_is_file_lru(struct page *page)
{
    return !PageSwapBacked(page);  // 文件页 = 非 swap 支持
}

// LRU 链表分类
// 文件页：LRU_INACTIVE_FILE / LRU_ACTIVE_FILE
// 匿名页：LRU_INACTIVE_ANON / LRU_ACTIVE_ANON
```

### 5.2 Page Cache 回收

```c
// mm/vmscan.c
// 回收 Page Cache 页面
static unsigned long shrink_page_list(struct list_head *page_list, ...)
{
    while (!list_empty(page_list)) {
        struct page *page = lru_to_page(page_list);
        
        // 文件页处理
        if (page_has_private(page)) {
            // 有 buffer_head，尝试释放
            if (!try_to_release_page(page, gfp_mask))
                goto keep_locked;
        }
        
        // 检查页面是否被映射
        if (page_mapped(page)) {
            // 需要解除映射
            try_to_unmap(page, ...);
        }
        
        // 检查是否脏页
        if (PageDirty(page)) {
            // 触发回写
            pageout(page, mapping);
            goto keep;
        }
        
        // 从 Page Cache 移除
        if (page_has_private(page) || page_mapped(page))
            goto keep_locked;
        
        __delete_from_page_cache(page, NULL);
        
        // 释放页面
        free_unref_page(page);
        nr_reclaimed++;
    }
    
    return nr_reclaimed;
}
```

### 5.3 读取预热

```c
// mm/readahead.c
// 预读机制

void page_cache_ra_unbounded(struct readahead_control *ractl,
                             unsigned long nr_to_read,
                             unsigned long lookahead_size)
{
    struct address_space *mapping = ractl->mapping;
    LIST_HEAD(page_pool);
    
    // 分配页面
    for (i = 0; i < nr_to_read; i++) {
        struct page *page = __page_cache_alloc(gfp_mask);
        if (!page)
            break;
        
        // 添加到缓存
        if (add_to_page_cache_lru(page, mapping, index + i, gfp_mask) < 0) {
            put_page(page);
            continue;
        }
        
        list_add(&page->lru, &page_pool);
    }
    
    // 触发实际读取
    if (!list_empty(&page_pool)) {
        mapping->a_ops->readahead(ractl);
    }
}
```

---

## 六、/proc 接口

### 6.1 /proc/meminfo 中的 Page Cache

```bash
$ cat /proc/meminfo
Cached:          2765440 kB    # Page Cache 大小
Buffers:          162652 kB    # Buffer Cache
Active(file):    1582180 kB    # 活跃文件页
Inactive(file):  1803260 kB    # 不活跃文件页
Dirty:             12345 kB    # 脏页大小
Writeback:             0 kB    # 正在回写
```

### 6.2 /proc/sys/vm/ 参数

```bash
# 脏页控制
/proc/sys/vm/dirty_ratio              # 20 (%)
/proc/sys/vm/dirty_background_ratio   # 10 (%)
/proc/sys/vm/dirty_expire_centisecs   # 3000 (30秒)
/proc/sys/vm/dirty_writeback_centisecs # 500 (5秒)

# 内存压力下的行为
/proc/sys/vm/vfs_cache_pressure       # 100 (默认)
```

---

## 总结

### 核心概念

1. **Page Cache**：文件数据的内存缓存
2. **address_space**：管理文件页面的数据结构
3. **脏页**：被修改但未写回的页面
4. **回写**：将脏页写回存储设备

### 关键数据结构

| 结构 | 作用 |
|-----|------|
| address_space | 管理文件的所有缓存页 |
| address_space_operations | 读写回调函数 |
| writeback_control | 回写控制参数 |

### 与其他模块的交互

| 模块 | 交互点 |
|-----|-------|
| VFS | 通过 address_space 访问缓存 |
| 内存回收 | 回收 Page Cache 页面 |
| Block 层 | 提交 I/O 请求 |
| mmap | 文件映射共享 Page Cache |

### 后续学习

- [页面回收框架详解](13-页面回收框架详解.md) - 了解 Page Cache 回收
- [内存与其他子系统交互](19-内存与其他子系统交互.md) - 了解完整交互关系

## 参考资源

- 内核源码：
  - `mm/filemap.c` - Page Cache 核心
  - `mm/page-writeback.c` - 回写机制
- 内核文档：`Documentation/filesystems/vfs.rst`

## 更新记录

- 2026-01-28：初始创建，包含 Page Cache 与内存交互详解
