# 06-Activity 启动和调度分析

启动 `Activity` 的方式大致有以下几种：

- 在 Launcher 应用上点击 APP 图标，会打开 APP 的主 `Activity`
- 在应用中调用 `startActivity/startActivityForResult` 启动一个 `Activity`，该 `Activity` 可属于当前应用中，也可以不是
- 按 Back 键，结束当前 `Activity`，回到上一个 `Activity`
- 点击最近任务页面显示的 Task 列表，启动这个 Task 的当前 `Activity`
- adb shell 登录手机执行 `am start` 命令启动指定的 `Activity`

在继续下面的叙述之前，先明确一些类的主要作用，因为在下文会经常设计这些概念：

* `ProcessRecord`：代表一个进程信息，包括其里面的 `Activity` 和 `Service` 等
* `ServiceRecord`：代表一个 `Service`，对应客户端的一个 `Service`
* `BroadcastRecord`：代表一个 `Broadcast`，对应客户端的一个 `Broadcast`
* `ActivityRecord`：代表一个 `Activity`，对应客户端的一个 `Activity`
* `TaskRecord`：代表一个 Task，其中有一个变量 `mActivities`，这是一个 `ArrayList`，用来存放 `ActivityRecord`，变量 `mStack` 代表所属的 ActivityStack。Task是指在执行特定作业时与用户交互的一系列 Activity。 这些 Activity 按照各自的打开顺序排列在堆栈（即返回栈）中。设备主屏幕是大多数 Task 的起点。当用户触摸应用启动器中的图标（或主屏幕上的快捷方式）时，该应用的 Task 将出现在前台。 如果应用不存在 Task（应用最近未曾使用），则会创建一个新 Task，并且该应用的主 Activity 将作为堆栈中的根 Activity 打开。
* `ActivityStack`：管理 Task，其中有一个变量 `mTaskHistory`，是一个 `TaskRecod` 类型的 `ArrayList`
* `ActivityStackSupervisor`：主要管理 `ActivityStack`，android 4.4 版本后会有两个 `ActivityStack`，Home Stack 用来放 Launcher 和 SystemUI，Application Stack 用来放其他的应用

Launcher 是一个应用，所以从 Launcher 启动我们的 APP，其实也是通过 Launcher 中的一个 `Activity` 打开我们的 APP 中的 主 `Activity`，所以启动的流程从 `Activity.startActivity()` 开始：

```Java
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

这个方法实际上是去调用了 `startActivityForResult()`，并且 `requestCode` 传递的是 -1，现在去看 ``startActivityForResult()` 的代码：

```Java
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this, 			mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                        mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                        ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

首先这判断了 `mParent` 对象是否为空，目前先不讨论 `ParentActivity` 相关的问题，先看其为空的情况，这里传参里过来的名为 `options` 的 `Bundle`，是转场动画相关的内容，也不是本文需要讨论的内容。接下来的 `mInstrumentation.execStartActivity` 是重点戏，不过在看它的源码之前先把后面的代码说了，我们可以看到  `requestCode` 只有大于等于 0 的时候，才会将 `mStartedActivity` 设为 `true`，所以 `startActivity` 方法是不会走到这里来的。而再后面点有个 `cancelInputsAndStartExitTransition()` 方法，看看它的源码：

```Java
    private void cancelInputsAndStartExitTransition(Bundle options) {
        final View decor = mWindow != null ? mWindow.peekDecorView() : null;
        if (decor != null) {
            decor.cancelPendingInputEvents();
        }
        if (options != null && !isTopOfTask()) {
            mActivityTransitionState.startExitOutTransition(this, options);
        }
    }
```

从方法名和注释可以看出，这个方法是用来取消掉还未执行的输入事件和启动转场动画的，转场动画本文不讨论，后面会专门出文章分析，我们看看 `decor.cancelPendingInputEvents()`，`decor` 是 `Window` 中的根 `ViewGroup`，所以我们看看 `View` 和 `ViewGroup` 中这个方法及与其相关的方法的源码：

```Java
    class View implements ...{
        // ...
        public final void cancelPendingInputEvents() {
            dispatchCancelPendingInputEvents();
        }

        void dispatchCancelPendingInputEvents() {
            mPrivateFlags3 &= ~PFLAG3_CALLED_SUPER;
            onCancelPendingInputEvents();
            if ((mPrivateFlags3 & PFLAG3_CALLED_SUPER) != PFLAG3_CALLED_SUPER) {
                throw new SuperNotCalledException("View " + getClass().getSimpleName() +
                        " did not call through to super.onCancelPendingInputEvents()");
            }
        }
        //...
    }

    class ViewGroup extends View implements ...{
        @Override
        void dispatchCancelPendingInputEvents() {
            super.dispatchCancelPendingInputEvents();

            final View[] children = mChildren;
            final int count = mChildrenCount;
            for (int i = 0; i < count; i++) {
                children[i].dispatchCancelPendingInputEvents();
            }
        }
    }
