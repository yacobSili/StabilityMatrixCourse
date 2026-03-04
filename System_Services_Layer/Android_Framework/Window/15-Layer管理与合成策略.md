# Layer 管理与合成策略

## 引言

Layer 是 SurfaceFlinger 中最核心的概念，代表一个可合成的图形层。SurfaceFlinger 需要管理多个 Layer 的层级关系，并选择合适的合成策略。本文将详细介绍 Layer 管理和合成策略。

---

## 1. Layer 类型

### 1.1 Layer 类层次

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Layer 类层次                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                          Layer (基类)                               │
│                             │                                       │
│         ┌───────────────────┼───────────────────┐                  │
│         │                   │                   │                   │
│         ▼                   ▼                   ▼                   │
│    BufferLayer       ContainerLayer       EffectLayer              │
│    (有缓冲区)         (容器层)            (效果层)                  │
│         │                                       │                   │
│    ┌────┴────┐                            ┌────┴────┐              │
│    │         │                            │         │              │
│ BufferState BufferQueue                 Color     Blur            │
│   Layer       Layer                     Layer     Layer           │
│                                                                     │
│  Layer 类型说明：                                                    │
│  • BufferLayer: 有图形缓冲区，用于应用窗口                          │
│  • ContainerLayer: 只包含子 Layer，不显示内容                       │
│  • EffectLayer: 显示效果（颜色填充、模糊等）                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Layer 核心属性

```cpp
// Layer.h
class Layer : public virtual RefBase {
protected:
    // ======== 身份标识 ========
    std::string mName;               // Layer 名称
    uint32_t mSequence;              // 序列号
    sp<IBinder> mHandle;             // Binder 句柄
    
    // ======== 层级关系 ========
    wp<Layer> mParent;               // 父 Layer
    SortedVector<sp<Layer>> mChildren;  // 子 Layer 列表
    
    // ======== 几何属性 ========
    Rect mBounds;                    // 边界
    Rect mCrop;                      // 裁剪区域
    ui::Transform mTransform;        // 变换矩阵
    
    // ======== 视觉属性 ========
    float mAlpha;                    // 透明度
    half4 mColor;                    // 颜色
    bool mIsColorspaceAgnostic;
    
    // ======== 状态 ========
    State mCurrentState;             // 当前状态
    State mDrawingState;             // 绘制状态
    
    // ======== 合成相关 ========
    compositionengine::LayerFECompositionState mCompositionState;
};
```

---

## 2. Layer 生命周期

### 2.1 Layer 创建

```cpp
// SurfaceFlinger.cpp
status_t SurfaceFlinger::createLayer(const String8& name, ...,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, ...) {
    
    sp<Layer> layer;
    
    // 根据类型创建 Layer
    switch (type) {
        case eFXSurfaceBufferQueue:
            // 使用 BufferQueue 的 Layer
            result = createBufferQueueLayer(client, name, w, h, flags, 
                    format, &handle, &gbp, &layer);
            break;
            
        case eFXSurfaceBufferState:
            // 使用 BufferState 的 Layer（现代方式）
            result = createBufferStateLayer(client, name, w, h, flags,
                    &handle, &layer);
            break;
            
        case eFXSurfaceEffect:
            // 效果 Layer
            result = createEffectLayer(client, name, w, h, flags,
                    &handle, &layer);
            break;
            
        case eFXSurfaceContainer:
            // 容器 Layer
            result = createContainerLayer(client, name, w, h, flags,
                    &handle, &layer);
            break;
    }
    
    // 添加到列表
    if (result == NO_ERROR) {
        addClientLayer(client, *handle, layer, ...);
    }
    
    return result;
}
```

### 2.2 Layer 销毁

```cpp
void SurfaceFlinger::onLayerDestroyed(Layer* layer) {
    // 从列表移除
    mCurrentState.layersSortedByZ.remove(layer);
    
    // 标记需要更新
    setTransactionFlags(eTransactionNeeded);
}
```

---

## 3. Layer 树管理

### 3.1 父子关系

```cpp
// Layer.cpp
void Layer::addChild(const sp<Layer>& layer) {
    // 设置父子关系
    layer->setParent(this);
    mChildren.add(layer);
}

void Layer::removeChild(const sp<Layer>& layer) {
    mChildren.remove(layer);
    layer->setParent(nullptr);
}

// 遍历 Layer 树
void Layer::traverse(const std::function<void(Layer*)>& visitor) {
    visitor(this);
    for (const sp<Layer>& child : mChildren) {
        child->traverse(visitor);
    }
}
```

