# VMA 管理机制详解

## 学习目标

- 理解 VMA (虚拟内存区域) 的概念和作用
- 掌握 VMA 的创建、查找、合并、拆分操作
- 了解 VMA 的数据结构组织方式
- 理解 VMA 与内存映射的关系

## 一、VMA 概述

### 1.1 什么是 VMA

VMA（Virtual Memory Area，虚拟内存区域）描述进程地址空间中一段连续的虚拟地址区域。每个 VMA 具有：
- 相同的权限属性
- 相同的映射类型（文件/匿名）
- 相同的后备存储

```
进程地址空间由多个 VMA 组成：

地址
高  ┌─────────────────────────┐
    │  VMA 5: 栈              │ VM_GROWSDOWN
    ├─────────────────────────┤
    │       空闲区域           │
    ├─────────────────────────┤
    │  VMA 4: libc.so 数据    │ VM_READ | VM_WRITE
    ├─────────────────────────┤
    │  VMA 3: libc.so 代码    │ VM_READ | VM_EXEC
    ├─────────────────────────┤
    │       空闲区域           │
    ├─────────────────────────┤
    │  VMA 2: 堆              │ VM_READ | VM_WRITE
    ├─────────────────────────┤
    │  VMA 1: 数据段          │ VM_READ | VM_WRITE
    ├─────────────────────────┤
    │  VMA 0: 代码段          │ VM_READ | VM_EXEC
低  └─────────────────────────┘
```

### 1.2 VMA 的作用

1. **内存管理单元**：进程内存操作的基本单位
2. **权限控制**：不同区域有不同的访问权限
3. **按需分配**：缺页时知道如何处理
4. **资源跟踪**：跟踪文件映射、匿名映射

---

## 二、VMA 数据结构

### 2.1 vm_area_struct

```c
// include/linux/mm_types.h
struct vm_area_struct {
    /* ===== 地址范围 ===== */
    unsigned long vm_start;         // 起始地址（包含）
    unsigned long vm_end;           // 结束地址（不包含）
    
    /* ===== 所属内存描述符 ===== */
    struct mm_struct *vm_mm;        // 所属 mm_struct
    
    /* ===== 页保护属性 ===== */
    pgprot_t vm_page_prot;          // 页表项保护位
    
    /* ===== VMA 标志 ===== */
    unsigned long vm_flags;         // VM_READ, VM_WRITE, VM_EXEC 等
    
    /* ===== 链接结构（旧版本）===== */
    struct vm_area_struct *vm_next; // 链表下一个
    struct vm_area_struct *vm_prev; // 链表上一个
    
    /* ===== 树结构 ===== */
    struct rb_node vm_rb;           // 红黑树节点（旧版本）
    
    /* ===== 匿名映射 ===== */
    struct list_head anon_vma_chain;// 匿名 VMA 链表
    struct anon_vma *anon_vma;      // 匿名 VMA 描述符
    
    /* ===== 操作函数 ===== */
    const struct vm_operations_struct *vm_ops;
    
    /* ===== 文件映射 ===== */
    unsigned long vm_pgoff;         // 文件偏移（页为单位）
    struct file *vm_file;           // 映射的文件（NULL=匿名）
    void *vm_private_data;          // 私有数据
    
    /* ===== NUMA 相关 ===== */
#ifdef CONFIG_NUMA
    struct mempolicy *vm_policy;    // NUMA 内存策略
#endif
};
```

### 2.2 VMA 组织方式

#### 旧版本：红黑树 + 链表

```c
// Linux 6.0 之前
struct mm_struct {
    struct vm_area_struct *mmap;    // VMA 链表头
    struct rb_root mm_rb;           // VMA 红黑树根
    // ...
};

// 链表用于顺序遍历
// 红黑树用于快速查找
```

#### 新版本：Maple Tree (6.1+)

```c
// Linux 6.1+
struct mm_struct {
    struct maple_tree mm_mt;        // VMA Maple Tree
    // ...
};

// Maple Tree 优点：
// - 更好的缓存局部性
// - 原生支持范围查询
// - 简化代码
```

### 2.3 vm_flags 标志