```

从上面贴出的源码可以看出，这个 `cancelPendingInputEvents()` 会调用 `dispatchCancelPendingInputEvents()` 方法，`ViewGroup` 中的 `dispatchCancelPendingInputEvents()` 方法除了执行了父类即 `View` 中原本方法，还会把这个找到它自己的子 `View` 们，调用它们的 `dispatchCancelPendingInputEvents()`。刚才在 `Activity` 中调用这个方法的是 `decor`，所以会导致这个 `Activity` 中所有的 `View` 都会去执行 `dispatchCancelPendingInputEvents()` 方法，看看它的源码：

```Java
    class View implements ...{
        public void onCancelPendingInputEvents() {
            removePerformClickCallback();
            cancelLongPress();
            mPrivateFlags3 |= PFLAG3_CALLED_SUPER;
        }

        private void removePerformClickCallback() {
            if (mPerformClick != null) {
                removeCallbacks(mPerformClick);
            }
        }

        public boolean removeCallbacks(Runnable action) {
            if (action != null) {
                final AttachInfo attachInfo = mAttachInfo;
                if (attachInfo != null) {
                    attachInfo.mHandler.removeCallbacks(action);
                    attachInfo.mViewRootImpl.mChoreographer.removeCallbacks(
                            Choreographer.CALLBACK_ANIMATION, action, null);
                }
                getRunQueue().removeCallbacks(action);
            }
            return true;
        }
    }
