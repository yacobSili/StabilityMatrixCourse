# SurfaceFlinger 架构概述

## 引言

SurfaceFlinger (SF) 是 Android 图形栈的核心服务，负责将多个应用的 Surface 合成为最终显示的帧。本文将详细介绍 SurfaceFlinger 的架构设计、核心组件和工作流程。

---

## 1. SurfaceFlinger 在系统中的位置

### 1.1 系统架构定位

```
┌─────────────────────────────────────────────────────────────────────┐
│                      应用进程 (多个)                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │   App 1     │  │   App 2     │  │  SystemUI   │                 │
│  │  (Surface)  │  │  (Surface)  │  │  (Surface)  │                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
└─────────┼────────────────┼────────────────┼─────────────────────────┘
          │                │                │
          │ BufferQueue    │ BufferQueue    │ BufferQueue
          │                │                │
┌─────────▼────────────────▼────────────────▼─────────────────────────┐
│                     SurfaceFlinger 进程                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     SurfaceFlinger                             │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │  │
│  │  │ Layer 1 │  │ Layer 2 │  │ Layer 3 │  │  ...    │          │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘          │  │
│  │                         │                                      │  │
│  │                         ▼                                      │  │
│  │              ┌─────────────────────┐                          │  │
│  │              │  CompositionEngine  │                          │  │
│  │              │  (合成引擎)          │                          │  │
│  │              └──────────┬──────────┘                          │  │
│  └──────────────────────────┼────────────────────────────────────┘  │
└─────────────────────────────┼────────────────────────────────────────┘
                              │ HWC HAL
┌─────────────────────────────▼────────────────────────────────────────┐
│                        Hardware                                       │
│       ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│       │ HWComposer  │    │     GPU     │    │   Display   │         │
│       └─────────────┘    └─────────────┘    └─────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心职责

| 职责 | 描述 |
|-----|------|
| Layer 管理 | 管理所有 Surface 对应的 Layer |
| 帧调度 | 根据 VSYNC 调度合成时机 |
| 图形合成 | 将多个 Layer 合成为一帧 |
| 显示输出 | 将合成结果送显 |
| 事务处理 | 处理 Surface 属性变更 |
| 电源管理 | 显示屏开关、亮度控制 |

---

## 2. SurfaceFlinger 核心组件

### 2.1 组件架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SurfaceFlinger 核心组件                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    SurfaceFlinger (主类)                     │   │
│  │                                                              │   │
│  │   • 服务入口点                                                │   │
│  │   • Binder 接口实现 (ISurfaceComposer)                       │   │
│  │   • 事件循环管理                                              │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │                                       │
│  ┌──────────────────────────┼──────────────────────────────────┐   │
│  │                          │                                   │   │
│  │  ┌───────────────┐  ┌────▼─────────┐  ┌───────────────┐    │   │
│  │  │               │  │              │  │               │    │   │
│  │  │    Layer      │  │  Scheduler   │  │ MessageQueue  │    │   │
│  │  │   (图层)      │  │  (调度器)     │  │  (消息队列)    │    │   │
│  │  │               │  │              │  │               │    │   │
│  │  └───────────────┘  └──────────────┘  └───────────────┘    │   │
│  │                                                              │   │
│  │  ┌───────────────┐  ┌──────────────┐  ┌───────────────┐    │   │
│  │  │               │  │              │  │               │    │   │
│  │  │ Composition   │  │DisplayDevice │  │ RenderEngine  │    │   │
│  │  │   Engine      │  │  (显示设备)   │  │  (渲染引擎)    │    │   │
│  │  │  (合成引擎)    │  │              │  │               │    │   │
│  │  └───────────────┘  └──────────────┘  └───────────────┘    │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心类概述

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.h
class SurfaceFlinger : public BnSurfaceComposer,
        public PriorityDumper,
        private IBinder::DeathRecipient,
        private HWC2::ComposerCallback {
    
    // ======== Layer 管理 ========
    // 所有 Layer 的列表
    std::vector<sp<Layer>> mDrawingState.layersSortedByZ;
    
    // ======== 调度器 ========
    std::unique_ptr<scheduler::Scheduler> mScheduler;
    
    // ======== 消息队列 ========
    std::unique_ptr<MessageQueue> mEventQueue;
    
    // ======== 合成引擎 ========
    std::unique_ptr<compositionengine::CompositionEngine> mCompositionEngine;
    
    // ======== 显示设备 ========
    std::unordered_map<PhysicalDisplayId, sp<DisplayDevice>> mDisplays;
    
    // ======== 渲染引擎 ========
    std::unique_ptr<renderengine::RenderEngine> mRenderEngine;
    
    // ======== HWComposer ========
    std::unique_ptr<HWComposer> mHwc;
    
    // ======== 事务状态 ========
    State mCurrentState;  // 当前状态
    State mDrawingState;  // 绘制状态
};
```

