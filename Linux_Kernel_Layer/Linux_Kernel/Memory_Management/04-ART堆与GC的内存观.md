# ART 堆与 GC 的内存观

## 学习目标

- 理解 ART 运行时堆在“内存地图”中的位置与形态
- 掌握对象分配路径与 RSS/PSS 的关系（工程可观测视角）
- 了解 GC 对进程内存（PSS/RSS）的影响方式
- 能结合 smaps/dumpsys 解读 Java Heap 与 Large Object 等分类

## 概述

本文从 **内存视角** 看 ART：堆在 `/proc/<pid>/maps` 与 `smaps` 中如何呈现、分配路径如何落到物理页、GC 如何影响 PSS/RSS，以及 dumpsys meminfo 中 Java Heap、Large Object 等指标的含义。不深入 GC 算法实现，侧重“可观测性”与“和内核/进程内存的衔接”。

---

## 一、ART 堆在地址空间中的形态

### 1.1 典型 maps 中的 ART 区间

ART 为 Java 堆、Large Object Space **等预留虚拟地址区间，通常表现为：**

- **匿名、可读写的连续区间**，pathname 常为 `[anon:dalvik-*]` 或类似标记
- 可能有多段：主堆、Large Object Space、映像空间等（视 ART 版本与配置而定）

示例（概念性）：

```
... 
7f1234000000-7f1234800000 rw-p 00000000 00:00 0   [anon:dalvik-main space]
7f1234800000-7f1234c00000 rw-p 00000000 00:00 0   [anon:dalvik-large object space]
...
```

这些区间的 **Rss/Pss** 在 smaps 中对应“当前已提交到物理内存”的部分；虚拟大小（Size）可能远大于 Rss（按需提交）。

```
TECNO-LI9:/proc/2233 # cat maps
14000000-34000000 rw-p 00000000 00:00 0                                  [anon:dalvik-main space]
366d8000-566d8000 rw-p 00000000 00:00 0                                  [anon:dalvik-free list large object space]
566d8000-586d8000 r--s 00000000 00:01 5126                               /memfd:jit-zygote-cache (deleted)
586d8000-5a6d8000 r-xs 02000000 00:01 5126                               [anon_shmem:dalvik-zygote-jit-code-cache]
5a6d8000-5c6d8000 r--s 00000000 00:01 5191                               /memfd:jit-cache (deleted)
5c6d8000-5e6d8000 r-xs 02000000 00:01 5191                               [anon_shmem:dalvik-jit-code-cache]
5e6d8000-5ec60000 rw-p 00000000 00:00 0
6f37c000-70a54000 rw-p 00000000 00:00 0                                  [anon:dalvik-/data/misc/apexdata/com.android.art/dalvik-cache/boot.art]
70a54000-70dc8000 r--p 00000000 fe:4e 24401                              /data/misc/apexdata/com.android.art/dalvik-cache/arm64/boot.oat
70dc8000-7199a000 r-xp 00374000 fe:4e 24401                              /data/misc/apexdata/com.android.art/dalvik-cache/arm64/boot.oat
7199a000-7199c000 ---p 00000000 00:00 0                                  [anon:dalvik-Boot image reservation]
前面的地址是VMA吗
```



**是的，/proc/[pid]/maps 中每一行最前面的**`起始地址-结束地址`**段，就是对应进程的一个**VMA（虚拟内存区域）**的**虚拟地址区间**，每一行整体就代表内核中一个`struct vm_area_struct`实例的可视化输出**。

简单来说：`/proc/[pid]/maps` 是内核把目标进程的VMA红黑树/链表解析后，以文本形式暴露的**VMA清单**，你贴的这份TECNO手机（Android系统）进程2233的maps，每一行都对应该进程的一个独立VMA，前面的地址段是这个VMA的核心标识（划定虚拟地址范围）。

### 补充：Android下/proc/[pid]/maps每行格式与VMA属性的对应
结合你贴的ART/Dalvik进程的实际场景，maps每行的字段完全对应`struct vm_area_struct`的核心成员，格式拆解如下：
```
[虚拟地址起始]-[虚拟地址结束] [权限] [页表偏移] [设备号:次设备号] [inode]  [VMA说明/映射文件路径]
```
对应你贴的示例行（比如dalvik主空间）：
`14000000-34000000 rw-p 00000000 00:00 0  [anon:dalvik-main space]`
- 地址段`14000000-34000000`：该VMA的**虚拟地址范围**（连续的，符合VMA是“连续虚拟地址区间”的定义）；
- 权限`rw-p`：该VMA的访问权限（读/写/不可执行/私有），对应VMA的`vm_flags`；
- `00:00 0`：无对应物理设备和inode，说明是**匿名VMA**（如Dalvik/ART的堆、栈、匿名映射，对应之前讲的“匿名映射型VMA”）；
- 说明`[anon:dalvik-main space]`：Android内核对该VMA的用途标注（Dalvik主内存空间）。