```

从这里可以看到这里面取消了点击事件和长按事件，并且看得出来点击事件和长按事件在 `View` 中是被添加到消息队列中的具体消息的 `callback` 变量的，这部分的具体内容在后面分析触摸事件分发流程的时候叙述。所以这里取消未发生的点击和长按事件就是从消息队列里移除相应的消息。所以在启动新的 `Activity` 并启动转场动画的时候会取消掉已按下但还未发生的点击事件，这是有必要的。好的，把这一大截的讲完，回到刚才 `mInstrumentation.execStartActivity()` ，讲下这个方法传入的参数：

* `mInstrumentation` 是一个 `Instrumentation` 对象，看注释描述，它用于监视应用程序和系统（主要是 AMS）的交互，可以在 `AndroidManifest.xml` 里通过 `instrumentation` 来描述。
* `mToken` 是一个 `IBinder` 对象，通过它可以访问当前 `Activity` （这里是讨论的是从 Launcher 启动，所以当前 `Activity` 是 `Launcher` 中的 `Activity`）在 AMS 中对应的 `ActivityRecord`。而 `ActivityRecord` 中记录了当前 `Activity` 的信息，AMS 需要这些信息。
* `mMainThread ` 是 `ActivityThread` 对象，名字像线程但并不是线程，这里面有 `main()` 方法，是应用程序的启动入口。`ActivityThread.getApplicationThread()` 获取到一个名为 `mAppThread` 的 `ApplicationThread` 对象，这继承于 `IApplicationThread.Stub` ，属于 `Binder` 的服务端。因为 AMS 和应用交互是多进程间通信，AMS 可以调用 `ApplicationThread` 中的方法来间接调用和通知 `ActivityThread` 做出操作。

讲了这几个主要参数就进入它的源码中看看吧：

```Java
    public ActivityResult execStartActivity(
                Context who, IBinder contextThread, IBinder token, Activity target,
                Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token, target != null ? target.mEmbeddedID : null,
                requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

上半部分是通过 `ActivityMonitor` 对启动 `Activity` 进行检查，找到匹配的 `Activity`，这个不是本文重点不表，看下面的重点。`ActivityManager.getService()` 在上篇文章已经讲过的，就是用来获取与 AMS 通信的  `Binder`，至此执行流程就从客户端转移到了服务端中了。当然后面还有一句是 `checkStartActivityResult(result, intent)`，看看它是怎么做的：

```java
    public static void checkStartActivityResult(int res, Object intent) {
        if (!ActivityManager.isStartResultFatalError(res)) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                        "Unable to find explicit activity class "
                        + ((Intent)intent).getComponent().toShortString()
                        + "; have you declared this activity in your AndroidManifest.xml?");
                    throw new ActivityNotFoundException(
                            "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                            + intent);
            case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                throw new AndroidRuntimeException(
                            "FORWARD_RESULT_FLAG used while also requesting a result");
            case ActivityManager.START_NOT_ACTIVITY:
                throw new IllegalArgumentException(
                            "PendingIntent is not an activity");
            case ActivityManager.START_NOT_VOICE_COMPATIBLE:
                throw new SecurityException(
                    "Starting under voice control not allowed for: " + intent);
            case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                    "Session calling startVoiceActivity does not match active session");
            case ActivityManager.START_VOICE_HIDDEN_SESSION:
                throw new IllegalStateException(
                    "Cannot start voice activity on a hidden session");
            case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                    "Session calling startAssistantActivity does not match active session");
            case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
                throw new IllegalStateException(
                    "Cannot start assistant activity on a hidden session");
            case ActivityManager.START_CANCELED:
                throw new AndroidRuntimeException("Activity could not be started for "
                    + intent);
            default:
                throw new AndroidRuntimeException("Unknown error code "
                    + res + " when starting " + intent);
        }
    }
```

这个方法的作用就是检查启动 `Activity` 是否成功，如果不成功就抛出异常，这里有一个我们很常见的异常信息，`Unable to find explicit activity class; have you declared this activity in your AndroidManifest.xml?`，这个错经常出现在没有在 `AndroidManifest.xml` 中声明 `Acitivity` 的情景。现在去看看 AMS 的 `startActivity` 怎么做的：

```Java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId, false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, "startActivityAsUser");
    }
```

可以看到这里又走到了 `mActivityStarter.startActivityMayWait()` 中：

```Java
    public void startActivityMayWait(......) {
        // ...
        intent = new Intent(intent);
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
        // ...
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
        // ...
        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                        aInfo, rInfo, voiceSession, voiceInteractor,
                        resultTo, resultWho, requestCode, callingPid,
                        callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                        options, ignoreTargetSecurity, componentSpecified, outRecord, inTask,
                        reason);
        // ...
    }
```

这个方法会经由 PMS 查询 `Activity` 信息，然后调用 `startActivityLocked` 方法，在这个方法中最后又调用了 `ActivityStarter.startActivity`，这个方法主要是首先用 PMS 查询 Activity 的信息，然后判断了各种可能出错的情景，如果出错就直接返回了错误码，否则就创建了 `ActivityRecord` 对象，之后走入了另一个 `ActivityStarter.startActivity()` 方法中，然后走入了 `startActivityUnchecked()` 中，这个方法的代码太长，说明一下它的过程：

1. 根据启动标记和模式，判断是否需要在新 Task 中运行目标 Activity
2. 判断有没有可以复用的 Task 或 Activity
3. 将复用或新建的 `TaskRecord` 与 `ActivityRecord` 相关联

之后的主线方法调用流程是 `ActivityStarter.startActivityUnchecked()` -> `ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()` -> `ActivityStack.resumeTopActivityUncheckedLocked` -> `ActivityStack.resumeTopActivityInnerLocked()` ，`ActivityStack.resumeTopActivityInnerLocked()` 中做了一个重要的操作：如果 `mResumedActivity` 不为空，则调用 `startPausingLocked()` 方法停止它。所以在 AActivity 打开 BActivity 的时候，会首先执行 AActivity 的 `onPause()`，后执行 BActivity 的 `onCreate()`。看看 `startPausingLocked()` 的源码：

```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
                ActivityRecord resuming, boolean pauseImmediately) {
        ActivityRecord prev = mResumedActivity;
        // ...
        mPausingActivity = prev;
        // ...
        prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                            userLeaving, prev.configChangeFlags, pauseImmediately);
        // ...
    }
```

在这里面可以看到首先把 `mPausingActivity` 变量指向了当前的 `mResumedActivity` ，`prev.app` 是将要暂停的 `Activity` 所对应的 `ActivityRecord` 中的变量 `app`，是一个 `ProcessRecord` 对象，它的 `thread` 对象是 `IApplicationThread` 类型，看名字就能猜到，是用来跟这个 APP 用来跨进程通信的工具。看看 `AcitivityThread` 中的代码：

```java
    public final class ActivityThread {
        // ...
        private class ApplicationThread extends IApplicationThread.Stub {
            public final void schedulePauseActivity(...){
                // ...
            }    
            public final void scheduleStopActivity(...){
                // ...
            }  
        }
    }
```

这里就是 `IApplicationThread` 所要进行通信的 ”服务端“，`ActivityThread` 需要通过 `IActivityManager` 与 AMS 通信，AMS 也需要一个桥梁来 ”调用“ `ActivityThread` 的方法。看看刚才 `startPausingLocked()` 中所调用的 `schedulePauseActivity()` 的代码：

```Java
    public class ActivityThread{
        // ...
        H mH = new H();
        // ...
        public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
            int seq = getLifecycleSeq();
            if (DEBUG_ORDER) Slog.d(TAG, "pauseActivity " + ActivityThread.this
                + " operation received seq: " + seq);
                sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? USER_LEAVING : 0) | (dontReport ? DONT_REPORT : 0),
                    configChanges,
                    seq);
        }
        // ...
        private void sendMessage(int what, Object obj, int arg1, int arg2, int seq) {
            if (DEBUG_MESSAGES) Slog.v(
                    TAG, "SCHEDULE " + mH.codeToString(what) + " arg1=" + arg1 + " arg2=" + arg2 + "seq= " + seq);
            Message msg = Message.obtain();
            msg.what = what;
            SomeArgs args = SomeArgs.obtain();
            args.arg1 = obj;
            args.argi1 = arg1;
            args.argi2 = arg2;
            args.argi3 = seq;
            msg.obj = args;
            mH.sendMessage(msg);
        }
        // ...
        private class H extends Handler {
            public static final int LAUNCH_ACTIVITY = 100;
            public static final int PAUSE_ACTIVITY = 101;
            // ...
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case LAUNCH_ACTIVITY: {
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                        final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                        r.packageInfo = getPackageInfoNoCheck(
                                r.activityInfo.applicationInfo, r.compatInfo);
                        handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    } break;
                    case RELAUNCH_ACTIVITY: {
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                        ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                        handleRelaunchActivity(r);
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    } break;
                    case PAUSE_ACTIVITY: {
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                        SomeArgs args = (SomeArgs) msg.obj;
                        handlePauseActivity((IBinder) args.arg1, false,
                                (args.argi1 & USER_LEAVING) != 0, args.argi2,
                                (args.argi1 & DONT_REPORT) != 0, args.argi3);
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    } 
                    case PAUSE_ACTIVITY_FINISHING: {
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                        SomeArgs args = (SomeArgs) msg.obj;
                        handlePauseActivity((IBinder) args.arg1, true, (args.argi1 & USER_LEAVING) != 0,
                                args.argi2, (args.argi1 & DONT_REPORT) != 0, args.argi3);
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    } break;      
                    // ...
            }
        }
    }
```

这的代码很好理解，`ActivityThread` 中有一个变量 `mH`，是一个主线程上的 `Handler`，在刚才的 `schedulePauseActivity` 中调用 `schedulePauseActivity` 是一次跨进程通信，所以会在 `Binder` 线程中执行，为了切回主线程所以使用 `Handler` 发消息到主线程中执行后续操作，所以最终会走到 `ActivityThread.handlePauseActivity()` 中：

```java
	private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport, int seq) {
        ActivityClientRecord r = mActivities.get(token);
        if (DEBUG_ORDER) Slog.d(TAG, "handlePauseActivity " + r + ", seq: " + seq);
        if (!checkAndUpdateLifecycleSeq(seq, r, "pauseActivity")) {
            return;
        }
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb(), "handlePauseActivity");

            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }

            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
                    ActivityManager.getService().activityPaused(token);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
```

上面这如果走到 `performUserLeavingActivity()` 最终会去回调 `Activity.onUserLeaving()` 方法，然后我们看 `performPauseActivity()` 相关的代码：

```java
	final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState, reason) : null;
    }

    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState, String reason) {
        if (r.paused) {
            if (r.activity.mFinished) {
                // If we are finishing, we won't call onResume() in certain cases.
                // So here we likewise don't want to call onPause() if the activity
                // isn't resumed.
                return null;
            }
            RuntimeException e = new RuntimeException(
                    "Performing pause of activity that is not resumed: "
                    + r.intent.getComponent().toShortString());
            Slog.e(TAG, e.getMessage(), e);
        }
        if (finished) {
            r.activity.mFinished = true;
        }

        // Next have the activity save its current state and managed dialogs...
        if (!r.activity.mFinished && saveState) {
            callCallActivityOnSaveInstanceState(r);
        }

        performPauseActivityIfNeeded(r, reason);

        // Notify any outstanding on paused listeners
        ArrayList<OnActivityPausedListener> listeners;
        synchronized (mOnPauseListeners) {
            listeners = mOnPauseListeners.remove(r.activity);
        }
        int size = (listeners != null ? listeners.size() : 0);
        for (int i = 0; i < size; i++) {
            listeners.get(i).onPaused(r.activity);
        }

        return !r.activity.mFinished && saveState ? r.state : null;
    }

    private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
        if (r.paused) {
            // You are already paused silly...
            return;
        }

        try {
            r.activity.mCalled = false;
            mInstrumentation.callActivityOnPause(r.activity);
            EventLog.writeEvent(LOG_AM_ON_PAUSE_CALLED, UserHandle.myUserId(),
                    r.activity.getComponentName().getClassName(), reason);
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException("Activity " + safeToComponentShortString(r.intent)
                        + " did not call through to super.onPause()");
            }
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to pause activity "
                        + safeToComponentShortString(r.intent) + ": " + e.toString(), e);
            }
        }
        r.paused = true;
    }
```

从上面的 3 个方法中可以看到最终调用了 `mInstrumentation.callActivityOnPause(r.activity);`：

```java
	public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
```

最终走到了 `activity.performPause()` 中：

```java
	final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }
```

这里面就看到了熟悉的 `onPause()` 方法。在 `handlePauseActivity()` 方法中，后面还有一部，就是去通知 AMS 已经完成了 "pause" 操作，看看 `ActivityManager.getService().activityPaused(token)` 在里面做了什么：

```java
	@Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```

接下来的方法调用是 `ActivityStack.getStackLocked()` -> `ActivityStack.activityPausedLocked()` -> `ActivityStack.completePauseLocked()` -> ``ActivityStackSupervisor.resumeFocusedStackTopActivityLocked` -> `ActivityStack.resumeTopActivityUncheckedLocked`，回到了这个熟悉的方法啦，这时候又去 `ActivityStack.resumeTopActivityInnerLocked()` 方法中，刚才调用了一次这方法中，现在是第二次调用，这回里面会调用 `ActivityStackSupervisor.startSpecificActivityLocked()`：

```java
    void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,  					r.info.applicationInfo.uid, true);

        r.getStack().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                                mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
    }