```c
// include/linux/mm.h

/* 基本权限 */
#define VM_READ         0x00000001  // 可读
#define VM_WRITE        0x00000002  // 可写
#define VM_EXEC         0x00000004  // 可执行
#define VM_SHARED       0x00000008  // 共享映射

/* 可修改权限 */
#define VM_MAYREAD      0x00000010  // 可以设置为可读
#define VM_MAYWRITE     0x00000020  // 可以设置为可写
#define VM_MAYEXEC      0x00000040  // 可以设置为可执行
#define VM_MAYSHARE     0x00000080  // 可以设置为共享

/* 增长方向 */
#define VM_GROWSDOWN    0x00000100  // 向下增长（栈）
#define VM_GROWSUP      0x00000200  // 向上增长

/* 特殊属性 */
#define VM_PFNMAP       0x00000400  // 纯 PFN 映射（无 struct page）
#define VM_LOCKED       0x00002000  // 锁定在内存中
#define VM_IO           0x00004000  // I/O 内存映射
#define VM_SEQ_READ     0x00008000  // 顺序读取提示
#define VM_RAND_READ    0x00010000  // 随机读取提示
#define VM_DONTCOPY     0x00020000  // fork 时不复制
#define VM_DONTEXPAND   0x00040000  // 不可扩展
#define VM_LOCKONFAULT  0x00080000  // 缺页时锁定
#define VM_ACCOUNT      0x00100000  // 记账
#define VM_NORESERVE    0x00200000  // 不预留 swap
#define VM_HUGETLB      0x00400000  // 大页
#define VM_SYNC         0x00800000  // 同步映射
#define VM_ARCH_1       0x01000000  // 架构特定
#define VM_WIPEONFORK   0x02000000  // fork 时清零
#define VM_DONTDUMP     0x04000000  // 不包含在 coredump
#define VM_SOFTDIRTY    0x08000000  // 软脏页跟踪
#define VM_MIXEDMAP     0x10000000  // 混合映射（有/无 struct page）
#define VM_HUGEPAGE     0x20000000  // 透明大页
#define VM_NOHUGEPAGE   0x40000000  // 禁止透明大页
#define VM_MERGEABLE    0x80000000  // KSM 可合并
```

---

## 三、VMA 操作

### 3.1 查找 VMA

```c
// mm/mmap.c

// 查找包含或紧接 addr 的 VMA
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
    struct vm_area_struct *vma;
    
    // 先检查缓存
    vma = vmacache_find(mm, addr);
    if (likely(vma))
        return vma;
    
    // 从 Maple Tree 查找（6.1+）
    vma = mt_find(&mm->mm_mt, &addr, ULONG_MAX);
    if (vma)
        vmacache_update(addr, vma);
    
    return vma;
}

// 查找 addr 所在的 VMA（严格）
struct vm_area_struct *find_vma_exact(struct mm_struct *mm,
                                      unsigned long addr, unsigned long len)
{
    struct vm_area_struct *vma = find_vma(mm, addr);
    
    if (!vma || vma->vm_start != addr || vma->vm_end != addr + len)
        return NULL;
    
    return vma;
}

// 查找与 [addr, addr+len) 相交的 VMA
struct vm_area_struct *find_vma_intersection(struct mm_struct *mm,
                                             unsigned long start_addr,
                                             unsigned long end_addr)
{
    struct vm_area_struct *vma = find_vma(mm, start_addr);
    
    if (vma && vma->vm_start < end_addr)
        return vma;
    
    return NULL;
}
```

### 3.2 创建 VMA