---

## 3. SurfaceFlinger 启动流程

### 3.1 启动序列

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SurfaceFlinger 启动流程                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  init.rc 启动 surfaceflinger 服务                                    │
│         │                                                           │
│         ▼                                                           │
│  main_surfaceflinger.cpp::main()                                    │
│         │                                                           │
│         ├─► 1. 设置进程优先级 (RT 优先级)                            │
│         │                                                           │
│         ├─► 2. 创建 SurfaceFlinger 实例                              │
│         │       sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger()
│         │                                                           │
│         ├─► 3. 初始化                                                │
│         │       flinger->init()                                     │
│         │         ├─ 初始化 RenderEngine (EGL/Vulkan)               │
│         │         ├─ 初始化 CompositionEngine                       │
│         │         ├─ 初始化 HWComposer                              │
│         │         ├─ 初始化 Scheduler                               │
│         │         └─ 初始化 MessageQueue                            │
│         │                                                           │
│         ├─► 4. 注册服务                                              │
│         │       sm->addService(String16("SurfaceFlinger"), flinger) │
│         │                                                           │
│         └─► 5. 启动主循环                                            │
│               flinger->run()                                        │
│                 └─ mEventQueue->run() (Looper)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 init() 详解

```cpp
// SurfaceFlinger.cpp
void SurfaceFlinger::init() {
    // 1. 创建 RenderEngine
    auto builder = renderengine::RenderEngineCreationArgs::Builder()
            .setPixelFormat(static_cast<int>(defaultCompositionPixelFormat))
            .setMaxTextureCacheSize(maxTextureCacheSize)
            .setEnableProtectedContext(enable);
    mCompositionEngine->setRenderEngine(
            renderengine::RenderEngine::create(builder.build()));
    
    // 2. 初始化 HWComposer
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(
            mHwcServiceName));
    mCompositionEngine->getHwComposer().setCallback(this);
    
    // 3. 处理热插拔（初始化显示设备）
    processDisplayHotplugEventsLocked();
    
    // 4. 初始化 Scheduler
    mScheduler = std::make_unique<scheduler::Scheduler>(
            *this, *mRefreshRateConfigs, *mVsyncSchedule);
    
    // 5. 初始化主显示
    initializeDisplays();
}
```

---

## 4. 主循环与消息处理

### 4.1 消息循环

```cpp
// MessageQueue.cpp
void MessageQueue::waitMessage() {
    do {
        IPCThreadState::self()->flushCommands();
        // 等待消息（使用 epoll）
        int32_t ret = mLooper->pollOnce(-1);
    } while (true);
}

// SurfaceFlinger 主循环
void SurfaceFlinger::run() {
    while (true) {
        mEventQueue->waitMessage();
    }
}
```

### 4.2 消息类型

```cpp
// MessageQueue.h
enum {
    INVALIDATE = 0,    // 刷新请求
    REFRESH = 1,       // 合成刷新
};

// 消息处理
void SurfaceFlinger::onMessageReceived(int32_t what) {
    switch (what) {
        case MessageQueue::INVALIDATE: {
            // 处理事务、Layer 更新
            onMessageInvalidate(vsyncId, expectedVsyncTime);
            break;
        }
        case MessageQueue::REFRESH: {
            // 执行合成
            onMessageRefresh();
            break;
        }
    }
}
```

### 4.3 VSYNC 驱动的工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VSYNC 驱动的工作流程                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  时间线 ─────────────────────────────────────────────────────────►  │
│                                                                     │
│  VSYNC  ↓           ↓           ↓           ↓                      │
│         │           │           │           │                       │
│         │           │           │           │                       │
│    INVALIDATE  REFRESH    INVALIDATE  REFRESH                       │
│         │           │           │           │                       │
│         ▼           ▼           ▼           ▼                       │
│    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                 │
│    │ 处理    │ │ 执行    │ │ 处理    │ │ 执行    │                 │
│    │ 事务    │ │ 合成    │ │ 事务    │ │ 合成    │                 │
│    │ Layer   │ │ 输出    │ │ Layer   │ │ 输出    │                 │
│    │ 更新    │ │ 显示    │ │ 更新    │ │ 显示    │                 │
│    └─────────┘ └─────────┘ └─────────┘ └─────────┘                 │
│                                                                     │
│  两阶段处理：                                                        │
│  1. INVALIDATE: 收集 Layer 变化、处理事务                            │
│  2. REFRESH: 执行实际合成、送显                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. 事务处理

