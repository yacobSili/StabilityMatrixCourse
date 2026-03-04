# Activity 与 Window 关系详解

## 引言

Activity 是 Android 四大组件之一，也是用户界面的主要载体。每个 Activity 都拥有一个 Window，通过 Window 来管理和显示 View。本文将详细介绍 Activity 与 Window 的关系，以及 PhoneWindow、DecorView、setContentView 的工作原理。

---

## 1. Activity 与 Window 的关系

### 1.1 整体关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Activity                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                        PhoneWindow                             │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │                      DecorView                           │  │  │
│  │  │  ┌─────────────────────────────────────────────────┐    │  │  │
│  │  │  │              LinearLayout                        │    │  │  │
│  │  │  │  ┌───────────────────────────────────────────┐  │    │  │  │
│  │  │  │  │            标题栏 (可选)                    │  │    │  │  │
│  │  │  │  ├───────────────────────────────────────────┤  │    │  │  │
│  │  │  │  │                                           │  │    │  │  │
│  │  │  │  │        FrameLayout (ContentParent)        │  │    │  │  │
│  │  │  │  │          id: android.R.id.content         │  │    │  │  │
│  │  │  │  │                                           │  │    │  │  │
│  │  │  │  │    ┌─────────────────────────────────┐   │  │    │  │  │
│  │  │  │  │    │     用户布局 (setContentView)   │   │  │    │  │  │
│  │  │  │  │    │                                 │   │  │    │  │  │
│  │  │  │  │    └─────────────────────────────────┘   │  │    │  │  │
│  │  │  │  │                                           │  │    │  │  │
│  │  │  │  └───────────────────────────────────────────┘  │    │  │  │
│  │  │  └─────────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

对象持有关系：
Activity.mWindow ──► PhoneWindow
PhoneWindow.mDecor ──► DecorView
PhoneWindow.mContentParent ──► FrameLayout (android.R.id.content)
```

### 1.2 核心类关系

```java
// 类关系
Activity
  └─► Window (mWindow) ─────► PhoneWindow (唯一实现)
        └─► DecorView (mDecor) ─────► 根 View
              └─► ContentParent (mContentParent) ─────► 内容容器

// 接口关系
Activity implements Window.Callback
  └─► 接收窗口事件（按键、触摸、焦点等）
```

---

## 2. Activity 创建 Window 的过程

### 2.1 attach() 阶段

```java
// Activity.java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback,
        IBinder assistToken, IBinder shareableActivityToken) {
    
    attachBaseContext(context);
    
    // 1. 创建 PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    
    // 2. 设置 Window 回调（Activity 自己）
    mWindow.setCallback(this);
    
    // 3. 设置窗口管理器
    mWindow.setWindowManager(
        (WindowManager) context.getSystemService(Context.WINDOW_SERVICE),
        mToken,      // Activity 的 token
        mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0
    );
    
    // 4. 保存父窗口（用于对话框等）
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    
    mWindowManager = mWindow.getWindowManager();
}
```

### 2.2 调用时机

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Activity 启动流程中的 Window 创建                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ActivityThread.handleLaunchActivity()                              │
│      │                                                              │
│      ├─► performLaunchActivity()                                    │
│      │       │                                                      │
│      │       ├─► Instrumentation.newActivity()                      │
│      │       │       └─► 创建 Activity 实例                          │
│      │       │                                                      │
│      │       ├─► activity.attach()        ◄─── 创建 PhoneWindow     │
│      │       │                                                      │
│      │       └─► Instrumentation.callActivityOnCreate()             │
│      │               └─► activity.onCreate()                        │
│      │                       └─► setContentView()                   │
│      │                                                              │
│      └─► handleResumeActivity()                                     │
│              │                                                      │
│              ├─► activity.onResume()                                │
│              │                                                      │
│              └─► wm.addView(decor, l)     ◄─── 添加窗口到 WMS       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. PhoneWindow 详解

### 3.1 PhoneWindow 核心结构

```java
// PhoneWindow.java
public class PhoneWindow extends Window implements MenuBuilder.Callback {
    
    // 装饰视图（根 View）
    private DecorView mDecor;
    
    // 内容容器（用户布局的父容器）
    ViewGroup mContentParent;
    
    // 布局加载器
    private LayoutInflater mLayoutInflater;
    
    // 窗口特性
    private int mFeatures = DEFAULT_FEATURES;
    private int mLocalFeatures = DEFAULT_FEATURES;
    
    // 标题
    private CharSequence mTitle;
    
    // 背景
    private Drawable mBackgroundDrawable;
    