```

这个代码中，首先 `mService.getProcessRecordLocked(r.processName, r.info.applicationInfo.uid, true)` 会查询是否已经存在指定的进程信息，如果存在则复用它，然后走 `realStartActivityLocked()` 方法，否则使用 `ActivityManagerService.startProcessLocked()` 在其中最终会使用 `Process.start()` 通过 zygote 启动一个新的进程，本质是 fork 出一个子进程，然后在子进程中去执行 `ActivityThread.main()` 方法。如果是从 Launcher 打开我们的 APP，就会走后面这个方式，我们继续分析这种情况，此时看看 `ActivityThread.main()` 的代码：

```Java
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

在这个方法里面做了两件事：

1. 创建了主线程的消息队列和开始消息循环，以及创建了相应的 `Handler`
2. 创建了 `ActivityThread` 对象，并调用它的 `attach()` 方法

```java
	private void attach(boolean system) {
        	// ...
			ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
  
        	// ...
        
    }
```

看到这里面调用了 AMS 的 `attachApplication` 方法，看起来是把 `IApplicationThread` 这个 `Binder` 对象传递给 AMS：

```java
	public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

在 AMS 中，这个方法进一步调用了 `attachApplicationLocked()`，在这里调用了`IApplicationThread.bindApplication()` 方法，其中在传参的时候有个 `getCommonServicesLocked()` 方法注意一下：

```java
	private HashMap<String, IBinder> getCommonServicesLocked(boolean isolated) {
        // Isolated processes won't get this optimization, so that we don't
        // violate the rules about which services they have access to.
        if (isolated) {
            if (mIsolatedAppBindArgs == null) {
                mIsolatedAppBindArgs = new HashMap<>();
                mIsolatedAppBindArgs.put("package", ServiceManager.getService("package"));
            }
            return mIsolatedAppBindArgs;
        }

        if (mAppBindArgs == null) {
            mAppBindArgs = new HashMap<>();

            // Setup the application init args
            mAppBindArgs.put("package", ServiceManager.getService("package"));
            mAppBindArgs.put("window", ServiceManager.getService("window"));
            mAppBindArgs.put(Context.ALARM_SERVICE,
                    ServiceManager.getService(Context.ALARM_SERVICE));
        }
        return mAppBindArgs;
    }
