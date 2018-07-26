# Activity 中的 Window 分析

上文叙述了 Activity 的启动流程，所以关于 Window 的相关分析，就从 Activity 的开始，但其实其他的也大同小异。Activity 的 UI 显示的过程中，有三个重要的对象，即 `Window`、`DecorView`、`ViewRootImpl`：`Window` 用来表示窗口；`DecorView` 是视图容器，包含了所有的控件；`ViewRootImpl` 用来给控件分发事件以及与 `WindowManagerService` 交互。下面回顾一下 Activity 的部分启动过程来分析这三个对象的创建。AMS 调用 `ActivityThread` 的 `scheduleLaunchActivity()`，之后会走到 `handleLaunchActivity()` 方法，之后会执行  `performLaunchActivity()` 和 `handleResumeActivity`。首先看看 `performLaunchActivity()` 的部分源码：

```java
	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) { 
    	
        // ...
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }       
        // ...
        
        if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
        	window = r.mPendingRemoveWindow;
            r.mPendingRemoveWindow = null;
            r.mPendingRemoveWindowManager = null;
        }
        appContext.setOuterContext(activity);
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
        
        // ...
        if (r.isPersistable()) {
        	mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
        	mInstrumentation.callActivityOnCreate(activity, r.state);
		}
        // ...
        
    }
```

从这个代码可以看出，在 `performLaunchActivity()` 这个步骤中，创建了具体 Activity 的对象，然后调用了 `Activity.attach()` 方法，当然从代码可以看出，这个步骤是在 `Activity.onCreate()` 之前发生的，看看 `Activity.attach()` 中是怎样的：

```java
	final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);
        mFragments.attachHost(null /*parent*/);
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
    }
```

这个方法所做的工作是初始化 Activity 中的一些重要对象，当然，绝大部分是从 `ActivityThread` 中传递过来的，`Window` 对象就是在这里创建的，它是一个 `abstract` 类，可以看到实际的对象是 `PhoneWindow`，它的构造函数也需要一个 `Window` 对象，我们看到在 `performLaunchActivity()` 中传入的 `Window` 对象是 `ActivityClientRecord` 中的 `mPendingRemoveWindow`，这个的作用是当一个 `Activity` 销毁时，会将 `Window` 对象保存在此，以备 `Activity` 再启动时候的使用，就不用再初始化 `mDecor`（后面介绍） 等变量，直接从旧的 `Window` 对象中获取。继续下面，调用了 `Window.setCallback()` 方法，这里需要传入的对象是 `Window.Callback` 接口对象

```java
public interface Callback {
	public boolean dispatchKeyEvent(KeyEvent event);
	public boolean dispatchKeyShortcutEvent(KeyEvent event);
    public boolean dispatchTouchEvent(MotionEvent event);
    public boolean dispatchTrackballEvent(MotionEvent event);
    // ...
}
```

这是一些回调事件，`Activity` 自己实现了这个接口，当系统给 `PhoneWindow` 分发键盘、触屏等事件时就可以通过这个回调分发给 `Activity` 了。接着往下走，又调用了 `Window.setWindowManager()`，去看看如何实现的：

```java
	public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
```

这里有个 `appToken`，是在 `scheduleLaunchActivity` 这一步从 AMS 传过来的，实质上这是一个位于 `ActivityRecord` 中的继承于 `IApplicationToken.Stub` 的 `ActivityRecord.Token` 类的对象：

```java
	static class Token extends IApplicationToken.Stub {
        private final WeakReference<ActivityRecord> weakActivity;

        Token(ActivityRecord activity) {
            weakActivity = new WeakReference<>(activity);
        }

        private static ActivityRecord tokenToActivityRecordLocked(Token token) {
            if (token == null) {
                return null;
            }
            ActivityRecord r = token.weakActivity.get();
            if (r == null || r.getStack() == null) {
                return null;
            }
            return r;
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder(128);
            sb.append("Token{");
            sb.append(Integer.toHexString(System.identityHashCode(this)));
            sb.append(' ');
            sb.append(weakActivity.get());
            sb.append('}');
            return sb.toString();
        }
    }

```

