# 内存压缩（Memory Compression）深度解析：swap 与压缩到底什么关系？

## 📋 文档概述

本文系统介绍 Linux 中的**内存压缩机制**，重点回答几个经常被混在一起、但本质不同的问题：

- 内核里的“内存压缩”究竟指什么？
- zram、zswap 分别是什么，它们各自处在什么层？
- 它们与传统 swap 分区 / swap 文件到底是什么关系？
- 在 ANR / 高 iowait / 高内存压力场景下，应该如何解读“压缩 + swap”这条链路？

阅读本文之前，建议你已经熟悉：

- `File_vs_Anonymous_Pages_Deep_Dive.md` 中的**文件页 / 匿名页**与 swap 基本机制
- `kswapd_Deep_Dive.md` 中的**内存回收流水线**（kswapd / direct reclaim）
- `Zone_Deep_Dive.md` 中的**水位线与页面回收触发条件**

---

## 第一章：内存压缩概览（What & Why）

### 1.1 内存压缩到底是什么？

从**本质**上讲，所谓“内存压缩（memory compression）”可以概括为一句话：

> 在把页面写到**更慢、更贵的介质**（通常是磁盘 / 闪存上的 swap）之前，先在**更快的介质**上以**压缩形式**保存，从而用 CPU 的压缩/解压开销，换取“更少的 I/O、更大的有效容量”。

注意几点：

- **它并不是虚拟内存机制本身**，而是在虚拟内存 + swap 这条通路上的**一层优化**。
- **它不改变匿名页必须通过 swap 才能被真正回收的这个事实**，只是改变：
  - 这些匿名页**最终落在什么介质上**（磁盘、闪存还是压缩内存块设备）；
  - 以及在这条路径里**有没有一个“压缩缓存层”**（zswap）。

### 1.2 为什么需要内存压缩？

如果只用传统 swap（机械盘 / SSD / eMMC），在内存紧张时会遇到几个老问题：

1. **I/O 延迟高**
   - 访问磁盘的延迟远高于内存：纳秒 vs 微秒/毫秒。
   - 大量匿名页 swap out / swap in 时，系统响应会明显变慢。

2. **闪存写放大与寿命问题**
   - 在移动设备（eMMC/UFS）上，频繁 swap 会增加写放大。
   - 闪存有写入寿命限制，过多 swap 会加速磨损。

3. **内存有限但 workload 爆发**
   - 当物理内存本身不大，应用/系统想容纳更多匿名页时，单靠未压缩的 swap 成本很高。

因此，Linux 引入了 zram、zswap 等机制，通过：

- **在内存中以压缩形式保存更多匿名页**（以空间换时间）；
- **减少真实块设备上的 I/O 量**（以 CPU 换 I/O）；

来缓解上述矛盾。

### 1.3 与传统 swap 的对比视角

先用一个非常粗略但很好记的对比：

- **没有压缩**时：

  > 匿名页 → 被回收 → 直接写入 swap 分区/文件（真实块设备） → 以后访问再从该设备读回。

- **有压缩**时（视具体配置而定）：

  > 匿名页 → 被回收 →  
  > 优先写入“压缩后介质”（zram 或 zswap pool） →  
  > 只有在压缩池不足 / 淘汰时，才进一步落到真实 swap 设备。

所以，从“swap 与压缩”的关系看：

- **swap 是逻辑概念：匿名页被换出到某个后端存储；**
- **压缩是 swap 通路上的实现策略：让这部分数据以更小体积、更低 I/O 成本的形式存在。**

---

## 第二章：基础回顾——匿名页与 swap 工作流

这一章只做**极简回顾**，重点是为“压缩插入在哪里”打地基。

### 2.1 匿名页为什么必须依赖 swap 才能回收？

结合 `File_vs_Anonymous_Pages_Deep_Dive.md` 中的结论：

- **文件页（file-backed）**：
  - 有磁盘/文件后备；
  - 脏页写回、干净页直接丢弃；
  - 回收不一定需要 swap。

- **匿名页（anonymous）**：
  - 没有文件后备；
  - 内容只存在内存中；
  - 要想释放物理页，必须先在某处**保存其内容**——这就是 swap 的角色。

因此，对匿名页的回收路径本质是：

> “先找个地方存起来（swap），然后才能把物理页让出来”。

### 2.2 不引入压缩时的经典 swap 流水线

简化一下 kswapd / direct reclaim 对匿名页的处理流程（不考虑压缩）：

```text
内存压力（zone 水位线低于 low）
    ↓
kswapd / direct reclaim 开始回收
    ↓
扫描 LRU：挑选 inactive_anon 页面
    ↓
为每个页面分配 swap 槽（swap slot）
    ↓
把页面内容写入 swap 设备（分区/文件）
    ↓
更新页表：标记该页在 swap 中，不在内存
    ↓
释放物理页面（page frame）
```

之后，当进程再次访问该虚拟地址时：