### 3.2 Z-Order 管理

```cpp
// 按 Z-Order 排序的 Layer 列表
void SurfaceFlinger::computeVisibleRegions(...) {
    // 自底向上遍历 Layer
    for (const auto& layer : mDrawingState.layersSortedByZ) {
        // 计算可见区域
        layer->computeVisibleRegions(...);
    }
}

// Layer 排序
void SurfaceFlinger::rebuildLayerStacks() {
    // 遍历所有 Layer，按 Z-Order 排序
    mDrawingState.traverse([&](Layer* layer) {
        // 根据层级和父子关系确定顺序
        displayLayersById[displayId].push_back(layer);
    });
}
```

---

## 4. 合成策略

### 4.1 Client Composition vs Device Composition

```
┌─────────────────────────────────────────────────────────────────────┐
│                       合成策略对比                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Device Composition (HWC 合成)                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   Layer 1 ────┐                                              │   │
│  │               │                                              │   │
│  │   Layer 2 ────┼───► HWComposer ───► Display                 │   │
│  │               │      (硬件合成)                               │   │
│  │   Layer 3 ────┘                                              │   │
│  │                                                              │   │
│  │   优点: 功耗低、性能好                                        │   │
│  │   缺点: 有限制（Layer 数量、格式、变换等）                    │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Client Composition (GPU 合成)                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │   Layer 1 ────┐                                              │   │
│  │               │                                              │   │
│  │   Layer 2 ────┼───► RenderEngine ───► HWC ───► Display      │   │
│  │               │      (GPU 合成)         (送显)               │   │
│  │   Layer 3 ────┘                                              │   │
│  │                                                              │   │
│  │   优点: 灵活、无限制                                          │   │
│  │   缺点: 功耗高、有延迟                                        │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 合成策略选择

```cpp
// CompositionEngine.cpp
void CompositionEngine::chooseCompositionStrategy() {
    // 1. 首先尝试全部使用 Device Composition
    for (auto& layer : outputLayers) {
        layer.compositionType = HWC2::Composition::Device;
    }
    
    // 2. 调用 HWC validate
    error = mHwc->validateDisplay(displayId, &numTypes, &numRequests);
    
    // 3. 如果 HWC 无法处理某些 Layer，改用 Client Composition
    if (error != HWC2::Error::None) {
        // 获取 HWC 的建议
        std::vector<HWC2::Layer*> changedLayers;
        mHwc->getChangedCompositionTypes(displayId, &changedLayers);
        
        for (auto hwcLayer : changedLayers) {
            // 将 Layer 设为 Client Composition
            auto& outputLayer = findOutputLayer(hwcLayer);
            outputLayer.compositionType = HWC2::Composition::Client;
        }
    }
}
```

### 4.3 混合合成

```
┌─────────────────────────────────────────────────────────────────────┐
│                        混合合成示例                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  实际场景：5 个 Layer，HWC 只能处理 3 个叠加层                       │
│                                                                     │
│       Layer 5 (系统 UI)   ─────┐                                    │
│       Layer 4 (悬浮窗)    ─────┼──► HWC 叠加层                      │
│                                │                                    │
│       Layer 3 (应用 C)    ─────┐                                    │
│       Layer 2 (应用 B)    ─────┼──► GPU 合成 ──► HWC 叠加层         │
│       Layer 1 (应用 A)    ─────┘    (合成为一个)                     │
│                                                                    │
│  合成结果：                                                          │
│  • Layer 1-3: 由 GPU 合成为一个缓冲区                               │
│  • Layer 4-5 + GPU 输出: 由 HWC 合成                                │
│  • 总共 3 个 HWC 叠加层                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. CompositionEngine 合成流程

### 5.1 合成主流程

```cpp
// CompositionEngine.cpp
void CompositionEngine::present(CompositionRefreshArgs& args) {
    // 1. 准备所有 Layer 的合成状态
    for (const auto& layer : args.layers) {
        layer->prepareForComposition();
    }
    
    // 2. 选择合成策略
    for (const auto& output : args.outputs) {
        output->chooseCompositionStrategy();
    }
    
    // 3. 执行 Client Composition（GPU 渲染）
    for (const auto& output : args.outputs) {
        if (output->hasClientComposition()) {
            output->renderClientLayers(*mRenderEngine);
        }
    }
    
    // 4. 提交到 HWC
    for (const auto& output : args.outputs) {
        output->finishFrame();
        output->presentFrame();
    }
}
```