### 5.1 事务流程

```cpp
// SurfaceFlinger.cpp
void SurfaceFlinger::setTransactionState(
        const FrameTimelineInfo& frameTimelineInfo,
        const Vector<ComposerState>& states,
        const Vector<DisplayState>& displays,
        uint32_t flags,
        const sp<IBinder>& applyToken,
        ...) {
    
    // 1. 加锁
    Mutex::Autolock lock(mStateLock);
    
    // 2. 应用 Layer 状态
    for (const ComposerState& state : states) {
        setClientStateLocked(frameTimelineInfo, state, ...
                
    // 3. 应用 Display 状态
    for (const DisplayState& display : displays) {
        setDisplayStateLocked(display);
    }
    
    // 4. 标记需要提交
    setTransactionFlags(eTransactionNeeded);
}
```

### 5.2 状态双缓冲

```cpp
// SurfaceFlinger 使用双缓冲管理状态
// mCurrentState: 接收新事务的状态
// mDrawingState: 用于合成的状态

void SurfaceFlinger::commitTransaction() {
    // 将 CurrentState 提交到 DrawingState
    mDrawingState = mCurrentState;
    
    // 更新各 Layer 状态
    mDrawingState.traverse([&](Layer* layer) {
        layer->commitTransaction(mDrawingState.layersSortedByZ);
    });
}
```

---

## 6. ISurfaceComposer 接口

### 6.1 主要接口

```cpp
// ISurfaceComposer.h
class ISurfaceComposer : public IInterface {
public:
    // 创建连接
    virtual sp<ISurfaceComposerClient> createConnection() = 0;
    
    // 创建显示
    virtual sp<IBinder> createDisplay(const String8& displayName, bool secure) = 0;
    
    // 销毁显示
    virtual void destroyDisplay(const sp<IBinder>& displayToken) = 0;
    
    // 设置事务状态
    virtual status_t setTransactionState(
            const FrameTimelineInfo& frameTimelineInfo,
            const Vector<ComposerState>& state,
            const Vector<DisplayState>& displays,
            uint32_t flags, ...) = 0;
    
    // 获取显示信息
    virtual status_t getDisplayInfo(const sp<IBinder>& display, 
            DisplayInfo* info) = 0;
    
    // 截屏
    virtual status_t captureScreen(const sp<IBinder>& display,
            sp<GraphicBuffer>* outBuffer, ...) = 0;
    
    // 电源控制
    virtual void setPowerMode(const sp<IBinder>& display, int mode) = 0;
};
```

---

## 7. 源码目录结构

```
frameworks/native/services/surfaceflinger/
├── SurfaceFlinger.cpp            # 主类实现
├── SurfaceFlinger.h
├── Layer.cpp                     # Layer 基类
├── Layer.h
├── BufferLayer.cpp               # 有缓冲区的 Layer
├── EffectLayer.cpp               # 效果 Layer
├── ContainerLayer.cpp            # 容器 Layer
├── DisplayDevice.cpp             # 显示设备
├── DisplayDevice.h
│
├── Scheduler/                    # 调度器
│   ├── Scheduler.cpp
│   ├── VSyncDispatch.cpp
│   └── VSyncTracker.cpp
│
├── CompositionEngine/            # 合成引擎
│   ├── CompositionEngine.cpp
│   ├── Output.cpp
│   ├── OutputLayer.cpp
│   └── RenderSurface.cpp
│
├── RenderEngine/                 # 渲染引擎
│   ├── RenderEngine.cpp
│   ├── GLESRenderEngine.cpp
│   └── SkiaGLRenderEngine.cpp
│
├── DisplayHardware/              # 显示硬件抽象
│   ├── HWComposer.cpp
│   ├── VsyncController.cpp
│   └── DisplayMode.cpp
│
└── main_surfaceflinger.cpp       # 入口
```

---

## 8. 本章小结

本文介绍了 SurfaceFlinger 的架构概述：

1. **系统定位**：图形合成服务，位于应用和硬件之间
2. **核心组件**：Layer、Scheduler、CompositionEngine、RenderEngine
3. **启动流程**：init() 初始化各组件，run() 进入主循环
4. **消息处理**：INVALIDATE 处理事务，REFRESH 执行合成
5. **事务机制**：双缓冲状态管理

下一篇文章将详细介绍 Layer 管理与合成策略。

---

## 参考资料

1. Android 源码：`frameworks/native/services/surfaceflinger/`
2. [SurfaceFlinger](https://source.android.com/docs/core/graphics/surfaceflinger-windowmanager)
3. [图形架构](https://source.android.com/docs/core/graphics/architecture)
