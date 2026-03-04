# HWComposer 与显示输出

## 引言

HWComposer (HWC) 是 SurfaceFlinger 与显示硬件之间的桥梁，负责硬件合成和显示输出。本文将详细介绍 HWC 的架构、合成策略选择和显示设备管理。

---

## 1. HWComposer 概述

### 1.1 HWC 在系统中的位置

```
┌─────────────────────────────────────────────────────────────────────┐
│                      HWC 在系统中的位置                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SurfaceFlinger                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 CompositionEngine                            │   │
│  │                        │                                     │   │
│  │     ┌──────────────────┼──────────────────┐                 │   │
│  │     │                  │                  │                 │   │
│  │     ▼                  ▼                  ▼                 │   │
│  │ RenderEngine      HWComposer        DisplayDevice           │   │
│  │ (GPU 合成)        (硬件合成)         (显示管理)              │   │
│  │     │                  │                  │                 │   │
│  └─────┼──────────────────┼──────────────────┼─────────────────┘   │
│        │                  │                  │                     │
│        │                  │ HAL              │                     │
│        │                  ▼                  │                     │
│  ┌─────┼────────────────────────────────────┼─────────────────┐   │
│  │     │          HWC HAL (HIDL/AIDL)       │                 │   │
│  │     │          IComposer Interface       │                 │   │
│  └─────┼────────────────────────────────────┼─────────────────┘   │
│        │                  │                  │                     │
│        │                  │ 驱动             │                     │
│        ▼                  ▼                  ▼                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Hardware                                │   │
│  │      GPU          Display Engine       Display Panel        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 HWC 职责

| 职责 | 描述 |
|-----|------|
| 硬件合成 | 使用显示引擎叠加 Layer |
| 显示管理 | 管理显示设备（连接、断开、模式） |
| VSYNC | 提供硬件 VSYNC 信号 |
| 热插拔 | 处理显示器热插拔事件 |
| 电源控制 | 控制显示电源状态 |
| 颜色管理 | HDR、色彩校准等 |

---

## 2. HWC HAL 接口

### 2.1 IComposer 接口

```cpp
// hardware/interfaces/graphics/composer/2.4/IComposer.hal
interface IComposer {
    // 创建客户端
    createClient() generates (Error error, IComposerClient client);
    
    // 获取能力
    getCapabilities() generates (vec<Capability> capabilities);
};

// IComposerClient 接口
interface IComposerClient {
    // ======== 显示管理 ========
    createVirtualDisplay(uint32_t width, uint32_t height, ...)
        generates (Error error, Display display);
    destroyVirtualDisplay(Display display) generates (Error error);
    
    // ======== Layer 管理 ========
    createLayer(Display display, uint32_t bufferSlotCount)
        generates (Error error, Layer layer);
    destroyLayer(Display display, Layer layer) generates (Error error);
    
    // ======== 合成 ========
    validateDisplay(Display display)
        generates (Error error, uint32_t numTypes, uint32_t numRequests);
    presentDisplay(Display display)
        generates (Error error, handle releaseFence);
    
    // ======== Layer 属性 ========
    setLayerBuffer(Display display, Layer layer, handle buffer, int32_t fence)
        generates (Error error);
    setLayerBlendMode(Display display, Layer layer, BlendMode mode)
        generates (Error error);
    setLayerCompositionType(Display display, Layer layer, Composition type)
        generates (Error error);
    setLayerDisplayFrame(Display display, Layer layer, Rect frame)
        generates (Error error);
    
    // ======== VSYNC ========
    setVsyncEnabled(Display display, Vsync enabled) generates (Error error);
    
    // ======== 电源 ========
    setPowerMode(Display display, PowerMode mode) generates (Error error);
}
```

### 2.2 合成类型

```cpp
// 合成类型枚举
enum class Composition : int32_t {
    INVALID = 0,
    
    // 客户端合成（GPU）
    CLIENT = 1,
    
    // 设备合成（HWC 叠加层）
    DEVICE = 2,
    
    // 固态颜色
    SOLID_COLOR = 3,
    
    // 鼠标
    CURSOR = 4,
    
