# VSYNC 与帧调度机制

## 引言

VSYNC (Vertical Synchronization) 是协调应用绘制、合成和显示的核心时钟信号。理解 VSYNC 机制对于分析卡顿、丢帧问题至关重要。本文将详细介绍 VSYNC 的产生、分发和应用。

---

## 1. VSYNC 概述

### 1.1 什么是 VSYNC

```
┌─────────────────────────────────────────────────────────────────────┐
│                        VSYNC 概述                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  VSYNC = 垂直同步信号                                                │
│                                                                     │
│  来源：显示屏硬件                                                    │
│  频率：通常 60Hz (16.67ms) 或 90Hz/120Hz                            │
│  作用：告诉系统"可以刷新下一帧了"                                     │
│                                                                     │
│  时间线示意：                                                        │
│                                                                     │
│  VSYNC  ↓     ↓     ↓     ↓     ↓     ↓                            │
│         │     │     │     │     │     │                             │
│    ─────┼─────┼─────┼─────┼─────┼─────┼─────► 时间                  │
│         │     │     │     │     │     │                             │
│        16.67ms                                                      │
│        (60Hz)                                                       │
│                                                                     │
│  没有 VSYNC 的问题：                                                 │
│  • 撕裂 (Tearing): 显示两帧混合的画面                                │
│  • 不同步: 应用、合成、显示节奏混乱                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 VSYNC 的分发

```
┌─────────────────────────────────────────────────────────────────────┐
│                      VSYNC 分发架构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Hardware VSYNC (硬件信号)                                          │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    SurfaceFlinger                            │   │
│  │                                                              │   │
│  │   VSyncTracker: 跟踪 VSYNC 时间                              │   │
│  │   VSyncDispatch: 分发 VSYNC 信号                             │   │
│  │   Scheduler: 调度帧合成                                       │   │
│  │                                                              │   │
│  └────────────────────────┬────────────────────────────────────┘   │
│                           │                                         │
│          ┌────────────────┼────────────────┐                       │
│          │                │                │                        │
│          ▼                ▼                ▼                        │
│    VSYNC-sf         VSYNC-app        VSYNC-hw                      │
│    (SF 合成)        (应用绘制)        (显示刷新)                    │
│          │                │                                         │
│          │                ▼                                         │
│          │      ┌─────────────────┐                                │
│          │      │  Choreographer  │                                │
│          │      │   (编舞者)       │                                │
│          │      └────────┬────────┘                                │
│          │               │                                          │
│          ▼               ▼                                          │
│    SurfaceFlinger    应用进程                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. VSYNC 产生与跟踪

### 2.1 硬件 VSYNC

```cpp
// HWC 提供硬件 VSYNC 回调
// DisplayHardware/VsyncController.cpp
void VsyncController::onVsync(int64_t timestamp) {
    // 1. 通知 VSyncTracker
    mVSyncTracker->addVsyncTimestamp(timestamp);
    
    // 2. 如果收集够样本，可以关闭硬件 VSYNC
    if (mVSyncTracker->needsMoreSamples()) {
        mHwc->setVsyncEnabled(mDisplay, HWC2::Vsync::Enable);
    } else {
        mHwc->setVsyncEnabled(mDisplay, HWC2::Vsync::Disable);
    }
}
```

### 2.2 VSyncTracker - 软件模型

```cpp
// Scheduler/VSyncTracker.cpp
// VSyncTracker 基于历史数据预测 VSYNC 时间

class VSyncTracker {
    // VSYNC 周期
    nsecs_t mPeriod;
    
    // 历史时间戳
    std::vector<nsecs_t> mTimestamps;
    
    // 预测下一个 VSYNC 时间
    nsecs_t nextAnticipatedVSyncTimeFrom(nsecs_t timePoint) const {
        // 计算距离 timePoint 最近的下一个 VSYNC
        nsecs_t phase = (timePoint - mReferenceTime) % mPeriod;
        if (phase > 0) {
            return timePoint + (mPeriod - phase);
        }
        return timePoint;
    }
    
    // 添加样本
    void addVsyncTimestamp(nsecs_t timestamp) {
        mTimestamps.push_back(timestamp);
        // 更新周期估算
        updateModel();
    }
};
```

---

## 3. VSYNC 偏移

### 3.1 VSYNC 偏移概念

```
┌─────────────────────────────────────────────────────────────────────┐
│                      VSYNC 偏移机制                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  为什么需要偏移？                                                    │
│  • 应用绘制和 SF 合成需要时间                                        │
│  • 需要在显示 VSYNC 之前完成绘制和合成                               │
│                                                                     │
│  HW VSYNC  ↓               ↓               ↓                        │
│            │               │               │                        │
│       ─────┼───────────────┼───────────────┼─────► 时间             │
│            │               │               │                        │
│            │←──phase_app──►│               │                        │
│            │   offset      │               │                        │
│            │               │←─phase_sf────►│                        │
│            │               │   offset      │                        │
│            │               │               │                        │
│    VSYNC-app               VSYNC-sf        HW VSYNC                 │
│                                                                     │
│  典型配置：                                                          │
│  • App Offset: 0~4ms (让应用提前开始绘制)                           │
│  • SF Offset: 1~2ms (让 SF 在 HW VSYNC 前完成合成)                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 配置偏移

```cpp
// 系统属性配置
// ro.surface_flinger.vsync_event_phase_offset_ns  // App VSYNC 偏移
// ro.surface_flinger.vsync_sf_event_phase_offset_ns  // SF VSYNC 偏移

