# Surface 与 BufferQueue 详解

## 引言

Surface 是 Android 图形系统中最重要的概念之一，它代表了一块可以绘制的画布。而 BufferQueue 是 Surface 底层的缓冲区管理机制，实现了生产者-消费者模型。本文将详细介绍 Surface 和 BufferQueue 的工作原理。

---

## 1. Surface 概述

### 1.1 Surface 是什么

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Surface 的本质                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Surface = 一块可以绘制图形的画布                                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                         Surface                              │   │
│  │                                                              │   │
│  │   • 应用通过 Surface 获取 Canvas 进行绘制                     │   │
│  │   • 底层关联 BufferQueue                                      │   │
│  │   • 每个 Surface 对应 SurfaceFlinger 中的一个 Layer           │   │
│  │   • 支持硬件加速（通过 EGL/OpenGL ES）                         │   │
│  │                                                              │   │
│  │   主要用途：                                                  │   │
│  │   1. 窗口内容绘制（View 系统）                                │   │
│  │   2. 视频播放（MediaCodec）                                   │   │
│  │   3. 相机预览（Camera）                                       │   │
│  │   4. 游戏渲染（GLSurfaceView）                                │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Surface 类层次

```java
// Java 层
public class Surface implements Parcelable {
    // Native 对象指针
    long mNativeObject;
    
    // 绘制相关
    public Canvas lockCanvas(Rect dirtyRect);
    public void unlockCanvasAndPost(Canvas canvas);
    
    // 硬件加速
    public Canvas lockHardwareCanvas();
}

// Native 层
// frameworks/native/libs/gui/Surface.cpp
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase> {
    // BufferQueue 生产者
    sp<IGraphicBufferProducer> mGraphicBufferProducer;
    
    // 缓冲区槽
    BufferSlot mSlots[NUM_BUFFER_SLOTS];
    
    // ANativeWindow 接口实现
    int dequeueBuffer(...);
    int queueBuffer(...);
};
```

---

## 2. BufferQueue 机制

### 2.1 生产者-消费者模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                   BufferQueue 生产者-消费者模型                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   生产者 (Producer)              BufferQueue          消费者 (Consumer)
│   ┌─────────────┐            ┌─────────────────┐    ┌─────────────┐│
│   │             │            │                 │    │             ││
│   │    App      │            │    Slot Array   │    │ SurfaceFlinger
│   │  (Surface)  │            │  ┌───┬───┬───┐  │    │   (Layer)   ││
│   │             │            │  │ 0 │ 1 │ 2 │  │    │             ││
│   └──────┬──────┘            │  └───┴───┴───┘  │    └──────┬──────┘│
│          │                   │                 │           │       │
│          │ dequeueBuffer()   │                 │           │       │
│          │────────────────►  │  分配空闲槽     │           │       │
│          │ ◄──── Buffer ────│                 │           │       │
│          │                   │                 │           │       │
│          │ 绘制到 Buffer     │                 │           │       │
│          │                   │                 │           │       │
│          │ queueBuffer()     │                 │           │       │
│          │────────────────►  │  Buffer 入队   │           │       │
│          │                   │                 │ acquireBuffer()   │
│          │                   │                 │◄──────────│       │
│          │                   │   Buffer 出队  │           │       │
│          │                   │                 │           │       │
│          │                   │                 │  合成显示  │       │
│          │                   │                 │           │       │
│          │                   │                 │ releaseBuffer()   │
│          │                   │                 │◄──────────│       │
│          │                   │   Buffer 回收  │           │       │
│          │                   │                 │           │       │
│          │                   └─────────────────┘           │       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 BufferSlot 状态

```cpp
// BufferSlot.h
enum BufferState {
    // 空闲状态，可被生产者出队
    FREE = 0,
    
    // 已出队，生产者正在使用
    DEQUEUED = 1,
    
    // 已入队，等待消费者获取
    QUEUED = 2,
    
    // 已获取，消费者正在使用
    ACQUIRED = 3,
    
    // 共享状态（用于多消费者场景）
    SHARED = 4,
};

// BufferSlot 结构
struct BufferSlot {
    // 图形缓冲区
    sp<GraphicBuffer> mGraphicBuffer;
    
    // 当前状态
    BufferState mBufferState;
    
    // Fence（同步原语）
    sp<Fence> mFence;
    
    // 帧号
    uint64_t mFrameNumber;
};
```

### 2.3 BufferQueue 核心类