    // 转场动画
    private Transition mEnterTransition;
    private Transition mExitTransition;
}
```

### 3.2 窗口特性 (Features)

```java
// Window.java 定义的窗口特性
public static final int FEATURE_OPTIONS_PANEL = 0;      // 选项面板
public static final int FEATURE_NO_TITLE = 1;           // 无标题
public static final int FEATURE_PROGRESS = 2;           // 进度条
public static final int FEATURE_INDETERMINATE_PROGRESS = 5;
public static final int FEATURE_SWIPE_TO_DISMISS = 11;  // 滑动关闭
public static final int FEATURE_CONTENT_TRANSITIONS = 12; // 内容转场

// 使用方式
requestWindowFeature(Window.FEATURE_NO_TITLE);  // 必须在 setContentView 之前调用
```

---

## 4. setContentView 详解

### 4.1 三种重载形式

```java
// Activity.java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
}

public void setContentView(View view) {
    getWindow().setContentView(view);
}

public void setContentView(View view, ViewGroup.LayoutParams params) {
    getWindow().setContentView(view, params);
}
```

### 4.2 PhoneWindow.setContentView() 详解

```java
// PhoneWindow.java
@Override
public void setContentView(int layoutResID) {
    // 1. 确保 DecorView 和 ContentParent 已创建
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        // 如果已存在，清空旧内容
        mContentParent.removeAllViews();
    }
    
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        // 2a. 有转场动画时，延迟加载
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, ...);
        transitionTo(newScene);
    } else {
        // 2b. 直接加载布局到 ContentParent
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    
    // 3. 通知内容已改变
    mContentParent.requestApplyInsets();
    
    // 4. 回调 Activity
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

### 4.3 installDecor() 详解

```java
// PhoneWindow.java
private void installDecor() {
    if (mDecor == null) {
        // 1. 创建 DecorView
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
    }
    
    if (mContentParent == null) {
        // 2. 创建内容容器（根据窗口特性选择布局）
        mContentParent = generateLayout(mDecor);
        
        // 3. 设置标题
        mDecor.setWindowTitle(mTitle);
        
        // 4. 设置背景
        if (mBackgroundDrawable != null) {
            mDecor.setWindowBackground(mBackgroundDrawable);
        }
    }
}

// 创建 DecorView
protected DecorView generateDecor(int featureId) {
    return new DecorView(context, featureId, this, getAttributes());
}
```

### 4.4 generateLayout() - 选择系统布局

```java
// PhoneWindow.java
protected ViewGroup generateLayout(DecorView decor) {
    // 1. 根据窗口属性获取主题
    TypedArray a = getWindowStyle();
    
    // 2. 读取各种窗口属性
    mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
    mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);
    
    // 3. 根据特性选择系统布局
    int layoutResource;
    int features = getLocalFeatures();
    
    if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        // 有标题栏
        if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = R.layout.screen_action_bar;
        } else {
            layoutResource = R.layout.screen_title;
        }
    } else {
        // 无标题栏
        layoutResource = R.layout.screen_simple;
    }
    
    // 4. 加载系统布局到 DecorView
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    
    // 5. 找到内容容器
    ViewGroup contentParent = (ViewGroup) findViewById(ID_ANDROID_CONTENT);
    
    return contentParent;
}
```

### 4.5 系统布局示例

```xml
<!-- screen_simple.xml - 最简单的布局（无标题） -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    
    <FrameLayout
        android:id="@android:id/content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:foregroundInsidePadding="false"
        android:foregroundGravity="fill_horizontal|top" />
        
</LinearLayout>

<!-- screen_title.xml - 带标题栏的布局 -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <!-- 标题容器 -->
    <FrameLayout android:id="@android:id/title_container"
        android:layout_width="match_parent"
        android:layout_height="?android:attr/windowTitleSize">
        <TextView android:id="@android:id/title"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </FrameLayout>
    
    <!-- 内容容器 -->
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1" />
        
</LinearLayout>
```

---

## 5. DecorView 详解

### 5.1 DecorView 结构

```java
// DecorView.java
public class DecorView extends FrameLayout {
    
    // 所属的 PhoneWindow
    private PhoneWindow mWindow;
    
    // 窗口背景
    private Drawable mWindowBackground;
    
    // 状态栏/导航栏颜色
    private int mStatusBarColor;
    private int mNavigationBarColor;
    
    // 特性 ID
    private int mFeatureId = -1;
    
    // 根据系统布局创建后的结构：
    // DecorView (FrameLayout)
    //   └── LinearLayout (或其他系统布局根)
    //         ├── TitleBar (可选)
    //         └── ContentParent (FrameLayout, id: android.R.id.content)
    //               └── 用户布局
}
```

### 5.2 DecorView 的事件分发

```java
// DecorView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 1. 先给 Window.Callback 处理机会
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed()
            && cb.dispatchTouchEvent(ev)  // Activity.dispatchTouchEvent()
            ? true
            : super.dispatchTouchEvent(ev);
}

@Override
public boolean dispatchKeyEvent(KeyEvent event) {
    // 按键事件也类似
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed()
            && cb.dispatchKeyEvent(event)
            ? true
            : super.dispatchKeyEvent(event);
}
```

### 5.3 事件分发链

```
输入事件到达 DecorView 后的分发链：

┌─────────────────────────────────────────────────────────────────────┐
│                     触摸事件分发链                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. DecorView.dispatchTouchEvent()                                  │
│      │                                                              │
│      ├─► Window.Callback.dispatchTouchEvent()                       │
│      │       │                                                      │
│      │       └─► Activity.dispatchTouchEvent()                      │
│      │               │                                              │
│      │               └─► PhoneWindow.superDispatchTouchEvent()      │
│      │                       │                                      │
│      │                       └─► DecorView.superDispatchTouchEvent()│
│      │                               │                              │
│      │                               └─► ViewGroup.dispatchTouchEvent()
│      │                                       │                      │
│      │                                       └─► 子 View 处理      │
│      │                                                              │
│      └─► Activity.onTouchEvent() (如果没被消费)                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Window 添加到 WMS

### 6.1 添加时机

```java
// ActivityThread.java
@Override
public void handleResumeActivity(ActivityClientRecord r, ...) {
    // 1. 执行 onResume
    final Activity a = r.activity;
    performResumeActivity(r, ...);
    
    // 2. 添加窗口到 WMS
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);  // 先不可见
        
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        
        // 重要：添加 DecorView 到窗口管理器
        wm.addView(decor, l);
    }
    
    // 3. 设置可见
    if (!r.activity.mFinished && willBeVisible && ...) {
        r.activity.makeVisible();
    }
}