再比如有文件映射的VMA行：
`70a54000-70dc8000 r--p 00000000 fe:4e 24401  /data/misc/apexdata/com.android.art/dalvik-cache/arm64/boot.oat`
- 地址段是该VMA的虚拟地址范围；
- 权限`r--p`：只读/不可执行/私有；
- `fe:4e 24401`：映射文件的设备号和inode；
- 后面的路径是**文件映射的源文件**（boot.oat，ART的预编译文件），说明这是**文件映射型VMA**（内核将boot.oat文件映射到进程虚拟地址空间，对应VMA的`vm_file`成员）。

### 贴合Android/ART的关键细节（你贴的maps里的典型VMA类型）
你贴的这份maps是Android的ART/Dalvik进程，里面的VMA是Android特有的命名标注，都是内核为ART虚拟机创建的独立VMA，核心类型分两类：
1. **匿名VMA**：标注`[anon:xxx]`、设备号`00:00`、inode`0`，比如dalvik主空间、大对象空闲列表、boot镜像保留区，对应ART的内存堆、预留地址空间，无实际映射文件，是内核动态分配的匿名虚拟地址区间；
2. **文件映射VMA**：有具体文件路径（如boot.oat、memfd:jit-cache）、非0设备号和inode，比如JIT缓存、boot.oat/boot.art映射，是内核将磁盘文件（或memfd匿名文件）映射到进程虚拟地址空间的VMA，对应之前讲的“文件映射型VMA”。

### 再验证：VMA的“互不重叠、属性独立”特性
从你贴的maps里能清晰看到**所有VMA的地址段无重叠**（比如上一个VMA结束于34000000，下一个起始于366d8000，中间无交集），且每个VMA的**权限、映射类型、用途**都独立（比如dalvik主空间是rw-p匿名，boot.oat是r--p文件映射，JIT代码缓存是r-xs共享），这完全符合之前讲的VMA“**互不重叠、属性独立**”的核心特征。

### 核心总结
/proc/[pid]/maps 是**内核暴露进程VMA的核心调试文件**，对于Android内核调试/性能优化（比如排查进程地址空间泄漏、VMA过多导致的内存管理效率低、地址冲突），分析maps里的VMA清单是最直接的手段——**看maps就是看进程的VMA分布**，前面的地址段就是每个VMA的虚拟地址边界。

我可以帮你**分析这份maps里的VMA分布特征**（比如匿名VMA/文件映射VMA占比、关键ART/Dalvik VMA的地址范围和权限），贴合Android内核调试场景，需要吗？





### 1.2 与“应用进程内存地图”的对应关系

- 在 [03-应用进程内存地图与指标体系](03-应用进程内存地图与指标体系.md) 中，这些区间属于 **匿名页**，计入进程的 **Private_Dirty / Anonymous**。
- dumpsys meminfo 中的 **Java Heap** 主要对应这些 ART 堆区间（以及可能的其他 ART 管理区间）的 PSS 汇总。

---

## 二、分配路径与 RSS/PSS 的关系

### 2.1 从 new 到物理页

- **Java 侧**：`new Object()` 等由 ART 的分配器在堆上分配指针/对象。
- **ART 侧**：在已预留的堆虚拟区间内移动 bump pointer 或从 free list 取块，必要时向内核 **扩大可写区域**（如 mprotect 或扩展 mmap 区间）。
- **内核侧**：首次写入时发生 **缺页**，分配物理页并建立映射，此时该页才计入 **Rss**。

因此：**分配对象不一定立刻推高 RSS**，只有触及新页时才会增加驻留；RSS 的“阶梯式”增长与 GC 后的“可能回落”都与此有关。

### 2.2 Large Object Space

- 大对象（如大数组）往往单独放在 **Large Object Space**，减少主堆碎片。
- 在 maps/smaps 中对应单独一段（或几段）匿名区；在 dumpsys 中常单独列为 **Native Heap** 或 **Other** 中的一大块，取决于实现与分类方式。
- 大对象直接 mmap 时，可能表现为 **独立匿名段**，其 Rss 随访问而增加。