// Scheduler.cpp
void Scheduler::initVsync() {
    // 读取配置
    const nsecs_t appOffset = getPropertyValue(
            "ro.surface_flinger.vsync_event_phase_offset_ns", 0);
    const nsecs_t sfOffset = getPropertyValue(
            "ro.surface_flinger.vsync_sf_event_phase_offset_ns", 0);
    
    // 创建 VSYNC 源
    mAppVsync = mVsyncDispatch->createCallback(
            [this](nsecs_t, nsecs_t) { onVsyncApp(); },
            appOffset);
    
    mSfVsync = mVsyncDispatch->createCallback(
            [this](nsecs_t, nsecs_t) { onVsyncSf(); },
            sfOffset);
}
```

---

## 4. Choreographer

### 4.1 Choreographer 架构

```java
// frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    
    // 回调类型
    public static final int CALLBACK_INPUT = 0;        // 输入事件
    public static final int CALLBACK_ANIMATION = 1;    // 动画
    public static final int CALLBACK_INSETS_ANIMATION = 2;
    public static final int CALLBACK_TRAVERSAL = 3;    // View 遍历
    public static final int CALLBACK_COMMIT = 4;       // 提交
    
    // 回调队列
    private final CallbackQueue[] mCallbackQueues;
    
    // Native 显示事件接收器
    private FrameDisplayEventReceiver mDisplayEventReceiver;
    
    // 帧信息
    private long mLastFrameTimeNanos;
    private long mFrameIntervalNanos;  // 帧间隔（16.67ms）
    
    // 获取实例（每个 Looper 一个）
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
    
    // 注册回调
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
    
    // 请求 VSYNC
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            // 请求下一个 VSYNC
            mDisplayEventReceiver.scheduleVsync();
        }
    }
}
```

### 4.2 VSYNC 接收与分发

```java
// Choreographer.java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver {
    
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId,
            int frame, VsyncEventData vsyncEventData) {
        // 发送消息到主线程
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);  // 异步消息（跳过同步屏障）
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }
    
    @Override
    public void run() {
        // 在主线程执行
        doFrame(mTimestampNanos, mFrame, mVsyncEventData);
    }
}

void doFrame(long frameTimeNanos, int frame, ...) {
    // 1. 计算 jank
    long jitterNanos = frameTimeNanos - mLastFrameTimeNanos;
    if (jitterNanos >= mFrameIntervalNanos) {
        // 发生掉帧
        final long skippedFrames = jitterNanos / mFrameIntervalNanos;
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            Log.i(TAG, "Skipped " + skippedFrames + " frames!");
        }
    }
    
    // 2. 按顺序执行回调
    doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
}
```

---

## 5. 帧调度流程

### 5.1 一帧的完整流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                      一帧的完整流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Frame N 的生命周期：                                                │
│                                                                     │
│  时间 ─────────────────────────────────────────────────────────►    │
│                                                                     │
│  VSYNC-app     │                                                    │
│       │        │                                                    │
│       ▼        │                                                    │
│  ┌─────────────┴──────────────┐                                    │
│  │ 1. Choreographer.doFrame() │ ← 应用主线程                        │
│  │    • CALLBACK_INPUT        │                                    │
│  │    • CALLBACK_ANIMATION    │                                    │
│  │    • CALLBACK_TRAVERSAL    │                                    │
│  │      └─ performTraversals()│                                    │
│  │          ├─ measure        │                                    │
│  │          ├─ layout         │                                    │
│  │          └─ draw           │                                    │
│  └────────────────────────────┘                                    │
│            │                                                        │
│            │ 同步 DisplayList                                       │
│            ▼                                                        │
│  ┌────────────────────────────┐                                    │
│  │ 2. RenderThread            │ ← 渲染线程                          │
│  │    • 执行 DisplayList      │                                    │
│  │    • GPU 绘制              │                                    │
│  │    • queueBuffer()         │                                    │
│  └────────────────────────────┘                                    │
│            │                                                        │
│            │ Buffer 入队                                            │
│            ▼                                                        │
│  VSYNC-sf  │                                                        │
│       │    │                                                        │
│       ▼    ▼                                                        │
│  ┌────────────────────────────┐                                    │
│  │ 3. SurfaceFlinger          │                                    │
│  │    • latchBuffer()         │                                    │
│  │    • 合成所有 Layer         │                                    │
│  │    • presentDisplay()      │                                    │
│  └────────────────────────────┘                                    │
│            │                                                        │
│            │ 送显                                                    │
│            ▼                                                        │
│  VSYNC-hw  │                                                        │
│       │    │                                                        │
│       ▼    ▼                                                        │
│  ┌────────────────────────────┐                                    │
│  │ 4. Display                 │                                    │
│  │    • 显示 Frame N          │                                    │
│  └────────────────────────────┘                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 流水线并行

```
┌─────────────────────────────────────────────────────────────────────┐
│                        流水线并行                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  VSYNC  ↓       ↓       ↓       ↓       ↓                          │
│         │       │       │       │       │                           │
│         │       │       │       │       │                           │
│    App  │ ████  │       │       │       │ Frame N 绘制              │
│         │       │ ████  │       │       │ Frame N+1 绘制            │
│         │       │       │ ████  │       │ Frame N+2 绘制            │
│         │       │       │       │       │                           │
│    SF   │       │ ████  │       │       │ Frame N 合成              │
│         │       │       │ ████  │       │ Frame N+1 合成            │
│         │       │       │       │ ████  │ Frame N+2 合成            │
│         │       │       │       │       │                           │
│  Display│       │       │ ████  │       │ Frame N 显示              │
│         │       │       │       │ ████  │ Frame N+1 显示            │
│         │       │       │       │       │                           │
│                                                                     │
│  延迟：2 个 VSYNC 周期（约 33ms @ 60Hz）                             │
│  但吞吐量：每个 VSYNC 周期输出一帧                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 丢帧分析

