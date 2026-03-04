# WindowManagerService 架构详解

## 引言

WindowManagerService (WMS) 是 Android 系统中最重要的服务之一，负责管理所有窗口的创建、布局、层级和焦点。本文将详细介绍 WMS 的架构设计、核心组件和启动流程。

---

## 1. WMS 在系统中的位置

### 1.1 系统服务架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                       System Server 进程                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  核心服务 (Core Services)                     │   │
│  │                                                              │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐                │   │
│  │  │    AMS    │  │    WMS    │  │    PMS    │                │   │
│  │  │(Activity) │◄►│ (Window)  │◄►│ (Package) │                │   │
│  │  └───────────┘  └─────┬─────┘  └───────────┘                │   │
│  │                       │                                      │   │
│  │       ┌───────────────┼───────────────┐                     │   │
│  │       ▼               ▼               ▼                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐                │   │
│  │  │    IMS    │  │    DMS    │  │   ATMS    │                │   │
│  │  │  (Input)  │  │ (Display) │  │(Activity  │                │   │
│  │  │           │  │           │  │ Task)     │                │   │
│  │  └───────────┘  └───────────┘  └───────────┘                │   │
│  │                                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ Binder
┌───────────────────────────────▼─────────────────────────────────────┐
│                        SurfaceFlinger 进程                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 WMS 核心职责

| 职责 | 描述 | 相关方法 |
|-----|------|---------|
| 窗口管理 | 添加、移除、更新窗口 | addWindow/removeWindow |
| 布局计算 | 计算窗口大小和位置 | performLayoutLockedInner |
| 层级管理 | Z-Order 排序 | assignWindowLayers |
| 焦点管理 | 维护焦点窗口 | updateFocusedWindowLocked |
| 动画调度 | 窗口动画控制 | WindowAnimator |
| Surface 管理 | 创建和配置 Surface | SurfaceControl |
| 输入协调 | 与 IMS 协作 | InputMonitor |

---

## 2. WMS 核心架构

### 2.1 类图结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WindowManagerService                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  核心管理器:                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │RootWindowContain│  │  DisplayContent │  │   WindowToken   │     │
│  │      er         │──│                 │──│                 │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│           │                   │                    │                │
│           │                   │                    │                │
│           ▼                   ▼                    ▼                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │  WindowState[]  │  │ DisplayPolicy   │  │  WindowState    │     │
│  │   (所有窗口)     │  │  (显示策略)      │  │   (单个窗口)     │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
│  辅助组件:                                                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │  WindowAnimator │  │  InputMonitor   │  │    Session      │     │
│  │   (窗口动画)     │  │   (输入监控)     │  │   (客户端会话)   │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
│  策略组件:                                                           │
│  ┌─────────────────┐  ┌─────────────────┐                          │
│  │ WindowManager   │  │  TaskSnapshot   │                          │
│  │    Policy       │  │   Controller    │                          │
│  │   (窗口策略)     │  │   (任务快照)     │                          │
│  └─────────────────┘  └─────────────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心成员变量

```java
// WindowManagerService.java
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
    
    // ======== 全局锁 ========
    final WindowManagerGlobalLock mGlobalLock;
    
    // ======== 窗口容器根 ========
    RootWindowContainer mRoot;
    
    // ======== 策略 ========
    final WindowManagerPolicy mPolicy;  // PhoneWindowManager
    
    // ======== 动画 ========
    final WindowAnimator mAnimator;
    
    // ======== 输入 ========
    final InputManagerService mInputManager;
    final InputMonitor mInputMonitor;
    
    // ======== 会话管理 ========
    final ArrayMap<IBinder, Session> mSessions = new ArrayMap<>();
    
    // ======== 显示管理 ========
    DisplayManager mDisplayManager;
    
    // ======== Handler ========
    final H mH = new H(this);
    
    // ======== 事务 ========
    final SurfaceControl.Transaction mTransaction;
    
    // ======== 配置 ========
    boolean mDisplayReady;
    boolean mSystemBooted;
}
```

---

## 3. WMS 启动流程

### 3.1 启动序列