### 5.2 Client Composition 渲染

```cpp
// Output.cpp
void Output::renderClientLayers(RenderEngine& renderEngine) {
    // 1. 设置渲染目标
    auto& renderSurface = getDisplaySurface();
    sp<GraphicBuffer> buffer = renderSurface.dequeueBuffer();
    
    // 2. 开始渲染
    renderEngine.beginRender(RenderSettings{...});
    
    // 3. 逐层渲染
    for (const auto& layer : getClientCompositionLayers()) {
        // 绑定 Layer 的纹理
        auto& buffer = layer->getBuffer();
        renderEngine.bindExternalTextureBuffer(buffer);
        
        // 绘制
        renderEngine.drawLayers(layerSettings);
    }
    
    // 4. 结束渲染
    sp<Fence> fence = renderEngine.finishRender();
    
    // 5. 入队缓冲区
    renderSurface.queueBuffer(buffer, fence);
}
```

---

## 6. Layer 状态更新

### 6.1 状态双缓冲

```cpp
// Layer.h
struct State {
    // 几何属性
    Rect crop;
    ui::Transform transform;
    float z;
    
    // 视觉属性
    float alpha;
    half4 color;
    
    // 缓冲区
    sp<GraphicBuffer> buffer;
    sp<Fence> acquireFence;
    
    // 帧信息
    uint64_t frameNumber;
};

// 状态提交
void Layer::commitTransaction() {
    mDrawingState = mCurrentState;
}
```

### 6.2 Buffer 更新

```cpp
// BufferLayer.cpp
bool BufferLayer::latchBuffer() {
    // 从 BufferQueue 获取最新缓冲区
    BufferItem item;
    status_t err = mConsumer->acquireBuffer(&item, 0);
    
    if (err == NO_ERROR) {
        // 更新当前缓冲区
        mCurrentState.buffer = item.mGraphicBuffer;
        mCurrentState.acquireFence = item.mFence;
        mCurrentState.frameNumber = item.mFrameNumber;
        
        // 需要重新合成
        return true;
    }
    
    return false;
}
```

---

## 7. 可见性与遮挡计算

### 7.1 可见区域计算

```cpp
// Layer.cpp
void Layer::computeVisibleRegions(Region* outVisibleRegion,
        Region* outCoveredRegion) {
    
    // 1. 计算 Layer 的不透明区域
    Region opaqueRegion;
    if (isOpaque()) {
        opaqueRegion.set(mBounds);
    }
    
    // 2. 从上层已覆盖区域中排除
    Region visible = mBounds - coveredRegion;
    
    // 3. 更新结果
    *outVisibleRegion = visible;
    *outCoveredRegion = coveredRegion | opaqueRegion;
}
```

### 7.2 遮挡优化

```cpp
// 完全被遮挡的 Layer 不需要合成
void CompositionEngine::setVisibleLayers() {
    Region aboveOpaque;
    
    // 自顶向下遍历
    for (auto it = layers.rbegin(); it != layers.rend(); ++it) {
        auto& layer = *it;
        
        // 计算可见区域
        Region visible = layer->getBounds() - aboveOpaque;
        
        if (visible.isEmpty()) {
            // 完全被遮挡，跳过合成
            layer->setVisible(false);
        } else {
            layer->setVisible(true);
            layer->setVisibleRegion(visible);
        }
        
        // 累积不透明区域
        if (layer->isOpaque()) {
            aboveOpaque |= layer->getBounds();
        }
    }
}
```

---

## 8. 本章小结

本文详细介绍了 Layer 管理与合成策略：

1. **Layer 类型**：BufferLayer、ContainerLayer、EffectLayer
2. **Layer 树**：父子关系、Z-Order 管理
3. **合成策略**：Device Composition (HWC) vs Client Composition (GPU)
4. **合成流程**：选择策略 → GPU 渲染 → HWC 送显
5. **可见性计算**：遮挡剔除优化

下一篇文章将介绍 VSYNC 与帧调度机制。

---

## 参考资料

1. Android 源码：`frameworks/native/services/surfaceflinger/Layer.cpp`
2. Android 源码：`frameworks/native/services/surfaceflinger/CompositionEngine/`
3. [HWComposer](https://source.android.com/docs/core/graphics/implement-hwc)