```text
访问该虚拟地址
    ↓
发生缺页异常（page fault）
    ↓
页表发现：这是一个 swap entry（在 swap 中）
    ↓
从 swap 设备读取该页内容
    ↓
分配新的物理页面并填充
    ↓
更新页表，重新映射到物理页
```

这个过程里，**swap 设备**可以是：

- 传统磁盘分区（/dev/sdX）；
- swap 文件（/swapfile）；
- 以及后文要说到的 **zram 块设备**。

### 2.3 关键点小结

1. swap 是**匿名页**回收的**唯一后备存储**（除非直接杀进程 / 丢弃数据）。
2. kswapd / direct reclaim 只是负责**挑页 + 发起写出**，真正“写到哪里”由 swap 子系统和底层块设备决定。
3. 引入“内存压缩”时，改变的是：
   - **swap entry 在内核中的保存形式**（压缩 vs 未压缩）；
   - **后端设备的类型和行为**（zram 内存块设备 / 真实磁盘）。

---

## 第三章：Linux 中两类主流内存压缩机制

Linux 里常见的**通用内存压缩**方案主要有两类：

- **zram**：压缩块设备，最常见用法是“把它当作一个 swap 盘”；
- **zswap**：swap 子系统前面的压缩缓存（“swap 上再加一层 cache”）。

它们解决的问题相似，但层次、使用方式、实现细节都不同。

### 3.1 zram：压缩块设备

#### 3.1.1 基本概念

**zram** 是一个内核模块，提供一个**基于内存的压缩块设备**：

- 从上层看，它就是一个普通块设备，比如 `/dev/zram0`；
- 内核会在这个设备上按块存储数据，但每个块的内容在写入时会被**压缩**；
- 实际占用的是**物理内存**，不是磁盘；
- 常见用法：把 `/dev/zram0` 格式化成 swap：

```bash
echo 4G > /sys/block/zram0/disksize   # 把 zram 逻辑大小设为 4G
mkswap /dev/zram0
swapon /dev/zram0
```

此时，对 swap 子系统来说：

- `/dev/zram0` 就是一块“4G 的 swap 盘”；
- 但在物理上，这 4G 的数据是**压缩后存放在内存中**，实际占用可能远小于 4G。

#### 3.1.2 所在层次

从内核架构来看，zram 在**块设备层**：

```text
匿名页回收逻辑 (mm/vmscan.c, mm/swapfile.c)
        ↓
swap 子系统（分配 swap entry，发起 I/O）
        ↓
通用块 I/O 层 (block layer)
        ↓
块设备驱动：
    - 传统磁盘 / eMMC / SSD
    - zram（压缩内存盘）
```

对上层 swap 来说：

- zram 与普通磁盘 swap 分区**接口一致**，都是“一个块设备”；
- 区别在于：**zram 在驱动内部把要写入的页先压缩，再存入内存池**。

#### 3.1.3 数据流示意

以“匿名页被换出到 zram swap”为例：

```text
kswapd / direct reclaim 选中 inactive_anon 页面
    ↓
分配 swap entry，目标设备 = /dev/zram0
    ↓
swap write → block I/O → zram 驱动
    ↓
zram 压缩该页内容，并存在内存后备存储里
    ↓
释放原物理页面
```

之后 swap in 时：

```text
page fault → 发现 swap entry 指向 /dev/zram0
    ↓
从 zram 读取压缩数据 → 解压 → 填充到新页面
    ↓
更新页表，映射到新物理页
```

**关键认识：**

- 使用 zram 做 swap，本质上还是在用 swap，只是**后端介质从“磁盘/闪存”变成了“压缩内存块设备”**。
- I/O 延迟从“设备 I/O”变成“内存访问 + 压缩/解压 CPU 开销”。

### 3.2 zswap：swap 前端的压缩 cache

#### 3.2.1 基本概念

**zswap** 是另一个内核组件，被称为 “compressed swap cache”：

- 它不是一个块设备，而是 swap 子系统中的一个**前端 cache 层**；
- 当匿名页要被换出（swap out）时：
  - **优先被压缩后放入 zswap 内存池**；
  - 只有在 zswap 池满了 / 命中淘汰策略时，才会被写入“真实 swap 设备”（后端）。

也就是说，zswap 把“原本要写去磁盘的 swap 内容”，先在内存里以压缩形式缓存一份：

- 命中 zswap：读写都在内存中完成（存在 zswap pool 里）；
- 未命中 / 淘汰：才真正访问后端 swap（可能是磁盘、也可能是 zram）。

#### 3.2.2 所在层次

按逻辑层次看，zswap 在 **mm/swap 层内部**：

```text
匿名页回收逻辑 (mm/vmscan.c)
        ↓
swap 子系统 (mm/swapfile.c, mm/zswap.c)
        ↓
    1) 先尝试放入 zswap 内存池（压缩）
        ↓ miss 或淘汰
    2) 再写入后端 swap 设备（磁盘 / zram / 其它）
        ↓
块 I/O 层 + 具体块设备驱动
```

因此：

