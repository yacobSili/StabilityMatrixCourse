# Bio 机制详解

## 学习目标

- 理解 bio（Block IO）的设计理念和为什么需要 bio
- 掌握 bio 的结构和关键字段
- 理解 bio 的分配和释放机制
- 了解 bio 的合并和分割机制
- 理解 bio 向量（biovec）机制
- 了解 bio 与页缓存的关系

## 概述

bio（Block IO）是 Linux 内核 Block 层中最基本的数据结构，表示一个块设备 IO 操作。它是文件系统层与 Block 层交互的主要接口，也是整个 IO 路径中的核心数据结构。

本文档深入讲解 bio 的设计理念、实现机制和使用场景。

---

## 一、Bio 的设计理念

### 为什么需要 bio？

在早期的 Linux 内核中，IO 请求直接使用 `struct request` 表示。但随着内核的发展，这种设计遇到了以下问题：

1. **请求粒度太大**：一个 request 可能包含多个不连续的 IO 操作
2. **无法灵活处理**：难以处理 scatter-gather IO（分散-聚集 IO）
3. **与页缓存耦合**：request 与页缓存机制耦合过紧

### bio 的解决方案

bio 引入了以下设计：

1. **灵活的向量结构**：支持多个不连续的内存段（通过 biovec）
2. **与页缓存解耦**：bio 可以表示任意内存区域的 IO
3. **细粒度控制**：可以精确控制每个 IO 操作

### bio 在 IO 路径中的位置

```mermaid
graph LR
    subgraph VFS[VFS/文件系统层]
        FS[文件系统操作]
    end
    
    subgraph Bio[Bio 层]
        BioAlloc[bio 分配]
        BioConfig[bio 配置]
        BioSubmit[bio 提交]
    end
    
    subgraph Block[Block 层]
        Request[request 创建]
        Queue[队列管理]
    end
    
    FS -->|创建| BioAlloc
    BioAlloc --> BioConfig
    BioConfig --> BioSubmit
    BioSubmit -->|submit_bio| Request
    Request --> Queue
```

---

## 二、Bio 的结构和字段详解

### 完整结构定义

**定义位置**：`include/linux/blk_types.h`

```c
struct bio {
    // 链表相关
    struct bio *bi_next;           // bio 链表中的下一个 bio
    
    // 设备相关
    struct block_device *bi_bdev;  // 目标块设备
    
    // IO 操作相关
    unsigned short bi_opf;          // 操作标志（REQ_OP_READ/WRITE等）
    unsigned short bi_flags;        // bio 标志
    unsigned short bi_ioprio;       // IO 优先级
    
    // 数据相关
    struct bvec_iter bi_iter;      // 迭代器（当前位置）
    unsigned short bi_vcnt;         // biovec 数量
    unsigned short bi_max_vecs;     // 最大 biovec 数量
    struct bio_vec *bi_io_vec;      // biovec 数组指针
    struct bio_vec bi_inline_vecs[]; // 内联 biovec 数组（小 bio）
    
    // 完整性检查
    struct bio_integrity_payload *bi_integrity;
    
    // 回调函数
    bio_end_io_t *bi_end_io;        // IO 完成回调
    void *bi_private;               // 私有数据
    
    // 统计相关
    atomic_t __bi_remaining;        // 剩余引用计数
    struct bio_crypt_ctx *bi_crypt_context; // 加密上下文
    
    // 其他
    struct bvec_iter bi_iter;       // 迭代器
    // ...
};
```

### 关键字段详解

#### 1. bi_bdev - 块设备指针

**作用**：指向目标块设备

**使用场景**：
```c
// 文件系统层创建 bio 时设置
struct bio *bio = bio_alloc(GFP_NOFS, 1);
bio->bi_bdev = inode->i_sb->s_bdev;  // 设置目标设备
```

#### 2. bi_opf - 操作标志

**作用**：指定 IO 操作类型和标志

**操作类型**（`include/linux/blk_types.h`）：
```c
enum req_op {
    REQ_OP_READ,        // 读操作
    REQ_OP_WRITE,       // 写操作
    REQ_OP_FLUSH,       // 刷新操作
    REQ_OP_DISCARD,     // 丢弃操作
    REQ_OP_SECURE_ERASE, // 安全擦除
    REQ_OP_WRITE_SAME,  // 写入相同数据
    REQ_OP_WRITE_ZEROES, // 写入零
    // ...
};
```

