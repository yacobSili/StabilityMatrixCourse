# View 系统与窗口绑定

## 引言

View 系统是 Android UI 的核心，而 ViewRootImpl 是 View 系统与窗口系统的桥梁。本文将详细介绍 ViewRootImpl 的工作原理、Surface 创建过程以及绘制触发机制。

---

## 1. ViewRootImpl 概述

### 1.1 ViewRootImpl 的角色

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ViewRootImpl 的桥梁角色                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                          View 系统                                   │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                        View 树                                 │  │
│  │    DecorView ─► ViewGroup ─► View ─► View                     │  │
│  │        │                                                       │  │
│  │    测量 (measure) / 布局 (layout) / 绘制 (draw)                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              │ mView (根 View)                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      ViewRootImpl                              │  │
│  │                                                                │  │
│  │   • 管理 View 树的根                                            │  │
│  │   • 与 WMS 通信（IWindowSession）                               │  │
│  │   • 管理 Surface                                                │  │
│  │   • 处理输入事件（InputChannel）                                 │  │
│  │   • 调度测量/布局/绘制                                          │  │
│  │   • 管理 Choreographer 回调                                     │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                 │                              │                    │
│    IWindowSession                        InputChannel               │
│                 │                              │                    │
│                 ▼                              ▼                    │
│  ┌──────────────────────────┐    ┌──────────────────────────┐      │
│  │          WMS             │    │    InputDispatcher       │      │
│  │   (窗口管理)              │    │    (输入分发)            │      │
│  └──────────────────────────┘    └──────────────────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 ViewRootImpl 核心成员

```java
// ViewRootImpl.java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    
    // ======== View 相关 ========
    View mView;                           // 根 View (DecorView)
    final View.AttachInfo mAttachInfo;    // 附加信息，传递给整个 View 树
    
    // ======== Surface 相关 ========
    final Surface mSurface = new Surface();           // 绘图表面
    private final SurfaceControl mSurfaceControl;     // Surface 控制句柄
    final SurfaceSession mSurfaceSession;             // Surface 会话
    
    // ======== WMS 通信 ========
    final IWindowSession mWindowSession;  // 与 WMS 的会话
    final W mWindow;                      // IWindow 实现，WMS 回调
    
    // ======== 输入事件 ========
    InputChannel mInputChannel;           // 输入通道
    WindowInputEventReceiver mInputEventReceiver;  // 输入事件接收器
    
    // ======== 绘制调度 ========
    Choreographer mChoreographer;         // 帧调度器
    final TraversalRunnable mTraversalRunnable;  // 遍历任务
    boolean mTraversalScheduled;          // 是否已调度遍历
    
    // ======== 窗口属性 ========
    final WindowManager.LayoutParams mWindowAttributes;
    int mWidth, mHeight;                  // 窗口大小
    final Rect mWinFrame = new Rect();    // 窗口位置
    
    // ======== 状态标记 ========
    boolean mFirst = true;                // 是否首次遍历
    boolean mAdded;                       // 是否已添加到 WMS
    boolean mRemoved;                     // 是否已移除
}
```

---

## 2. ViewRootImpl 创建与初始化

### 2.1 创建时机

```java
// WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params, ...) {
    ViewRootImpl root;
    View panelParentView = null;
    
    synchronized (mLock) {
        // 1. 创建 ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);
        
        // 2. 保存引用
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        
        // 3. 设置 View（触发窗口添加）
        root.setView(view, wparams, panelParentView, userId);
    }
}
```

### 2.2 ViewRootImpl 构造函数

```java
// ViewRootImpl.java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    
    // 1. 获取 WMS 会话
    mWindowSession = WindowManagerGlobal.getWindowSession();
    
    // 2. 创建 Surface（空 Surface，后续填充）
    mSurface = new Surface();
    mSurfaceControl = new SurfaceControl();
    
    // 3. 创建 AttachInfo
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display,
            this, mHandler, this, context);
    
    // 4. 获取 Choreographer
    mChoreographer = Choreographer.getInstance();
    
    // 5. 创建 IWindow 实现
    mWindow = new W(this);
    
    // 6. 初始化密度等信息
    mDisplayManager = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
}
```