```c
// mm/mmap.c
// mmap 创建 VMA 的核心流程

unsigned long mmap_region(struct file *file, unsigned long addr,
                          unsigned long len, vm_flags_t vm_flags,
                          unsigned long pgoff, struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;
    unsigned long charged = 0;
    
    // 1. 检查是否需要解除现有映射
    if (find_vma_intersection(mm, addr, addr + len)) {
        if (do_munmap(mm, addr, len, uf))
            return -ENOMEM;
    }
    
    // 2. 尝试与相邻 VMA 合并
    vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
                    NULL, file, pgoff, NULL, NULL);
    if (vma)
        goto out;
    
    // 3. 分配新 VMA
    vma = vm_area_alloc(mm);
    if (!vma)
        return -ENOMEM;
    
    // 4. 初始化 VMA
    vma->vm_start = addr;
    vma->vm_end = addr + len;
    vma->vm_flags = vm_flags;
    vma->vm_page_prot = vm_get_page_prot(vm_flags);
    vma->vm_pgoff = pgoff;
    
    // 5. 文件映射设置
    if (file) {
        vma->vm_file = get_file(file);
        error = call_mmap(file, vma);  // 调用文件系统的 mmap
        if (error)
            goto unmap_and_free_vma;
    }
    
    // 6. 插入 VMA 到 mm
    if (vma_link(mm, vma))
        goto unmap_and_free_vma;
    
out:
    return addr;
}
```

### 3.3 VMA 合并

```c
// mm/mmap.c
// 合并相邻的兼容 VMA

struct vm_area_struct *vma_merge(struct mm_struct *mm,
                                 struct vm_area_struct *prev,
                                 unsigned long addr, unsigned long end,
                                 unsigned long vm_flags,
                                 struct anon_vma *anon_vma,
                                 struct file *file, pgoff_t pgoff,
                                 struct mempolicy *policy,
                                 struct vm_userfaultfd_ctx vm_userfaultfd_ctx)
{
    struct vm_area_struct *area, *next;
    int err;
    
    // 检查是否可以与前一个 VMA 合并
    if (prev && prev->vm_end == addr &&
        can_vma_merge_after(prev, vm_flags, anon_vma, file, pgoff, policy)) {
        // 可以合并
    }
    
    // 检查是否可以与后一个 VMA 合并
    next = prev ? prev->vm_next : mm->mmap;
    if (next && end == next->vm_start &&
        can_vma_merge_before(next, vm_flags, anon_vma, file, pgoff + (end - addr) >> PAGE_SHIFT, policy)) {
        // 可以合并
    }
    
    // 执行合并
    // ...
    
    return merged_vma;
}

// 检查两个 VMA 是否可以合并
static bool can_vma_merge_after(struct vm_area_struct *vma,
                                unsigned long vm_flags,
                                struct anon_vma *anon_vma,
                                struct file *file, pgoff_t vm_pgoff,
                                struct mempolicy *policy)
{
    // 条件：
    // 1. 相同的 vm_flags
    // 2. 相同的文件（或都是匿名）
    // 3. 连续的文件偏移
    // 4. 相同的 NUMA 策略
    // 5. 无特殊属性阻止合并
    
    if (vma->vm_flags != vm_flags)
        return false;
    if (vma->vm_file != file)
        return false;
    if (vma->vm_pgoff + vma_pages(vma) != vm_pgoff)
        return false;
    if (!vma_policy_equal(vma, policy))
        return false;
    
    return true;
}
```

### 3.4 VMA 拆分

```c
// mm/mmap.c
// 在指定地址拆分 VMA

int __split_vma(struct mm_struct *mm, struct vm_area_struct *vma,
                unsigned long addr, int new_below)
{
    struct vm_area_struct *new;
    int err;
    
    // 分配新 VMA
    new = vm_area_dup(vma);
    if (!new)
        return -ENOMEM;
    
    if (new_below) {
        // 新 VMA 在下面
        new->vm_end = addr;
        vma->vm_start = addr;
        vma->vm_pgoff += ((addr - new->vm_start) >> PAGE_SHIFT);
    } else {
        // 新 VMA 在上面
        new->vm_start = addr;
        new->vm_pgoff += ((addr - vma->vm_start) >> PAGE_SHIFT);
        vma->vm_end = addr;
    }
    
    // 插入新 VMA
    err = vma_link(mm, new);
    if (err) {
        vm_area_free(new);
        return err;
    }
    
    return 0;
}

// 使用场景示例：
// 原 VMA: [0x1000, 0x5000)
// 调用 split_vma(mm, vma, 0x3000, 1)
// 结果：
//   VMA 1: [0x1000, 0x3000)  (new_below)
//   VMA 2: [0x3000, 0x5000)  (原 VMA)
```

### 3.5 删除 VMA