```cpp
// BufferQueueCore.h - BufferQueue 核心状态
class BufferQueueCore {
    // 槽数组
    BufferQueueDefs::SlotsType mSlots;
    
    // 队列（QUEUED 状态的缓冲区）
    Fifo mQueue;
    
    // 最大缓冲区数
    int mMaxAcquiredBufferCount;
    int mMaxDequeuedBufferCount;
    
    // 默认大小
    uint32_t mDefaultWidth;
    uint32_t mDefaultHeight;
    uint32_t mDefaultBufferFormat;
};

// IGraphicBufferProducer - 生产者接口
class IGraphicBufferProducer : public IInterface {
public:
    // 请求缓冲区
    virtual status_t requestBuffer(int slot, sp<GraphicBuffer>* buf) = 0;
    
    // 出队空闲缓冲区
    virtual status_t dequeueBuffer(int* outSlot, sp<Fence>* outFence,
            uint32_t width, uint32_t height, PixelFormat format,
            uint64_t usage, FrameEventHistoryDelta* outTimestamps) = 0;
    
    // 入队已绘制缓冲区
    virtual status_t queueBuffer(int slot, const QueueBufferInput& input,
            QueueBufferOutput* output) = 0;
    
    // 取消缓冲区
    virtual status_t cancelBuffer(int slot, const sp<Fence>& fence) = 0;
};

// IGraphicBufferConsumer - 消费者接口
class IGraphicBufferConsumer : public IInterface {
public:
    // 获取已入队的缓冲区
    virtual status_t acquireBuffer(BufferItem* outBuffer,
            nsecs_t expectedPresent, uint64_t maxFrameNumber = 0) = 0;
    
    // 释放缓冲区
    virtual status_t releaseBuffer(int slot, uint64_t frameNumber,
            const sp<Fence>& releaseFence, ...) = 0;
};
```

---

## 3. Surface 绘制流程

### 3.1 软件绘制流程

```java
// ViewRootImpl.java - 软件绘制
private boolean drawSoftware(Surface surface, ...) {
    // 1. 锁定 Canvas
    Canvas canvas;
    try {
        canvas = surface.lockCanvas(dirty);
    } catch (IllegalArgumentException | OutOfResourcesException e) {
        return false;
    }
    
    try {
        // 2. 清除背景
        if (!canvas.isOpaque() || yoff != 0) {
            canvas.drawColor(0, PorterDuff.Mode.CLEAR);
        }
        
        // 3. 绘制 View 树
        mView.draw(canvas);
        
    } finally {
        // 4. 解锁并提交
        surface.unlockCanvasAndPost(canvas);
    }
    return true;
}
```

### 3.2 lockCanvas 实现

```java
// Surface.java
public Canvas lockCanvas(Rect inOutDirty) {
    synchronized (mLock) {
        // 调用 Native
        mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
        return mCanvas;
    }
}

// frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    
    // 1. 出队缓冲区
    ANativeWindow_Buffer outBuffer;
    status_t err = surface->lock(&outBuffer, dirtyRect);
    
    // 2. 创建 SkBitmap
    SkImageInfo info = SkImageInfo::Make(
            outBuffer.width, outBuffer.height,
            convertPixelFormat(outBuffer.format),
            kPremul_SkAlphaType);
    
    SkBitmap bitmap;
    bitmap.setInfo(info, outBuffer.stride * bytesPerPixel(outBuffer.format));
    bitmap.setPixels(outBuffer.bits);
    
    // 3. 设置到 Canvas
    Canvas* nativeCanvas = getNativeCanvas(env, canvasObj);
    nativeCanvas->setBitmap(bitmap);
    
    return (jlong) lockedSurface.get();
}
```

### 3.3 unlockCanvasAndPost 实现

```java
// Surface.java
public void unlockCanvasAndPost(Canvas canvas) {
    synchronized (mLock) {
        // 解锁并入队
        nativeUnlockCanvasAndPost(mLockedObject, canvas);
        mLockedObject = 0;
    }
}

// Native 实现
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    
    // 1. 清理 Canvas
    Canvas* nativeCanvas = getNativeCanvas(env, canvasObj);
    nativeCanvas->setBitmap(SkBitmap());
    
    // 2. 入队缓冲区
    status_t err = surface->unlockAndPost();
}
```

---

## 4. BufferQueue 缓冲区管理

### 4.1 dequeueBuffer 流程