### 2.3 setView() - 绑定 View 并添加窗口

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs,
        View panelParentView, int userId) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            
            // 1. 请求布局（会在下一帧执行）
            requestLayout();
            
            // 2. 创建 InputChannel
            InputChannel inputChannel = new InputChannel();
            
            // 3. 添加窗口到 WMS
            res = mWindowSession.addToDisplayAsUser(
                    mWindow,             // IWindow 回调
                    mWindowAttributes,   // 窗口属性
                    getHostVisibility(),
                    mDisplay.getDisplayId(),
                    userId,
                    mInsetsController.getRequestedVisibilities(),
                    inputChannel,        // 输出：输入通道
                    mTempInsets,
                    mTempControls,
                    mSurfaceControl      // 输出：SurfaceControl
            );
            
            // 4. 设置输入通道
            if (inputChannel != null) {
                mInputChannel = inputChannel;
                mInputEventReceiver = new WindowInputEventReceiver(
                        inputChannel, Looper.myLooper());
            }
            
            // 5. 关联 View
            view.assignParent(this);
            
            mAdded = true;
        }
    }
}
```

---

## 3. Surface 创建与管理

### 3.1 Surface 创建时机

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Surface 创建时机                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  setView() 阶段：                                                    │
│  └─► addToDisplayAsUser()                                           │
│       └─► WMS.addWindow()                                           │
│            └─► 创建 SurfaceControl（只是句柄，无实际缓冲区）           │
│                                                                     │
│  performTraversals() 阶段：                                          │
│  └─► relayoutWindow()                                               │
│       └─► WMS.relayoutWindow()                                      │
│            └─► 分配实际的 Surface 缓冲区                              │
│                 └─► 应用可以开始绘制                                  │
│                                                                     │
│  为什么分两阶段？                                                     │
│  1. addWindow 时窗口大小可能还未确定                                  │
│  2. relayoutWindow 时大小已确定，可分配正确大小的缓冲区                │
│  3. 延迟分配节省内存                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 relayoutWindow 详解

```java
// ViewRootImpl.java
private int relayoutWindow(WindowManager.LayoutParams params,
        int viewVisibility, boolean insetsPending) throws RemoteException {
    
    // 调用 WMS 重新布局
    int relayoutResult = mWindowSession.relayout(
            mWindow,                    // IWindow
            params,                     // 窗口属性
            requestedWidth,             // 请求宽度
            requestedHeight,            // 请求高度
            viewVisibility,             // 可见性
            insetsPending,
            mWinFrame,                  // 输出：窗口位置
            mPendingDisplayCutout,      // 输出：刘海区域
            mPendingMergedConfiguration,
            mSurfaceControl,            // 输出：SurfaceControl
            mTempInsets,
            mTempControls,
            ...
    );
    
    // 从 SurfaceControl 获取 Surface
    if (mSurfaceControl.isValid()) {
        if (!useBLAST()) {
            // 传统方式：直接从 SurfaceControl 复制
            mSurface.copyFrom(mSurfaceControl);
        } else {
            // BLAST 方式：使用 BLASTBufferQueue
            updateBlastSurfaceIfNeeded();
        }
    }
    
    return relayoutResult;
}
```

### 3.3 WMS 中的 Surface 创建

```java
// WindowManagerService.java
public int relayoutWindow(Session session, IWindow client,
        WindowManager.LayoutParams attrs, int requestedWidth, int requestedHeight,
        int viewVisibility, ..., SurfaceControl outSurfaceControl, ...) {
    
    synchronized (mGlobalLock) {
        WindowState win = windowForClientLocked(session, client, false);
        
        // 计算窗口大小和位置
        win.setRequestedSize(requestedWidth, requestedHeight);
        performLayoutLockedInner(...);
        
        // 创建或更新 Surface
        if (shouldRelayoutSurface(win, ...)) {
            result = createSurfaceControl(outSurfaceControl, win, winAnimator);
        }
    }
}

// WindowState.java
void createSurfaceControl(SurfaceControl.Builder b) {
    // 创建 SurfaceControl
    mSurfaceControl = b.setName(mWinAnimator.mSurfaceTitle)
            .setContainerLayer()
            .setCallsite("WindowState.createSurfaceControl")
            .build();
    
    // 通过 Transaction 设置属性
    mWmService.mTransactionFactory.get()
            .setLayer(mSurfaceControl, mLayer)
            .apply();
}
```

---

## 4. 绘制触发机制

### 4.1 Choreographer 机制

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Choreographer 帧调度                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  requestLayout() / invalidate()                                     │
│         │                                                           │
│         ▼                                                           │
│  scheduleTraversals()                                               │
│         │                                                           │
│         ▼                                                           │
│  mChoreographer.postCallback(CALLBACK_TRAVERSAL, mTraversalRunnable)│
│         │                                                           │
│         │  等待 VSYNC                                               │
│         ▼                                                           │
│  ──── VSYNC-app 信号到达 ────                                       │
│         │                                                           │
│         ▼                                                           │
│  Choreographer.doFrame()                                            │
│         │                                                           │
│         ├─► CALLBACK_INPUT      (输入事件)                          │
│         ├─► CALLBACK_ANIMATION  (动画)                              │
│         ├─► CALLBACK_INSETS_ANIMATION                               │
│         ├─► CALLBACK_TRAVERSAL  ─► doTraversal() ─► performTraversals()
│         └─► CALLBACK_COMMIT                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 scheduleTraversals()

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        
        // 1. 设置同步屏障（阻止同步消息）
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        
        // 2. 向 Choreographer 注册回调
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL,
                mTraversalRunnable,  // 回调任务
                null
        );
        
        // 3. 通知渲染线程
        notifyRendererOfFramePending();
    }
}

// 遍历任务
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        
        // 1. 移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        
        // 2. 执行遍历
        performTraversals();
    }
}
```