可以看到里面持有了一个弱引用来持有对应的 `ActivityRecord`，对应的 `Activity` 和 `Window` 经过传递也拥有这个 `Token` 就形成了与 `ActivityRecord` 的关联（在 `Activity.attach()` 这一步），从之前所学可知 `ActivityRecord` 代表着 `Activity` 的运行状态。刚才 `setWindowManager()` 方法中的最后调用了 `WindowManagerImpl.createLocalWindowManager()` 方法。`WindowManager` 是个接口，继承于一个叫 `ViewManager` 的接口，而 `WindowManagerImpl` 才是最终的实现类：

```java
	public final class WindowManagerImpl implements WindowManager {
        private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
        // ...
		private WindowManagerImpl(Context context, Window parentWindow) {
            mContext = context;
            mParentWindow = parentWindow;
        }
        // ...
        public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        	return new WindowManagerImpl(mContext, parentWindow);
    	}
        // ...
    }
```

结合之前的代码来看，我们是使用的一个已有的 `WindowManagerImpl` 来创建的新的 `WindowManagerImpl`，已有的这个的来历在之前分析获取系统服务中提过，它是在 `SystemServiceRegistry` 类中通过 `new WindowManagerImpl(ctx)` 的方式创建的，并且对应一个 `Context` 派生类有一个相应的实例，所以 `Application`、`Activity`、`Service` 有各自的 `WindowManagerImpl` 实例。在继续刚才的代码，调用 `WindowManagerImpl.createLocalWindowManager()` 创建新的 `WindowManagerImpl` ，并将 `PhoneWindow` 对象传递进去了，作为它的 `parentWindow` 变量，在 `Activity.attach()` 之后的工作中，又将新创建的 `WindowManagerImpl` 赋给了 `mWindowManager` 变量，`Activity` 其实是重写了 `getSystemService()` 方法的：

```java
	@Override
    public Object getSystemService(@ServiceName @NonNull String name) {
        if (getBaseContext() == null) {
            throw new IllegalStateException(
                    "System services not available to Activities before onCreate()");
        }

        if (WINDOW_SERVICE.equals(name)) {
            return mWindowManager;
        } else if (SEARCH_SERVICE.equals(name)) {
            ensureSearchManager();
            return mSearchManager;
        }
        return super.getSystemService(name);
    }
```

所以之后再通过 `Activity` 来获取 `WindowManger`，获取到的是刚才创建的这个 `parentWindow` 为 该 Activity 对应的 `PhoneWindow` 对象的 `WindowManagerImpl`，直接用 `ContextImpl.getSystemService()` 所获取到的 `WindowManagerImpl` 的 `parentWindow` 是 `null`，例如 `Application` 这种情况的也会。在  `WindowManagerImpl` 这个类中可以看到有个 `WindowManagerGlobal` 单例对象，之后在进行 `addView()/updateViewLayout()/removeView()` 等操作时都会实际去调用 `WindowManagerGlobal` 对象来进行操作。回到 `performLaunchActivity()` 中，在 `Activity.attach()` 之后所做的事是调用 Activity 的 `onCreate()`、`onStart()`、`onRestoreInstanceState()`、`onPostCreate()` 等回调方法。回想一下，我们平时在 `onCreate()` 中经常干的一件事就是 `Activity.setContentView()` ：

```java
	public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
	
	public void setContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().setContentView(view, params);
        initWindowDecorActionBar();
    }
```

有多种 `setContentView()` 方法，但无论怎样，都会最终走到 `Window` 的实现类 `PhoneWindow` 中去处理，看看 `PhoneWindow.setContentView()`：

```java
	@Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }

	private void installDecor() {
    	// ...
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
        // ...
    }

	public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
	protected ViewGroup generateLayout(DecorView decor) {
    	// ...
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        // ...
    }
```

