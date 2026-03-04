# SurfaceControl 与 Transaction

## 引言

SurfaceControl 是 Surface 的控制句柄，用于管理 Surface 的属性和层级。Transaction 机制则允许批量更新多个 Surface 的属性，保证原子性。本文将详细介绍 SurfaceControl 和 Transaction 的工作原理。

---

## 1. SurfaceControl 概述

### 1.1 SurfaceControl 与 Surface 的关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                SurfaceControl 与 Surface 的关系                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  应用进程                           SurfaceFlinger                   │
│  ┌─────────────────────────┐       ┌─────────────────────────┐     │
│  │                         │       │                         │     │
│  │   SurfaceControl        │       │        Layer            │     │
│  │   ┌─────────────────┐   │       │   ┌─────────────────┐   │     │
│  │   │                 │   │       │   │                 │   │     │
│  │   │  Handle (句柄)  │───┼───────┼──►│   属性/状态     │   │     │
│  │   │                 │   │ Binder│   │                 │   │     │
│  │   └─────────────────┘   │       │   └────────┬────────┘   │     │
│  │           │             │       │            │            │     │
│  │           │ getSurface()│       │            │            │     │
│  │           ▼             │       │            ▼            │     │
│  │   ┌─────────────────┐   │       │   ┌─────────────────┐   │     │
│  │   │     Surface     │   │       │   │   BufferQueue   │   │     │
│  │   │ (绘制接口)       │───┼───────┼──►│   (缓冲区管理)   │   │     │
│  │   └─────────────────┘   │ 共享   │   └─────────────────┘   │     │
│  │                         │ 内存   │                         │     │
│  └─────────────────────────┘       └─────────────────────────┘     │
│                                                                     │
│  职责区分：                                                          │
│  • SurfaceControl: 控制 Layer 属性（位置、大小、透明度、Z-Order）    │
│  • Surface: 绘制内容（获取 Canvas、提交帧）                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 SurfaceControl 类结构

```java
// Java 层 - SurfaceControl.java
public final class SurfaceControl implements Parcelable {
    // Native 对象指针
    public long mNativeObject;
    
    // Layer 名称
    private String mName;
    
    // 宽高
    private int mWidth;
    private int mHeight;
    
    // 获取 Surface（用于绘制）
    public Surface getSurface() { ... }
    
    // Builder 模式创建
    public static class Builder {
        public Builder setName(String name) { ... }
        public Builder setParent(SurfaceControl parent) { ... }
        public Builder setBufferSize(int width, int height) { ... }
        public Builder setFormat(int format) { ... }
        public SurfaceControl build() { ... }
    }
}

// Native 层 - SurfaceControl.cpp
class SurfaceControl : public RefBase {
    // SurfaceFlinger 客户端
    sp<SurfaceComposerClient> mClient;
    
    // Layer 句柄
    sp<IBinder> mHandle;
    
    // GraphicBuffer 生产者
    sp<IGraphicBufferProducer> mGraphicBufferProducer;
};
```

---

## 2. SurfaceControl 创建

### 2.1 创建流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SurfaceControl 创建流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  应用/WMS                        SurfaceFlinger                      │
│      │                                │                             │
│      │ new SurfaceControl.Builder()   │                             │
│      │     .setName("xxx")            │                             │
│      │     .build()                   │                             │
│      │         │                      │                             │
│      │         │ nativeCreate()       │                             │
│      │         │──────────────────────│                             │
│      │         │                      │                             │
│      │         │            SurfaceComposerClient                   │
│      │         │              ::createSurface()                     │
│      │         │                      │                             │
│      │         │                      ▼                             │
│      │         │              ISurfaceComposer                      │
│      │         │              ::createLayer()                       │
│      │         │                      │                             │
│      │         │                      ▼                             │
│      │         │              创建 Layer 对象                        │
│      │         │              创建 BufferQueue                       │
│      │         │                      │                             │
│      │         │◄─────────────────────│                             │
│      │         │   返回 Handle + GBP  │                             │
│      │◄────────│                      │                             │
│      │                                │                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 创建代码分析

