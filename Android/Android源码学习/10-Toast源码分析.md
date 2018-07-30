# Toast 源码分析

Toast 是 Android 中出现频率特别高的东西，这个词原意是吐司面包的意思，大概这东西给人的感觉就是像面包片一样出来一片又一片的。最简单快捷的使用 Toast 的是这样的：

```java
Toast.makeText(this, "—_-。。。", Toast.LENGTH_SHORT).show();
```

先去看看 `Toast.makeText()` 方法的源码:

```java
public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
            @NonNull CharSequence text, @Duration int duration) {
	Toast result = new Toast(context, looper);

    LayoutInflater inflate = (LayoutInflater)
    	context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
	View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);
	result.mNextView = v;
    result.mDuration = duration;

    return result;
}
```

这里面创建了要被 Toast 显示出来的 View，并把传入的文字设置给这个 View 中的 TextView。此外，还创建了 Toast 对象，并将这个 View 设置给 Toast 对象的 mNextView 变量，看看 Toast 的构造器源码:

```java
	public Toast(@NonNull Context context, @Nullable Looper looper) {
        mContext = context;
        mTN = new TN(context.getPackageName(), looper);
        mTN.mY = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.toast_y_offset);
        mTN.mGravity = context.getResources().getInteger(
                com.android.internal.R.integer.config_toastDefaultGravity);
    }
```

这里面新建了 TN 对象，并设置了两个值，第一个 mY，设置了 24dp，第二个 mGravity，设置为 Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM，稍后看它的作用，先看看 TN 类：

```java
	private static class TN extends ITransientNotification.Stub {
        // ...
        
    	TN(String packageName, @Nullable Looper looper) {
            // XXX This should be changed to use a Dialog, with a Theme.Toast
            // defined that sets up the layout params appropriately.
            final WindowManager.LayoutParams params = mParams;
            params.height = WindowManager.LayoutParams.WRAP_CONTENT;
            params.width = WindowManager.LayoutParams.WRAP_CONTENT;
            params.format = PixelFormat.TRANSLUCENT;
            params.windowAnimations = com.android.internal.R.style.Animation_Toast;
            params.type = WindowManager.LayoutParams.TYPE_TOAST;
            params.setTitle("Toast");
            params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;

            mPackageName = packageName;

            if (looper == null) {
                // Use Looper.myLooper() if looper is not specified.
                looper = Looper.myLooper();
                if (looper == null) {
                    throw new RuntimeException(
                            "Can't toast on a thread that has not called Looper.prepare()");
                }
            }
            mHandler = new Handler(looper, null) {
                @Override
                public void handleMessage(Message msg) {
                    switch (msg.what) {
                        case SHOW: {
                            IBinder token = (IBinder) msg.obj;
                            handleShow(token);
                            break;
                        }
                        case HIDE: {
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            break;
                        }
                        case CANCEL: {
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            try {
                                getService().cancelToast(mPackageName, TN.this);
                            } catch (RemoteException e) {
                            }
                            break;
                        }
                    }
                }
            };
        }
        // ...
    }
```

从类名就可以看出来，这是一个 Binder 的服务端，来拿充当回调用。在 TN 的构造器中，设置了 WindowManager.LayoutParams 的各种参数，type 是 TYPE_TOAST，随后检查了当前线程的 Looper 是否已准备后，否则抛出异常，所以直接在一个新线程里显示 Toast 会抛出异常，并不是在非主线程不能显示 Toast，而是没有开启 Looper 的消息循环。为什么会这样？因为后面这就新建了使用该 Looper 对象的 Handler，处理 SHOW、HIDE、CANCEL 等消息，后面再进行分析。现在来看 `Toast.show()` 方法：

```java
	public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }

	private static INotificationManager sService;

    static private INotificationManager getService() {
        if (sService != null) {
            return sService;
        }
        sService = INotificationManager.Stub
            .asInterface(ServiceManager.getService("notification"));
        return sService;
    }
```