**常见标志**：
- `REQ_SYNC` - 同步 IO
- `REQ_META` - 元数据 IO
- `REQ_PRIO` - 高优先级 IO
- `REQ_NOMERGE` - 禁止合并
- `REQ_IDLE` - 空闲时处理

#### 3. bi_iter - 迭代器

**作用**：跟踪当前处理位置

**结构**：
```c
struct bvec_iter {
    sector_t bi_sector;      // 当前扇区号
    unsigned int bi_size;    // 剩余大小（字节）
    unsigned int bi_idx;     // 当前 biovec 索引
    unsigned int bi_bvec_done; // 当前 biovec 已处理大小
};
```

**使用示例**：
```c
// 遍历 bio 的所有段
struct bvec_iter iter;
struct bio_vec bv;

bio_for_each_segment(bv, bio, iter) {
    // 处理每个段
    process_bvec(&bv);
}
```

#### 4. bi_io_vec - Bio 向量数组

**作用**：存储不连续的内存段

**特点**：
- 一个 bio 可以包含多个不连续的内存段
- 每个内存段用 `struct bio_vec` 表示
- 支持 scatter-gather IO

**bio_vec 结构**：
```c
struct bio_vec {
    struct page *bv_page;    // 页指针
    unsigned int bv_len;      // 长度（字节）
    unsigned int bv_offset;    // 页内偏移（字节）
};
```

**内存布局示例**：
```
bio:
  bi_io_vec[0]: page1, offset=0, len=4096
  bi_io_vec[1]: page2, offset=0, len=4096
  bi_io_vec[2]: page3, offset=0, len=2048
```

---

## 三、Bio 的分配和释放机制

### Bio 分配

#### 1. bio_alloc() - 分配 bio

**函数定义**：`block/bio.c`

```c
struct bio *bio_alloc(gfp_t gfp_mask, unsigned short nr_vecs)
{
    return bio_alloc_bioset(gfp_mask, nr_vecs, &fs_bio_set);
}
```

**参数说明**：
- `gfp_mask`：内存分配标志（如 `GFP_NOFS`、`GFP_KERNEL`）
- `nr_vecs`：需要的 biovec 数量

**分配策略**：
1. **小 bio**（≤4 个 biovec）：使用内联 biovec 数组（`bi_inline_vecs`）
2. **大 bio**（>4 个 biovec）：动态分配 biovec 数组

#### 2. bio_alloc_bioset() - 从指定 bio_set 分配

**函数实现**（简化）：
```c
struct bio *bio_alloc_bioset(gfp_t gfp_mask, unsigned short nr_vecs,
                             struct bio_set *bs)
{
    struct bio *bio;
    void *p;
    
    // 1. 尝试从 per-CPU 缓存分配
    if (bs->cache) {
        struct bio_alloc_cache *cache = get_cpu_ptr(bs->cache);
        
        if (!list_empty(&cache->free_list)) {
            bio = list_first_entry(&cache->free_list, struct bio, bi_list);
            list_del(&bio->bi_list);
            cache->nr--;
            put_cpu_ptr(cache);
            goto found;
        }
        put_cpu_ptr(cache);
    }
    
    // 2. 从 slab 分配 bio 结构
    p = kmem_cache_alloc(bs->bio_slab, gfp_mask);
    bio = p + bs->front_pad;
    
    // 3. 分配 biovec 数组
    if (nr_vecs > BIO_INLINE_VECS) {
        bio->bi_io_vec = bvec_alloc(bs->bvec_pool, &nr_vecs, gfp_mask);
        if (!bio->bi_io_vec)
            goto err_free;
    } else {
        // 使用内联 biovec
        bio->bi_io_vec = bio->bi_inline_vecs;
    }
    
    // 4. 初始化 bio
    bio_init(bio, bio->bi_io_vec, nr_vecs);
    
    return bio;
    
err_free:
    kmem_cache_free(bs->bio_slab, p);
    return NULL;
}
```

#### 3. Bio 缓存机制

**Per-CPU 缓存**：
- 每个 CPU 维护一个 bio 缓存列表
- 减少锁竞争，提高分配效率
- 缓存大小有限制，避免内存浪费

**缓存管理**：
```c
struct bio_alloc_cache {
    struct bio_list free_list;  // 空闲 bio 列表
    unsigned int nr;             // 缓存数量
};
```

### Bio 释放

#### 1. bio_put() - 释放 bio

**函数实现**（简化）：
```c
void bio_put(struct bio *bio)
{
    if (unlikely(atomic_dec_and_test(&bio->__bi_remaining))) {
        // 引用计数为 0，释放 bio
        bio_free(bio);
    }
}
```

