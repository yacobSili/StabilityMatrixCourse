# GraphicBuffer 与显存管理

## 引言

GraphicBuffer 是 Android 图形系统中实际存储像素数据的缓冲区。它涉及跨进程共享内存、GPU 内存分配等底层机制。本文将详细介绍 GraphicBuffer 的工作原理和显存管理机制。

---

## 1. GraphicBuffer 概述

### 1.1 GraphicBuffer 是什么

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GraphicBuffer 概述                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  GraphicBuffer = 图形缓冲区                                          │
│                                                                     │
│  特点：                                                              │
│  • 存储实际的像素数据                                                │
│  • 可跨进程共享（通过 native_handle）                                │
│  • 可被 GPU 和 CPU 访问                                              │
│  • 由 Gralloc HAL 分配                                               │
│                                                                     │
│  组成：                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    GraphicBuffer                             │   │
│  │                                                              │   │
│  │   width: 宽度（像素）                                         │   │
│  │   height: 高度（像素）                                        │   │
│  │   stride: 行跨度（字节）                                      │   │
│  │   format: 像素格式（RGBA_8888 等）                            │   │
│  │   usage: 使用标志（GPU 读写、CPU 读写等）                      │   │
│  │   handle: native_handle（用于跨进程共享）                     │   │
│  │                                                              │   │
│  │   ┌──────────────────────────────────────┐                  │   │
│  │   │           实际像素数据                │                  │   │
│  │   │      (GPU 可访问的内存区域)           │                  │   │
│  │   └──────────────────────────────────────┘                  │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 GraphicBuffer 类结构

```cpp
// frameworks/native/libs/ui/include/ui/GraphicBuffer.h
class GraphicBuffer : public ANativeObjectBase<
        ANativeWindowBuffer, GraphicBuffer, RefBase> {
public:
    // 缓冲区属性
    uint32_t width;
    uint32_t height;
    PixelFormat format;
    uint32_t usage;
    uint32_t stride;
    
    // Native 句柄（用于跨进程传递）
    buffer_handle_t handle;
    
    // 分配器
    sp<GraphicBufferAllocator> mAllocator;
    
    // 构造函数
    GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
            PixelFormat inFormat, uint32_t inUsage);
    
    // CPU 访问
    status_t lock(uint32_t usage, void** vaddr);
    status_t unlock();
    
    // 带 Fence 的访问
    status_t lockAsync(uint32_t usage, void** vaddr, int fenceFd);
    status_t unlockAsync(int* fenceFd);
    
    // 从句柄创建（用于跨进程接收）
    static sp<GraphicBuffer> from(ANativeWindowBuffer* buffer);
};
```

---

## 2. Gralloc HAL

### 2.1 Gralloc 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Gralloc 架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  应用/Framework                                                      │
│       │                                                             │
│       │ GraphicBufferAllocator                                      │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  libui / libgui                              │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│                            │ IAllocator (HIDL/AIDL)                 │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Gralloc HAL                               │   │
│  │                                                              │   │
│  │   allocate():  分配图形缓冲区                                 │   │
│  │   free():      释放图形缓冲区                                 │   │
│  │   lock():      锁定缓冲区（CPU 访问）                         │   │
│  │   unlock():    解锁缓冲区                                     │   │
│  │   registerBuffer(): 注册外部缓冲区                            │   │
│  │   unregisterBuffer(): 注销缓冲区                              │   │
│  │                                                              │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│                            │ 驱动调用                               │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    GPU 驱动 / DRM                            │   │
│  │                                                              │   │
│  │   分配 GPU 可访问的内存                                       │   │
│  │   创建 dma-buf fd                                            │   │
│  │   映射到用户空间                                              │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Gralloc 接口