- zswap 是“逻辑层上的增强”，不会出现在 `/dev` 下面；
- 它始终需要一个**后端 swap 设备**作为兜底：
  - 常见组合：zswap + 传统磁盘 swap；
  - 也可以是：zswap + zram（较少见但可以配置）。

#### 3.2.3 数据流示意

以 “zswap + 磁盘 swap” 为例：

```text
kswapd / direct reclaim 选中 inactive_anon 页面
    ↓
swap 子系统准备换出该页
    ↓
zswap 压缩该页，并尝试放入 zswap pool
    ↓
如果成功：
    - 只在内存中记录一个压缩后的对象
    - swap entry 指向 zswap 元数据
    - 不发生后端 I/O

如果 zswap pool 已满 / 策略要求淘汰：
    - 压缩数据被写入后端 swap 设备
    - 或者直接把原始页写到后端
```

swap in 时：

- 如果 swap entry 命中 zswap：

  ```text
  page fault → 发现该 entry 在 zswap pool 里
      ↓
  从 zswap pool 取出压缩数据 → 解压 → 填充到新页面
      ↓
  更新页表
  ```

- 如果不在 zswap（已淘汰去了后端）：

  ```text
  page fault → entry 指向后端 swap 设备
      ↓
  走传统 swap in 流程，从设备读取 → 填充 → 更新页表
  ```

#### 3.2.4 小结：zram vs zswap

可以用一句话区分：

- **zram 更接近“压缩的 swap 设备”（block device）；**
- **zswap 更像“swap 上的一层内存压缩缓存”（前端 cache）。**

从“swap 与压缩”的关系看：

- zram 是把 “swap 的后端介质” 从物理盘替换为“压缩内存盘”；
- zswap 是在“swap 逻辑通路前面”再加一层压缩缓存，减少真正落到后端 swap 的数据量。

---

## 第四章：swap 与压缩的关系——分层视角

这一章是**整篇文章的核心**：从分层视角回答“swap 和压缩是什么关系”。

### 4.1 三层逻辑视图

从高到低，把相关机制拆成三层：

```text
L1：匿名页回收逻辑
    - kswapd / direct reclaim
    - 负责：选页、决定回收多少

L2：swap 子系统
    - 管理 swap entry 的分配与索引
    - 决定：某个被换出的页，逻辑上“存到 swap 的哪一块”
    - 在此层可以挂接 zswap（前端压缩 cache）

L3：具体 swap 后端设备
    - 传统 swap 分区 / swap 文件
    - zram 块设备（压缩内存盘）
    - 其它自定义后端
```

**压缩插入的位置**：

- zswap：在 **L2 内部**，是逻辑上的压缩缓存；
- zram：在 **L3**，是一种特殊的“基于内存、会压缩”的 swap 设备。

### 4.2 几种典型组合场景

#### 4.2.1 只用传统 swap 分区（无压缩）

配置：

- 启用：`/dev/sdaX` 作为 swap 分区；
- 不启用 zram / zswap。

数据流：

```text
匿名页 → 回收逻辑 → swap 子系统
    → 直接分配 swap entry，后端 = /dev/sdaX
    → 写磁盘 I/O
```

特点：

- 实现简单、行为易理解；
- 当内存压力大、匿名页大量换出时，**磁盘 I/O 与 iowait 可能飙升**；
- 在 eMMC/SSD 上可能带来明显的性能抖动和写放大。

#### 4.2.2 只用 zram 做 swap（无磁盘 swap）

配置：

- `mkswap /dev/zram0 && swapon /dev/zram0`；
- 不再启用传统磁盘/闪存 swap。

数据流：

```text
匿名页 → 回收逻辑 → swap 子系统
    → swap entry 的后端设备 = /dev/zram0
    → I/O 实际发给 zram，写入时被压缩后放在内存池中
```

特点：

- **所有被换出的匿名页，都最终以压缩形式存在内存里**；
- 几乎没有真实块设备 I/O，iowait 对应的 swap I/O 大幅下降；
- 代价：
  - zram 内存池本身会占据一部分物理内存（逻辑大小通常大于物理占用）；
  - 压缩/解压有 CPU 开销；
  - 如果 zram 配得过大，可能挤占 page cache，反而影响文件 I/O 性能。

这是一种**常见的移动设备/嵌入式方案**：没有或尽量少用“慢且易磨损”的外部存储 swap，而是把内存“压缩再利用”。

#### 4.2.3 zswap + 磁盘 swap

配置：

- 启用 zswap；
- 后端 swap 设备为传统磁盘分区/文件。

数据流：

```text
匿名页 → 回收逻辑 → swap 子系统
    → 先尝试放入 zswap pool（压缩存内存）
        → 如果成功：swap entry 仅指向 zswap 元数据，不发生磁盘 I/O
        → 如果失败/淘汰：再将页（压缩或未压缩）写入后端磁盘 swap
```

特点：

- 在内存仍略微有余的情况下，大部分匿名页会以“压缩形式”留在 zswap pool 中；
- 只在压力进一步增大时，才会真正写入磁盘；
- 典型用于**服务器/桌面**：减少磁盘 swap 量，提高响应速度。