这里可以看出来，在 `PhoneWindow.setContentView()` 中判断了 `mContentParent` 是否为空，如果是则初始化 `mDecor`，`mDecor` 是 `DecorView` 类的实例，`DecorView` 继承于 `FrameLayout`，代表的根部局。然后 `mContentParent` 是 `mDecor` 中 id 为 `com.android.internal.R.id.content` 的 `ViewGroup`，实质是个 `FrameLayout`。在 `PhoneWindow.setContentView()` 之后的代码，会移除 `mContentParent` 的所有子 View，然后再将你指定 layoutId 的布局插入进去。其实还有一个 `addContentView()` 方法，它就直接将 View 添加到 `mContentParent` 中，没有移除所有子 View 的步骤。通过前面的分析，我们可以勾勒出这些类之间的一些关系了：一个应用程序中有多个 Activity，一个 Activity 对应一个 PhoneWindow、一个 ViewRootImpl、一个根视图 DecorView，一个应用只有一个 WindowManagerGlobal。WindowManagerGlobal 中有三个 `ArrayList`，分别是 `mViews`、`mRoots`、`mParams`，分别用来装 `View`、`ViewRootImpl`、`WindowManager.LayoutParams`。一个 `WindowState` 对应一个 `PhoneWindow`，一个 `Activity` 对应一个 `AppWindowToken`，但一个 `AppWindowToken` 可能对应多个 `PhoneWindow` 对象，从一个 `Activity` 中添加新的 `Window` 的时候就是这样。继续回到 `ActivityThread` 中，现在来看 `handleResumeActivity()` 中的处理：

```java
    final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
		// ...
        r = performResumeActivity(token, clearHide, reason);
        // ...
        final Activity a = r.activity;
        // ...
        boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManager.getService().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient) {
                    if (!a.mWindowAdded) {
                        a.mWindowAdded = true;
                        wm.addView(decor, l);
                    } else {
                        // The activity will get a callback for this {@link LayoutParams} change
                        // earlier. However, at that time the decor will not be set (this is set
                        // in this method), so no action will be taken. This call ensures the
                        // callback occurs with the decor set.
                        a.onWindowAttributesChanged(l);
                    }
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r, false /* force */);

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                if (r.newConfig != null) {
                    performConfigurationChangedForActivity(r, r.newConfig);
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                            + r.activityInfo.name + " with newConfig " + r.activity.mCurrentConfig);
                    r.newConfig = null;
                }
                if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                        + isForward);
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);
                    }
                }

                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }
        // ...
    }
```

首先通过 `performResumeActivity()` 去执行了 `Activity` 的 `onResume()` 生命周期，并返回了 `ActivityClientRecord` 对象。随后，获得了该 `Activity` 的 `Window` 及其下的 `DecorView`，设置其为 `INVISIBLE` 的状态，把这个 `Window` 的 `LayoutParam` 中的 `type` 设置为 `TYPE_BASE_APPLICATION`。一个 `Window` 有三个重要的属性，X 和 Y 分别代表横坐标和纵坐标，还有一个 Z轴 代表 `Window` 所在的 layer 层级，就由这个 `type` 决定，值大的就显示在上面，比如 `TYPE_BASE_APPLICATION` 的值是 1，而 `TYPE_TOAST` 的值是 2005，所以 toast 会显示在应用的上层。系统将窗口类型分成 3 种类型，Application-Window(1-99)、Sub-Window(1000-1999)、System-Window(2000-2999)，就不列举具体的 type 了，反正 type 数值大的会在上面。`handleResumeActivity()` 接下来做的事是 `WindowManager.addView()`，通过前面的学习可知实际调用的是 `WindowManagerImpl.addView()`：

```java
	public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```

所以，这又使用 `WindowManagerGlobal.addView()` 来处理了，前面已说过这是一个单例类，一个 APP 只有一个，这里的 `mParentWindow` 是创建 `Activity` 时在创建 `WindowManagerImpl` 时通过构造函数传给它的 `Activity` 的 `PhoneWindow` 对象：

```java
	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```

这里有一步骤调用了 `Window.adjustLayoutParamsForSubWindow()`：