```java
// Java 层创建
SurfaceControl surfaceControl = new SurfaceControl.Builder()
        .setName("MyWindow")
        .setParent(parentSurface)
        .setBufferSize(width, height)
        .setFormat(PixelFormat.TRANSLUCENT)
        .build();

// Builder.build() 实现
public SurfaceControl build() {
    return new SurfaceControl(
            mSession, mName, mWidth, mHeight, mFormat,
            mFlags, mParent, mMetadata, mCallsite);
}

// SurfaceControl 构造函数
private SurfaceControl(SurfaceSession session, String name, 
        int w, int h, int format, int flags, SurfaceControl parent, ...) {
    
    // 调用 Native 创建
    mNativeObject = nativeCreate(session, name, w, h, format, flags,
            parent != null ? parent.mNativeObject : 0, metaParcel);
    
    mName = name;
    mWidth = w;
    mHeight = h;
}
```

```cpp
// Native 层创建
// android_view_SurfaceControl.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags,
        jlong parentObject, jobject metadataParcel) {
    
    sp<SurfaceComposerClient> client;
    if (sessionObj != nullptr) {
        client = android_view_SurfaceSession_getClient(env, sessionObj);
    } else {
        client = SurfaceComposerClient::getDefault();
    }
    
    sp<SurfaceControl> parent;
    if (parentObject != 0) {
        parent = reinterpret_cast<SurfaceControl*>(parentObject);
    }
    
    // 创建 SurfaceControl
    sp<SurfaceControl> surface = client->createSurface(
            String8(name), w, h, format, flags, parent, metadata);
    
    return reinterpret_cast<jlong>(surface.get());
}
```

---

## 3. Transaction 机制

### 3.1 Transaction 概述

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Transaction 机制                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  为什么需要 Transaction？                                            │
│  • 批量更新多个 Surface 属性                                         │
│  • 保证原子性（全部生效或全部不生效）                                 │
│  • 减少 IPC 次数，提升性能                                           │
│                                                                     │
│  Transaction 用法：                                                  │
│                                                                     │
│    Transaction t = new Transaction()                                │
│        .setPosition(surfaceA, x1, y1)    // 设置位置                │
│        .setAlpha(surfaceA, 0.5f)         // 设置透明度              │
│        .setLayer(surfaceB, 10)           // 设置层级                │
│        .setVisibility(surfaceC, false)   // 设置可见性              │
│        .apply();                         // 提交到 SF               │
│                                                                     │
│  执行时机：                                                          │
│  • apply(): 立即提交                                                 │
│  • mergeInto(): 合并到另一个 Transaction                             │
│  • 通过 Choreographer 在 VSYNC 时提交                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Transaction 类结构

```java
// SurfaceControl.Transaction
public static class Transaction implements Closeable, Parcelable {
    // Native 对象
    private long mNativeObject;
    
    // 已修改的 Surface 列表（用于释放）
    private final ArrayMap<SurfaceControl, Point> mResizedSurfaces = new ArrayMap<>();
    
    public Transaction() {
        mNativeObject = nativeCreateTransaction();
    }
    
    // ======== 属性设置方法 ========
    
    // 设置位置
    public Transaction setPosition(SurfaceControl sc, float x, float y) {
        nativeSetPosition(mNativeObject, sc.mNativeObject, x, y);
        return this;
    }
    
    // 设置大小
    public Transaction setBufferSize(SurfaceControl sc, int w, int h) {
        nativeSetSize(mNativeObject, sc.mNativeObject, w, h);
        return this;
    }
    
    // 设置透明度
    public Transaction setAlpha(SurfaceControl sc, float alpha) {
        nativeSetAlpha(mNativeObject, sc.mNativeObject, alpha);
        return this;
    }
    
    // 设置层级
    public Transaction setLayer(SurfaceControl sc, int z) {
        nativeSetLayer(mNativeObject, sc.mNativeObject, z);
        return this;
    }
    
    // 设置可见性
    public Transaction setVisibility(SurfaceControl sc, boolean visible) {
        if (visible) {
            nativeShow(mNativeObject, sc.mNativeObject);
        } else {
            nativeHide(mNativeObject, sc.mNativeObject);
        }
        return this;
    }
    
    // 设置变换矩阵
    public Transaction setMatrix(SurfaceControl sc, float dsdx, float dtdx,
            float dtdy, float dsdy) {
        nativeSetMatrix(mNativeObject, sc.mNativeObject, dsdx, dtdx, dtdy, dsdy);
        return this;
    }
    
    // 设置裁剪区域
    public Transaction setCrop(SurfaceControl sc, Rect crop) {
        nativeSetWindowCrop(mNativeObject, sc.mNativeObject,
                crop.left, crop.top, crop.right, crop.bottom);
        return this;
    }
    
    // ======== 提交方法 ========
    
    // 立即提交
    public void apply() {
        applyResizedSurfaces();
        nativeApplyTransaction(mNativeObject, false);
    }
    
    // 同步提交（等待完成）
    public void apply(boolean sync) {
        applyResizedSurfaces();
        nativeApplyTransaction(mNativeObject, sync);
    }
    
    // 合并到另一个 Transaction
    public Transaction merge(Transaction other) {
        nativeMergeTransaction(mNativeObject, other.mNativeObject);
        return this;
    }
}
```