```

这个 `getCommonServicesLocked()` 返回的结果传递给了下面代码中的方法的倒数第三个参数 `Map services` 

```java
	public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            sendMessage(H.BIND_APPLICATION, data);
        }
```

可以看到这里传递了 `PackageManagerService`、`WindowManagerService`、`AlarmManagerService` 作为通用服务传递给了应用，在应用中使用了 `ServiceManager.initServiceCache(services)` 保存起来作为通用服务，这里可以看到新建了一个 `AppBindData` 对象封装 AMS 传递过来的数据，然后通过 `Handler` 调用了 `ActivityThread.handleBindApplication()`，代码太长不贴，讲一下里面的重点步骤：

1. 为应用程序设置进程名
2. 创建应用的 `Application`，并设置进程的初始 `Application`
3. 安装 `ContentProvider`，所以 `ContentProvider` 的创建先于其他组件
4. 执行 `Instrumentation.onCreate()` 方法
5. 执行 `Application.onCreate()` 方法

这几个步骤执行完就说明加载应用程序完成。回到刚才调用 AMS 中的 `attachApplicationLocked()` 中，除了初始化应用，还进行了其他的操作，主要方法调用是

`ActivityStackSuperVisor.attachApplicationLocked()` -> `ActivityStackSuperVisor.realStartActivityLocked()`，这个方法又是一个熟悉的方法，回想上文 `ActivityStackSupervisor.startSpecificActivityLocked()` 处，如果进程已存在，则就直接调用 `realStartActivityLocked()` 来进行真正的启动 `Activity` 了：

```Java
	final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
		// ...
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
			System.identityHashCode(r), r.info,
        	mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, !andResume,
            mService.isNextTransitionForward(), profilerInfo);    		
        // ...
    }