    // Sideband（视频直通）
    SIDEBAND = 5,
};
```

---

## 3. 合成流程详解

### 3.1 validate-present 流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HWC validate-present 流程                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SurfaceFlinger                              HWC HAL                │
│       │                                         │                   │
│       │  1. 设置所有 Layer 属性                  │                   │
│       │     setLayerBuffer()                    │                   │
│       │     setLayerBlendMode()                 │                   │
│       │     setLayerCompositionType(DEVICE)     │                   │
│       │ ─────────────────────────────────────►  │                   │
│       │                                         │                   │
│       │  2. validateDisplay()                   │                   │
│       │ ─────────────────────────────────────►  │                   │
│       │                                         │ 检查是否所有      │
│       │                                         │ Layer 都能硬件合成│
│       │  ◄──────────────────────────────────── │                   │
│       │     返回需要改变的 Layer 数量            │                   │
│       │                                         │                   │
│       │  3. getChangedCompositionTypes()        │                   │
│       │ ─────────────────────────────────────►  │                   │
│       │  ◄──────────────────────────────────── │                   │
│       │     返回需要改为 CLIENT 的 Layer        │                   │
│       │                                         │                   │
│       │  4. 如果有 CLIENT Layer:                │                   │
│       │     └─ GPU 渲染到 framebuffer          │                   │
│       │     └─ setClientTarget()                │                   │
│       │ ─────────────────────────────────────►  │                   │
│       │                                         │                   │
│       │  5. presentDisplay()                    │                   │
│       │ ─────────────────────────────────────►  │                   │
│       │                                         │ 执行硬件合成      │
│       │                                         │ 输出到显示器      │
│       │  ◄──────────────────────────────────── │                   │
│       │     返回 release fence                  │                   │
│       │                                         │                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 SurfaceFlinger 调用 HWC

```cpp
// HWComposer.cpp
status_t HWComposer::prepare(Display displayId, const compositionengine::Output& output) {
    // 1. 获取 Layer 列表
    const auto& outputLayers = output.getOutputLayersOrderedByZ();
    
    // 2. 设置每个 Layer 的属性
    for (const auto& layer : outputLayers) {
        // 设置缓冲区
        auto& buffer = layer->getBuffer();
        mComposer->setLayerBuffer(displayId, layer->getHwcLayer(),
                buffer.handle, buffer.acquireFence);
        
        // 设置混合模式
        mComposer->setLayerBlendMode(displayId, layer->getHwcLayer(),
                layer->getBlendMode());
        
        // 设置显示帧
        mComposer->setLayerDisplayFrame(displayId, layer->getHwcLayer(),
                layer->getDisplayFrame());
        
        // 初始设为 Device 合成
        mComposer->setLayerCompositionType(displayId, layer->getHwcLayer(),
                HWC2::Composition::Device);
    }
    
    // 3. 验证
    uint32_t numTypes, numRequests;
    auto error = mComposer->validateDisplay(displayId, &numTypes, &numRequests);
    
    // 4. 获取需要改变的 Layer
    if (numTypes > 0) {
        std::vector<HWC2::Layer*> changedLayers;
        std::vector<HWC2::Composition> changedTypes;
        mComposer->getChangedCompositionTypes(displayId, &changedLayers, &changedTypes);
        
        // 更新 Layer 的合成类型
        for (size_t i = 0; i < changedLayers.size(); ++i) {
            findLayer(changedLayers[i])->setCompositionType(changedTypes[i]);
        }
    }
    
    return NO_ERROR;
}
```

---

## 4. 显示设备管理

### 4.1 DisplayDevice

```cpp
// DisplayDevice.cpp
class DisplayDevice : public RefBase {
    // 显示 ID
    DisplayId mId;
    
    // 显示信息
    DisplayInfo mDisplayInfo;
    
    // HWC 显示句柄
    std::optional<HWC2::Display> mHwcDisplay;
    
    // 显示模式
    DisplayModePtr mActiveMode;
    std::vector<DisplayModePtr> mSupportedModes;
    
    // 电源状态
    PowerMode mPowerMode;
    
    // 渲染表面
    std::unique_ptr<compositionengine::RenderSurface> mRenderSurface;
};
```

### 4.2 热插拔处理

```cpp
// SurfaceFlinger.cpp
void SurfaceFlinger::onHotplugReceived(int32_t sequenceId, HWC2::Display& display,
        HWC2::Connection connection) {
    
    if (connection == HWC2::Connection::Connected) {
        // 显示器连接
        createDisplay(display);
    } else {
        // 显示器断开
        destroyDisplay(display);
    }
    
    // 通知客户端
    mScheduler->onHotplugReceived(connection);
}