#### 4.2.4 zswap + zram / 其它后端

从设计上看：

- zswap 后端可以是任何合法的 swap 设备，包括 zram；
- 于是形成链路：

```text
匿名页 → zswap（内存压缩缓存）
    → 淘汰时 → 后端 zram（压缩块设备）
        → 仍然在内存里，但以“二次压缩形式”存在
```

这种组合在通用发行版里不多见，但在某些定制系统上可能出现。核心认识还是：

- **zswap 负责“第一层缓存”；**
- **后端（无论是磁盘还是 zram）负责“最终落地”。**

### 4.3 关键结论：压缩是 swap 通路中的一层优化

现在可以回到最关键的那个问题：

> swap 和内存压缩到底是什么关系？

可以用几点来“钉死”这个概念：

1. **swap 是逻辑抽象，压缩是实现细节**
   - swap 的职责：为匿名页提供后备存储，使其可以被回收；
   - 是否压缩、如何压缩，是“这块后备存储如何高效实现”的问题。

2. **压缩不会改变“匿名页必须经由 swap 才能回收”的事实**
   - 有无 zram/zswap，匿名页想释放物理页，最终都要“挂接一个 swap entry”；
   - 不同的是：这个 entry 背后，指向的是：
     - 未压缩的磁盘块；
     - 还是 zram/zswap 中的压缩对象。

3. **压缩可以插在 swap 通路的不同位置**
   - zram：把 swap 的后端设备变成“压缩内存盘”；
   - zswap：在写入后端 swap 前，先在内存里压缩缓存一次；
   - 它们可以单独使用，也可以叠加。

4. **从性能角度看，是在做“三角平衡”：CPU ↔ 内存 ↔ I/O**
   - 通过增加 CPU 压缩/解压负担 + 占用部分内存，来换取：
     - 更少的真实块设备 I/O；
     - 更大的“可容纳匿名页的总量”。

后续章节会在此基础上继续展开：

- 第五章：从内核源码与伪代码角度，再看一次“swap + 压缩”的实现分支；
- 第六章、第七章：讨论性能与调优参数（CPU/内存/I/O 三者的权衡）；
- 第八章、第九章：结合 ANR / iowait 案例，讲怎么在实战中正确解读这些机制。

---

## 第五章：内核实现视角（简化说明）

> 本章只做“高层源码地图 + 简化伪代码”，方便你在 `mm/` 和 `drivers/block/` 中按图索骥。

### 5.1 相关源码模块定位

- **swap 管理核心**
  - `mm/swapfile.c`：swap 设备的注册、启用、swap entry 管理。
  - `mm/swap_state.c`：swap cache、swapin/swapout 相关状态处理。

- **匿名页回收入口**
  - `mm/vmscan.c`：kswapd / direct reclaim 的主逻辑。

- **zram**
  - `drivers/block/zram/`：zram 块设备驱动，实现压缩存储。

- **zswap**
  - `mm/zswap.c`：zswap pool 管理、压缩/解压逻辑、与 swap entry 交互。

### 5.2 简化伪代码：匿名页被换出时的分支

以下为高度简化、仅用于帮助理解的伪代码：

```c
// vmscan.c 中，回收匿名页时触发 swap out
void shrink_page_list(struct list_head *page_list)
{
    list_for_each_page(page, page_list) {
        if (PageAnon(page)) {
            try_to_unmap(page);
            swap_out(page);   // 关键入口
        } else {
            // 文件页的回收路径，见 File_vs_Anonymous_Pages_Deep_Dive.md
        }
    }
}

// swap 子系统中的换出逻辑（伪代码）
int swap_out(struct page *page)
{
    swp_entry_t entry = get_swap_page(); // 分配 swap entry
    if (!entry)
        return -ENOSPC;

    if (zswap_enabled) {
        if (zswap_store(entry, page) == ZSWAP_STORED)
            goto out_free_page;    // 已经存入 zswap，不必立刻写后端
    }

    // 如果 zswap 失败 / 未开启，则写入后端 swap 设备
    swap_writepage(page, entry);

out_free_page:
    free_page(page);
    return 0;
}
```

对应关系：

- `get_swap_page()`：在 swap 子系统中为该页分配一个逻辑“槽位”；
- `zswap_store()`：如果启用了 zswap，尝试把该页的内容压缩后放入 zswap pool；
- `swap_writepage()`：
  - 如果没有 zswap 或 zswap 存储失败；
  - 则通过块 I/O 层，把数据写到后端 swap 设备（可能是磁盘，也可能是 zram）。

### 5.3 swap entry 与 zswap/zram 的关系（高层）

在不考虑实现细节的前提下，可以这样理解：

- **swap entry 是逻辑句柄**：
  - 用来唯一标识“某个被换出的页面”；
  - 不关心具体是压缩还是未压缩、在哪个物理介质上。