这里 INotificationManager 是一个 Binder 接口，用来与 NotificationManagerService（后面称之为 NMS） 通信。这里把刚才创建好的 Toast 的 View 赋给 TN 对象，然后调用 `INotificationManager.enqueueToast()` 方法，这是一个跨进程调用，传递了包名、TN 对象和时间 mDuration，现在去 NMS 看看：

```java
public class NotificationManagerService extends SystemService {
	// ...
    final ArrayList<ToastRecord> mToastQueue = new ArrayList<>();
	
    // ...
    public void onStart() {
        // ...
        
    	publishBinderService(Context.NOTIFICATION_SERVICE, mService);
    }
    
    private final IBinder mService = new INotificationManager.Stub() {
    	public void enqueueToast(String pkg, ITransientNotification callback, int duration){
            if (DBG) {
                Slog.i(TAG, "enqueueToast pkg=" + pkg + " callback=" + callback
                        + " duration=" + duration);
            }

            if (pkg == null || callback == null) {
                Slog.e(TAG, "Not doing toast. pkg=" + pkg + " callback=" + callback);
                return ;
            }
            final boolean isSystemToast = isCallerSystemOrPhone() || ("android".equals(pkg));
            final boolean isPackageSuspended =
                    isPackageSuspendedForUser(pkg, Binder.getCallingUid());

            if (ENABLE_BLOCKED_TOASTS && !isSystemToast &&
                    (!areNotificationsEnabledForPackage(pkg, Binder.getCallingUid())
                            || isPackageSuspended)) {
                Slog.e(TAG, "Suppressing toast from package " + pkg
                        + (isPackageSuspended
                                ? " due to package suspended by administrator."
                                : " by user request."));
                return;
            }

            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    int index;
                    // All packages aside from the android package can enqueue one toast at a time
                    if (!isSystemToast) {
                        index = indexOfToastPackageLocked(pkg);
                    } else {
                        index = indexOfToastLocked(pkg, callback);
                    }

                    // If the package already has a toast, we update its toast
                    // in the queue, we don't move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                        record.update(callback);
                    } else {
                        Binder token = new Binder();
                        mWindowManagerInternal.addWindowToken(token, TYPE_TOAST, DEFAULT_DISPLAY);
                        record = new ToastRecord(callingPid, pkg, callback, duration, token);
                        mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                    }
                    keepProcessAliveIfNeededLocked(callingPid);
                    // If it's at index 0, it's the current toast.  It doesn't matter if it's
                    // new or just been updated.  Call back and tell it to show itself.
                    // If the callback fails, this will remove it from the list, so don't
                    // assume that it's valid after this.
                    if (index == 0) {
                        showNextToastLocked();
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }

    }
    
    private static final class ToastRecord {
        final int pid;
        final String pkg;
        ITransientNotification callback;
        int duration;
        Binder token;

        // ...
    }
}
```

前面的文章分析过系统服务 SystemService，这里 mService 是作为 Binder 的服务端，`onStart()` 中调用 `publishBinderService()` 来发布这个服务。然后判断了是不是系统 Toast，之后分别进行遍历:

```java
	int indexOfToastPackageLocked(String pkg){
        ArrayList<ToastRecord> list = mToastQueue;
        int len = list.size();
        for (int i=0; i<len; i++) {
            ToastRecord r = list.get(i);
            if (r.pkg.equals(pkg)) {
                return i;
            }
        }
        return -1;
    }

    int indexOfToastLocked(String pkg, ITransientNotification callback){
        IBinder cbak = callback.asBinder();
        ArrayList<ToastRecord> list = mToastQueue;
        int len = list.size();
        for (int i=0; i<len; i++) {
            ToastRecord r = list.get(i);
            if (r.pkg.equals(pkg) && r.callback.asBinder().equals(cbak)) {
                return i;
            }
        }
        return -1;
    }
```

