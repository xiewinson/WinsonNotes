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

所以之后再通过 `Activity` 来获取 `WindowManger`，获取到的是刚才创建的这个 `parentWindow` 为 该 Activity 对应的 `PhoneWindow` 对象的 `WindowManagerImpl`，直接用 `ContextImpl.getSystemService()` 所获取到的 `WindowManagerImpl` 的 `parentWindow` 是 `null`，例如 `Application` 这种情况的也会。在  `WindowManagerImpl` 这个类中可以看到有个 `WindowManagerGlobal` 单例对象，之后在进行 `addView()/updateViewLayout()/removeView()` 等操作时都会实际去调用 `WindowManagerGlobal` 对象来进行操作。



`performLaunchActivity()` 在 `Activity.attach()` 之后所做的事是调用 Activity 的 `onCreate()`、`onStart()`、`onRestoreInstanceState()`、`onPostCreate()` 等回调方法。回想一下，我们平时在 `onCreate()` 中经常干的一件事就是 `Activity.setContentView()` ：

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

这里可以看出来，在 `PhoneWindow.setContentView()` 中判断了 `mContentParent` 是否为空，如果是则初始化 `mDecor`，`mDecor` 是 `DecorView` 类的实例，`DecorView` 继承于 `FrameLayout`，代表的根部局。然后 `mContentParent` 是 `mDecor` 中 id 为 `com.android.internal.R.id.content` 的 `ViewGroup`，实质是个 `FrameLayout`。在 `PhoneWindow.setContentView()` 之后的代码，会移除 `mContentParent` 的所有子 View，然后再将你指定 layoutId 的布局插入进去。其实还有一个 `addContentView()` 方法，它就直接将 View 添加到 `mContentParent` 中，没有移除所有子 View 的步骤。通过前面的分析，我们可以勾勒出这些类之间的一些关系了：一个应用程序中有多个 Activity，一个 Activity 对应一个 PhoneWindow、一个 ViewRootImpl





现在来看 `handleResumeActivity()` 中的处理：

```java
    final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
		// ...
        r = performResumeActivity(token, clearHide, reason);
        // ...
    }
```