```c
// mm/mmap.c
// munmap 删除 VMA

int do_munmap(struct mm_struct *mm, unsigned long start, size_t len,
              struct list_head *uf)
{
    struct vm_area_struct *vma;
    unsigned long end = start + len;
    
    // 查找受影响的 VMA
    vma = find_vma(mm, start);
    if (!vma)
        return 0;
    
    // 可能需要拆分 VMA
    if (start > vma->vm_start) {
        int error = __split_vma(mm, vma, start, 0);
        if (error)
            return error;
        vma = vma->vm_next;
    }
    
    if (end < vma->vm_end) {
        int error = __split_vma(mm, vma, end, 1);
        if (error)
            return error;
    }
    
    // 解除映射
    detach_vmas_to_be_unmapped(mm, vma, prev, end);
    unmap_region(mm, vma, prev, start, end);
    
    // 释放 VMA
    remove_vma_list(mm, vma);
    
    return 0;
}
```

---

## 四、VMA 操作函数

### 4.1 vm_operations_struct

```c
// include/linux/mm.h
struct vm_operations_struct {
    // VMA 打开时调用（如 fork 复制）
    void (*open)(struct vm_area_struct *area);
    
    // VMA 关闭时调用
    void (*close)(struct vm_area_struct *area);
    
    // 拆分前检查
    int (*may_split)(struct vm_area_struct *area, unsigned long addr);
    
    // 复制时调用
    int (*mremap)(struct vm_area_struct *area);
    
    // 缺页异常处理
    vm_fault_t (*fault)(struct vm_fault *vmf);
    
    // 大页缺页
    vm_fault_t (*huge_fault)(struct vm_fault *vmf, enum page_entry_size pe_size);
    
    // 批量预映射
    vm_fault_t (*map_pages)(struct vm_fault *vmf,
                           pgoff_t start_pgoff, pgoff_t end_pgoff);
    
    // 页面变为脏页时调用
    int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
    
    // PFN 映射变为脏页
    int (*pfn_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
    
    // 内存访问（用于 /proc/pid/mem）
    int (*access)(struct vm_area_struct *vma, unsigned long addr,
                  void *buf, int len, int write);
    
    // 获取页面名称（用于 /proc/pid/maps）
    const char *(*name)(struct vm_area_struct *vma);
    
    // NUMA 策略
    int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
    struct mempolicy *(*get_policy)(struct vm_area_struct *vma, unsigned long addr);
};
```

### 4.2 常见 vm_ops 实现

```c
// 文件映射的 vm_ops
// fs/ext4/file.c
const struct vm_operations_struct ext4_file_vm_ops = {
    .fault          = ext4_filemap_fault,
    .map_pages      = filemap_map_pages,
    .page_mkwrite   = ext4_page_mkwrite,
};

// 共享内存的 vm_ops
// mm/shmem.c
static const struct vm_operations_struct shmem_vm_ops = {
    .fault          = shmem_fault,
    .map_pages      = filemap_map_pages,
};

// 匿名映射没有 vm_ops（或使用默认处理）
```

---

## 五、匿名 VMA 管理

### 5.1 anon_vma 结构

```c
// include/linux/rmap.h
struct anon_vma {
    struct anon_vma *root;          // 根 anon_vma
    struct rw_semaphore rwsem;      // 读写信号量
    atomic_t refcount;              // 引用计数
    unsigned degree;                // 合并的 VMA 数
    struct anon_vma *parent;        // 父 anon_vma
    struct rb_root_cached rb_root;  // anon_vma_chain 红黑树
};

struct anon_vma_chain {
    struct vm_area_struct *vma;     // 关联的 VMA
    struct anon_vma *anon_vma;      // 关联的 anon_vma
    struct list_head same_vma;      // VMA 的链表
    struct rb_node rb;              // anon_vma 的红黑树
    unsigned long rb_subtree_last;
};
```

### 5.2 anon_vma 的作用

```
anon_vma 用于反向映射（rmap），找到映射某个物理页的所有 PTE

场景：fork 后的写时复制

父进程:                子进程:
VMA_parent             VMA_child
    │                      │
    └──► anon_vma ◄────────┘
            │
            ▼
        物理页 P

当需要回收页面 P 时：
1. 通过 page->mapping 找到 anon_vma
2. 遍历 anon_vma 的所有 VMA
3. 在每个 VMA 中找到对应的 PTE
4. 解除所有映射
```