```cpp
// hardware/interfaces/graphics/allocator/4.0/IAllocator.hal
interface IAllocator {
    // 分配缓冲区
    allocate(BufferDescriptor descriptor, uint32_t count)
        generates (Error error, uint32_t stride, vec<handle> buffers);
};

// hardware/interfaces/graphics/mapper/4.0/IMapper.hal
interface IMapper {
    // 导入缓冲区句柄（跨进程接收后）
    importBuffer(handle rawHandle)
        generates (Error error, handle buffer);
    
    // 释放缓冲区
    freeBuffer(handle buffer) generates (Error error);
    
    // 锁定缓冲区（CPU 访问）
    lock(handle buffer, uint64_t cpuUsage, Rect accessRegion, handle acquireFence)
        generates (Error error, pointer data);
    
    // 解锁缓冲区
    unlock(handle buffer) generates (Error error, handle releaseFence);
};
```

---

## 3. 缓冲区分配流程

### 3.1 分配流程

```cpp
// GraphicBufferAllocator.cpp
status_t GraphicBufferAllocator::allocate(uint32_t width, uint32_t height,
        PixelFormat format, uint32_t usage, buffer_handle_t* handle,
        uint32_t* stride, std::string requestorName) {
    
    // 1. 创建缓冲区描述符
    IMapper::BufferDescriptorInfo descriptorInfo;
    descriptorInfo.width = width;
    descriptorInfo.height = height;
    descriptorInfo.layerCount = 1;
    descriptorInfo.format = static_cast<hardware::graphics::common::V1_2::PixelFormat>(format);
    descriptorInfo.usage = usage;
    
    // 2. 调用 Gralloc HAL 分配
    hardware::hidl_vec<hardware::hidl_handle> buffers;
    status_t error = mAllocator->allocate(
            descriptor, 1 /* count */, 
            [&](auto err, auto newStride, auto newBuffers) {
                error = static_cast<status_t>(err);
                *stride = newStride;
                buffers = newBuffers;
            });
    
    // 3. 导入句柄
    if (error == NO_ERROR) {
        error = mMapper->importBuffer(buffers[0], handle);
    }
    
    return error;
}
```

### 3.2 GraphicBuffer 创建

```cpp
// GraphicBuffer.cpp
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inUsage)
    : BASE(), mOwner(ownHandle), mBufferMapper(GraphicBufferMapper::get()),
      mInitCheck(NO_ERROR) {
    
    width = inWidth;
    height = inHeight;
    format = inFormat;
    usage = inUsage;
    
    // 分配缓冲区
    mInitCheck = initWithSize(inWidth, inHeight, inFormat, 1, inUsage, "[unknown]");
}

status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
        std::string requestorName) {
    
    // 调用 GraphicBufferAllocator
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inUsage,
            &handle, &stride, requestorName);
    
    return err;
}
```

---

## 4. 跨进程共享

### 4.1 native_handle 机制

```cpp
// native_handle 是跨进程传递 GraphicBuffer 的关键
// 它包含文件描述符（fd）和整数数据

// system/core/include/cutils/native_handle.h
typedef struct native_handle {
    int version;        // 版本
    int numFds;         // 文件描述符数量
    int numInts;        // 整数数量
    int data[0];        // 实际数据（fd 在前，int 在后）
} native_handle_t;

// GraphicBuffer 的 handle 包含：
// - dma-buf fd（用于内存共享）
// - 缓冲区元数据（宽高、格式、stride 等）
```

### 4.2 跨进程传递流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                  GraphicBuffer 跨进程传递                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  进程 A (生产者)                      进程 B (消费者)                 │
│                                                                     │
│  ┌──────────────────┐               ┌──────────────────┐           │
│  │  GraphicBuffer   │               │  GraphicBuffer   │           │
│  │                  │               │                  │           │
│  │  handle ─────────┼───────────────┼─► handle         │           │
│  │    ├─ fd         │   Binder      │    ├─ fd (dup)   │           │
│  │    └─ metadata   │   传递        │    └─ metadata   │           │
│  │                  │               │                  │           │
│  └──────────────────┘               └──────────────────┘           │
│           │                                  │                      │
│           │ 映射到用户空间                    │ 映射到用户空间       │
│           ▼                                  ▼                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    共享物理内存 (dma-buf)                      │  │
│  │                                                              │  │
│  │                     实际像素数据                              │  │
│  │                                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  关键点：                                                           │
│  • native_handle 通过 Binder 传递                                   │
│  • fd 会被 dup 到接收进程                                           │
│  • 两个进程映射同一块物理内存                                        │
│  • 使用 Fence 同步访问                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 Flatten/Unflatten

