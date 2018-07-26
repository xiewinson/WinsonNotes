# Dialog 中的 Window 分析

从上一篇文章中的分析可知，一个窗口所对应的并不是一个 Window 类，而是由 WindowManagerGlobal 调用 `addView()` 方法所添加的那个 View，其实用 Layer 来描述它或许更为准确，不过这篇文章中仍沿袭窗口的说法分析一下 Dialog。

### Activity

上篇文章中着重介绍了 Activity 与 Window 的关系，这篇文章中不再赘述，简而言之，ActivityThread 在 `performLaunchActivity()` 这一步骤中调用 Activity 的 `attach()`，在此创建了 PhoneWindow 对象和该 Activity 的 WindowManager。随后在 `handleResumeActivity()` 中对 PhoneWindow 的 DecorView 进行 `WindowManager.addView()` ，WindowManagerGlobal 的 `addView()` 又创建了相应的 ViewRootImpl 对象，自此创建了窗口。这个窗口的 WindowManager.LayoutParams 中，token 为该 Activity 对应的 AppToken，type 为 TYPE_BASE_APPLICATION，所以是一个应用级别窗口，在其下还可以进行添加子窗口的添加

### Dialog

常用的 Dialog 是 AlertDialog，当然它也不过是继承自 Dialog，所以直接看 Dialog 的源码吧:

```java
	Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        if (createContextThemeWrapper) {
            if (themeResId == ResourceId.ID_NULL) {
                final TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                themeResId = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, themeResId);
        } else {
            mContext = context;
        }

        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
        w.setOnWindowSwipeDismissedCallback(() -> {
            if (mCancelable) {
                cancel();
            }
        });
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);
    }
```

在这里面取得了 WindowManager，这个 context 需要是 Activity 对象，原因后面分析，所以取得的 WindowManager 对象是 Activity 中的那个。随后创建了 PhoneWindow 对象，并为其设置各种回调，以让 Dialog 中能收到各种回调。随后 `w.setWindowManager` 为这个 Window 真正新建并设置了属于它的 WindowManager，和 Activity 中的处理情况一样，就不贴这的源码了，这里创建的 WindowManagerImpl 对象中的 parentWindow 依然是 PhoneWindow 对象本身。再后面设置了这个 Window 的方位是居中，所以 Dialog 默认呈现出来的样子都是居中的。Dialog 中的 `setContentView()` 和 Activity 中的也没啥区别，也是去调用了 Window 中的同名方法。我们在创建 Dialog 的最后一步一般是去调用其 `create()` 方法或者直接调用 `show()` 方法：

```java
	public void create() {
        if (!mCreated) {
            dispatchOnCreate(null);
        }
    }

	void dispatchOnCreate(Bundle savedInstanceState) {
        if (!mCreated) {
            onCreate(savedInstanceState);
            mCreated = true;
        }
    }

	public void show() {
        if (mShowing) {
            if (mDecor != null) {
                if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                    mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
                }
                mDecor.setVisibility(View.VISIBLE);
            }
            return;
        }

        mCanceled = false;

        if (!mCreated) {
            dispatchOnCreate(null);
        } else {
            // Fill the DecorView in on any configuration changes that
            // may have occured while it was removed from the WindowManager.
            final Configuration config = mContext.getResources().getConfiguration();
            mWindow.getDecorView().dispatchConfigurationChanged(config);
        }

        onStart();
        mDecor = mWindow.getDecorView();

        if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
            final ApplicationInfo info = mContext.getApplicationInfo();
            mWindow.setDefaultIcon(info.icon);
            mWindow.setDefaultLogo(info.logo);
            mActionBar = new WindowDecorActionBar(this);
        }

        WindowManager.LayoutParams l = mWindow.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
            WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
            nl.copyFrom(l);
            nl.softInputMode |=
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
            l = nl;
        }

        mWindowManager.addView(mDecor, l);
        mShowing = true;

        sendShowMessage();
    }
```

