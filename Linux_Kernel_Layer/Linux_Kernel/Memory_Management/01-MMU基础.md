# MMU 基础 - ARM64 内存管理单元

## 学习目标
- [x] 理解 ARM64 架构下的内存管理单元（MMU）作用
- [x] 掌握虚拟地址到物理地址的转换机制
- [x] 理解 PAGE_OFFSET、PAGE_END、VMALLOC_START 的作用和含义
- [ ] 理解线性映射区域的实现机制
- [ ] 掌握页表的创建和管理

## 源码位置
- 主文件：`../common/arch/arm64/mm/mmu.c`
- 头文件：`../common/arch/arm64/include/asm/memory.h`
- 版本：`android13-5.15-2024-11_r3`

## 关键概念

### 1. 线性映射区域（Linear Mapping）

**定义：**
线性映射是内核中最重要的虚拟地址区域，用于直接映射物理内存。在 ARM64 架构中，这是一个 1:1 的映射关系，即物理地址（PA）直接加上偏移量得到对应的虚拟地址（VA）。

**关键常量：**
- `PAGE_OFFSET`：线性映射区域的起始虚拟地址
- `PAGE_END`：线性映射区域的结束虚拟地址

**代码位置：**

```c
// 文件：../common/arch/arm64/mm/mmu.c
// 行号：1527-1531

/*
 * Linear mapping region is the range [PAGE_OFFSET..(PAGE_END - 1)]
 * accommodating both its ends but excluding PAGE_END. Max physical
 * range which can be mapped inside this linear mapping range, must
 * also be derived from its end points.
 */
```

**理解：**
线性映射提供了一个简单高效的物理地址访问方式。当内核需要访问物理内存时，可以直接通过 `PAGE_OFFSET + 物理地址` 的方式计算出虚拟地址，无需复杂的页表查找。

**地址转换公式：**
```
虚拟地址 (VA) = PAGE_OFFSET + 物理地址 (PA)
```

**关键要点：**
- 线性映射区域覆盖了大部分可用的物理内存
- 这个区域在内核启动时就已经建立
- 对于内核代码访问物理内存非常重要

### 2. VMALLOC 区域

**定义：**
`VMALLOC_START` 是内核动态内存分配（vmalloc）区域的起始地址。这个区域位于线性映射区域之后，用于分配非连续的虚拟地址空间。

**代码位置：**

```c
// 文件：../common/arch/arm64/include/asm/pgtable.h
// 行号：20, 24

/*
 * VMALLOC_START: beginning of the kernel vmalloc space
 */
#define VMALLOC_START		(MODULES_END)
```

**理解：**
vmalloc 区域用于：
- 分配大的、不连续的虚拟地址空间
- 设备驱动需要大块虚拟地址但物理内存可能不连续的场景
- I/O 缓冲区和 DMA 缓冲区

### 3. 地址空间布局

**ARM64 内核地址空间布局：**

```
低地址                                    高地址
  |--------|--------|--------|--------|--------|
    用户空间   线性映射   VMALLOC   ...    内核代码
           ↑         ↑         ↑
        PAGE_OFFSET  PAGE_END  VMALLOC_START
```

**关键区域说明：**

1. **用户空间**：低于 `PAGE_OFFSET`，用户进程的虚拟地址空间
2. **线性映射区域**：`[PAGE_OFFSET, PAGE_END)`，直接映射物理内存
3. **VMALLOC 区域**：从 `VMALLOC_START` 开始，用于动态分配

### 4. 页表管理

**代码位置：**

```c
// 文件：../common/arch/arm64/mm/mmu.c
// 行号：439-466

// 检查虚拟地址是否在有效范围内
if (virt < PAGE_OFFSET) {
    // 处理用户空间地址
}

// 检查虚拟地址是否在线性映射区域
if (virt < PAGE_OFFSET) {
    // 错误：地址在线性映射区域之前
}
```

## 源码分析

### 核心函数：arch_add_memory

**功能：**
添加物理内存到内核的内存管理系统中。

**关键代码：**

```c
// 文件：../common/arch/arm64/mm/mmu.c
// 行号：1538-1545

int arch_add_memory(int nid, u64 start, u64 size,
		    struct mhp_params *params)
{
	u64 start_linear_pa, end_linear_pa;
	
	// 计算线性映射区域的物理地址范围
	start_linear_pa = __pa(_PAGE_OFFSET(vabits_actual));
	end_linear_pa = __pa(PAGE_END - 1);
	
	// ...
}
```

**理解：**
这个函数负责将新的物理内存区域添加到系统中，需要确保新内存在线性映射范围内。

## 实践与验证

### 查看源码

```powershell
# 切换到源码目录
cd ../common

# 查看 MMU 相关代码
code arch/arm64/mm/mmu.c

# 搜索 PAGE_OFFSET 的定义和使用
git grep "PAGE_OFFSET" arch/arm64/include/asm/memory.h

# 查看内存布局相关常量
git grep -E "PAGE_OFFSET|PAGE_END|VMALLOC_START" arch/arm64/include/asm/

# 查看相关提交历史
git log --oneline -- arch/arm64/mm/mmu.c | Select-Object -First 10
```

### 验证地址布局

```powershell
# 查看地址布局定义
cd ../common
git grep "PAGE_OFFSET\|PAGE_END\|VMALLOC_START" arch/arm64/include/asm/pgtable.h

# 查看内存初始化代码
git show android13-5.15-2024-11_r3:arch/arm64/mm/mmu.c | Select-String "PAGE_OFFSET" -Context 3
```

### 查看线性映射相关代码

```powershell
cd ../common
# 查看线性映射区域的计算
git show android13-5.15-2024-11_r3:arch/arm64/mm/mmu.c | Select-String "Linear mapping" -Context 5
```

## 实际应用

### 问题场景

在你的升级问题中，从 `2024-05_r2` 升级到 `2024-11_r3` 后出现性能劣化。可能的原因：

1. **内存布局变更**：如果 `PAGE_OFFSET` 或线性映射区域大小发生变化，可能影响内存访问模式
2. **页表操作变化**：MMU 相关代码的修改可能影响地址转换效率
3. **内存碎片化**：VMALLOC 区域的使用方式改变可能导致内存碎片

### 解决方案

使用你已有的分析脚本检查变更：

```powershell
cd ../common
git diff android13-5.15-2024-05_r2 android13-5.15-2024-11_r3 -- arch/arm64/mm/mmu.c
```

重点关注：
- `PAGE_OFFSET` 相关的变更
- 线性映射区域的计算逻辑
- 页表操作相关函数

## 总结

### 核心要点

- **线性映射**：提供高效的物理地址访问，公式为 `VA = PAGE_OFFSET + PA`
- **地址布局**：内核虚拟地址空间分为线性映射、VMALLOC 等多个区域
- **页表管理**：MMU 通过页表完成虚拟地址到物理地址的转换
- **版本升级影响**：MMU 相关变更可能影响内存访问性能和布局

### 后续学习

- [ ] 深入理解页表的结构和创建过程
- [ ] 学习内存回收机制（vmscan.c）
- [ ] 理解页面写回机制（page-writeback.c）
- [ ] 分析升级问题中 MMU 相关变更的影响

## 参考资料

- 官方文档：`../common/Documentation/arm64/memory.rst`
- ARM Architecture Reference Manual
- 相关 commit：使用 `git log --oneline -- arch/arm64/mm/mmu.c` 查看

## 更新记录

- 2024-01-21：初始创建，包含 MMU 基础概念和线性映射区域说明