---

## 六、VMA 与 /proc 接口

### 6.1 /proc/pid/maps

```bash
$ cat /proc/self/maps
# 地址范围          权限 偏移     设备  inode  路径
00400000-00452000 r--p 00000000 fd:01 123456 /bin/cat
00452000-00480000 r-xp 00052000 fd:01 123456 /bin/cat
00480000-00498000 r--p 00080000 fd:01 123456 /bin/cat
00498000-0049a000 r--p 00097000 fd:01 123456 /bin/cat
0049a000-0049b000 rw-p 00099000 fd:01 123456 /bin/cat
01a00000-01a21000 rw-p 00000000 00:00 0      [heap]
7f8a10000000-7f8a10021000 rw-p 00000000 00:00 0
7f8a20000000-7f8a201c8000 r--p 00000000 fd:01 234567 /lib/x86_64-linux-gnu/libc.so.6
7ffc8a500000-7ffc8a521000 rw-p 00000000 00:00 0      [stack]
7ffc8a5fe000-7ffc8a602000 r--p 00000000 00:00 0      [vvar]
7ffc8a602000-7ffc8a604000 r-xp 00000000 00:00 0      [vdso]

# 权限说明：
# r = 可读 (VM_READ)
# w = 可写 (VM_WRITE)
# x = 可执行 (VM_EXEC)
# p = 私有 (非 VM_SHARED)
# s = 共享 (VM_SHARED)
```

### 6.2 /proc/pid/smaps

```bash
$ cat /proc/self/smaps
00400000-00452000 r--p 00000000 fd:01 123456 /bin/cat
Size:                328 kB      # VMA 大小
KernelPageSize:        4 kB      # 内核页大小
MMUPageSize:           4 kB      # MMU 页大小
Rss:                 328 kB      # 驻留内存
Pss:                 164 kB      # 比例共享内存
Shared_Clean:        328 kB      # 共享的干净页
Shared_Dirty:          0 kB      # 共享的脏页
Private_Clean:         0 kB      # 私有的干净页
Private_Dirty:         0 kB      # 私有的脏页
Referenced:          328 kB      # 被引用的页
Anonymous:             0 kB      # 匿名页
LazyFree:              0 kB      # 延迟释放页
AnonHugePages:         0 kB      # 匿名大页
ShmemPmdMapped:        0 kB      # 共享内存 PMD 映射
FilePmdMapped:         0 kB      # 文件 PMD 映射
Shared_Hugetlb:        0 kB      # 共享 Hugetlb
Private_Hugetlb:       0 kB      # 私有 Hugetlb
Swap:                  0 kB      # 换出大小
SwapPss:               0 kB      # 比例换出大小
Locked:                0 kB      # 锁定大小
VmFlags: rd mr mw me sd          # VMA 标志
```

---

## 总结

### 核心概念

1. **VMA**：进程地址空间的基本管理单元
2. **vm_flags**：控制 VMA 的权限和行为
3. **vm_ops**：VMA 的操作函数集
4. **anon_vma**：匿名映射的反向映射支持
5. **Maple Tree**：新的 VMA 组织数据结构

### 关键操作

| 操作 | 函数 | 说明 |
|-----|------|------|
| 查找 | find_vma() | 查找包含地址的 VMA |
| 创建 | mmap_region() | 创建新的 VMA |
| 合并 | vma_merge() | 合并相邻的兼容 VMA |
| 拆分 | __split_vma() | 在指定地址拆分 VMA |
| 删除 | do_munmap() | 删除 VMA |

### 后续学习

- [缺页异常处理机制](10-缺页异常处理机制.md) - 了解 VMA 在缺页处理中的作用
- [文件映射与匿名映射](11-文件映射与匿名映射.md) - 了解不同类型的 VMA

## 参考资源

- 内核源码：
  - `include/linux/mm_types.h` - VMA 定义
  - `mm/mmap.c` - VMA 操作实现
- 内核文档：`Documentation/mm/`

## 更新记录

- 2026-01-28：初始创建，包含 VMA 管理机制详解