### 2.3 与 Native 的边界

- JNI 分配的 Native 内存（malloc/mmap）在 maps 中通常为 **[heap]** 或其它匿名/文件段，**不计入 Java Heap**。
- 若 ART 通过 Native 分配器为大对象或内部结构分配内存，可能出现在 **Native Heap** 或 **Other** 中；区分时需结合 maps 的 pathname 与 dumpsys 的分类说明。

---

## 三、GC 对进程内存（PSS/RSS）的影响

### 3.1 为什么 GC 后 PSS 不一定明显下降

- **物理页不会仅因 GC 就立刻归还内核**：ART 回收的是“Java 对象”，空闲的堆空间通常 **仍被进程持有**（虚拟区间保留），物理页可能被复用或仍标记为已用，因此 **RSS/PSS 可能暂时不变甚至略升**（复制/压缩时的临时使用）。
- **只有“释放”回内核**（如 munmap、收缩堆、或内核回收匿名页并 swap）时，RSS 才会明显下降；Android 上多数情况下 ART 不会频繁做这类释放，以换取后续分配速度。

### 3.2 何时可能看到 PSS 下降

- 发生 **堆收缩**（heap trimming）或 **大块 unmap** 时。
- 系统内存压力大时，内核回收匿名页、swap 到 zram 等，**RSS 可能下降**（部分页被换出）；PSS 仍算本进程“分摊”的部分，可能随 swap 行为略有变化。
- 进程被 **杀** 或 **大量 unmap**（如关闭 WebView、释放大型缓存）时，PSS/RSS 会明显下降。

### 3.3 可观测性小结

- **dumpsys meminfo** 的 Java Heap PSS：看的是“当前 ART 堆占用的物理内存（按比例）”。
- **GC 频率/耗时** 与 **PSS 曲线** 结合看：若 GC 后 PSS 长期不降，多为正常（堆保留）；若希望“尽量少占物理内存”，需看 trim 策略与系统压力（详见 [06-AMS与进程内存治理框架](06-AMS与进程内存治理框架.md)）。

---

## 四、dumpsys meminfo 中的相关分类

- **Java Heap**：ART 管理的 Java 堆（主堆 + 可能的大对象区等）对应的 PSS。
- **Native Heap**：本进程 Native 分配（malloc 等）的 PSS；若 ART 内部用 Native 分配器，部分可能归在这里或 **Other**。
- **Stack**：线程栈；与 [03-应用进程内存地图与指标体系](03-应用进程内存地图与指标体系.md) 中的 [stack] 等对应。
- **Code**：可执行与 so 等文件映射的 PSS（共享多，PSS 通常不高）。
- **Graphics**：与 Gralloc/GPU 相关，见 [20-Gralloc与BufferQueue的内存路径](20-Gralloc与BufferQueue的内存路径.md)、[21-图形内存问题定位框架](21-图形内存问题定位框架.md)。

结合 [08-系统内存统计与可观测性](08-系统内存统计与可观测性.md) 可进一步理解各列含义与使用场景。

---

## 五、实践建议

1. **看堆是否在涨**：对比多次 `dumpsys meminfo <pid>` 的 Java Heap / Native Heap；结合 `smaps` 中 `[anon:dalvik-*]` 的 Rss 趋势。
2. **看 GC 与 PSS**：GC 后短时间 PSS 不降属常见；若要做“低内存占用”优化，需看 trim、大对象与缓存释放策略。
3. **区分 Java 与 Native**：Java Heap 涨多为对象未释放或缓存过大；Native Heap 涨需结合 [09-malloc到内核：brk与mmap及分配策略](09-malloc到内核-brk与mmap及分配策略.md)、JNI 与 so 分析。

---

## 总结

- ART 堆在 maps 中多为 **匿名段**（如 `[anon:dalvik-*]`），在 smaps/dumpsys 中对应 **Java Heap** 等 PSS。
- **分配路径**：new → ART 堆分配 → 触及新页时缺页 → RSS 增加；Large Object 可能单独区间或 mmap。
- **GC** 主要回收对象，一般不立刻释放物理页，故 **PSS 不一定在 GC 后明显下降**；下降多发生在堆收缩、unmap 或系统回收/swap 时。
- 工程上用 **dumpsys meminfo + smaps** 观察 Java/Native 占比与趋势，再结合 [05-应用内存优化与常见坑](05-应用内存优化与常见坑.md) 做优化与排查。