- 当启用 zswap 时：
  - entry 可能指向一块 zswap pool 中的压缩对象；
  - 也可能已经被写入后端 swap 设备；
  - swap 子系统负责在这两者间转换、淘汰、迁移。

- 当使用 zram 作为后端时：
  - entry 逻辑上仍然是“这个页在某个 swap 逻辑地址”；
  - 只是这个地址最终落在 `/dev/zram0` 的某个逻辑块上；
  - 而该块在 zram 驱动内部又被映射到“内存 + 压缩算法管理的对象”。

---

## 第六章：性能与权衡——CPU、内存、I/O 的三角平衡

### 6.1 内存压缩的主要收益

1. **减少真实块设备 I/O**
   - 对“慢设备”（机械盘、eMMC、某些 UFS 型号）尤为关键；
   - 对 ANR / 卡顿中常见的 iowait 升高有明显帮助。

2. **提高“可承载匿名页”的总量**
   - 相同物理内存下，压缩后可容纳更多匿名页；
   - 对内存紧张、进程数较多的系统（例如手机）非常重要。

3. **平滑内存压力**
   - 在达到 OOM / LMK 之前，多了一层缓冲空间；
   - 可以给系统和调度策略更多回旋余地。

### 6.2 付出的代价

1. **CPU 开销**
   - 压缩/解压算法（如 lzo、lz4、zstd 等）都需要 CPU 时间；
   - 在 CPU 已经接近瓶颈的设备上，滥用压缩可能适得其反。

2. **内存占用**
   - zram 设备本身占用一个内存池，虽然是“压缩后容量 > 实际物理占用”，但仍然要抢占一部分物理内存；
   - zswap pool 也会消耗内存，需要设置上限（如 max_pool_percent）。

3. **可能挤压 Page Cache**
   - 如果 zram/zswap 占用过多内存，Page Cache 空间被压缩挤出；
   - 导致文件 I/O 需要更频繁地访问底层文件系统与块设备，反而**增加** iowait。

### 6.3 极端/失败场景示例

1. **“CPU 被压缩打满”**
   - 内存极度紧张，系统不停在“压缩 → 解压 → 再压缩”的循环中；
   - CPU 使用率高、loadavg 飙升，但真实进程并没有获得更多可用内存；
   - ANR 中可能看到 CPU 满、iowait 中等、PSI memory/IO 各有表现。

2. **“zram 过大 + 强挤 Page Cache”**
   - zram 配置为物理内存的很大比例（例如 75%）；
   - 匿名页是“压缩后非常省空间”，但文件 I/O 严重依赖 Page Cache；
   - 结果：匿名页“很好活着”，但应用频繁读写文件，被迫走真实块 I/O，整体体验却变差。

3. **zswap 命中率偏低**
   - 工作负载中匿名页被访问模式高度随机，zswap pool 很快就被冷数据占满并频繁淘汰；
   - 实际上大多数 swap in 请求都落在后端设备上；
   - 这时 zswap 的收益有限，但 CPU 仍然付出了压缩/解压的成本。

---

## 第七章：参数、配置与调优思路（高层）

本章只给出调优的方向和核心参数，具体数值需要结合设备和 workload 实验。

### 7.1 zram 关键配置

常见接口（以 zram0 为例）：

```bash
# 设置逻辑磁盘大小（通常设置为物理内存的一定倍数）
echo 4G > /sys/block/zram0/disksize

# 查看或设置压缩算法
cat /sys/block/zram0/comp_algorithm
echo lz4 > /sys/block/zram0/comp_algorithm
```

一般思路：

- **容量大小**：
  - 移动设备常见经验值：物理内存的 25%～50% 左右；
  - 过小：压缩空间不足，频繁回落到其它手段（杀进程 / OOM）；
  - 过大：挤压 page cache 和其他内存用途。

- **压缩算法选择**：
  - lz4：压缩率一般，但速度快、对 CPU 友好；
  - zstd：压缩率好，但 CPU 开销更大；
  - 在手机/嵌入式上，常优先考虑“解压/压缩速度”而非极致压缩率。

### 7.2 zswap 关键配置

常见 sysfs 参数（以模块参数为例）：

- `/sys/module/zswap/parameters/enabled`
- `/sys/module/zswap/parameters/max_pool_percent`
- `/sys/module/zswap/parameters/compressor`
- `/sys/module/zswap/parameters/zpool`

高层含义：

- `enabled`：是否启用 zswap；
- `max_pool_percent`：zswap pool 最大可占系统内存的百分比；
- `compressor`：使用哪种压缩算法（同样是速度 vs 压缩率的 trade-off）；
- `zpool`：底层内存分配器类型。

基本调优思路：

- 如果 I/O 成本高、CPU 相对富余 → 可适当提高 `max_pool_percent`，提高命中率；
- 如果 CPU 常年高负载 → 谨慎使用 zswap 或选用非常轻量的压缩算法；
- 实战中要结合 `/sys/kernel/debug/zswap` 相关统计（如 stored pages、pool size、reject count）来判断收益。