```cpp
// GraphicBuffer.cpp

// 序列化（发送前）
status_t GraphicBuffer::flatten(void*& buffer, size_t& size,
        int*& fds, size_t& count) const {
    
    // 写入元数据
    FlattenableUtils::write(buffer, size, width);
    FlattenableUtils::write(buffer, size, height);
    FlattenableUtils::write(buffer, size, stride);
    FlattenableUtils::write(buffer, size, format);
    FlattenableUtils::write(buffer, size, usage);
    
    // 写入 handle 中的 fd
    if (handle != nullptr) {
        native_handle_t const* h = handle;
        for (int i = 0; i < h->numFds; i++) {
            fds[i] = h->data[i];
        }
        // 写入 handle 中的整数数据
        for (int i = 0; i < h->numInts; i++) {
            FlattenableUtils::write(buffer, size, h->data[h->numFds + i]);
        }
    }
    
    return NO_ERROR;
}

// 反序列化（接收后）
status_t GraphicBuffer::unflatten(void const*& buffer, size_t& size,
        int const*& fds, size_t& count) {
    
    // 读取元数据
    FlattenableUtils::read(buffer, size, width);
    FlattenableUtils::read(buffer, size, height);
    FlattenableUtils::read(buffer, size, stride);
    FlattenableUtils::read(buffer, size, format);
    FlattenableUtils::read(buffer, size, usage);
    
    // 重建 handle
    native_handle* h = native_handle_create(numFds, numInts);
    for (int i = 0; i < numFds; i++) {
        h->data[i] = fds[i];  // fd 已由 Binder 自动 dup
    }
    
    // 导入到 Gralloc
    status_t err = mBufferMapper.importBuffer(h, &handle);
    
    return err;
}
```

---

## 5. CPU 访问与锁定

### 5.1 lock/unlock 机制

```cpp
// GraphicBuffer.cpp

// 锁定缓冲区（CPU 读写）
status_t GraphicBuffer::lock(uint32_t inUsage, void** vaddr) {
    const Rect lockBounds(width, height);
    return lock(inUsage, lockBounds, vaddr);
}

status_t GraphicBuffer::lock(uint32_t inUsage, const Rect& rect, void** vaddr) {
    // 检查 usage 兼容性
    if ((usage & USAGE_SW_READ_MASK) == 0 && (inUsage & USAGE_SW_READ_MASK) != 0) {
        return INVALID_OPERATION;
    }
    
    // 调用 Gralloc mapper
    status_t err = mBufferMapper.lock(handle, inUsage, rect, vaddr);
    
    return err;
}

// 解锁缓冲区
status_t GraphicBuffer::unlock() {
    return mBufferMapper.unlock(handle);
}

// 带 Fence 的锁定（异步）
status_t GraphicBuffer::lockAsync(uint32_t inUsage, void** vaddr, int fenceFd) {
    // 等待 Fence 信号后再锁定
    return mBufferMapper.lockAsync(handle, inUsage, rect, fenceFd, vaddr);
}
```

### 5.2 使用示例

```cpp
// CPU 写入示例
sp<GraphicBuffer> buffer = new GraphicBuffer(
        width, height,
        HAL_PIXEL_FORMAT_RGBA_8888,
        GRALLOC_USAGE_SW_WRITE_OFTEN);

void* pixels = nullptr;
if (buffer->lock(GRALLOC_USAGE_SW_WRITE_OFTEN, &pixels) == NO_ERROR) {
    // 写入像素数据
    uint32_t* p = static_cast<uint32_t*>(pixels);
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            p[y * buffer->stride + x] = 0xFFFF0000;  // 红色
        }
    }
    
    buffer->unlock();
}
```

---

## 6. Usage 标志

### 6.1 常用 Usage 标志