```
┌─────────────────────────────────────────────────────────────────────┐
│                      WMS 启动流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SystemServer.startOtherServices()                                  │
│         │                                                           │
│         ▼                                                           │
│  1. WindowManagerService.main()                                     │
│         │  创建 WMS 实例                                             │
│         │  初始化 DisplayContent                                     │
│         │  初始化 WindowAnimator                                     │
│         ▼                                                           │
│  2. WMS.onInitReady()                                               │
│         │  初始化策略 (PhoneWindowManager)                           │
│         │  设置 DisplayPolicy                                        │
│         ▼                                                           │
│  3. WMS.displayReady()                                              │
│         │  配置显示参数                                              │
│         │  初始化 TaskSnapshotController                             │
│         ▼                                                           │
│  4. WMS.systemReady()                                               │
│         │  mPolicy.systemReady()                                    │
│         │  启动完成，可以接收窗口请求                                  │
│         ▼                                                           │
│  5. 系统启动完成                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 main() 方法

```java
// WindowManagerService.java
public static WindowManagerService main(final Context context,
        final InputManagerService im, final boolean showBootMsgs,
        final boolean onlyCore, WindowManagerPolicy policy,
        ActivityTaskManagerService atm, ...) {
    
    // 在 DisplayThread 中创建 WMS
    DisplayThread.getHandler().runWithScissors(() -> {
        sInstance = new WindowManagerService(context, im, showBootMsgs,
                onlyCore, policy, atm, ...);
    }, 0);
    
    return sInstance;
}

// 构造函数
private WindowManagerService(Context context, InputManagerService inputManager,
        boolean showBootMsgs, boolean onlyCore,
        WindowManagerPolicy policy, ActivityTaskManagerService atm, ...) {
    
    mContext = context;
    mInputManager = inputManager;
    mPolicy = policy;
    mAtmService = atm;
    
    // 1. 创建全局锁
    mGlobalLock = atm.getGlobalLock();
    
    // 2. 初始化 Handler
    mAnimationHandler = new Handler(AnimationThread.getHandler().getLooper());
    
    // 3. 创建窗口容器根
    mRoot = new RootWindowContainer(this);
    
    // 4. 创建动画器
    mAnimator = new WindowAnimator(this);
    
    // 5. 创建事务工厂
    mTransactionFactory = SurfaceControl.Transaction::new;
    mTransaction = mTransactionFactory.get();
    
    // 6. 初始化策略
    mPolicy.init(context, this);
    
    // 7. 获取 DisplayManager
    mDisplayManager = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
    mDisplayManager.registerDisplayListener(this, null);
    
    // 8. 初始化所有显示屏
    mDisplayReady = true;
    mRoot.forAllDisplays(dc -> dc.initializeDisplayBaseInfo());
}
```

### 3.3 displayReady() 和 systemReady()

```java
// WindowManagerService.java
public void displayReady() {
    synchronized (mGlobalLock) {
        // 1. 配置所有显示屏
        mRoot.forAllDisplays(DisplayContent::reconfigureDisplayLocked);
        
        // 2. 请求遍历
        requestTraversal();
    }
}