```java
	void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
        CharSequence curTitle = wp.getTitle();
        if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            if (wp.token == null) {
                View decor = peekDecorView();
                if (decor != null) {
                    wp.token = decor.getWindowToken();
                }
            }
            if (curTitle == null || curTitle.length() == 0) {
                final StringBuilder title = new StringBuilder(32);
                if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA) {
                    title.append("Media");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA_OVERLAY) {
                    title.append("MediaOvr");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {
                    title.append("Panel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_SUB_PANEL) {
                    title.append("SubPanel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_ABOVE_SUB_PANEL) {
                    title.append("AboveSubPanel");
                } else if (wp.type == WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG) {
                    title.append("AtchDlg");
                } else {
                    title.append(wp.type);
                }
                if (mAppName != null) {
                    title.append(":").append(mAppName);
                }
                wp.setTitle(title);
            }
        } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
                wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
            // We don't set the app token to this system window because the life cycles should be
            // independent. If an app creates a system window and then the app goes to the stopped
            // state, the system window should not be affected (can still show and receive input
            // events).
            if (curTitle == null || curTitle.length() == 0) {
                final StringBuilder title = new StringBuilder(32);
                title.append("Sys").append(wp.type);
                if (mAppName != null) {
                    title.append(":").append(mAppName);
                }
                wp.setTitle(title);
            }
        } else {
            if (wp.token == null) {
                wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
            }
            if ((curTitle == null || curTitle.length() == 0)
                    && mAppName != null) {
                wp.setTitle(mAppName);
            }
        }
        if (wp.packageName == null) {
            wp.packageName = mContext.getPackageName();
        }
        if (mHardwareAccelerated ||
                (mWindowAttributes.flags & FLAG_HARDWARE_ACCELERATED) != 0) {
            wp.flags |= FLAG_HARDWARE_ACCELERATED;
        }
    }
```

`WindowManagerGlobal.addView()` 中，如果 `parentWindow` 不为空，就会走到 `adjustLayoutParamsForSubWindow()` 中，添加好 `token`， 之后主要就是创建了 `ViewRootImpl` 对象，然后分别把 `View`、`ViewRootImpl`、`LayoutParams` 添加到相应的 3 个 `ArrayList` 中，接着调用 `ViewRootImpl.setView()` 方法，将 `View` 和 `LayoutParams` 设置进去，如果是子窗口，那么还会查找出父窗口所在的 `panelParentView` 然后通过 `ViewRootImpl.setView()` 传递。

```java
	public final class ViewRootImpl implements ViewParent, View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
	    // ...
        final W mWindow; 
        final IWindowSession mWindowSession;
        // ...
        
        public ViewRootImpl(Context context, Display display) {
        	// ...
            mWindowSession = WindowManagerGlobal.getWindowSession();
            // ...
            mWindow = new W(this);
            // ...
        }
        
        // ...
        
        static class W extends IWindow.Stub {
            private final WeakReference<ViewRootImpl> mViewAncestor;
            private final IWindowSession mWindowSession;

            W(ViewRootImpl viewAncestor) {
                mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
                mWindowSession = viewAncestor.mWindowSession;
            }
        }
        
        // ...
    }
```

在构造函数里初始化了 `mWindowSession ` 对象，用来请求 WMS 操作，看看 `WindowManagerGlobal.getWindowSession()`：

```java
	public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    IWindowManager windowManager = getWindowManagerService();
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
	
	public static IWindowManager getWindowManagerService() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowManagerService == null) {
                sWindowManagerService = IWindowManager.Stub.asInterface(
                        ServiceManager.getService("window"));
                try {
                    if (sWindowManagerService != null) {
                        ValueAnimator.setDurationScale(
                                sWindowManagerService.getCurrentAnimatorScale());
                    }
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowManagerService;
        }
    }
	 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
     	// ...
		requestLayout();
 		// ...
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
        // ...
     }
```

可以看出这里先取得 WMS 的客户端对象，然后通过这个对象的 `openSession()` 创建一个 `WindowSession` 对象，`session` 这个词就有会话的意思，`ViewRootImpl` 就能靠它来进行与 WMS 通信。然后 `IWindow` 在 `ViewRootImpl` 里是服务端，明显是用来接受 WMS 对 `ViewRootImpl` 的通信，在 `setView()` 中有一个步骤 `mWindowSession.addToDisplay()` 把这个 ，这就实现了双向通信。在 WMS 里，可以看到我们前面说的 `WindowState`，它对应于客户端的 `PhoneWindow`， `IWindow` 的客户端就是在它之中持有。接下来的需要分析的环节就是 `requestLayout()` 方法，但这个的具体流程在后面分析 `View` 的 `measure`、`layout`、`draw` 三大流程的再谈。现在先看看 `addToDisplay()`，这个方法对应的在 `Binder` 服务端中，类名叫 `Session`：