```cpp
// BufferQueueProducer.cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<Fence>* outFence,
        uint32_t width, uint32_t height, PixelFormat format, uint64_t usage, ...) {
    
    { // 自动锁
        Mutex::Autolock lock(mCore->mMutex);
        
        // 1. 查找空闲槽
        int found = BufferItem::INVALID_BUFFER_SLOT;
        while (found == BufferItem::INVALID_BUFFER_SLOT) {
            for (int s = 0; s < BufferQueueDefs::NUM_BUFFER_SLOTS; ++s) {
                if (mSlots[s].mBufferState == BufferState::FREE) {
                    found = s;
                    break;
                }
            }
            
            if (found == BufferItem::INVALID_BUFFER_SLOT) {
                // 没有空闲槽，等待
                mCore->mDequeueCondition.wait(mCore->mMutex);
            }
        }
        
        // 2. 检查是否需要重新分配
        const sp<GraphicBuffer>& buffer = mSlots[found].mGraphicBuffer;
        if (buffer == nullptr ||
                buffer->width != width ||
                buffer->height != height ||
                buffer->format != format) {
            // 需要分配新缓冲区
            mSlots[found].mNeedsReallocation = true;
        }
        
        // 3. 更新状态
        mSlots[found].mBufferState = BufferState::DEQUEUED;
        
        *outSlot = found;
        *outFence = mSlots[found].mFence;
    }
    
    // 4. 分配缓冲区（如需要）
    if (mSlots[*outSlot].mNeedsReallocation) {
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
                width, height, format, usage);
        mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
    }
    
    return NO_ERROR;
}
```

### 4.2 queueBuffer 流程

```cpp
// BufferQueueProducer.cpp
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput& input, QueueBufferOutput* output) {
    
    { // 自动锁
        Mutex::Autolock lock(mCore->mMutex);
        
        // 1. 验证槽状态
        if (mSlots[slot].mBufferState != BufferState::DEQUEUED) {
            return BAD_VALUE;
        }
        
        // 2. 创建 BufferItem
        BufferItem item;
        item.mGraphicBuffer = mSlots[slot].mGraphicBuffer;
        item.mSlot = slot;
        item.mFence = input.fence;
        item.mCrop = input.crop;
        item.mTransform = input.transform;
        item.mTimestamp = input.timestamp;
        item.mFrameNumber = ++mCore->mFrameCounter;
        
        // 3. 更新状态
        mSlots[slot].mBufferState = BufferState::QUEUED;
        mSlots[slot].mFrameNumber = item.mFrameNumber;
        
        // 4. 加入队列
        mCore->mQueue.push_back(item);
        
        // 5. 通知消费者
        mCore->mDequeueCondition.broadcast();
        
        // 6. 调用监听器
        if (mCore->mConsumerListener != nullptr) {
            mCore->mConsumerListener->onFrameAvailable(item);
        }
    }
    
    return NO_ERROR;
}
```

### 4.3 acquireBuffer 流程

```cpp
// BufferQueueConsumer.cpp
status_t BufferQueueConsumer::acquireBuffer(BufferItem* outBuffer,
        nsecs_t expectedPresent, uint64_t maxFrameNumber) {
    
    Mutex::Autolock lock(mCore->mMutex);
    
    // 1. 检查队列
    if (mCore->mQueue.empty()) {
        return NO_BUFFER_AVAILABLE;
    }
    
    // 2. 获取队首
    BufferItem& front = mCore->mQueue[0];
    
    // 3. 检查时间戳
    if (expectedPresent != 0 && front.mTimestamp > expectedPresent) {
        return PRESENT_LATER;
    }
    
    // 4. 更新状态
    int slot = front.mSlot;
    mSlots[slot].mBufferState = BufferState::ACQUIRED;
    mSlots[slot].mAcquireCalled = true;
    
    // 5. 复制到输出
    *outBuffer = front;
    
    // 6. 从队列移除
    mCore->mQueue.erase(mCore->mQueue.begin());
    
    return NO_ERROR;
}
```

---

## 5. Fence 同步机制

### 5.1 Fence 概述

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Fence 同步机制                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Fence = GPU/显示同步原语                                            │
│                                                                     │
│  问题：CPU 和 GPU/Display 并行工作，如何同步？                        │
│                                                                     │
│  解决：                                                              │
│  • Acquire Fence: 生产者完成绘制的信号（GPU 渲染完成）                │
│  • Release Fence: 消费者使用完成的信号（显示完成）                    │
│                                                                     │
│  流程：                                                              │
│                                                                     │
│      App (CPU)              GPU                  Display            │
│         │                    │                      │               │
│    dequeue Buffer            │                      │               │
│    (等待 Release Fence)      │                      │               │
│         │                    │                      │               │
│    提交绘制命令 ──────────────►│                      │               │
│         │                    │                      │               │
│    queue Buffer              │                      │               │
│    (携带 Acquire Fence)      │                      │               │
│         │                    │ 渲染完成             │               │
│         │                    │ (signal Acquire)    │               │
│         │                    │                      │               │
│         │                    │          SF acquire Buffer           │
│         │                    │          (等待 Acquire Fence)         │
│         │                    │                      │               │
│         │                    │                      │ 合成显示      │
│         │                    │                      │               │
│         │                    │          SF release Buffer           │
│         │                    │          (携带 Release Fence)         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Fence 使用