public void systemReady() {
    // 1. 策略就绪
    mPolicy.systemReady();
    
    // 2. 开始处理任务快照
    mTaskSnapshotController.systemReady();
    
    // 3. 标记系统启动完成
    mSystemBooted = true;
}
```

---

## 4. 窗口添加流程

### 4.1 addWindow 完整流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    addWindow 完整流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  App: ViewRootImpl.setView()                                        │
│         │                                                           │
│         │ IWindowSession.addToDisplayAsUser()                       │
│         ▼                                                           │
│  WMS: Session.addToDisplayAsUser()                                  │
│         │                                                           │
│         │ 转发                                                       │
│         ▼                                                           │
│  WMS: addWindow()                                                   │
│         │                                                           │
│         ├─► 1. 权限检查 (mPolicy.checkAddPermission)                 │
│         │                                                           │
│         ├─► 2. 获取 DisplayContent                                   │
│         │       getDisplayContentOrCreate(displayId)                │
│         │                                                           │
│         ├─► 3. 获取或创建 WindowToken                                │
│         │       displayContent.getWindowToken(attrs.token)          │
│         │                                                           │
│         ├─► 4. 创建 WindowState                                      │
│         │       new WindowState(this, session, client, token, ...)  │
│         │                                                           │
│         ├─► 5. 添加到 WindowToken                                    │
│         │       win.mToken.addWindow(win)                           │
│         │                                                           │
│         ├─► 6. 创建 SurfaceControl                                   │
│         │       win.createSurfaceControl(outSurfaceControl)         │
│         │                                                           │
│         ├─► 7. 配置输入通道                                          │
│         │       win.openInputChannel(outInputChannel)               │
│         │                                                           │
│         ├─► 8. 更新焦点                                              │
│         │       updateFocusedWindowLocked()                         │
│         │                                                           │
│         └─► 9. 返回结果                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 addWindow 源码分析

```java
// WindowManagerService.java
public int addWindow(Session session, IWindow client, LayoutParams attrs,
        int viewVisibility, int displayId, int requestUserId,
        InsetsVisibilities requestedVisibilities, InputChannel outInputChannel,
        InsetsState outInsetsState, InsetsSourceControl[] outActiveControls,
        Rect outAttachedFrame, float[] outSizeCompatScale) {
    
    // 1. 参数校验
    if (client == null || attrs == null) {
        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
    }
    
    final int type = attrs.type;
    
    synchronized (mGlobalLock) {
        // 2. 权限检查
        int res = mPolicy.checkAddPermission(type, ...);
        if (res != ADD_OKAY) {
            return res;
        }
        
        // 3. 获取 DisplayContent
        final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);
        if (displayContent == null) {
            return WindowManagerGlobal.ADD_INVALID_DISPLAY;
        }
        
        // 4. 获取 WindowToken
        WindowToken token = displayContent.getWindowToken(attrs.token);
        
        // 5. Token 检查和创建
        if (token == null) {
            if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
                // 应用窗口必须有有效 token
                return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
            }
            // 其他类型可以自动创建 token
            token = new WindowToken.Builder(this, attrs.token, type)
                    .setDisplayContent(displayContent)
                    .build();
        }
        
        // 6. 创建 WindowState
        final WindowState win = new WindowState(this, session, client, token,
                parentWindow, appOp[0], attrs, viewVisibility, session.mUid,
                userId, session.mCanAddInternalSystemWindow);
        
        // 7. 检查窗口是否可以添加
        res = displayPolicy.validateAddingWindowLw(attrs, ...);
        if (res != ADD_OKAY) {
            return res;
        }
        
        // 8. 打开输入通道
        if (outInputChannel != null) {
            win.openInputChannel(outInputChannel);
        }
        
        // 9. 添加窗口
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        win.mToken.addWindow(win);
        
        // 10. 创建 SurfaceControl
        win.createSurfaceControl(mTransaction);
        
        // 11. 更新焦点
        if (win.canReceiveKeys()) {
            updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS, false);
        }
        
        // 12. 分配层级
        displayContent.assignWindowLayers(false);
        
        return res;
    }
}
```

---

## 5. WMS Handler 消息处理

### 5.1 Handler H

```java
// WindowManagerService.java 内部类
final class H extends Handler {
    // 消息类型
    public static final int REPORT_FOCUS_CHANGE = 2;
    public static final int REPORT_LOSING_FOCUS = 3;
    public static final int WINDOW_FREEZE_TIMEOUT = 11;
    public static final int PERSIST_ANIMATION_SCALE = 14;
    public static final int FORCE_GC = 15;
    public static final int ENABLE_SCREEN = 16;
    public static final int APP_FREEZE_TIMEOUT = 17;
    public static final int WALLPAPER_DRAW_PENDING_TIMEOUT = 39;
    public static final int UPDATE_MULTI_WINDOW_STACKS = 41;
    public static final int WINDOW_REPLACEMENT_TIMEOUT = 46;
    
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case REPORT_FOCUS_CHANGE: {
                // 报告焦点变化
                final WindowState lastFocus;
                final WindowState newFocus;
                synchronized (mGlobalLock) {
                    lastFocus = mLastFocus;
                    newFocus = mCurrentFocus;
                }
                // 通知应用
                if (lastFocus != null) {
                    lastFocus.reportFocusChangedSerialized(false);
                }
                if (newFocus != null) {
                    newFocus.reportFocusChangedSerialized(true);
                }
                break;
            }
            
            case WINDOW_FREEZE_TIMEOUT: {
                // 窗口冻结超时
                synchronized (mGlobalLock) {
                    DisplayContent dc = (DisplayContent) msg.obj;
                    dc.onWindowFreezeTimeout();
                }
                break;
            }
            