### 6.1 丢帧原因

```
┌─────────────────────────────────────────────────────────────────────┐
│                        丢帧原因分析                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. App 绘制超时                                                    │
│     ┌───────────────────────────────────────┐                      │
│     │ VSYNC → doFrame → 绘制 → RenderThread │                      │
│     │        │←─────── > 16.67ms ──────────►│ ← 超时               │
│     └───────────────────────────────────────┘                      │
│     原因：主线程阻塞、布局复杂、过度绘制                             │
│                                                                     │
│  2. SF 合成超时                                                     │
│     ┌───────────────────────────────────────┐                      │
│     │ VSYNC-sf → 合成 → presentDisplay      │                      │
│     │           │←─── > 截止时间 ──────────►│ ← 超时               │
│     └───────────────────────────────────────┘                      │
│     原因：Layer 过多、GPU 合成复杂                                   │
│                                                                     │
│  3. Buffer 不足                                                     │
│     ┌───────────────────────────────────────┐                      │
│     │ App: dequeueBuffer() → 等待...        │                      │
│     │                        │← 无空闲 Buffer                       │
│     └───────────────────────────────────────┘                      │
│     原因：三缓冲不够、SF 消费慢                                      │
│                                                                     │
│  4. 调度延迟                                                        │
│     ┌───────────────────────────────────────┐                      │
│     │ VSYNC → 回调延迟执行                   │                      │
│     │        │← CPU 繁忙、线程被阻塞                                │
│     └───────────────────────────────────────┘                      │
│     原因：CPU 负载高、主线程被阻塞                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Jank 检测

```java
// Choreographer.java
void doFrame(long frameTimeNanos, ...) {
    // 计算与上一帧的间隔
    long jitterNanos = frameTimeNanos - mLastFrameTimeNanos;
    
    if (jitterNanos >= mFrameIntervalNanos) {
        // 跳过的帧数
        final long skippedFrames = jitterNanos / mFrameIntervalNanos;
        
        if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
            // 打印警告
            Log.i(TAG, "Skipped " + skippedFrames + " frames! "
                    + "The application may be doing too much work on its main thread.");
        }
        
        // 调整帧时间
        mLastFrameTimeNanos = frameTimeNanos - (jitterNanos % mFrameIntervalNanos);
    }
}
```

---

## 7. 可变刷新率

### 7.1 VRR (Variable Refresh Rate)

```cpp
// Scheduler.cpp
// 支持可变刷新率（如 60Hz/90Hz/120Hz 切换）

void Scheduler::setRefreshRateConfigs(
        std::shared_ptr<RefreshRateConfigs> configs) {
    mRefreshRateConfigs = configs;
}

// 根据内容选择刷新率
void Scheduler::updateFrameRateOverrides(const scheduler::RefreshRateConfigs& configs) {
    // 静态内容可降低刷新率节省功耗
    // 动画/视频可提高刷新率提升流畅度
    
    auto frameRate = configs.getBestRefreshRate(layers, ...);
    if (frameRate != mCurrentRefreshRate) {
        mCurrentRefreshRate = frameRate;
        mHwc->setActiveConfig(displayId, frameRate.configId);
    }
}
```

---

## 8. 本章小结

本文详细介绍了 VSYNC 与帧调度机制：

1. **VSYNC 来源**：硬件信号 → VSyncTracker 跟踪预测
2. **VSYNC 分发**：App VSYNC、SF VSYNC、偏移配置
3. **Choreographer**：应用端帧调度，按序执行回调
4. **帧流水线**：绘制 → 合成 → 显示 并行
5. **丢帧分析**：App 超时、SF 超时、Buffer 不足

下一篇文章将介绍 HWComposer 与显示输出。

---

## 参考资料

1. Android 源码：`frameworks/native/services/surfaceflinger/Scheduler/`
2. Android 源码：`frameworks/base/core/java/android/view/Choreographer.java`
3. [VSYNC](https://source.android.com/docs/core/graphics/arch-sf-hwc#vsync)