**引用计数机制**：
- 每个 bio 有一个引用计数（`__bi_remaining`）
- `bio_get()` 增加引用计数
- `bio_put()` 减少引用计数
- 引用计数为 0 时释放 bio

#### 2. bio_free() - 实际释放

**函数实现**（简化）：
```c
static void bio_free(struct bio *bio)
{
    struct bio_set *bs = bio->bi_pool;
    void *p = bio;
    
    // 1. 释放 biovec 数组
    if (bio->bi_io_vec != bio->bi_inline_vecs)
        bvec_free(bs->bvec_pool, bio->bi_io_vec, bio->bi_vcnt);
    
    // 2. 释放完整性检查数据
    if (bio->bi_integrity)
        bio_integrity_free(bio);
    
    // 3. 释放加密上下文
    if (bio->bi_crypt_context)
        bio_crypt_free_ctx(bio);
    
    // 4. 释放 bio 结构
    p -= bs->front_pad;
    kmem_cache_free(bs->bio_slab, p);
}
```

#### 3. Bio 释放策略

**释放到缓存**：
```c
// 如果缓存未满，释放到 per-CPU 缓存
if (bs->cache && cache->nr < ALLOC_CACHE_MAX) {
    list_add(&bio->bi_list, &cache->free_list);
    cache->nr++;
    return;
}
```

**直接释放**：
- 如果缓存已满，直接释放到 slab
- 避免缓存占用过多内存

---

## 四、Bio 的合并和分割

### Bio 合并

#### 1. 为什么需要合并？

**优势**：
- 减少 IO 请求数量
- 提高 IO 效率
- 减少设备寻道时间

**合并条件**：
1. 相邻扇区
2. 相同操作类型
3. 可以合并的标志（没有 `REQ_NOMERGE`）

#### 2. bio_mergeable() - 检查是否可以合并

**函数实现**：
```c
static inline bool bio_mergeable(struct bio *bio)
{
    if (bio->bi_opf & REQ_NOMERGE_FLAGS)
        return false;
    
    return true;
}
```

#### 3. 合并场景

**场景 1：Plug 合并**
```c
// 在 plug 中合并相邻的 bio
static bool blk_attempt_plug_merge(struct request_queue *q,
                                   struct bio *bio,
                                   unsigned int nr_segs,
                                   struct request **same_queue_rq)
{
    struct blk_plug *plug = current->plug;
    
    if (!plug)
        return false;
    
    // 尝试与 plug 中的请求合并
    list_for_each_entry_reverse(rq, &plug->mq_list, queuelist) {
        if (blk_attempt_bio_merge(q, rq, bio, nr_segs, false) ==
            BIO_MERGE_OK) {
            *same_queue_rq = rq;
            return true;
        }
    }
    
    return false;
}
```

**场景 2：调度器合并**
```c
// 在调度器队列中合并
static bool blk_mq_sched_bio_merge(struct request_queue *q,
                                   struct bio *bio,
                                   unsigned int nr_segs)
{
    struct elevator_queue *e = q->elevator;
    
    if (e && e->type->ops.mq.bio_merge) {
        struct request *rq = e->type->ops.mq.bio_merge(q, bio, nr_segs);
        if (rq) {
            *same_queue_rq = rq;
            return true;
        }
    }
    
    return false;
}
```

### Bio 分割

#### 1. 为什么需要分割？

**原因**：
- 设备限制：单个 IO 不能超过设备的最大扇区数
- 队列限制：单个请求不能超过队列限制
- 边界对齐：某些操作需要对齐到特定边界

#### 2. __blk_queue_split() - 分割 bio

**函数实现**（简化）：
```c
void __blk_queue_split(struct bio **bio, unsigned int *nr_segs)
{
    struct request_queue *q = (*bio)->bi_bdev->bd_disk->queue;
    unsigned int max_sectors = queue_max_sectors(q);
    unsigned int sectors = bio_sectors(*bio);
    
    // 如果 bio 大小超过限制，需要分割
    if (sectors > max_sectors) {
        struct bio *split = bio_split(*bio, max_sectors,
                                      GFP_NOIO, &q->bio_split);
        bio_chain(split, *bio);
        submit_bio_noacct(*bio);
        *bio = split;
    }
}
```

#### 3. bio_split() - 实际分割