### 4.3 performTraversals() - 核心遍历流程

```java
// ViewRootImpl.java
private void performTraversals() {
    final View host = mView;
    
    // ======== 阶段 0: 准备 ========
    if (mFirst) {
        // 首次遍历的初始化
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
    }
    
    // ======== 阶段 1: 窗口属性处理 ========
    if (mFirst || mAttachInfo.mViewVisibilityChanged) {
        // 处理可见性变化
    }
    
    // ======== 阶段 2: 请求 WMS 布局（如需要）========
    boolean windowShouldResize = ...;
    if (mFirst || windowShouldResize || viewVisibilityChanged || ...) {
        // 调用 WMS 重新布局，获取 Surface
        relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
    }
    
    // ======== 阶段 3: 测量 ========
    if (mFirst || mLayoutRequested || ...) {
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    
    // ======== 阶段 4: 布局 ========
    final boolean didLayout = layoutRequested && ...;
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
    }
    
    // ======== 阶段 5: 绘制 ========
    boolean cancelDraw = ...;
    if (!cancelDraw) {
        performDraw();
    }
    
    mFirst = false;
}
```

### 4.4 测量、布局、绘制三部曲

```java
// ViewRootImpl.java

// 1. 测量
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

// 2. 布局
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mInLayout = true;
    final View host = mView;
    
    // 布局根 View
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    
    // 处理布局期间的 requestLayout 请求
    if (mLayoutRequesters != null) {
        // 二次布局
    }
    
    mInLayout = false;
}

// 3. 绘制
private void performDraw() {
    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    mFullRedrawNeeded = false;
    
    // 实际绘制
    draw(fullRedrawNeeded);
}

private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        // 硬件加速绘制
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 软件绘制
        drawSoftware(surface, mAttachInfo, ...);
    }
}

private boolean drawSoftware(Surface surface, ...) {
    // 1. 锁定 Canvas
    Canvas canvas = surface.lockCanvas(dirty);
    
    // 2. 绘制 View 树
    mView.draw(canvas);
    
    // 3. 解锁并提交
    surface.unlockCanvasAndPost(canvas);
    
    return true;
}
```

---

## 5. View.AttachInfo 详解

### 5.1 AttachInfo 作用

```java
// View.java 内部类
final static class AttachInfo {
    // ViewRootImpl 引用
    final ViewRootImpl mViewRootImpl;
    
    // WMS 会话
    final IWindowSession mSession;
    
    // 窗口引用
    final IWindow mWindow;
    
    // Handler（用于 post 操作）
    final Handler mHandler;
    
    // 硬件加速渲染器
    ThreadedRenderer mThreadedRenderer;
    
    // 窗口属性
    boolean mHardwareAccelerated;
    float mApplicationScale;
    
    // 显示信息
    Display mDisplay;
    
    // 标记
    boolean mHasWindowFocus;
    boolean mWindowVisibility;
    int mSystemUiVisibility;
    
    // 用于 invalidate 区域收集
    final Rect mTmpInvalRect = new Rect();
    final int[] mTmpLocation = new int[2];
}
```

### 5.2 AttachInfo 传递

```java
// View.java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    
    // 回调
    onAttachedToWindow();
    
    // 传递给子 View
    // ViewGroup.dispatchAttachedToWindow() 会遍历子 View
}

// 通过 AttachInfo，View 可以访问：
// 1. 获取 Handler 执行 post 操作
public boolean post(Runnable action) {
    if (mAttachInfo != null) {
        return mAttachInfo.mHandler.post(action);
    }
    // 未 attach 时存入队列
    getRunQueue().post(action);
    return true;
}

// 2. 获取窗口可见性
public int getWindowVisibility() {
    return mAttachInfo != null ? mAttachInfo.mWindowVisibility : GONE;
}

// 3. 判断是否硬件加速
public boolean isHardwareAccelerated() {
    return mAttachInfo != null && mAttachInfo.mHardwareAccelerated;
}
```