// Activity.java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

### 6.2 addView 流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WindowManager.addView() 流程                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WindowManagerImpl.addView(decorView, params)                       │
│      │                                                              │
│      └─► WindowManagerGlobal.addView()                              │
│              │                                                      │
│              ├─► 创建 ViewRootImpl                                  │
│              │       new ViewRootImpl(context, display)             │
│              │                                                      │
│              ├─► 保存到列表                                          │
│              │       mViews.add(view)                               │
│              │       mRoots.add(root)                               │
│              │       mParams.add(wparams)                           │
│              │                                                      │
│              └─► ViewRootImpl.setView(decorView, params)            │
│                      │                                              │
│                      ├─► requestLayout() ─► 触发首次遍历             │
│                      │                                              │
│                      └─► IWindowSession.addToDisplayAsUser()        │
│                              │                                      │
│                              └─► WMS.addWindow()                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Activity 与 Window 的回调交互

### 7.1 Window.Callback 接口

```java
// Window.java
public interface Callback {
    // 事件分发
    boolean dispatchKeyEvent(KeyEvent event);
    boolean dispatchTouchEvent(MotionEvent event);
    boolean dispatchTrackballEvent(MotionEvent event);
    boolean dispatchGenericMotionEvent(MotionEvent event);
    
    // 窗口焦点
    void onWindowFocusChanged(boolean hasFocus);
    
    // 窗口属性
    void onWindowAttributesChanged(WindowManager.LayoutParams attrs);
    
    // 内容变化
    void onContentChanged();
    
    // 搜索请求
    boolean onSearchRequested();
    
    // 菜单相关
    boolean onCreatePanelMenu(int featureId, Menu menu);
    boolean onPreparePanel(int featureId, View view, Menu menu);
    boolean onMenuItemSelected(int featureId, MenuItem item);
}
```

### 7.2 Activity 实现 Callback

```java
// Activity.java
public class Activity implements Window.Callback, ... {
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();  // 用户交互回调
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;  // View 树消费了事件
        }
        return onTouchEvent(ev);  // Activity 处理
    }
    
    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        // 窗口获得/失去焦点
        // 子类可重写
    }
    
    @Override
    public void onContentChanged() {
        // setContentView 调用后回调
        // 通常用于 ButterKnife 等框架绑定 View
    }
}
```

---

## 8. 本章小结

本文详细介绍了 Activity 与 Window 的关系：

1. **创建关系**：Activity.attach() 时创建 PhoneWindow
2. **层次结构**：Activity → PhoneWindow → DecorView → ContentParent → 用户布局
3. **setContentView**：installDecor() + inflate() 加载布局
4. **添加到 WMS**：handleResumeActivity() 时调用 addView()
5. **回调交互**：Activity 实现 Window.Callback 接收窗口事件

下一篇文章将介绍各种应用窗口类型（Dialog、Toast、PopupWindow 等）。

---

## 参考资料

1. Android 源码：`frameworks/base/core/java/android/app/Activity.java`
2. Android 源码：`frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java`
3. Android 源码：`frameworks/base/core/java/com/android/internal/policy/DecorView.java`
4. [Activity 窗口管理](https://developer.android.com/guide/components/activities/activity-lifecycle)