```java
	@Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```

然后这里又去调用了 WMS 的 `addWindow()` 方法：

```java
	public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
    	int[] appOp = new int[1];
        int res = mPolicy.checkAddPermission(attrs, appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }   
        
        // ...
        
        WindowState parentWindow = null;
        
        // ...
		if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
        	parentWindow = windowForClientLocked(null, attrs.token, false);
            if (parentWindow == null) {
            	Slog.w(TAG_WM, "Attempted to add window with token that is not a window: "
                          + attrs.token + ".  Aborting.");
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
			}
            if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
            	&& parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                Slog.w(TAG_WM, "Attempted to add window with token that is a sub-window: "
                + attrs.token + ".  Aborting.");
                return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
			}
		}
        
        // ...
        WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
        
        if (token == null) {
                if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
        	// ...
        } else if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                atoken = token.asAppWindowToken();
                if (atoken == null) {
                    Slog.w(TAG_WM, "Attempted to add window with non-application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } else if (atoken.removed) {
                    Slog.w(TAG_WM, "Attempted to add window with exiting application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_APP_EXITING;
                }
        }
        
        // ...
        final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
        // ...
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        
        // ...
        win.mToken.addWindow(win);

        // ...
    }
```

这里首先通过 `mPolicy.checkAddPermission()` 检查了权限，如果是应用窗口，`WindowToken` 就一定不能为空，而且必须是对应 `Activity` 的 `AppToken`，且 `Activity` 不能已经 finish 掉的。然后判断了如果 `type` 为子窗口的情况下 `parentWindow` 是否为空，然后又判断了 `parentWindow` 的类型是否是子 `Window`，可知只允许 `Window` 的结构最多两层，子窗口使用所在父窗口的 `WindowToken`。对于系统类型的窗口，一部分需要 token 一部分不需要。窗口在 WMS 中通过一个 `WindowState` 进行管理，一个 `WindowState` 对应一个窗口，创建之后会保存到 `mWindowMap` 这个 `WindowHashMap` 之中，使用的 key 是 `ViewRootImpl` 中的那个 `W` 类的对象即 `IWindow`，便于查找。最终将创建的 `WindowState` 通过 `WindowToken.addWindow()` 添加进去了。至于更里面的细节，这篇文章不进行分析了，后面有时间再更加深入去看 WMS 里的代码。回到 `handleResumeActivity()` 中：

```java
	final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    	// ...
		r = performResumeActivity(token, clearHide, reason);
        // ...
        View decor = r.window.getDecorView();
		decor.setVisibility(View.INVISIBLE);
        // ...
        r.activity.makeVisible();
        // ...
    }
```

从这可以看出，确实是在 `onResume` 生命周期之后 `Activity` 的界面才是可见的。

最后总结一下：

* 窗口并不是 `Window`，而应该理解为一个根 `View`，WMS 用 `WindowState` 代表一个窗口，而应用中用 `ViewRootImpl` 和 `WindowManager.LayoutParams` 来描述。通过 `WindowManager/WindowManagerImpl/WindowManagerGlobal` 来添加根 `View` 来添加一个窗口，WMS 来管理着这些窗口的层次、位置等，每个窗口有自己的 `Surface`，`SurfaceFlinger` 负责把这些 `Surface` 合成一块 `FrameBuffer` 让屏幕显示出来

* Token 在应用窗口是必须的，并且需要是 `Activity` 的 AppToken，而子窗口的 Token 是父窗口的 `ViewRootImpl` 的 `IWindow`，系统窗口有一部分不需要 Token
* 从 `Activity` 使用 `getWindowManager()` 获取到的 `WindowMangerImpl` 对象是有 `mParentWindow` 的，即是该 `Activity` 的 `PhoneWindow` 对象，而从 `Application` 和 `Service` 获得的 `WindowMangerImpl` 对象是没有 `mParentWindow` 的。`mParentWindow` 不为空时，添加子窗口时会默认把 `PhoneWindow` 中的 `DecorView` 所在的 `WindowToken` 即 `ViewRootImpl` 中的 `IWindow` 那个 `W` 对象赋值给 `WindowManager.LayoutParams` 的 `token` 