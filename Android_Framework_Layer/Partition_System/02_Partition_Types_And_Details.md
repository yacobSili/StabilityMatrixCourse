# Android 16 分区类型详解

> 本文档详细介绍 Android 16 中存在的所有分区，确保全面覆盖每个分区的作用、大小、文件系统、挂载点等信息

## 📋 目录

1. [核心启动分区](#1-核心启动分区)
2. [系统分区](#2-系统分区)
3. [供应商分区](#4-供应商分区)
4. [DLKM分区](#5-dlkm分区动态加载内核模块)
5. [验证和引导分区](#6-验证和引导分区)
6. [数据分区](#7-数据分区)
7. [特殊分区](#8-特殊分区)
8. [分区快速参考表](#9-分区快速参考表)

---

## 1. 核心启动分区

### 1.1 boot 分区

**分区名称**：`boot` / `boot_a` / `boot_b`

**作用**：
- 包含 Linux 内核镜像（GKI 或设备特定内核）
- 在 Android 12 及之前，还包含通用 ramdisk
- 在 Android 13+，ramdisk 已分离到 `init_boot` 分区

**典型大小**：
- 32MB - 128MB（取决于内核大小和压缩方式）

**文件系统**：
- 特殊格式：boot.img 格式（不是传统文件系统）
- 包含：内核镜像 + ramdisk（Android 12-）或仅内核（Android 13+）

**挂载点**：
- 不挂载为文件系统，由 bootloader 直接加载

**可读写性**：
- 只读（运行时）
- 可通过 fastboot 刷写更新

**对应镜像文件**：
- `boot.img` 或 `Image.lz4`（GKI设备）

**编译生成**：
- 内核编译：`make ARCH=arm64`
- 打包：`mkbootimg` 工具

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ⚠️ 更新需要签名验证

**Android版本演进**：
- Android 1.0+：存在
- Android 13+：ramdisk 分离到 `init_boot`

**查看方法**：
```bash
# 查看boot分区信息
adb shell ls -l /dev/block/by-name/boot*

# 查看当前使用的槽位
adb shell getprop ro.boot.slot_suffix
```

---

### 1.2 init_boot 分区

**分区名称**：`init_boot` / `init_boot_a` / `init_boot_b`

**作用**：
- 包含通用 ramdisk（Android 13+ 引入）
- 包含 init 可执行文件和初始启动脚本
- 从 `boot` 分区分离出来，实现更好的模块化

**典型大小**：
- 8MB - 32MB

**文件系统**：
- 特殊格式：init_boot.img 格式
- 包含：ramdisk（cpio 格式）

**挂载点**：
- 不挂载为文件系统，由内核在启动时解压到内存

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `init_boot.img`

**编译生成**：
- ramdisk 编译：`build/make/core/Makefile`
- 打包：`mkbootimg` 工具（指定 ramdisk）

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新

**Android版本演进**：
- Android 13+：新引入
- 之前版本：ramdisk 包含在 `boot` 分区中

**查看方法**：
```bash
# 查看init_boot分区
adb shell ls -l /dev/block/by-name/init_boot*
```

---

### 1.3 vendor_boot 分区

**分区名称**：`vendor_boot` / `vendor_boot_a` / `vendor_boot_b`

**作用**：
- 包含供应商特定的启动组件（Android 11+ 引入）
- 包含供应商 ramdisk
- 包含设备树 Blob（DTB）
- 包含供应商 ramdisk 表
- 支持 GKI 架构，实现内核与供应商代码分离

**典型大小**：
- 16MB - 64MB

**文件系统**：
- 特殊格式：vendor_boot.img 格式（boot header v3+）
- 包含：供应商 ramdisk、DTB、ramdisk 表

**挂载点**：
- 不挂载为文件系统
- 供应商 ramdisk 在启动时合并到通用 ramdisk

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vendor_boot.img`

**编译生成**：
- 供应商 ramdisk 编译：`device/*/vendor_boot/`
- DTB 编译：设备树编译器
- 打包：`mkbootimg` 工具（vendor_boot 模式）

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新

**Android版本演进**：
- Android 11+：新引入
- 之前版本：供应商代码包含在 `boot` 分区中

**查看方法**：
```bash
# 查看vendor_boot分区
adb shell ls -l /dev/block/by-name/vendor_boot*

# 查看vendor_boot内容（需要root）
adb shell ls -R /vendor_boot
```

---

### 1.4 dtbo 分区

**分区名称**：`dtbo` / `dtbo_a` / `dtbo_b`

**作用**：
- 包含设备树覆盖层（Device Tree Overlay）
- 允许在运行时动态修改设备树配置
- 用于支持不同的硬件配置（如不同的屏幕、相机等）

**典型大小**：
- 1MB - 8MB

**文件系统**：
- 特殊格式：dtbo.img 格式
- 包含：一个或多个设备树覆盖层（.dtbo 文件）

**挂载点**：
- 不挂载为文件系统
- 由 bootloader 在启动时应用

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `dtbo.img`

**编译生成**：
- 设备树覆盖层编译：`dtc` 工具
- 打包：`mkdtimg` 工具

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新

**Android版本演进**：
- Android 9+：广泛使用
- 之前版本：设备树配置固定在内核中

**查看方法**：
```bash
# 查看dtbo分区
adb shell ls -l /dev/block/by-name/dtbo*

# 提取dtbo内容（需要root和工具）
adb shell dd if=/dev/block/by-name/dtbo of=/sdcard/dtbo.img
```

---

## 2. 系统分区

### 2.1 system 分区

**分区名称**：`system` / `system_a` / `system_b`

**作用**：
- 包含 Android 框架核心代码
- 包含系统服务（SystemServer）
- 包含 AOSP 核心应用和库
- 包含系统级 API
- 是 Android 系统的核心部分

**典型大小**：
- 2GB - 4GB（取决于 Android 版本和设备）

**文件系统**：
- **erofs**（Android 9+，推荐，只读压缩文件系统）
- **ext4**（传统选择，兼容性好）

**挂载点**：
- `/system`（只读）

**可读写性**：
- 只读（运行时）
- 可通过 OTA 或 fastboot 更新

**对应镜像文件**：
- `system.img`（可能是 sparse 格式）

**编译生成**：
- AOSP 源码编译：`m -j systemimage`
- 输出：`out/target/product/{device}/system.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ✅ 支持增量更新

**Android版本演进**：
- Android 1.0+：存在
- Android 9+：推荐使用 erofs
- Android 10+：可能位于 super 分区中（动态分区）

**查看方法**：
```bash
# 查看system分区挂载
adb shell mount | grep system

# 查看system分区大小
adb shell df -h /system

# 查看system分区内容
adb shell ls -l /system
```

---

### 2.2 system_ext 分区

**分区名称**：`system_ext` / `system_ext_a` / `system_ext_b`

**作用**：
- 包含系统扩展模块（Android 11+ 引入）
- 包含与 system 紧密耦合的专有系统模块
- 包含需要访问系统内部 API 的扩展功能
- 目标是未来迁移到 `product` 分区

**典型大小**：
- 100MB - 500MB

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/system_ext`（只读）
- 通常挂载在 `/system/system_ext`

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `system_ext.img`

**编译生成**：
- 扩展模块编译：`PRODUCT_PACKAGES += system_ext:...`
- 输出：`out/target/product/{device}/system_ext.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ⚠️ 通常与 system 分区一起更新

**Android版本演进**：
- Android 11+：新引入
- 之前版本：内容包含在 `system` 分区中

**查看方法**：
```bash
# 查看system_ext分区
adb shell mount | grep system_ext
adb shell ls -l /system_ext
```

---

### 2.3 product 分区

**分区名称**：`product` / `product_a` / `product_b`

**作用**：
- 包含产品特定模块（Android 9+ 引入，Android 11+ 强化）
- 包含 OEM 产品定制功能
- 包含设备特定的应用和库
- 与 system 和 vendor 有严格的接口限制

**典型大小**：
- 200MB - 1GB

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/product`（只读）
- 通常挂载在 `/system/product`

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `product.img`

**编译生成**：
- 产品模块编译：`PRODUCT_PACKAGES += product:...`
- 输出：`out/target/product/{device}/product.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新

**Android版本演进**：
- Android 9+：引入
- Android 11+：接口限制强化

**接口限制**：
- 只能使用 VNDK 接口
- 只能使用公共/基础 SDK API
- 不能依赖 system 的不稳定或隐藏部分

**查看方法**：
```bash
# 查看product分区
adb shell mount | grep product
adb shell ls -l /product
```

---

## 3. 供应商分区

### 3.1 vendor 分区

**分区名称**：`vendor` / `vendor_a` / `vendor_b`

**作用**：
- 包含硬件抽象层（HAL）实现
- 包含供应商特定的二进制文件
- 包含设备驱动和固件
- 包含供应商特定的库和工具
- 实现 Android Treble 架构的关键部分

**典型大小**：
- 200MB - 1GB

**文件系统**：
- **erofs**（推荐）或 **ext4**

**挂载点**：
- `/vendor`（只读）

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vendor.img`

**编译生成**：
- HAL 代码编译：`device/*/vendor/`
- 输出：`out/target/product/{device}/vendor.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ⚠️ 需要与 system 分区版本兼容

**Android版本演进**：
- Android 8.0+：强制要求（Treble 架构）
- 之前版本：内容包含在 `system` 分区中

**HAL接口**：
- 必须实现稳定的 HAL 接口
- 通过 VNDK 与 system 交互

**查看方法**：
```bash
# 查看vendor分区
adb shell mount | grep vendor
adb shell ls -l /vendor

# 查看HAL实现
adb shell ls -l /vendor/lib*/hw/
```

---

### 3.2 odm 分区

**分区名称**：`odm` / `odm_a` / `odm_b`

**作用**：
- 包含 ODM（原始设计制造商）定制化
- 包含板级特定的配置和二进制文件
- 包含 ODM 特定的 HAL 实现
- 可选分区，某些设备可能没有

**典型大小**：
- 50MB - 200MB（如果存在）

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/odm`（只读，如果存在）

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `odm.img`（如果存在）

**编译生成**：
- ODM 代码编译：`device/*/odm/`
- 输出：`out/target/product/{device}/odm.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新（如果存在）
- ✅ 支持 OTA 更新（如果存在）

**Android版本演进**：
- Android 9+：引入
- 可选分区，不是所有设备都有

**查看方法**：
```bash
# 检查odm分区是否存在
adb shell ls -l /dev/block/by-name/odm* 2>/dev/null

# 如果存在，查看内容
adb shell mount | grep odm
adb shell ls -l /odm
```

---

## 4. DLKM分区（动态加载内核模块）

### 4.1 system_dlkm 分区

**分区名称**：`system_dlkm` / `system_dlkm_a` / `system_dlkm_b`

**作用**：
- 包含系统级内核模块（Android 13+ 引入）
- 用于 GKI 设备，实现内核模块化
- 包含系统级可加载内核模块（.ko 文件）

**典型大小**：
- 10MB - 50MB

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/system_dlkm`（只读）
- 模块加载路径：`/system_dlkm/lib/modules/`

**可读写性**：
- 只读（运行时）
- 模块可以动态加载到内核

**对应镜像文件**：
- `system_dlkm.img`

**编译生成**：
- 内核模块编译：`make modules`
- 打包：系统构建流程
- 输出：`out/target/product/{device}/system_dlkm.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ⚠️ 需要与内核 KMI 版本兼容

**Android版本演进**：
- Android 13+：新引入
- 之前版本：内核模块包含在内核镜像中

**GKI支持**：
- 仅 GKI 设备需要
- 非 GKI 设备可能没有此分区

**查看方法**：
```bash
# 查看system_dlkm分区
adb shell mount | grep system_dlkm
adb shell ls -l /system_dlkm/lib/modules/

# 查看已加载的模块
adb shell lsmod
```

---

### 4.2 vendor_dlkm 分区

**分区名称**：`vendor_dlkm` / `vendor_dlkm_a` / `vendor_dlkm_b`

**作用**：
- 包含供应商内核模块（Android 13+ 引入）
- 包含硬件特定的内核驱动模块
- 用于 GKI 设备，实现供应商驱动模块化

**典型大小**：
- 20MB - 100MB

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/vendor_dlkm`（只读）
- 模块加载路径：`/vendor_dlkm/lib/modules/`

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vendor_dlkm.img`

**编译生成**：
- 供应商内核模块编译：`device/*/vendor_dlkm/`
- 输出：`out/target/product/{device}/vendor_dlkm.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ✅ 支持 OTA 更新
- ⚠️ 需要与内核 KMI 版本兼容

**Android版本演进**：
- Android 13+：新引入
- GKI 设备必需

**查看方法**：
```bash
# 查看vendor_dlkm分区
adb shell mount | grep vendor_dlkm
adb shell ls -l /vendor_dlkm/lib/modules/
```

---

### 4.3 odm_dlkm 分区

**分区名称**：`odm_dlkm` / `odm_dlkm_a` / `odm_dlkm_b`

**作用**：
- 包含 ODM 内核模块（Android 13+ 引入）
- 包含板级特定的内核驱动模块
- 可选分区，某些设备可能没有

**典型大小**：
- 5MB - 30MB（如果存在）

**文件系统**：
- **erofs** 或 **ext4**

**挂载点**：
- `/odm_dlkm`（只读，如果存在）

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `odm_dlkm.img`（如果存在）

**编译生成**：
- ODM 内核模块编译：`device/*/odm_dlkm/`
- 输出：`out/target/product/{device}/odm_dlkm.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新（如果存在）
- ✅ 支持 OTA 更新（如果存在）

**Android版本演进**：
- Android 13+：新引入
- 可选分区

**查看方法**：
```bash
# 检查odm_dlkm分区是否存在
adb shell ls -l /dev/block/by-name/odm_dlkm* 2>/dev/null
```

---

## 5. 验证和引导分区

### 5.1 vbmeta 分区

**分区名称**：`vbmeta` / `vbmeta_a` / `vbmeta_b`

**作用**：
- 包含 Android Verified Boot (AVB) 元数据
- 包含分区验证信息（哈希值、签名等）
- 包含验证链的根
- 是 AVB 验证的起点

**典型大小**：
- 64KB - 1MB

**文件系统**：
- 特殊格式：vbmeta 格式（不是文件系统）

**挂载点**：
- 不挂载为文件系统
- 由 bootloader 读取用于验证

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vbmeta.img`

**编译生成**：
- AVB 工具生成：`avbtool make_vbmeta_image`
- 输出：`out/target/product/{device}/vbmeta.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ⚠️ 更新需要签名
- ⚠️ 更新失败可能导致设备无法启动

**Android版本演进**：
- Android 8.0+：引入 AVB
- Android 9+：强制要求

**查看方法**：
```bash
# 查看vbmeta分区
adb shell ls -l /dev/block/by-name/vbmeta*

# 提取vbmeta内容（需要root）
adb shell dd if=/dev/block/by-name/vbmeta of=/sdcard/vbmeta.img
```

---

### 5.2 vbmeta_system 分区

**分区名称**：`vbmeta_system` / `vbmeta_system_a` / `vbmeta_system_b`

**作用**：
- 包含 system 分区的验证元数据
- 用于验证 system 分区的完整性
- 支持 system 分区的独立验证

**典型大小**：
- 64KB - 1MB

**文件系统**：
- 特殊格式：vbmeta 格式

**挂载点**：
- 不挂载为文件系统

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vbmeta_system.img`

**编译生成**：
- AVB 工具生成：`avbtool make_vbmeta_image --include_descriptors_from_image system.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ⚠️ 需要与 system 分区一起更新

**Android版本演进**：
- Android 10+：引入

---

### 5.3 vbmeta_vendor 分区

**分区名称**：`vbmeta_vendor` / `vbmeta_vendor_a` / `vbmeta_vendor_b`

**作用**：
- 包含 vendor 分区的验证元数据
- 用于验证 vendor 分区的完整性

**典型大小**：
- 64KB - 1MB

**文件系统**：
- 特殊格式：vbmeta 格式

**挂载点**：
- 不挂载为文件系统

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vbmeta_vendor.img`

**编译生成**：
- AVB 工具生成：`avbtool make_vbmeta_image --include_descriptors_from_image vendor.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ⚠️ 需要与 vendor 分区一起更新

---

### 5.4 vbmeta_vendor_dlkm 分区

**分区名称**：`vbmeta_vendor_dlkm` / `vbmeta_vendor_dlkm_a` / `vbmeta_vendor_dlkm_b`

**作用**：
- 包含 vendor_dlkm 分区的验证元数据
- 用于验证 vendor_dlkm 分区的完整性

**典型大小**：
- 64KB - 1MB

**文件系统**：
- 特殊格式：vbmeta 格式

**挂载点**：
- 不挂载为文件系统

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `vbmeta_vendor_dlkm.img`

**编译生成**：
- AVB 工具生成：`avbtool make_vbmeta_image --include_descriptors_from_image vendor_dlkm.img`

**更新策略**：
- ✅ 支持 A/B 槽位更新
- ⚠️ 需要与 vendor_dlkm 分区一起更新

**Android版本演进**：
- Android 13+：引入（与 vendor_dlkm 分区一起）

---

### 5.5 generic_bootloader 分区

**分区名称**：`generic_bootloader`

**作用**：
- 包含初始引导加载程序
- 负责初始化硬件和加载 boot 分区
- 通常不槽位化（单槽位）

**典型大小**：
- 1MB - 16MB

**文件系统**：
- 特殊格式：bootloader 二进制格式

**挂载点**：
- 不挂载为文件系统
- 由硬件直接加载

**可读写性**：
- 只读（运行时）
- ⚠️ 更新需要特殊工具和权限

**对应镜像文件**：
- `bootloader.img` 或设备特定名称

**编译生成**：
- Bootloader 源码编译（U-Boot、LK 等）
- 输出：设备特定的 bootloader 镜像

**更新策略**：
- ❌ 通常不支持 OTA 更新
- ⚠️ 更新需要解锁 bootloader
- ⚠️ 更新失败可能导致设备变砖

**Android版本演进**：
- Android 1.0+：存在
- 不同设备使用不同的 bootloader

**查看方法**：
```bash
# 查看bootloader版本
adb shell getprop ro.bootloader

# 查看bootloader分区（需要root）
adb shell ls -l /dev/block/by-name/bootloader*
```

---

## 6. 数据分区

### 6.1 userdata 分区

**分区名称**：`userdata`

**作用**：
- 包含用户安装的应用
- 包含用户数据和设置
- 包含应用私有数据
- 是用户可见的主要存储空间

**典型大小**：
- 16GB - 512GB+（取决于设备存储容量）
- 通常是设备上最大的分区

**文件系统**：
- **f2fs**（推荐，针对闪存优化）
- **ext4**（传统选择）

**挂载点**：
- `/data`（读写）

**可读写性**：
- 读写（运行时）
- 用户和应用可以写入

**对应镜像文件**：
- `userdata.img`（初始空镜像）

**编译生成**：
- 初始镜像生成：`make userdataimage`
- 输出：`out/target/product/{device}/userdata.img`

**更新策略**：
- ⚠️ 通常不通过 OTA 更新（会丢失数据）
- ⚠️ 可以通过 fastboot 擦除和重新格式化

**加密**：
- 支持文件级加密（FBE）
- 支持全盘加密（FDE，已弃用）

**查看方法**：
```bash
# 查看userdata分区
adb shell df -h /data

# 查看用户数据
adb shell ls -l /data
```

---

### 6.2 cache 分区

**分区名称**：`cache`

**作用**：
- 存储临时数据
- OTA 更新包下载缓存
- Recovery 模式使用的临时文件
- 在某些 A/B 设备上可能不存在（使用 userdata 代替）

**典型大小**：
- 100MB - 1GB
- 在某些设备上可能不存在

**文件系统**：
- **f2fs** 或 **ext4**

**挂载点**：
- `/cache`（读写，如果存在）

**可读写性**：
- 读写（运行时）
- 可以安全擦除

**对应镜像文件**：
- `cache.img`（如果存在）

**编译生成**：
- 初始镜像生成：`make cacheimage`
- 输出：`out/target/product/{device}/cache.img`

**更新策略**：
- ⚠️ 不通过 OTA 更新
- ✅ 可以安全擦除

**Android版本演进**：
- Android 1.0+：存在
- Android 7.0+：A/B 设备可能不使用（使用 userdata 代替）

**查看方法**：
```bash
# 检查cache分区是否存在
adb shell mount | grep cache

# 如果存在，查看内容
adb shell ls -l /cache
```

---

### 6.3 metadata 分区

**分区名称**：`metadata`

**作用**：
- 存储加密元数据（如果使用元数据加密）
- 存储文件系统加密密钥
- 存储加密相关的配置信息

**典型大小**：
- 16MB - 64MB

**文件系统**：
- **ext4** 或特殊格式

**挂载点**：
- `/metadata`（读写，如果使用元数据加密）

**可读写性**：
- 读写（运行时）

**对应镜像文件**：
- `metadata.img`

**编译生成**：
- 初始镜像生成：`make metadataimage`
- 输出：`out/target/product/{device}/metadata.img`

**更新策略**：
- ⚠️ 不通过 OTA 更新
- ⚠️ 擦除会导致加密数据无法访问

**Android版本演进**：
- Android 7.0+：引入（用于文件级加密）

**查看方法**：
```bash
# 查看metadata分区
adb shell mount | grep metadata
```

---

### 6.4 misc 分区

**分区名称**：`misc`

**作用**：
- 存储 recovery/bootloader/OTA 逻辑状态
- 存储 OTA 更新状态信息
- 存储 bootloader 消息
- 用于 recovery 和 bootloader 之间的通信

**典型大小**：
- 1MB - 4MB

**文件系统**：
- 特殊格式：misc 分区格式（不是标准文件系统）

**挂载点**：
- 不挂载为文件系统
- 由 recovery 和 bootloader 直接访问

**可读写性**：
- 读写（由 recovery 和 bootloader）

**对应镜像文件**：
- `misc.img`

**编译生成**：
- 初始镜像生成：`make miscimage`
- 输出：`out/target/product/{device}/misc.img`

**更新策略**：
- ⚠️ 不通过 OTA 更新
- ✅ 可以安全擦除（会重置 OTA 状态）

**Android版本演进**：
- Android 1.0+：存在

**查看方法**：
```bash
# 查看misc分区
adb shell ls -l /dev/block/by-name/misc

# 查看OTA状态（需要root）
adb shell cat /dev/block/by-name/misc | hexdump -C
```

---

### 6.5 persist 分区

**分区名称**：`persist`

**作用**：
- 存储持久化数据
- 存储校准数据（传感器、相机等）
- 存储安全 OS 状态
- 存储设备特定的持久化配置
- 在恢复出厂设置时保留

**典型大小**：
- 8MB - 64MB

**文件系统**：
- **ext4**

**挂载点**：
- `/mnt/vendor/persist` 或 `/persist`（读写）

**可读写性**：
- 读写（运行时）

**对应镜像文件**：
- `persist.img`（如果预填充）

**编译生成**：
- 可能包含预填充数据：`device/*/persist/`
- 输出：`out/target/product/{device}/persist.img`

**更新策略**：
- ⚠️ 不通过 OTA 更新
- ⚠️ 擦除会导致校准数据丢失

**Android版本演进**：
- Android 早期版本：存在
- 不同厂商实现可能不同

**查看方法**：
```bash
# 查看persist分区
adb shell mount | grep persist
adb shell ls -l /persist
```

---

### 6.6 frp 分区

**分区名称**：`frp`

**作用**：
- Factory Reset Protection（工厂重置保护）
- 存储 FRP 锁定状态
- 防止未授权用户重置设备
- 某些设备可能使用其他机制（如 eFuse）

**典型大小**：
- 1MB - 4MB

**文件系统**：
- 特殊格式：FRP 分区格式

**挂载点**：
- 不挂载为文件系统
- 由系统服务访问

**可读写性**：
- 读写（由系统服务）

**对应镜像文件**：
- `frp.img`（如果存在）

**编译生成**：
- 初始镜像生成：`make frpimage`
- 输出：`out/target/product/{device}/frp.img`

**更新策略**：
- ⚠️ 不通过 OTA 更新
- ⚠️ 擦除会清除 FRP 锁定（需要特殊权限）

**Android版本演进**：
- Android 5.1+：引入 FRP
- 不同设备实现可能不同

**查看方法**：
```bash
# 查看frp分区（如果存在）
adb shell ls -l /dev/block/by-name/frp* 2>/dev/null
```

---

## 7. 特殊分区

### 7.1 recovery 分区

**分区名称**：`recovery`（某些设备）

**作用**：
- 包含恢复模式系统
- 用于系统恢复和 OTA 更新安装
- 在某些 A/B 设备上可能不存在（使用 boot 分区代替）

**典型大小**：
- 32MB - 128MB（如果存在）

**文件系统**：
- 特殊格式：recovery.img 格式（类似 boot.img）

**挂载点**：
- 不挂载为文件系统
- 由 bootloader 在 recovery 模式下加载

**可读写性**：
- 只读（运行时）

**对应镜像文件**：
- `recovery.img`（如果存在）

**编译生成**：
- Recovery 源码编译：`make recoveryimage`
- 输出：`out/target/product/{device}/recovery.img`

**更新策略**：
- ✅ 支持 OTA 更新（如果存在）
- ⚠️ 在某些 A/B 设备上不存在

**Android版本演进**：
- Android 1.0+：存在
- Android 7.0+：A/B 设备可能不使用（使用 boot 分区代替）

**查看方法**：
```bash
# 检查recovery分区是否存在
adb shell ls -l /dev/block/by-name/recovery* 2>/dev/null

# 进入recovery模式
adb reboot recovery
```

---

### 7.2 super 分区

**分区名称**：`super`

**作用**：
- 动态分区容器（Android 10+ 引入）
- 包含多个逻辑分区（system, vendor, product 等）
- 允许动态调整分区大小
- 支持更灵活的 OTA 更新

**典型大小**：
- 4GB - 16GB+（取决于包含的逻辑分区）

**文件系统**：
- 特殊格式：LVM 风格的分区容器
- 内部包含多个逻辑分区

**挂载点**：
- 不直接挂载
- 内部的逻辑分区被挂载（如 /system, /vendor）

**可读写性**：
- 只读（运行时，由系统管理）

**对应镜像文件**：
- `super.img`

**编译生成**：
- 动态分区工具生成：`lpmake`
- 输出：`out/target/product/{device}/super.img`

**更新策略**：
- ✅ 支持 OTA 更新
- ⚠️ 更新会更新内部的逻辑分区

**Android版本演进**：
- Android 10+：引入动态分区
- 之前版本：使用静态分区布局

**查看方法**：
```bash
# 查看super分区
adb shell ls -l /dev/block/by-name/super

# 查看动态分区信息（需要root和工具）
adb shell lpdump /dev/block/by-name/super
```

---

## 8. 分区快速参考表

### 8.1 所有分区汇总表

| 分区名称 | 类型 | A/B槽位 | 大小范围 | 文件系统 | 挂载点 | 可更新 |
|---------|------|---------|---------|---------|--------|--------|
| **boot** | 启动 | ✅ | 32-128MB | boot.img | - | ✅ |
| **init_boot** | 启动 | ✅ | 8-32MB | init_boot.img | - | ✅ |
| **vendor_boot** | 启动 | ✅ | 16-64MB | vendor_boot.img | - | ✅ |
| **dtbo** | 启动 | ✅ | 1-8MB | dtbo.img | - | ✅ |
| **system** | 系统 | ✅ | 2-4GB | erofs/ext4 | /system | ✅ |
| **system_ext** | 系统 | ✅ | 100-500MB | erofs/ext4 | /system_ext | ✅ |
| **product** | 系统 | ✅ | 200MB-1GB | erofs/ext4 | /product | ✅ |
| **vendor** | 供应商 | ✅ | 200MB-1GB | erofs/ext4 | /vendor | ✅ |
| **odm** | 供应商 | ✅ | 50-200MB | erofs/ext4 | /odm | ✅ |
| **system_dlkm** | DLKM | ✅ | 10-50MB | erofs/ext4 | /system_dlkm | ✅ |
| **vendor_dlkm** | DLKM | ✅ | 20-100MB | erofs/ext4 | /vendor_dlkm | ✅ |
| **odm_dlkm** | DLKM | ✅ | 5-30MB | erofs/ext4 | /odm_dlkm | ✅ |
| **vbmeta** | 验证 | ✅ | 64KB-1MB | vbmeta | - | ⚠️ |
| **vbmeta_system** | 验证 | ✅ | 64KB-1MB | vbmeta | - | ⚠️ |
| **vbmeta_vendor** | 验证 | ✅ | 64KB-1MB | vbmeta | - | ⚠️ |
| **vbmeta_vendor_dlkm** | 验证 | ✅ | 64KB-1MB | vbmeta | - | ⚠️ |
| **generic_bootloader** | 引导 | ❌ | 1-16MB | bootloader | - | ❌ |
| **userdata** | 数据 | ❌ | 16GB-512GB+ | f2fs/ext4 | /data | ⚠️ |
| **cache** | 数据 | ❌ | 100MB-1GB | f2fs/ext4 | /cache | ⚠️ |
| **metadata** | 数据 | ❌ | 16-64MB | ext4 | /metadata | ⚠️ |
| **misc** | 数据 | ❌ | 1-4MB | misc | - | ⚠️ |
| **persist** | 数据 | ❌ | 8-64MB | ext4 | /persist | ⚠️ |
| **frp** | 数据 | ❌ | 1-4MB | frp | - | ⚠️ |
| **recovery** | 特殊 | ❌ | 32-128MB | recovery.img | - | ✅ |
| **super** | 特殊 | ❌ | 4-16GB+ | lvm | - | ✅ |

**图例**：
- ✅ = 支持
- ❌ = 不支持
- ⚠️ = 条件支持或需要特殊处理

### 8.2 分区按重要性分类

**关键分区**（损坏会导致设备无法启动）：
- `generic_bootloader`
- `boot`
- `vbmeta`
- `system`

**重要分区**（损坏会导致功能异常）：
- `vendor`
- `init_boot`
- `vendor_boot`

**可选分区**（某些设备可能没有）：
- `odm`
- `odm_dlkm`
- `recovery`（A/B设备）
- `cache`（A/B设备）

---

## 总结

本文档详细介绍了 Android 16 中存在的所有分区（30+个），包括：

1. **核心启动分区**：boot, init_boot, vendor_boot, dtbo
2. **系统分区**：system, system_ext, product
3. **供应商分区**：vendor, odm
4. **DLKM分区**：system_dlkm, vendor_dlkm, odm_dlkm
5. **验证分区**：vbmeta系列
6. **数据分区**：userdata, cache, metadata, misc, persist, frp
7. **特殊分区**：recovery, super, generic_bootloader

每个分区都包含详细的作用、大小、文件系统、挂载点、更新策略等信息。

**下一步学习**：
- 了解分区布局和结构，请阅读 [分区布局和结构](03_Partition_Layout_And_Structure.md)
- 了解分区与镜像的关系，请阅读 [分区与镜像的关系](04_Partition_vs_Image_Relationship.md)