```

这里又是使用 `IApplicationThread`去跟 `AppThread` 通信，调用其 `scheduleLaunchActivity()` 方法：

```java
	 public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
	}
```

这个方法将传来的参数封装到一个新建的 `ActivityClientRecord` 之中，然后又使用 `Handler` 去通知：

```java
	public void handleMessage(Message msg) {
		switch (msg.what) {
        	case LAUNCH_ACTIVITY: {
            	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                	r.activityInfo.applicationInfo, r.compatInfo);
				handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
             //...
		}
	}
```

这里首先根据信息找到 APK 文件信息，即一个 `LoadApk` 对象，然后调用 `handleLaunchActivity` 进一步处理：

```java
	private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        // ...
    	Activity a = performLaunchActivity(r, customIntent);
        // ...
        handleResumeActivity(r.token, false, r.isForward,
        	!r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
    	// ...
    }
```

这里面把 `Activity` 的启动过程又分为了 Launch 和 Resume 两个阶段，分别由 `performLaunchActivity()` 和 `handlerResumeActivity()` 完成。看看它的源码：

```java

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	// ...
    	ContextImpl appContext = createBaseContextForActivity(r);
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
        }
        // ... 
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
	 	if (!r.activity.mFinished) {
        	activity.performStart();
            r.stopped = false;
        }
        if (!r.activity.mFinished) {
        	if (r.isPersistable()) {
            	if (r.state != null || r.persistentState != null) {
                	mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
				}
            } else if (r.state != null) {
            	mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        if (!r.activity.mFinished) {
        	activity.mCalled = false;
            if (r.isPersistable()) {
            	mInstrumentation.callActivityOnPostCreate(activity, r.state,
                	r.persistentState);
            } else {
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
            }
            if (!activity.mCalled) {
            	throw new SuperNotCalledException(
                	"Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onPostCreate()");
            }
        }	
	// ...
    }
```

上面的代码中，首先创建了 `Activity` 的 `Context`，然后使用了 `mInstrumentation.newActivity()` 看看源码：

```java
	public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
```

原来 `Activity` 的实例就是在这个地方创建的。在回到刚才的代码中，调用了 `Activity.attach()`：

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

可以看出，这个方法就是 `Activity` 用来将大量信息附加到自身，构造出一个完整的运行框架。`performLaunchActivity()` 再接下来的重要方法是执行 ``mInstrumentation.callActivityOnCreate()`，这个和之前分析 `onPause/onStop` 时类似，很熟悉了，会执行 `Activity.onCreate()` 方法。在之后顺着 `performLaunchActivity()` 走下去，调用了 `activity.performStart()` 、`mInstrumentation.callActivityOnRestoreInstanceState()`、`mInstrumentation.callActivityOnPostCreate()` 相继去调用了 `onStart()`、`onRestoreInstanceState()`、`onPostCreate()` 几个回调方法。按我们的认知中，接下来就应该去走 `onResume` 了，这一步放在了 `handleLaunchActivity()` 中后面调用的 `handlerResumeActivity()` 里面，这里做了两件重要的事：

1. 调用 `performResumeActivity()` 
2. 处理 decorView，设置可见，这个在后面分析 `View` 三大流程时再进行分析

先进入 `performResumeActivity()` 的源码中：

```java
	public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide, String reason) {
        ActivityClientRecord r = mActivities.get(token);
        if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                + " finished=" + r.activity.mFinished);
        if (r != null && !r.activity.mFinished) {
            if (clearHide) {
                r.hideForNow = false;
                r.activity.mStartedActivity = false;
            }
            try {
                r.activity.onStateNotSaved();
                r.activity.mFragments.noteStateNotSaved();
                checkAndBlockForNetworkAccess();
                if (r.pendingIntents != null) {
                    deliverNewIntents(r, r.pendingIntents);
                    r.pendingIntents = null;
                }
                if (r.pendingResults != null) {
                    deliverResults(r, r.pendingResults);
                    r.pendingResults = null;
                }
                r.activity.performResume();

                synchronized (mResourcesManager) {
                    // If there is a pending local relaunch that was requested when the activity was
                    // paused, it will put the activity into paused state when it finally happens.
                    // Since the activity resumed before being relaunched, we don't want that to
                    // happen, so we need to clear the request to relaunch paused.
                    for (int i = mRelaunchingActivities.size() - 1; i >= 0; i--) {
                        final ActivityClientRecord relaunching = mRelaunchingActivities.get(i);
                        if (relaunching.token == r.token
                                && relaunching.onlyLocalRequest && relaunching.startsNotResumed) {
                            relaunching.startsNotResumed = false;
                        }
                    }
                }

                EventLog.writeEvent(LOG_AM_ON_RESUME_CALLED, UserHandle.myUserId(),
                        r.activity.getComponentName().getClassName(), reason);

                r.paused = false;
                r.stopped = false;
                r.state = null;
                r.persistentState = null;
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                        "Unable to resume activity "
                        + r.intent.getComponent().toShortString()
                        + ": " + e.toString(), e);
                }
            }
        }
        return r;
    }
	private void deliverNewIntents(ActivityClientRecord r, List<ReferrerIntent> intents) {
        final int N = intents.size();
        for (int i=0; i<N; i++) {
            ReferrerIntent intent = intents.get(i);
            intent.setExtrasClassLoader(r.activity.getClassLoader());
            intent.prepareToEnterProcess();
            r.activity.mFragments.noteStateNotSaved();
            mInstrumentation.callActivityOnNewIntent(r.activity, intent);
        }
    }

	private void deliverResults(ActivityClientRecord r, List<ResultInfo> results) {
        final int N = results.size();
        for (int i=0; i<N; i++) {
            ResultInfo ri = results.get(i);
            try {
                if (ri.mData != null) {
                    ri.mData.setExtrasClassLoader(r.activity.getClassLoader());
                    ri.mData.prepareToEnterProcess();
                }
                if (DEBUG_RESULTS) Slog.v(TAG,
                        "Delivering result to activity " + r + " : " + ri);
                r.activity.dispatchActivityResult(ri.mResultWho,
                        ri.mRequestCode, ri.mResultCode, ri.mData);
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                            "Failure delivering result " + ri + " to activity "
                            + r.intent.getComponent().toShortString()
                            + ": " + e.toString(), e);
                }
            }
        }
    }
```

在 `performResumeActivity()` 中， 通过 `deliverNewIntents()` 方法去使用 `mInstrumentation.callActivityOnNewIntent()` 处理了 `onNewIntent()` 回调，通过 `deliverResults()` 方法中调用 `Activity.dispatchActivityResult()` 去处理 `onActivityResult()` 回调，在后面通过 `r.activity.performResume()` ：

```java
	final void performResume() {
        performRestart();

        mFragments.execPendingActions();

        mLastNonConfigurationInstances = null;

        mCalled = false;
        // mResumed is set by the instrumentation
        mInstrumentation.callActivityOnResume(this);
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onResume()");
        }

        // invisible activities must be finished before onResume() completes
        if (!mVisibleFromClient && !mFinished) {
            Log.w(TAG, "An activity without a UI must call finish() before onResume() completes");
            if (getApplicationInfo().targetSdkVersion
                    > android.os.Build.VERSION_CODES.LOLLIPOP_MR1) {
                throw new IllegalStateException(
                        "Activity " + mComponent.toShortString() +
                        " did not call finish() prior to onResume() completing");
            }
        }

        // Now really resume, and install the current status bar and menu.
        mCalled = false;

        mFragments.dispatchResume();
        mFragments.execPendingActions();

        onPostResume();
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPostResume()");
        }
    }

	final void performRestart() {
        mCanEnterPictureInPicture = true;
        mFragments.noteStateNotSaved();

        if (mToken != null && mParent == null) {
            // No need to check mStopped, the roots will check if they were actually stopped.
            WindowManagerGlobal.getInstance().setStoppedState(mToken, false /* stopped */);
        }

        if (mStopped) {
            mStopped = false;

            synchronized (mManagedCursors) {
                final int N = mManagedCursors.size();
                for (int i=0; i<N; i++) {
                    ManagedCursor mc = mManagedCursors.get(i);
                    if (mc.mReleased || mc.mUpdated) {
                        if (!mc.mCursor.requery()) {
                            if (getApplicationInfo().targetSdkVersion
                                    >= android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                                throw new IllegalStateException(
                                        "trying to requery an already closed cursor  "
                                        + mc.mCursor);
                            }
                        }
                        mc.mReleased = false;
                        mc.mUpdated = false;
                    }
                }
            }

            mCalled = false;
            mInstrumentation.callActivityOnRestart(this);
            if (!mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onRestart()");
            }
            performStart();
        }
    }
```

这里面首先调用 `performRestart()` 方法，在里面如果 `mStopped` 为 `true`，会执行 `onRestart()` 和 `onStart()` 方法，然后 `performResume()` 中之后执行了 `onResume()` 和 `onPostResume()`。回到 `handlerResumeActivity()` 后面的代码中：

```java
			
	final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        	// ...
			if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                Looper.myQueue().addIdleHandler(new Idler());
            }
            r.onlyLocalRequest = false;

            // Tell the activity manager we have resumed.
            if (reallyResume) {
                try {
                    ActivityManager.getService().activityResumed(token);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }
    	// ...
    }
```

这里最后调用了 `ActivityManager.getService().activityResumed()` 去做了一些通知和收尾的工作。当然前面有一行代码 `Looper.myQueue().addIdleHandler(new Idler())`，`Idle` 的用法在前面分析消息机制时说过，在队列中消息处理完处于空闲期的时候会去调用，看看这个 `Idler` 类：

```java
	private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            boolean stopProfiling = false;
            if (mBoundApplication != null && mProfiler.profileFd != null
                    && mProfiler.autoStopProfiler) {
                stopProfiling = true;
            }
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManager.getService();
                ActivityClientRecord prev;
                do {
                    if (localLOGV) Slog.v(
                        TAG, "Reporting idle of " + a +
                        " finished=" +
                        (a.activity != null && a.activity.mFinished));
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) {
                            throw ex.rethrowFromSystemServer();
                        }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            if (stopProfiling) {
                mProfiler.stopProfiling();
            }
            ensureJitEnabled();
            return false;
        }
    }
```

也就是执行完 resume 之后，这里去调用了 AMS 的 `activityIdle()` ，然后其中又走到了 `ActivityStackSupervisor.activityIdleInternalLocked()` 方法，再走到 `ActivityStack.stopActivityLocked()` 方法，这里面走到了我们已经很熟悉的那一套，调用 `IApplicationThread.scheduleStopActivity()`，这里来到 `ActivityThread` 中了，类似于 pause 的流程，方法执行顺序为 `ActivityThread.handleStopActivity()` -> `ActivityThread.performStopActivityInner()` -> `ActivityThread.performPauseActivityIfNeeded()` -> `Instrumentation.callActivityOnPause`，最终会回调 `Activity.onStop()`，所在 stop 的步骤完成。把 `onStop()` 放在消息队列空闲时执行，有利于提高打开新 `Activity` 的效率。