### 7.3 与 `vm.swappiness`、kswapd 行为的联动

- `vm.swappiness`（0～100）：
  - 越高，内核越倾向于使用 swap（包括压缩 swap）来回收匿名页；
  - 越低，更多倾向于回收文件页 / 缓存。

- 当引入 zram / zswap 后：
  - 较高的 swappiness 会让系统更愿意“把匿名页挪进压缩池”，以换取更多可用物理内存；
  - 但如果压缩本身已经成为 CPU 瓶颈，则需要降低 swappiness，减轻匿名页换出的节奏。

- kswapd 视角：
  - kswapd 并不知道“后面是不是压缩设备”，它只关心 page reclaim 成功与否；
  - 但压缩成功率、zswap 命中率会间接影响：
    - kswapd 需要跑多快、多久；
    - 以及 direct reclaim / OOM 是否更容易被触发。

---

## 第八章：在 ANR / 高 iowait 场景下如何解读“压缩 + swap”

### 8.1 与现有 ANR 分析经验的结合

结合你在 `ANR_Case_IO_Wait_High_Analysis.md` 等文档中的经验，可以归纳几类典型情况：

1. **iowait 很高，但 pswpin/pswpout 并不夸张**
   - 多数 I/O 来自普通文件读写、F2FS GC、dm-verity 校验、blk_crypto 等路径；
   - 即使启用了 zram/zswap，swap 本身对 iowait 的贡献可能很有限；
   - 此时“内存压缩”更多影响的是可用内存和 OOM 风险，而不是 iowait 核心来源。

2. **pswpin/pswpout 很高，同时 IO PSI 较高**
   - 说明 swap I/O 是阻塞任务的重要来源；
   - 如果后端是磁盘/闪存，swap I/O 会直接体现为较高 iowait；
   - 若后端为 zram，则 pswpin/pswpout 高不一定导致 iowait 高，但会推高 CPU、可能影响调度延迟。

3. **Memory PSI 不高，但 kswapd 活跃、IO PSI 高**
   - 可能是：
     - kswapd 在后台持续做压缩/回收，但大部分工作并没有直接阻塞前台任务（PSI 只统计“任务被阻塞时间”）；
     - I/O 压力主要集中在文件系统/块设备链路（而非 swap），例如 F2FS 的 GC/写放大；
   - 需要同时看：
     - `/proc/vmstat` 中的 pswpin/pswpout；
     - zram/zswap 统计；
     - `/proc/pressure/memory` 与 `/proc/pressure/io` 的 some/full 指标。

### 8.2 实战判断方法（示例思路）

在一个现场 ANR / 卡顿场景中，可以按如下步骤判断“压缩 + swap”的角色：

1. **查看 swap 使用及 I/O**
   - `/proc/meminfo` 中的 `SwapTotal / SwapFree`；
   - `/proc/vmstat` 中的 `pswpin / pswpout`；
   - `iostat -x` 中是否有明显 swap 设备 I/O（例如磁盘/闪存上的 swap 分区）。

2. **查看 zram / zswap 统计**
   - zram：
     - `/sys/block/zram0/mm_stat`（包括压缩率、orig_data_size、compr_data_size 等）；
   - zswap：
     - `/sys/kernel/debug/zswap`（或对应统计接口），查看 stored pages、pool size、rejects。

3. **对照 PSI 和 kswapd 活动**
   - `/proc/pressure/memory`：Memory PSI 是否明显升高；
   - `/proc/pressure/io`：IO PSI 是否明显升高；
   - `/proc/vmstat` 中的 `pgscan_anon / pgsteal_anon / kswapd_*` 等指标，看 kswapd 是否在剧烈工作。

粗略判断：

- 如果：
  - pswpin/pswpout 高；
  - swap 后端是磁盘/闪存；
  - IO PSI、iowait 都明显升高；
  - 则 **swap I/O 很可能是 iowait 的重要组成部分，此时引入/加强压缩可能有帮助**。

- 如果：
  - pswpin/pswpout 不高；
  - 但 IO PSI、F2FS 相关 kworker、dm-verity、blk_crypto 等线程非常活跃；
  - 则 **问题核心更偏向“普通文件 I/O 路径”，压缩对 swap 的优化影响有限**。

### 8.3 小结：压缩既可能是“救火队”，也可能是“新瓶颈”

在 ANR / 性能分析中，关于“开不开压缩”有几个易踩的坑：

1. **不能简单认为：开了 zram/zswap 就一定减少 iowait**
   - 如果原本 swap I/O 就不重，真实瓶颈在文件系统 / F2FS / dm-verity；
   - 那么压缩对 iowait 的边际收益会很小。

2. **也不能简单认为：压缩一定提升整体性能**
   - 在 CPU 已经很紧的场景下，过多压缩可能推高 CPU 使用，增加调度延迟；
   - 在 Page Cache 需求很高的场景下，过大的 zram/zswap pool 可能抢走缓存空间，导致 I/O 更频繁直达设备。