**函数实现**（简化）：
```c
struct bio *bio_split(struct bio *bio, int sectors,
                      gfp_t gfp, struct bio_set *bs)
{
    struct bio *split;
    struct bvec_iter iter;
    
    // 分配新的 bio
    split = bio_alloc_bioset(gfp, bio->bi_max_vecs, bs);
    
    // 复制 bio 的元数据
    __bio_clone_fast(split, bio);
    
    // 设置分割后的扇区数
    split->bi_iter.bi_size = sectors << 9;
    
    // 更新原 bio 的迭代器
    bio_advance_iter(bio, &bio->bi_iter, sectors << 9);
    
    return split;
}
```

---

## 五、Bio 向量（biovec）机制

### Biovec 的作用

**支持 scatter-gather IO**：
- 一个 bio 可以包含多个不连续的内存段
- 每个内存段用 biovec 表示
- 支持高效的分散-聚集 IO

### Biovec 的结构

```c
struct bio_vec {
    struct page *bv_page;    // 页指针
    unsigned int bv_len;      // 长度（字节）
    unsigned int bv_offset;    // 页内偏移（字节）
};
```

### Biovec 的分配

#### 1. 内联 biovec（小 bio）

**特点**：
- bio 结构体中包含内联 biovec 数组
- 适用于小 IO（≤4 个 biovec）
- 无需额外分配内存

**定义**：
```c
struct bio {
    // ...
    struct bio_vec bi_inline_vecs[BIO_INLINE_VECS]; // 内联数组
};
```

#### 2. 动态 biovec（大 bio）

**特点**：
- 动态分配 biovec 数组
- 适用于大 IO（>4 个 biovec）
- 从 biovec slab 或 mempool 分配

**分配策略**：
```c
struct biovec_slab {
    int nr_vecs;
    char *name;
    struct kmem_cache *slab;
} bvec_slabs[] = {
    { .nr_vecs = 16, .name = "biovec-16" },
    { .nr_vecs = 64, .name = "biovec-64" },
    { .nr_vecs = 128, .name = "biovec-128" },
    { .nr_vecs = BIO_MAX_VECS, .name = "biovec-max" },
};
```

### Biovec 的遍历

#### 1. bio_for_each_segment() - 遍历所有段

```c
struct bio_vec bv;
struct bvec_iter iter;

bio_for_each_segment(bv, bio, iter) {
    // 处理每个段
    void *addr = page_address(bv.bv_page) + bv.bv_offset;
    size_t len = bv.bv_len;
    
    // 执行 IO 操作
    do_io(addr, len);
}
```

#### 2. bio_iter_advance() - 推进迭代器

```c
// 推进迭代器，跳过已处理的字节
void bio_advance_iter(const struct bio *bio,
                      struct bvec_iter *iter,
                      unsigned int bytes)
{
    iter->bi_sector += bytes >> 9;
    
    if (bio_no_advance_iter(bio))
        iter->bi_size -= bytes;
    else
        bvec_iter_advance(bio->bi_io_vec, iter, bytes);
}
```

---

## 六、Bio 与页缓存的关系

### 页缓存到 Bio 的转换

#### 1. 文件系统层创建 bio

**典型场景**：读取文件页

```c
// fs/ext4/inode.c (简化)
static int ext4_readpage(struct file *file, struct page *page)
{
    struct inode *inode = page->mapping->host;
    struct bio *bio;
    
    // 分配 bio
    bio = bio_alloc(GFP_NOFS, 1);
    
    // 设置设备
    bio->bi_bdev = inode->i_sb->s_bdev;
    
    // 设置操作类型
    bio->bi_opf = REQ_OP_READ;
    
    // 添加页到 bio
    bio_add_page(bio, page, PAGE_SIZE, 0);
    
    // 设置完成回调
    bio->bi_end_io = end_page_read;
    bio->bi_private = page;
    
    // 提交到 Block 层
    submit_bio(bio);
    
    return 0;
}
```

#### 2. bio_add_page() - 添加页到 bio

**函数实现**（简化）：
```c
int bio_add_page(struct bio *bio, struct page *page,
                 unsigned int len, unsigned int offset)
{
    struct bio_vec *bv;
    
    // 检查 bio 是否已满
    if (bio_full(bio, len))
        return 0;
    
    // 获取下一个 biovec
    bv = &bio->bi_io_vec[bio->bi_vcnt];
    
    // 设置 biovec
    bv->bv_page = page;
    bv->bv_len = len;
    bv->bv_offset = offset;
    
    // 更新 bio
    bio->bi_vcnt++;
    bio->bi_iter.bi_size += len;
    
    return len;
}
```