            // ... 其他消息处理
        }
    }
}
```

---

## 6. WMS 与其他服务的交互

### 6.1 与 AMS/ATMS 的交互

```java
// WMS 与 Activity 管理的交互
public class WindowManagerService {
    final ActivityTaskManagerService mAtmService;
    
    // 窗口可见性变化时通知 AMS
    void notifyAppVisibilityChange(ActivityRecord activityRecord, boolean visible) {
        mAtmService.notifyAppVisibilityChange(activityRecord, visible);
    }
    
    // 窗口焦点变化时通知 AMS
    void updateFocusedWindowLocked(...) {
        // 找到新的焦点窗口
        WindowState newFocus = computeFocusedWindowLocked();
        if (mCurrentFocus != newFocus) {
            // 通知 ATMS 焦点变化
            mAtmService.setFocusedActivityLocked(...);
        }
    }
}
```

### 6.2 与 IMS 的交互

```java
// WMS 与 Input 系统的交互
public class WindowManagerService {
    final InputManagerService mInputManager;
    
    // 更新输入窗口
    void updateInputWindowsLw(boolean force) {
        mInputMonitor.updateInputWindowsLw(force);
    }
    
    // 设置输入焦点
    void setFocusedWindow(String reason, ...) {
        mInputManager.setFocusedWindow(request);
    }
}

// InputMonitor - 输入监控
class InputMonitor {
    // 更新输入窗口信息到 InputDispatcher
    void updateInputWindowsLw(boolean force) {
        // 遍历所有窗口，收集输入窗口信息
        mService.mRoot.forAllWindows(this::populateInputWindowHandle, true);
        
        // 提交到 InputManager
        mService.mInputManager.setInputWindows(mInputWindowHandles, ...);
    }
}
```

### 6.3 与 SurfaceFlinger 的交互

```java
// WMS 与 SF 的交互
public class WindowManagerService {
    // 通过 Transaction 与 SF 通信
    final SurfaceControl.Transaction mTransaction;
    
    // 应用 Transaction
    void applyTransaction() {
        // 收集所有窗口的 Surface 变化
        mRoot.forAllWindows(w -> {
            w.prepareSurfaceLocked(mTransaction);
        }, false);
        
        // 提交到 SurfaceFlinger
        SurfaceControl.mergeToGlobalTransaction(mTransaction);
    }
}
```

---

## 7. WMS 核心配置

### 7.1 重要系统属性

```java
// WMS 相关系统属性
ro.surface_flinger.max_frame_buffer_acquired_buffers  // 最大缓冲区数
ro.surface_flinger.vsync_event_phase_offset_ns        // VSYNC 偏移
persist.debug.wm.verbose                               // WMS 详细日志
debug.wm.enabled                                       // 调试开关
```

### 7.2 窗口动画配置

```java
// 窗口动画时长配置 (Settings.Global)
Settings.Global.WINDOW_ANIMATION_SCALE     // 窗口动画缩放
Settings.Global.TRANSITION_ANIMATION_SCALE // 过渡动画缩放
Settings.Global.ANIMATOR_DURATION_SCALE    // 动画时长缩放

// WMS 中使用
void setAnimationsDisabled(boolean disabled) {
    mAnimationsDisabled = disabled;
}

float getWindowAnimationScaleLocked() {
    return mAnimationsDisabled ? 0 : mWindowAnimationScaleSetting;
}
```

---

## 8. 本章小结

本文详细介绍了 WMS 的架构设计：

1. **系统位置**：System Server 核心服务，与 AMS、IMS、SF 紧密协作
2. **核心架构**：RootWindowContainer → DisplayContent → WindowToken → WindowState
3. **启动流程**：main() → onInitReady() → displayReady() → systemReady()
4. **addWindow**：权限检查 → 创建 WindowState → 创建 SurfaceControl → 更新焦点
5. **服务交互**：与 AMS/IMS/SF 的协作机制

下一篇文章将详细介绍窗口层次结构与容器管理。

---

## 参考资料

1. Android 源码：`frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java`
2. Android 源码：`frameworks/base/services/core/java/com/android/server/wm/Session.java`
3. [WindowManager 概述](https://source.android.com/docs/core/graphics/arch-wm)