3. **正确姿势：用数据说话**
   - 通过多维指标（PSI、pswpin/pswpout、zram/zswap 统计、块设备 I/O 指标、kworker/kswapd 行为）一起看；
   - 明确区分：哪些是**已知事实（从日志、统计中读到的）**，哪些只是**推测**；
   - 在不同 workload 上做 A/B 对比实验，而不是只靠“理论上”判断压缩效果。

---

## 第九章：实践建议与常见误区

### 9.1 不同设备类型下的策略差异（高层）

1. **服务器**
   - 大内存 + 相对较快的存储；
   - 常见方案：**zswap + 磁盘 swap**；
   - 目标：在内存压力瞬时升高时，用 zswap 缓冲，减少真正落盘的 swap I/O。

2. **桌面/笔记本**
   - 可能有较快 SSD，也可能接入机械盘；
   - 方案灵活：可用 zswap、也可用 zram 或传统 swap；
   - 更关注交互体验，避免明显卡顿。

3. **移动设备 / 嵌入式（Android 等）**
   - 内存和闪存都相对有限；
   - 常见方案：**只用 zram 做 swap、不启用磁盘 swap**；
   - 目标：尽量避免对闪存的大量 swap I/O，减轻 iowait 和写放大。

### 9.2 常见误解澄清

1. “开了 zram 就不会再有 swap I/O 了”
   - 只有在**完全没有其它 swap 设备**、且所有 swap entry 都落在 zram 上时，这句话才近似成立；
   - 一旦有其它 swap 后端，或 zram 与之组合使用，就仍然可能产生块设备 I/O。

2. “只要开压缩，性能一定变好”
   - 忽略了 CPU、调度、Page Cache 等多维度因素；
   - 在某些场景里，压缩只是把 I/O 瓶颈“搬家”成 CPU 瓶颈，用户感知不一定更好。

3. “Memory PSI 低就说明内存没问题”
   - PSI 统计的是**任务被阻塞的时间**，而不是“资源消耗的多少”；
   - kswapd 在后台大量工作、zram/zswap 频繁压缩匿名页，但如果没有同步阻塞前台任务，PSI memory 也可能不高；
   - 仍需要结合 pswpin/pswpout、zram/zswap 统计等一起看。

### 9.3 建议的分析/验证方法

1. **观测面要尽量全面**
   - `/proc/meminfo`, `/proc/vmstat`, `/proc/zoneinfo`；
   - `/proc/pressure/memory`, `/proc/pressure/io`；
   - zram/zswap 相关 sysfs / debugfs；
   - 块设备 I/O 指标（iostat、blkio cgroup 统计、fio 压测等）。

2. **结合工具做实验**
   - `perf`, `ftrace`, eBPF，观察压缩相关函数的 CPU 时间；
   - 构造典型场景（内存紧张 + IO 重负载）对比“开启压缩”和“关闭压缩”。

3. **把结论写回知识库**
   - 对特定硬件平台（某款 eMMC、某个 UFS 控制器）；
   - 对特定 workload（视频播放、大图浏览、后台同步）；
   - 把压缩方案的效果沉淀为可复用的配置与排障经验。

---

## 附录：关键接口与命令速查

### A.1 swap 相关

```bash
# 查看内存与 swap 整体使用情况
free -h

# 查看每个 swap 设备
swapon -s
cat /proc/swaps

# 查看 swap 统计
cat /proc/meminfo | grep -i swap
cat /proc/vmstat | grep -E 'pswpin|pswpout'
```

### A.2 zram 相关

```bash
# 查看 zram 设备
ls /sys/block/zram*

# 查看 zram 统计（压缩率、内存占用等）
cat /sys/block/zram0/mm_stat
cat /sys/block/zram0/comp_algorithm
```

### A.3 zswap 相关

```bash
# 查看 zswap 是否开启
cat /sys/module/zswap/parameters/enabled

# 查看 zswap 池大小与统计（不同内核版本路径可能有差异）
grep . /sys/kernel/debug/zswap/*
```

### A.4 PSI 与回收相关

```bash
# 内存压力
cat /proc/pressure/memory

# IO 压力
cat /proc/pressure/io

# 回收与 swap 相关统计
cat /proc/vmstat | grep -E 'pgscan|pgsteal|pswpin|pswpout|kswapd'
```

---

## 附录 B：常见疑问（Q&A）——结合“8G 内存 + 8G swap + zram”的场景

本小节专门整理你在阅读过程中提出的几个关键问题，用更接近“心里怎么想”的方式再解释一遍。

### B.1 “8G 内存 + 8G swap，是不是就等于我有 16G 内存？”

**不是。**

- 8G **物理内存（RAM）**：  
  - 进程运行、Page Cache、内核、zram 内存池等，都只能从这 8G 里分。
- 8G **swap 分区（磁盘上的一块区域）**：  
  - 只是一个“后备存储空间”，用来放被换出的匿名页；  
  - 它让系统在物理内存不足时，有地方暂存一部分页的内容，但访问延迟远高于 RAM。