### Bio 到页缓存的回调

#### 1. bio 完成回调

**典型场景**：页读取完成

```c
// fs/ext4/inode.c (简化)
static void end_page_read(struct bio *bio)
{
    struct page *page = bio->bi_private;
    
    if (bio->bi_status == BLK_STS_OK) {
        // 标记页为最新
        SetPageUptodate(page);
    } else {
        // 标记页为错误
        ClearPageUptodate(page);
        SetPageError(page);
    }
    
    // 解锁页
    unlock_page(page);
    
    // 释放 bio
    bio_put(bio);
}
```

---

## 七、Bio 的使用场景

### 场景 1：文件系统读取

```c
// 文件系统读取文件页
static int ext4_readpage(struct file *file, struct page *page)
{
    struct bio *bio = bio_alloc(GFP_NOFS, 1);
    
    bio->bi_bdev = inode->i_sb->s_bdev;
    bio->bi_opf = REQ_OP_READ;
    bio_add_page(bio, page, PAGE_SIZE, 0);
    bio->bi_end_io = end_page_read;
    bio->bi_private = page;
    
    submit_bio(bio);
    return 0;
}
```

### 场景 2：直接 IO

```c
// 直接 IO（绕过页缓存）
static ssize_t ext4_direct_IO(struct kiocb *iocb, struct iov_iter *iter)
{
    struct bio *bio;
    struct bio_vec *bvec;
    int ret;
    
    // 创建 bio
    bio = bio_alloc(GFP_KERNEL, iter->nr_segs);
    
    // 设置设备
    bio->bi_bdev = inode->i_sb->s_bdev;
    bio->bi_opf = REQ_OP_READ | REQ_SYNC;
    
    // 添加所有段
    iov_iter_for_each_range(iter, bvec, i) {
        bio_add_page(bio, bvec->bv_page, bvec->bv_len, bvec->bv_offset);
    }
    
    // 提交
    submit_bio(bio);
    
    return ret;
}
```

### 场景 3：多段 IO

```c
// 处理多个不连续的页
static int write_multiple_pages(struct inode *inode, struct page **pages, int nr_pages)
{
    struct bio *bio;
    int i;
    
    bio = bio_alloc(GFP_NOFS, nr_pages);
    bio->bi_bdev = inode->i_sb->s_bdev;
    bio->bi_opf = REQ_OP_WRITE;
    
    // 添加所有页
    for (i = 0; i < nr_pages; i++) {
        bio_add_page(bio, pages[i], PAGE_SIZE, 0);
    }
    
    submit_bio(bio);
    return 0;
}
```

---

## 总结

### 核心要点

1. **Bio 的设计理念**：
   - 灵活的向量结构，支持 scatter-gather IO
   - 与页缓存解耦，可以表示任意内存区域的 IO
   - 细粒度控制，精确控制每个 IO 操作

2. **Bio 的关键机制**：
   - **分配和释放**：使用 per-CPU 缓存和 slab 分配器
   - **合并和分割**：支持相邻 bio 的合并和超大 bio 的分割
   - **向量机制**：支持多个不连续内存段的 IO

3. **Bio 与页缓存的关系**：
   - 文件系统层创建 bio 表示页的 IO
   - bio 完成时回调文件系统层更新页状态

### 关键函数

- `bio_alloc()` - 分配 bio
- `bio_put()` - 释放 bio
- `bio_add_page()` - 添加页到 bio
- `bio_split()` - 分割 bio
- `submit_bio()` - 提交 bio 到 Block 层

### 后续学习

- [Bio 完整性检查与加密](05-Bio完整性检查与加密.md) - 理解 bio 的数据完整性和加密支持
- [Request 机制详解](06-Request机制详解.md) - 理解 bio 如何转换为 request
- [Block 层 IO 路径总览](03-Block层IO路径总览.md) - 理解 bio 在完整 IO 路径中的作用

## 参考资源

- 内核源码：
  - `block/bio.c` - bio 的分配、释放、合并、分割
  - `include/linux/bio.h` - bio 数据结构定义
  - `include/linux/blk_types.h` - bio 类型定义
- 相关文章：
  - [Block 层核心数据结构](02-Block层核心数据结构.md) - bio 数据结构详解
  - [Block 层 IO 路径总览](03-Block层IO路径总览.md) - bio 在 IO 路径中的作用

## 更新记录

- 2026-01-26：初始创建，包含 bio 机制的详细说明