void SurfaceFlinger::createDisplay(HWC2::Display& hwcDisplay) {
    // 1. 获取显示信息
    DisplayInfo info;
    hwcDisplay.getName(&info.name);
    hwcDisplay.getDisplayType(&info.type);
    
    // 2. 创建 DisplayDevice
    auto display = std::make_shared<DisplayDevice>(
            displayId, hwcDisplay, info);
    
    // 3. 初始化显示配置
    display->setActiveMode(getPreferredMode(hwcDisplay));
    
    // 4. 添加到列表
    mDisplays.emplace(displayId, display);
}
```

---

## 5. 电源管理

### 5.1 电源模式

```cpp
// 电源模式枚举
enum class PowerMode : int32_t {
    OFF = 0,          // 完全关闭
    DOZE = 1,         // 低功耗显示（AOD）
    DOZE_SUSPEND = 3, // 低功耗休眠
    ON = 2,           // 正常开启
    ON_SUSPEND = 4,   // 开启但暂停
};
```

### 5.2 电源状态切换

```cpp
// SurfaceFlinger.cpp
void SurfaceFlinger::setPowerMode(const sp<IBinder>& displayToken, int mode) {
    auto display = getDisplayDevice(displayToken);
    
    PowerMode currentMode = display->getPowerMode();
    PowerMode newMode = static_cast<PowerMode>(mode);
    
    if (currentMode == newMode) {
        return;
    }
    
    // 1. 调用 HWC
    mHwc->setPowerMode(display->getId(), newMode);
    
    // 2. 更新 VSYNC
    if (newMode == PowerMode::ON) {
        // 开启 VSYNC
        mScheduler->onScreenAcquired();
    } else if (currentMode == PowerMode::ON) {
        // 关闭 VSYNC
        mScheduler->onScreenReleased();
    }
    
    // 3. 更新状态
    display->setPowerMode(newMode);
}
```

---

## 6. 显示配置

### 6.1 显示模式

```cpp
// DisplayMode 表示一种显示配置
struct DisplayMode {
    // 模式 ID
    ui::DisplayModeId id;
    
    // 分辨率
    uint32_t width;
    uint32_t height;
    
    // 刷新率
    float refreshRate;
    
    // DPI
    float xDpi;
    float yDpi;
    
    // 组 ID（用于分辨率分组）
    int32_t group;
};

// 切换显示模式
void SurfaceFlinger::setDesiredDisplayModeSpecs(
        const sp<IBinder>& displayToken, ...) {
    
    auto display = getDisplayDevice(displayToken);
    
    // 选择最佳模式
    auto mode = selectBestMode(desiredModeId, allowGroupSwitching);
    
    // 应用模式
    mHwc->setActiveConfig(display->getId(), mode->getHwcId());
    
    display->setActiveMode(mode);
}
```

### 6.2 颜色模式

```cpp
// 颜色模式
enum class ColorMode : int32_t {
    NATIVE = 0,              // 原生
    STANDARD_BT601_625 = 1,  // BT.601 625
    STANDARD_BT601_625_UNADJUSTED = 2,
    STANDARD_BT601_525 = 3,  // BT.601 525
    STANDARD_BT709 = 4,      // BT.709
    DCI_P3 = 5,              // DCI-P3
    SRGB = 7,                // sRGB
    ADOBE_RGB = 8,           // Adobe RGB
    DISPLAY_P3 = 9,          // Display P3
    BT2020 = 10,             // BT.2020
    BT2100_PQ = 11,          // BT.2100 PQ (HDR)
    BT2100_HLG = 12,         // BT.2100 HLG (HDR)
};
```

---

## 7. HWC 叠加层限制

### 7.1 常见限制

```
┌─────────────────────────────────────────────────────────────────────┐
│                     HWC 叠加层限制                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  限制类型                    说明                                    │
│  ────────────────────────────────────────────────                   │
│                                                                     │
│  叠加层数量                  通常 4-8 个                             │
│                                                                     │
│  缩放因子                    最大 2x 或 4x                           │
│                                                                     │
│  旋转支持                    可能只支持 0°/90°/180°/270°            │
│                                                                     │
│  像素格式                    可能不支持某些格式（如 YUV）            │
│                                                                     │
│  混合模式                    可能只支持部分混合模式                   │
│                                                                     │
│  显存带宽                    受显存带宽限制                          │
│                                                                     │
│  Layer 大小                  可能有最小/最大尺寸限制                  │
│                                                                     │
│  当超出限制时：                                                      │
│  └─ HWC 在 validate 时返回需要 Client 合成                          │
│  └─ SF 使用 GPU 合成这些 Layer                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 调试 HWC

```bash
# 查看 HWC 信息
adb shell dumpsys SurfaceFlinger --hwc

# 强制使用 GPU 合成
adb shell service call SurfaceFlinger 1008 i32 1

# 恢复硬件合成
adb shell service call SurfaceFlinger 1008 i32 0
```

---

## 8. 本章小结

本文详细介绍了 HWComposer 与显示输出：

1. **HWC 架构**：SurfaceFlinger 与显示硬件的桥梁
2. **HAL 接口**：IComposer、IComposerClient
3. **合成流程**：validate → GPU 渲染 → present
4. **显示管理**：热插拔、电源控制、显示模式
5. **叠加层限制**：数量、格式、变换限制

下一篇文章将介绍窗口渲染与动画机制。

---

## 参考资料

1. Android 源码：`frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp`
2. Android 源码：`hardware/interfaces/graphics/composer/`
3. [Implement HWC HAL](https://source.android.com/docs/core/graphics/implement-hwc)