可以理解为：

> “8G RAM + 8G 盘上的备份空间”，  
> 而不是“8G + 8G = 16G 真·内存条”。

### B.2 `/dev/zram0` 是不是把 8G 物理内存硬切一块出来挂上去？

**资源关系上：zram 用的确实是这 8G 里的部分 RAM；实现方式上：不是一次性硬切，而是按需从同一个内存池中申请页面。**

- 配置 zram 时，你会设置一个逻辑大小，例如：

```bash
echo 2G > /sys/block/zram0/disksize
```

- 这 2G 是 zram 对外呈现的“逻辑块设备容量”，**不是立刻锁死 2G 物理内存**；
- 实际占用的物理内存：
  - 随着你往 `/dev/zram0` 写入数据逐渐增长；
  - 还会受压缩率影响（压缩得好，2G 逻辑容量可能只占几百 MB 实际 RAM）。

所以可以这么记：

- **看上去**：`/dev/zram0` 像是“从 8G 里切出一块，用来当压缩盘”；
- **实际上**：zram 是一个驱动，它按需从同一个 8G 内存池里申请物理页，组织成一个“压缩块设备”。

### B.3 `/dev/zram0` 算不算“磁盘”？它背后没有真实硬件吗？

从两个角度看：

1. **从接口形态/内核层次看**：
   - `/dev/zram0` 是一个**块设备节点**，挂在块 I/O 层下；  
   - 上层可以像对待普通硬盘那样对它：
     - `mkswap /dev/zram0` → 当 swap 盘用；
     - `mkfs.ext4 /dev/zram0 && mount` → 在上面建文件系统。
   - 这个意义上，它**“长得像一个磁盘”**。

2. **从物理介质看**：
   - 它没有一个独立的“硬盘芯片”或“eMMC 颗粒”与之对应；  
   - 它用的是**系统已有的 RAM 页面 + 压缩算法**，通过 zram 驱动虚拟出来。

综合起来，更精确的说法是：

> `/dev/zram0` 是一个“**以 RAM 为后备存储的虚拟块设备**”，  
> 它在块设备层表现得像磁盘，但物理介质是内存而不是真正的磁盘。

### B.4 “先压缩再写磁盘”到底是谁在干？是 zram 还是 zswap？

你提到的这一句：

> “先做一次内存压缩，占用 CPU，然后再把压缩后的数据从内存换出到磁盘”

更符合的是 **“zswap + 磁盘 swap”** 的模型，而**不是“只用 zram 做 swap”**。

- **zswap + 磁盘 swap**：
  - 匿名页要换出时：
    - 先在 zswap pool 里压缩存一份（内存里，消耗 CPU + 一定内存）；
    - 当 zswap pool 不够 / 淘汰时，才把数据写入后端磁盘 swap；
  - 相比“直接不压缩就写 swap”：
    - 多了一次压缩/解压；
    - 但**真正落到磁盘的 I/O 量变少**。

- **只用 zram 做 swap（没有磁盘 swap）**：
  - 匿名页换出时：
    - 直接把页写到 `/dev/zram0`，由 zram 驱动压缩并存 RAM；
  - **整个 swap I/O 都停留在内存 + CPU 里，不再往磁盘落**。

因此：

- “先压缩，再从内存换出到磁盘” → 对应 **zswap 前端 + 磁盘后端**；
- “压缩后就停在内存设备里，不再落盘” → 对应 **zram 作为唯一 swap 设备**。

### B.5 简化心智模型：8G RAM + 8G swap + zram 时，怎么想才不乱？

假设：

- 物理内存：8G
- 磁盘有一个 8G 分区 `/dev/mmcblk0pX`，被 `mkswap` 当 swap 分区
- 还配置了一个 zram：

```bash
echo 2G > /sys/block/zram0/disksize
mkswap /dev/zram0
swapon /dev/zram0
```

可以这样总结：

1. **物理资源只有一份 8G RAM**，zram 使用的就是这 8G 中的一部分页面；
2. **swap 设备可以有多个**：
   - 8G 磁盘 swap（后端是真实 eMMC/UFS）；
   - 逻辑 2G 的 zram swap（后端是 RAM + 压缩）；
3. 内核在回收匿名页时，可以把页：
   - 写到磁盘 swap（慢，但不占 RAM）；
   - 或写到 zram swap（快但占 RAM + CPU，用压缩换容量）；
4. **zram 不是额外多出来的一块“硬件”；它是用这 8G RAM + 驱动 + 压缩算法，伪装出来的一个块设备。**

---

通过本篇文章，你可以把“swap 与压缩”的关系牢牢钉在三个点上：

- swap 是**匿名页的后备存储抽象**；
- 压缩是这条 swap 通路上的**实现策略和优化层**（zram = 压缩设备，zswap = 压缩缓存）；
- 在 ANR / 性能分析中，必须结合 CPU、内存、I/O 多个维度一起看，才能判断压缩到底是“救火队”还是“新瓶颈”。