首先，ToastRecord 这个类就是用来标志一个 Toast 的类，几个成员变量描述了 Toast 在 NMS 需要的几个属性，然后上面代码中二者的结果都是寻找所查包名是否已经有 ToastRecord 在队列之中了，如果能找到，就不新插入新的到列表尾部了，直接更新该 ToastRecord 的 duration 和 callback(这个 callback 即是 TN 对象)，否则就创建新的 ToastRecord 对象，并添加到列表尾部，然后判断下刚才的 index 是否为 0，如果为 0 就说明这个 package 的 Toast 处于第 0 个，该显示出来了，所以调用 `showNextToastLocked()` 方法，看下其代码：

```java
	void showNextToastLocked() {
        ToastRecord record = mToastQueue.get(0);
        while (record != null) {
            if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
            try {
                record.callback.show(record.token);
                scheduleTimeoutLocked(record);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Object died trying to show notification " + record.callback
                        + " in package " + record.pkg);
                // remove it from the list and let the process die
                int index = mToastQueue.indexOf(record);
                if (index >= 0) {
                    mToastQueue.remove(index);
                }
                keepProcessAliveIfNeededLocked(record.pid);
                if (mToastQueue.size() > 0) {
                    record = mToastQueue.get(0);
                } else {
                    record = null;
                }
            }
        }
    }

	private void scheduleTimeoutLocked(ToastRecord r){
        mHandler.removeCallbacksAndMessages(r);
        Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
        long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
        mHandler.sendMessageDelayed(m, delay);
    }

    protected class WorkerHandler extends Handler {
		// ...
        
        public void handleMessage(Message msg) {
            // ...
            switch (msg.what){
            	case MESSAGE_TIMEOUT:
                    handleTimeout((ToastRecord)msg.obj);
                    break;
                // ...
                
            }
            // ...
        }
    }

	private void handleTimeout(ToastRecord record){
        if (DBG) Slog.d(TAG, "Timeout pkg=" + record.pkg + " callback=" + record.callback);
        synchronized (mToastQueue) {
            int index = indexOfToastLocked(record.pkg, record.callback);
            if (index >= 0) {
                cancelToastLocked(index);
            }
        }
    }
	
	void cancelToastLocked(int index) {
        ToastRecord record = mToastQueue.get(index);
        try {
            record.callback.hide();
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to hide notification " + record.callback
                    + " in package " + record.pkg);
            // don't worry about this, we're about to remove it from
            // the list anyway
        }

        ToastRecord lastToast = mToastQueue.remove(index);
        mWindowManagerInternal.removeWindowToken(lastToast.token, true, DEFAULT_DISPLAY);

        keepProcessAliveIfNeededLocked(record.pid);
        if (mToastQueue.size() > 0) {
            // Show the next one. If the callback fails, this will remove
            // it from the list, so don't assume that the list hasn't changed
            // after this point.
            showNextToastLocked();
        }
    }
```

这里的代码很清晰，找到列表第一个 ToastRecord，然后调用其所属的 mCallback 即 TN 对象的 `show()` 方法，然后使用 `scheduleTimeoutLocked()` 方法，延时一个 Handler 的消息，在 Toast 指定的显示时间结束后发送一个 TimeOut 消息，然后在其中使用 `cancelToastLocaked()` 方法，这个方法调用了 TN 的 `hide()` 方法，然后从列表中移除了这个 ToastRecord。由此我们看出，Toast 的显示/隐藏的控制是由 NMS 完成的，然后通知客户端去做真正的操作。现在回到 TN 类，前面贴 TN 代码的时候，已经能看到了，调用TN 的  `show()` 和 `hide()` 的时候是发生在 Binder 线程的，所以不能直接在这里面进行 UI 的更新，所以这里通过发送 Handler 消息，去主线程执行 `handleShow()` 和 `handleHide()` 方法了:

```java
	public void handleShow(IBinder windowToken) {
            if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
                    + " mNextView=" + mNextView);
            // If a cancel/hide is pending - no need to show - at this point
            // the window token is already invalid and no need to do any work.
            if (mHandler.hasMessages(CANCEL) || mHandler.hasMessages(HIDE)) {
                return;
            }
            if (mView != mNextView) {
                // remove the old view if necessary
                handleHide();
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
                // We can resolve the Gravity here by using the Locale for getting
                // the layout direction
                final Configuration config = mView.getContext().getResources().getConfiguration();
                final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
                mParams.gravity = gravity;
                if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
                    mParams.horizontalWeight = 1.0f;
                }
                if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
                    mParams.verticalWeight = 1.0f;
                }
                mParams.x = mX;
                mParams.y = mY;
                mParams.verticalMargin = mVerticalMargin;
                mParams.horizontalMargin = mHorizontalMargin;
                mParams.packageName = packageName;
                mParams.hideTimeoutMilliseconds = mDuration ==
                    Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
                mParams.token = windowToken;
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeView(mView);
                }
                if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
                // Since the notification manager service cancels the token right
                // after it notifies us to cancel the toast there is an inherent
                // race and we may attempt to add a window after the token has been
                // invalidated. Let us hedge against that.
                try {
                    mWM.addView(mView, mParams);
                    trySendAccessibilityEvent();
                } catch (WindowManager.BadTokenException e) {
                    /* ignore */
                }
            }
        }

	public void handleHide() {
            if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
            if (mView != null) {
                // note: checking parent() just to make sure the view has
                // been added...  i have seen cases where we get here when
                // the view isn't yet added, so let's try not to crash.
                if (mView.getParent() != null) {
                    if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
                    mWM.removeViewImmediate(mView);
                }

                mView = null;
            }
        }
```

如果当前显示的 View 不是即将要显示的，那先使用  `handleHide()` 方法，这里面是调用 `WindowManager.removeViewImmediate()` 去移除这个窗口，上篇文章已经分析过这个方法了。接着 `handleShow()` 里面去调整了 WindowManager.LayoutParams 的多个参数，前面所说的 mGravity 是用来控制 Toast 所显示的位置的，而 mY 是用来控制相对的垂直方位上的偏移量的，默认是底部偏移 24dp。这个 LayoutParams 中还设置了一个 hideTimeoutMilliseconds，设置时间为该 Toast 的显示时间，按照注释所说，这个值用来控制显示 Toast 的超时时间，如果超过这个时间会被 hide，注释还说这样做是避免窗口被其他 APP 的 Toast 长期的盖住。我们去 WMS 的代码确实能够看到 `addWindow()` 中对 Toast 的特殊处理：

```java
	if (type == TYPE_TOAST) {
    	if (!getDefaultDisplayContentLocked().canAddToastWindowForUid(callingUid)) {
        	Slog.w(TAG_WM, "Adding more than one toast window for UID at a time.");
        	return WindowManagerGlobal.ADD_DUPLICATE_ADD;
		}
        // Make sure this happens before we moved focus as one can make the
        // toast focusable to force it not being hidden after the timeout.
        // Focusable toasts are always timed out to prevent a focused app to
        // show a focusable toasts while it has focus which will be kept on
        // the screen after the activity goes away.
        if (addToastWindowRequiresToken
        	|| (attrs.flags & LayoutParams.FLAG_NOT_FOCUSABLE) == 0
            || mCurrentFocus == null
            || mCurrentFocus.mOwnerUid != callingUid) {
        	mH.sendMessageDelayed(
            	mH.obtainMessage(H.WINDOW_HIDE_TIMEOUT, win),
                win.mAttrs.hideTimeoutMilliseconds);
    	}
	}
```

这里会发一个延时的消息来 hide 这个窗口。`handleShow()` 方法的最后是 `WindowManager.addView()` ，这一点在上篇文章已经分析过了。最后有一点需要提一下，在 Android 5.0 后，新增了每个 APP 的通知权限，如果用户禁用了某个 APP 的应用权限，因为 Toast 也使用了NMS，所以这个 APP 连 Toast 也没法显示了，所以现在官方其实更推荐使用 SnackBar 来替代 Toast。 