`create()` 方法很短小，就是判断是否已经创建若未创建就调用 `dispatchOnCreate()`，`dispatchOnCreate()` 中去调用了我们非常熟悉的 `onCreate()` 方法。而 `show()` 方法中，首先把 DecorView 设置为 VISIBLE，再判断是否创建了而决定是否去调用 `onCreate()` 方法，否则调用配置变更的操作。再然后调用了 `onStart()` 这个生命周期方法。`show()` 方法的最后一点使用 `WindowManager.addView()` 来添加窗口，和上一篇文章所描述的如出一辙。但是注意！这个 `mWindowManager` 并不是 Dialog 自己的 Window 所对应的 WindowManager，而是传进来的 Activity 所属的 WindowManager。我们在传入的过程中，也没有去设置过 token，所以它的 token 是在上篇文章所说的在 `parentWindow.adjustLayoutParamsForSubWindow()` 这一步骤时会把 token 设置成所在 Activity 的 AppToken，所以这就解释了为啥 Context 必须要是 Activity。此外，这个事也说明了 Dialog 并不是一个子窗口，应该是一个特殊的应用窗口，因为子窗口所用的 token 应该是父窗口的 IWindow，应用窗口用的应当是 AppToken。为啥没有 Dialog 里并没有看到设置 type 的地方？看上面的代码是直接 `mWindow.getAttributes()` 获得的 `WindowManager.LayoutParams` 对象，去 Window 中看看相关源码：

```java
	public final WindowManager.LayoutParams getAttributes() {
        return mWindowAttributes;
    }

	private final WindowManager.LayoutParams mWindowAttributes =
        new WindowManager.LayoutParams();

	public LayoutParams() {
    	super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
        type = TYPE_APPLICATION;
        format = PixelFormat.OPAQUE;
	}
```

恍然大悟！啥都不设置的时候就是默认为 TYPE_APPLICATION。上一篇文章只分析了添加窗口的情况，没有分析移除窗口的情况，这篇文章就来分析一下移除窗口的相关源码。我们知道，Dialog 有 `hide()` 和 `dismiss()` 两个方法：

```java
	public void hide() {
        if (mDecor != null) {
            mDecor.setVisibility(View.GONE);
        }
    }

	public void cancel() {
        if (!mCanceled && mCancelMessage != null) {
            mCanceled = true;
            // Obtain a new message so this dialog can be re-used
            Message.obtain(mCancelMessage).sendToTarget();
        }
        dismiss();
    }

	public void dismiss() {
        if (Looper.myLooper() == mHandler.getLooper()) {
            dismissDialog();
        } else {
            mHandler.post(mDismissAction);
        }
    }
	
	void dismissDialog() {
        if (mDecor == null || !mShowing) {
            return;
        }

        if (mWindow.isDestroyed()) {
            Log.e(TAG, "Tried to dismissDialog() but the Dialog's window was already destroyed!");
            return;
        }

        try {
            mWindowManager.removeViewImmediate(mDecor);
        } finally {
            if (mActionMode != null) {
                mActionMode.finish();
            }
            mDecor = null;
            mWindow.closeAllPanels();
            onStop();
            mShowing = false;

            sendDismissMessage();
        }
    }
```

这个 `hide()` 方法做的事就是设置 DecorView 为 GONE，没什么好讲了。`cancel()` 其实就是去调用了 `dismiss()`，不过先触发了一下 cancel 的回调。而 `dismiss()` 调用了 `dismissDialog()` 方法，而其中又调用了 `WindowManager.removeViewImmediate()` 方法。其实上篇文章没提，在 ActivityThread 的 `handleDestroyActivity()` 方法中，也是调用了同样的方法来移除窗口。下面就去 WindowManagerImpl 中看看：

```java
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
	
	public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
```

所以又毫无例外的通过 WindowManagerGlobal 去处理这个东西了:

```java
	private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();

	private int findViewLocked(View view, boolean required) {
        final int index = mViews.indexOf(view);
        if (required && index < 0) {
            throw new IllegalArgumentException("View=" + view + " not attached to window manager");
        }
        return index;
    }

	public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }

	private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```

WindowManagerGlobal 中有三个 Arraylist 分别保存了 View、LayoutParams、ViewRootImpl，所以就很容易地通过 View 对象找到它的 index 再找到 View 对应的 ViewRootImpl，再去调用其 `die()` 方法：

```java
	boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }

	void doDie() {
        checkThread();
        if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
        synchronized (this) {
            if (mRemoved) {
                return;
            }
            mRemoved = true;
            if (mAdded) {
                dispatchDetachedFromWindow();
            }

            if (mAdded && !mFirst) {
                destroyHardwareRenderer();

                if (mView != null) {
                    int viewVisibility = mView.getVisibility();
                    boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                    if (mWindowAttributesChanged || viewVisibilityChanged) {
                        // If layout params have been changed, first give them
                        // to the window manager to make sure it has the correct
                        // animation info.
                        try {
                            if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                    & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                                mWindowSession.finishDrawing(mWindow);
                            }
                        } catch (RemoteException e) {
                        }
                    }

                    mSurface.release();
                }
            }

            mAdded = false;
        }
        WindowManagerGlobal.getInstance().doRemoveView(this);
    }

	void dispatchDetachedFromWindow() {
        if (mView != null && mView.mAttachInfo != null) {
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
            mView.dispatchDetachedFromWindow();
        }

        mAccessibilityInteractionConnectionManager.ensureNoConnection();
        mAccessibilityManager.removeAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mAccessibilityManager.removeHighTextContrastStateChangeListener(
                mHighContrastTextManager);
        removeSendWindowContentChangedCallback();

        destroyHardwareRenderer();

        setAccessibilityFocus(null, null);

        mView.assignParent(null);
        mView = null;
        mAttachInfo.mRootView = null;

        mSurface.release();

        if (mInputQueueCallback != null && mInputQueue != null) {
            mInputQueueCallback.onInputQueueDestroyed(mInputQueue);
            mInputQueue.dispose();
            mInputQueueCallback = null;
            mInputQueue = null;
        }
        if (mInputEventReceiver != null) {
            mInputEventReceiver.dispose();
            mInputEventReceiver = null;
        }
        try {
            mWindowSession.remove(mWindow);
        } catch (RemoteException e) {
        }

        // Dispose the input channel after removing the window so the Window Manager
        // doesn't interpret the input channel being closed as an abnormal termination.
        if (mInputChannel != null) {
            mInputChannel.dispose();
            mInputChannel = null;
        }

        mDisplayManager.unregisterDisplayListener(mDisplayListener);

        unscheduleTraversals();
    }
```

如果 immediate 为 true，则立即调用 `doDie()`，否则会发一个 MSG_DIE，在 ViewRootImpl 的 ViewRootHandler 这个 Handler 实例在接收到这个消息后再调用 `doDie()` 方法。这里面通过 `dispatchDetachedFromWindow()` 去分发 View Detached 事件，通过 `WindowSession.remove()` 去通知 WMS 移除窗口，将 Surface 释放掉等工作。在 `doDie()` 的最后调用了 `WindowManagerGlobal.getInstance().doRemoveView(this)` 方法：

```java
	 void doRemoveView(ViewRootImpl root) {
        synchronized (mLock) {
            final int index = mRoots.indexOf(root);
            if (index >= 0) {
                mRoots.remove(index);
                mParams.remove(index);
                final View view = mViews.remove(index);
                mDyingViews.remove(view);
            }
        }
        if (ThreadedRenderer.sTrimForeground && ThreadedRenderer.isAvailable()) {
            doTrimForeground();
        }
    }
```

这一步去将 WindowManagerGlobal 中保存的已经 die 掉的那个 View 及其 WindowManager.LayoutParams 和 ViewRootImpl 从相应的 ArrayList 中移除掉。