```cpp
// 生产者出队时等待 Release Fence
status_t Surface::dequeueBuffer(ANativeWindowBuffer** buffer, int* fenceFd) {
    int buf = -1;
    sp<Fence> fence;
    
    // 出队
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, ...);
    
    // 返回 Fence fd
    if (fence != nullptr && fence->isValid()) {
        *fenceFd = fence->dup();
    } else {
        *fenceFd = -1;
    }
    
    return OK;
}

// 生产者入队时携带 Acquire Fence
status_t Surface::queueBuffer(ANativeWindowBuffer* buffer, int fenceFd) {
    // 创建 Fence
    sp<Fence> fence(fenceFd >= 0 ? new Fence(fenceFd) : Fence::NO_FENCE);
    
    // 入队
    IGraphicBufferProducer::QueueBufferInput input(..., fence, ...);
    status_t err = mGraphicBufferProducer->queueBuffer(slot, input, &output);
    
    return err;
}
```

---

## 6. 三缓冲与帧调度

### 6.1 三缓冲机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                        三缓冲机制                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  时间线 ─────────────────────────────────────────────────────────►  │
│                                                                     │
│  VSYNC   ↓           ↓           ↓           ↓                     │
│          │           │           │           │                      │
│     ┌────┴────┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐               │
│     │         │ │         │ │         │ │         │                │
│     │ 显示 A  │ │ 显示 B  │ │ 显示 C  │ │ 显示 A  │  ← Display    │
│     │         │ │         │ │         │ │         │                │
│     └─────────┘ └─────────┘ └─────────┘ └─────────┘                │
│                                                                     │
│     ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│     │         │ │         │ │         │ │         │                │
│     │ 合成 B  │ │ 合成 C  │ │ 合成 A  │ │ 合成 B  │  ← SF        │
│     │         │ │         │ │         │ │         │                │
│     └─────────┘ └─────────┘ └─────────┘ └─────────┘                │
│                                                                     │
│     ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│     │         │ │         │ │         │ │         │                │
│     │ 绘制 C  │ │ 绘制 A  │ │ 绘制 B  │ │ 绘制 C  │  ← App        │
│     │         │ │         │ │         │ │         │                │
│     └─────────┘ └─────────┘ └─────────┘ └─────────┘                │
│                                                                     │
│  三个 Buffer 轮换使用：                                              │
│  • 一个正在显示                                                      │
│  • 一个正在 SF 合成                                                  │
│  • 一个正在 App 绘制                                                 │
│                                                                     │
│  优点：流水线并行，避免等待                                          │
│  缺点：增加一帧延迟                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 缓冲区数量配置

```cpp
// BufferQueueCore.cpp
// 默认最大缓冲区数
static constexpr int DEFAULT_MAX_DEQUEUED_BUFFERS = 1;
static constexpr int DEFAULT_MAX_ACQUIRED_BUFFERS = 1;

// 设置缓冲区数量
status_t BufferQueueProducer::setMaxDequeuedBufferCount(int maxDequeuedBuffers) {
    // 一般设置为 2（双缓冲）或 3（三缓冲）
    mCore->mMaxDequeuedBufferCount = maxDequeuedBuffers;
    return OK;
}
```

---

## 7. 本章小结

本文详细介绍了 Surface 与 BufferQueue：

1. **Surface**：可绘制的画布，关联 BufferQueue
2. **BufferQueue**：生产者-消费者模型的缓冲区管理
3. **BufferSlot 状态**：FREE → DEQUEUED → QUEUED → ACQUIRED
4. **绘制流程**：lockCanvas → draw → unlockCanvasAndPost
5. **Fence 同步**：GPU/Display 同步原语
6. **三缓冲**：流水线并行提升效率

下一篇文章将介绍 SurfaceControl 与 Transaction 机制。

---

## 参考资料

1. Android 源码：`frameworks/native/libs/gui/Surface.cpp`
2. Android 源码：`frameworks/native/libs/gui/BufferQueueCore.cpp`
3. [图形架构](https://source.android.com/docs/core/graphics/architecture)