```cpp
// hardware/libhardware/include/hardware/gralloc.h

// CPU 访问标志
GRALLOC_USAGE_SW_READ_NEVER     // CPU 不读
GRALLOC_USAGE_SW_READ_RARELY    // CPU 偶尔读
GRALLOC_USAGE_SW_READ_OFTEN     // CPU 频繁读
GRALLOC_USAGE_SW_WRITE_NEVER    // CPU 不写
GRALLOC_USAGE_SW_WRITE_RARELY   // CPU 偶尔写
GRALLOC_USAGE_SW_WRITE_OFTEN    // CPU 频繁写

// GPU 访问标志
GRALLOC_USAGE_HW_TEXTURE        // GPU 纹理读取
GRALLOC_USAGE_HW_RENDER         // GPU 渲染目标
GRALLOC_USAGE_HW_2D             // 2D 硬件加速
GRALLOC_USAGE_HW_COMPOSER       // HWComposer 使用
GRALLOC_USAGE_HW_FB             // Framebuffer
GRALLOC_USAGE_HW_VIDEO_ENCODER  // 视频编码器输入
GRALLOC_USAGE_HW_VIDEO_DECODER  // 视频解码器输出
GRALLOC_USAGE_HW_CAMERA_READ    // Camera 读取
GRALLOC_USAGE_HW_CAMERA_WRITE   // Camera 写入

// 特殊标志
GRALLOC_USAGE_PROTECTED         // 受保护内容
GRALLOC_USAGE_CURSOR            // 鼠标光标
```

### 6.2 Usage 与内存位置

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Usage 与内存位置关系                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Usage 组合                          内存位置                        │
│  ───────────────────────────────────────────────────────           │
│                                                                     │
│  SW_READ + SW_WRITE                  系统内存（可 CPU 访问）         │
│                                                                     │
│  HW_TEXTURE + HW_RENDER              显存（GPU 专用）                │
│                                                                     │
│  SW_READ + HW_TEXTURE               共享内存（CPU+GPU 可访问）       │
│                                      可能使用 Cached 或 Uncached    │
│                                                                     │
│  HW_COMPOSER + HW_RENDER            显存（GPU+Display 可访问）       │
│                                                                     │
│  PROTECTED                          安全内存（DRM 保护）            │
│                                                                     │
│  Gralloc HAL 根据 usage 选择最优内存类型                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 内存压力与回收

### 7.1 BufferQueue 缓冲区管理

```cpp
// BufferQueueCore.cpp
// 缓冲区数量限制
static constexpr int NUM_BUFFER_SLOTS = 64;  // 最大槽数
static constexpr int DEFAULT_MAX_DEQUEUED_BUFFERS = 1;
static constexpr int DEFAULT_MAX_ACQUIRED_BUFFERS = 1;

// 实际分配的缓冲区数 = maxDequeued + maxAcquired + 1
// 三缓冲时通常是 3 个
```

### 7.2 内存释放策略

```cpp
// 当应用不再使用 Surface 时
void Surface::disconnect(int api) {
    // 1. 取消所有已出队的缓冲区
    for (int i = 0; i < NUM_BUFFER_SLOTS; i++) {
        if (mSlots[i].mBufferState == BufferSlot::DEQUEUED) {
            cancelBuffer(mSlots[i].mGraphicBuffer.get(), mSlots[i].mFence);
        }
    }
    
    // 2. 断开连接
    mGraphicBufferProducer->disconnect(api);
}

// BufferQueueConsumer 释放
void BufferQueueConsumer::freeBufferLocked(int slot) {
    // 释放 GraphicBuffer
    mCore->mSlots[slot].mGraphicBuffer.clear();
    // 内存由 Gralloc 回收
}
```

---

## 8. 本章小结

本文详细介绍了 GraphicBuffer 与显存管理：

1. **GraphicBuffer**：存储像素数据的缓冲区
2. **Gralloc HAL**：图形内存分配器
3. **跨进程共享**：通过 native_handle 和 dma-buf
4. **CPU 访问**：lock/unlock 机制
5. **Usage 标志**：决定内存类型和访问方式
6. **内存管理**：BufferQueue 控制缓冲区数量

下一篇文章将介绍 SurfaceFlinger 架构概述。

---

## 参考资料

1. Android 源码：`frameworks/native/libs/ui/GraphicBuffer.cpp`
2. Android 源码：`hardware/interfaces/graphics/`
3. [Gralloc HAL](https://source.android.com/docs/core/graphics/implement-gralloc)