---

## 6. 输入事件接收

### 6.1 InputChannel 建立

```java
// ViewRootImpl.java setView() 中
// 1. 创建 InputChannel
InputChannel inputChannel = new InputChannel();

// 2. 添加窗口时传给 WMS
mWindowSession.addToDisplayAsUser(..., inputChannel, ...);

// 3. 创建事件接收器
mInputEventReceiver = new WindowInputEventReceiver(
        inputChannel, Looper.myLooper());
```

### 6.2 事件接收与分发

```java
// ViewRootImpl.java
final class WindowInputEventReceiver extends InputEventReceiver {
    
    @Override
    public void onInputEvent(InputEvent event) {
        // 接收到输入事件
        enqueueInputEvent(event, this, 0, true);
    }
}

void enqueueInputEvent(InputEvent event, ...) {
    // 加入队列
    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
    
    if (last == null) {
        mPendingInputEventHead = q;
    } else {
        last.mNext = q;
    }
    mPendingInputEventCount += 1;
    
    // 处理事件
    doProcessInputEvents();
}

void doProcessInputEvents() {
    while (mPendingInputEventHead != null) {
        QueuedInputEvent q = mPendingInputEventHead;
        mPendingInputEventHead = q.mNext;
        
        // 通过 InputStage 链处理
        deliverInputEvent(q);
    }
}
```

### 6.3 InputStage 责任链

```
┌─────────────────────────────────────────────────────────────────────┐
│                    InputStage 处理链                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  NativePreImeInputStage                                             │
│         │                                                           │
│         ▼                                                           │
│  ViewPreImeInputStage     ─► View.dispatchKeyEventPreIme()          │
│         │                                                           │
│         ▼                                                           │
│  ImeInputStage            ─► 输入法处理                              │
│         │                                                           │
│         ▼                                                           │
│  EarlyPostImeInputStage                                             │
│         │                                                           │
│         ▼                                                           │
│  NativePostImeInputStage                                            │
│         │                                                           │
│         ▼                                                           │
│  ViewPostImeInputStage    ─► View.dispatchKeyEvent()                │
│         │                    View.dispatchTouchEvent()              │
│         ▼                                                           │
│  SyntheticInputStage      ─► 合成事件处理                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 硬件加速渲染

### 7.1 ThreadedRenderer

```java
// ViewRootImpl.java
private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
    // 创建硬件渲染器
    mAttachInfo.mThreadedRenderer = ThreadedRenderer.create(mContext, translucent, ...);
}

// 硬件加速绘制
private void draw(boolean fullRedrawNeeded) {
    if (mAttachInfo.mThreadedRenderer != null && 
            mAttachInfo.mThreadedRenderer.isEnabled()) {
        // 硬件加速路径
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 软件绘制路径
        drawSoftware(surface, ...);
    }
}
```

### 7.2 RenderThread 工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    硬件加速渲染流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  主线程 (UI Thread)              渲染线程 (RenderThread)             │
│         │                                  │                        │
│  View.draw()                               │                        │
│    └─► 记录绘制指令                         │                        │
│         (DisplayList)                      │                        │
│         │                                  │                        │
│         ├──── 同步 DisplayList ───────────►│                        │
│         │                                  │                        │
│         │                         执行 DisplayList                  │
│         │                           │                               │
│         │                         OpenGL ES / Vulkan 调用           │
│         │                           │                               │
│         │                         SwapBuffers                       │
│         │                           │                               │
│         │                         提交到 SurfaceFlinger              │
│         │                                  │                        │
│  准备下一帧 ◄───── 同步完成 ────────────────│                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. 本章小结

本文详细介绍了 View 系统与窗口的绑定：

1. **ViewRootImpl**：View 树与 WMS 的桥梁
2. **Surface 创建**：addWindow 创建句柄，relayoutWindow 分配缓冲区
3. **绘制触发**：Choreographer 调度，performTraversals 执行
4. **输入事件**：InputChannel 接收，InputStage 链处理
5. **硬件加速**：RenderThread 异步渲染

下一篇文章将详细介绍 WindowManagerService 的架构设计。

---

## 参考资料

1. Android 源码：`frameworks/base/core/java/android/view/ViewRootImpl.java`
2. Android 源码：`frameworks/base/core/java/android/view/Choreographer.java`
3. Android 源码：`frameworks/base/graphics/java/android/graphics/ThreadedRenderer.java`
4. [硬件加速](https://developer.android.com/guide/topics/graphics/hardware-accel)