### 3.3 Transaction Native 实现

```cpp
// frameworks/native/libs/gui/SurfaceComposerClient.cpp

class SurfaceComposerClient::Transaction {
    // 状态缓存
    std::unordered_map<sp<IBinder>, ComposerState> mComposerStates;
    
    // 显示状态
    std::unordered_map<sp<IBinder>, DisplayState> mDisplayStates;
    
public:
    // 设置位置
    Transaction& setPosition(const sp<SurfaceControl>& sc, float x, float y) {
        layer_state_t* s = getLayerState(sc);
        s->what |= layer_state_t::ePositionChanged;
        s->x = x;
        s->y = y;
        return *this;
    }
    
    // 设置透明度
    Transaction& setAlpha(const sp<SurfaceControl>& sc, float alpha) {
        layer_state_t* s = getLayerState(sc);
        s->what |= layer_state_t::eAlphaChanged;
        s->alpha = alpha;
        return *this;
    }
    
    // 提交事务
    status_t apply(bool synchronous = false) {
        // 1. 收集所有更改
        Vector<ComposerState> composerStates;
        for (auto const& [handle, state] : mComposerStates) {
            composerStates.add(state);
        }
        
        // 2. 调用 SurfaceFlinger
        sp<ISurfaceComposer> sf(ComposerService::getComposerService());
        sf->setTransactionState(
                FrameTimelineInfo{},
                composerStates,
                displayStates,
                flags,
                applyToken,
                inputWindowCommands,
                desiredPresentTime,
                ...);
        
        // 3. 清空状态
        mComposerStates.clear();
        
        return NO_ERROR;
    }
};
```

---

## 4. Transaction 在 WMS 中的使用

### 4.1 WMS 中的 Transaction 使用

```java
// WindowManagerService.java
public class WindowManagerService {
    
    // 全局 Transaction
    final SurfaceControl.Transaction mTransaction;
    
    // 窗口位置更新
    void setWindowPosition(WindowState win, float x, float y) {
        mTransaction.setPosition(win.getSurfaceControl(), x, y);
    }
    
    // 窗口大小更新
    void setWindowSize(WindowState win, int width, int height) {
        mTransaction.setBufferSize(win.getSurfaceControl(), width, height);
    }
    
    // 应用所有更改
    void applyTransaction() {
        // 收集所有窗口的 Surface 更新
        mRoot.forAllWindows(w -> {
            w.prepareSurfaceLocked(mTransaction);
        }, false);
        
        // 提交事务
        mTransaction.apply();
    }
}
```

### 4.2 WindowState 中的 Transaction 使用

```java
// WindowState.java
void prepareSurfaceLocked(SurfaceControl.Transaction t) {
    // 如果不需要更新，直接返回
    if (!mSurfaceControl.isValid()) {
        return;
    }
    
    // 1. 更新位置
    if (mWindowFrames.hasContentChanged()) {
        t.setPosition(mSurfaceControl, mWindowFrames.mFrame.left, 
                mWindowFrames.mFrame.top);
    }
    
    // 2. 更新透明度
    if (mAnimator.hasAlphaChanged()) {
        t.setAlpha(mSurfaceControl, mAnimator.mAlpha);
    }
    
    // 3. 更新变换
    if (mAnimator.hasMatrixChanged()) {
        t.setMatrix(mSurfaceControl, 
                mAnimator.mDsdx, mAnimator.mDtdx,
                mAnimator.mDtdy, mAnimator.mDsdy);
    }
    
    // 4. 更新可见性
    if (mWinAnimator.mLastHidden != mAnimatingExit) {
        if (mAnimatingExit) {
            t.hide(mSurfaceControl);
        } else {
            t.show(mSurfaceControl);
        }
    }
}
```

---

## 5. Layer 层级结构

### 5.1 SurfaceControl 层级

```java
// SurfaceControl 支持父子层级关系
// 子层级会继承父层级的属性（位置、透明度等）

// 创建子 SurfaceControl
SurfaceControl childSurface = new SurfaceControl.Builder()
        .setParent(parentSurface)  // 设置父层
        .setName("ChildWindow")
        .build();

// 或者通过 Transaction 设置
new Transaction()
        .reparent(childSurface, parentSurface)  // 重新设置父层
        .apply();
```

### 5.2 WMS 中的层级结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WMS SurfaceControl 层级                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  RootSurfaceControl (显示根)                                        │
│     │                                                               │
│     ├── DisplayContent.SurfaceControl                               │
│     │      │                                                        │
│     │      ├── TaskDisplayArea.SurfaceControl                       │
│     │      │      │                                                 │
│     │      │      ├── Task.SurfaceControl                           │
│     │      │      │      │                                          │
│     │      │      │      ├── ActivityRecord.SurfaceControl          │
│     │      │      │      │      │                                   │
│     │      │      │      │      └── WindowState.SurfaceControl      │
│     │      │      │      │             (应用窗口)                    │
│     │      │      │      │                                          │
│     │      │      │      └── ActivityRecord.SurfaceControl          │
│     │      │      │                                                 │
│     │      │      └── Task.SurfaceControl                           │
│     │      │                                                        │
│     │      ├── ImeContainer.SurfaceControl                          │
│     │      │      └── WindowState.SurfaceControl (输入法)           │
│     │      │                                                        │
│     │      └── SystemUIContainer.SurfaceControl                     │
│     │             ├── WindowState.SurfaceControl (状态栏)           │
│     │             └── WindowState.SurfaceControl (导航栏)           │
│     │                                                               │
│     └── ... (其他 Display)                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 同步与屏障

### 6.1 SyncTransaction

```java
// 同步提交（等待 SF 处理完成）
transaction.apply(true /* sync */);

// 或者使用回调
transaction.addTransactionCommittedListener(executor, () -> {
    // 事务提交完成
});
```

### 6.2 Barrier 机制

```java
// 等待特定 Surface 的帧
Transaction t = new Transaction();

// 添加屏障：等待 surface 的下一帧
t.setFrameTimeline(frameTimelineInfo);

// 设置期望显示时间
t.setDesiredPresentTime(expectedPresentTime);

t.apply();
```

---

## 7. 本章小结

本文详细介绍了 SurfaceControl 与 Transaction：

1. **SurfaceControl**：Surface 的控制句柄，管理 Layer 属性
2. **SurfaceControl vs Surface**：控制属性 vs 绘制内容
3. **Transaction**：批量、原子性更新 Surface 属性
4. **Layer 层级**：通过 parent 关系构建层级树
5. **WMS 使用**：通过 Transaction 批量更新所有窗口

下一篇文章将介绍 GraphicBuffer 与显存管理。

---

## 参考资料

1. Android 源码：`frameworks/base/core/java/android/view/SurfaceControl.java`
2. Android 源码：`frameworks/native/libs/gui/SurfaceComposerClient.cpp`
3. [SurfaceControl 文档](https://source.android.com/docs/core/graphics/arch-sf-hwc)
