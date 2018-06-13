# Activity 结束流程分析

当我们要结束一个 `Activity` 的时候通常有两种操作：一是在代码中调用 `Activity.finish()`，二是点击 Back 按键。我们看看第二种方式的源码：

```java
    public class Activity extends ContextThemeWrapper implements ... {
        // ...
        public boolean onKeyDown(int keyCode, KeyEvent event)  {
             if (keyCode == KeyEvent.KEYCODE_BACK) {
                if (getApplicationInfo().targetSdkVersion
                        >= Build.VERSION_CODES.ECLAIR) {
                    event.startTracking();
                } else {
                    onBackPressed();
                }
                return true;
            }
            // ...
        }
        
        public void onBackPressed() {
            if (mActionBar != null && mActionBar.collapseActionView()) {
                return;
            }

            FragmentManager fragmentManager = mFragments.getFragmentManager();

            if (fragmentManager.isStateSaved() ||
                !fragmentManager.popBackStackImmediate()) {
                finishAfterTransition();
            }
    	}
        public void finishAfterTransition() {
            if (!mActivityTransitionState.startExitBackTransition(this)) {
                finish();
            }
        }
    }
```

所以最终都会走到 `Activity.finish()` 方法中，看看它的源码：

```java
	public void finish() {
        finish(DONT_FINISH_TASK_WITH_ACTIVITY);
    }
    
    private void finish(int finishTask) {
        if (mParent == null) {
            int resultCode;
            Intent resultData;
            synchronized (this) {
                resultCode = mResultCode;
                resultData = mResultData;
            }
            if (false) Log.v(TAG, "Finishing self: token=" + mToken);
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess(this);
                }
                if (ActivityManager.getService()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
        } else {
            mParent.finishFromChild(this);
        }

        if (mIntent != null && 
        	mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)) {
			getAutofillManager().onPendingSaveUi(
                AutofillManager.PENDING_UI_OPERATION_RESTORE,
                mIntent.getIBinderExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN));
        }
    }
```

从上面代码可以看到最终走到 AMS 的 `finishActivity()` 方法中，接下来的方法调用步骤是 `ActivityManagerService.finishActivity` -> `ActivityStack.requestFinishActivityLocked()` -> `ActivityStack.finishActivityLocked()` ，看看这个方法中做了什么：

```java
	final boolean finishActivityLocked(ActivityRecord r, int resultCode,
		Intent resultData,String reason, boolean oomAdj, boolean pauseImmediately) {
    	// ...
        r.makeFinishingLocked();
        // ...
        startPausingLocked(false, false, null, pauseImmediately);
        // ...
    }
```

我们先看看 `ActivityRecord.makeFinishingLocked()` 方法：

```java
	void makeFinishingLocked() {
        if (finishing) {
            return;
        }
        finishing = true;
        if (stopped) {
            clearOptionsLocked();
        }

        if (service != null) {
            service.mTaskChangeNotificationController.notifyTaskStackChanged();
        }
    }
```

这个方法主要是把变量 `finishing` 设置为 `true`，之后会说到这个标志的作用，下面说  `startPausingLocked()`  开始的操作，分为三个部分来叙述：

1. 按我们上文的分解，这个 `startPausingLocked()` 接下来的方法调用想必已很清楚，即  `startPausingLocked()` -> `IApplicationThread.schedulePauseActivity()` -> `ActivityThread.handlePauseActivity()` -> `ActivityThread.performPauseActivity()` -> `ActivityThread.performPauseActivityIfNeeded()` -> `Instrumentation.callActivityOnPause()` -> `Activity.performPause()` -> `Activity.onPause()`，就这样一步一步地调用了这个被结束的 `Activity` 的 `onPause()` 生命周期方法。

2. 依旧是上篇文章的分析，在 `ActivityThread.handlePauseActivity()` 这一步骤中，会去调用 AMS 的 `activityPaused()` 方法，接下来的方法调用是 `ActivityStack.getStackLocked()` -> `ActivityStack.activityPausedLocked()` -> `ActivityStack.completePauseLocked()` -> ``ActivityStackSupervisor.resumeFocusedStackTopActivityLocked` -> `ActivityStack.resumeTopActivityUncheckedLocked` -> `IApplicationThread.scheduleResumeActivity() ` -> `ActivityThread.scheduleResumeActivity()` -> `ActivityThread.handleResumeMessage()` ->  `Activity.performResume()` -> `Activity.performRestart()` -> `Instrumentation.callActivityOnRestart()` -> `Activity.onRestart()` -> `Activity.performStart()`  -> `Instrumentation.callActivityOnStart()` -> `Activity.onStart()` ->  `Instrumentation.callActivityOnResume()` -> `Activity.onResume()`，这一片的方法下来，又完成了返回到的上一个 `Activity` 的操作，执行这个 `Activity` 的 `onRestart()`、`onStart()`、`onResume`。

3. 最后的步骤当然是被结束的这个 `Activity` 是怎样去执行 `onStop()` 和 `onDestroy()` 的。像上篇文章分析的那样，在 `ActivityThread.handleResumeActivity()` 中，最终会往消息队列中加入一个 `Idle` 以便在队列执行完前一系列的工作处于空闲的时候去调用 `ActivityManagerService.activityIdle()` 方法，随后会走到 `ActivityStackSupervisor.activityIdleInternalLocked()` 去处理 stop 和 destroy。首先我们回顾 `ActivityStack.completePauseLocked()` 方法：

   ```java
       private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
       	// ...
           ActivityRecord prev = mPausingActivity;
   		if (prev.finishing) {
               prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false);
   		} 
           // ...
       }
   ```

   这个方法判断了 `ActivityRecord.finishing`，对，就是文章前面分析过调用  `ActivityStack.finishActivityLocked()` 时通过 `ActivityRecord.makeFinishingLocked()` 设置了这个标志为 `true`，所以此时会执行 `ActivityStack.finishCurrentActivityLocked()` 方法：

   ```java
   	final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, 
   		int mode, boolean oomAdj) {
       	// ...
           mStackSupervisor.mFinishingActivities.add(r);
   		// ...
       }
   ```

   这个方法将 `ActivityRecord` 加入到了 `ActivityStackSupervisor` 的 `mFinishingActivities` 这个 `ArrayList` 之中，按注释所说，这个 `ArrayList` 是用来存储准备结束但在等待前面的 `Activity` 完成操作的 `Activity`。现在再来看  `ActivityStackSupervisor.activityIdleInternalLocked()` 的代码：

   ```java
   	final ActivityRecord activityIdleInternalLocked(final IBinder token,
   		boolean fromTimeout, boolean processPausingActivities, Configuration config) {
       	// ...
           ArrayList<ActivityRecord> finishes = null;
   		// ...
           if ((NF = mFinishingActivities.size()) > 0) {
               finishes = new ArrayList<>(mFinishingActivities);
               mFinishingActivities.clear();
           }
           // ...
           for (int i = 0; i < NF; i++) {
               r = finishes.get(i);
               final ActivityStack stack = r.getStack();
               if (stack != null) {
                   activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
               }
           }
       }
   ```

   从代码可以看到：针对需要 finish（即 `finishing` 标志为 `true`）的 `ActivityRecord`，是不会再去调用 `ActivityStack.stopActivityLocked()` 方法了，最终会调用 `ActivityStack.destroyActivityLocked()` 方法：

   ```java
   	final boolean destroyActivityLocked(ActivityRecord r, 
   		boolean removeFromApp, String reason) {
   		// ...
           r.app.thread.scheduleDestroyActivity(r.appToken, r.finishing,
                           r.configChangeFlags);
           // ...
       }
   ```

   好的，这里又是通过 `IApplicationThread` 去调用 `ActivityThread.scheduleDestroyActivity()`  -> `ActivityThread.handleDestroyActivity()` -> `ActivityThread.performDestroyActivity()` ，看看最后这个方法：

   ```java
   	private ActivityClientRecord performDestroyActivity(IBinder token,
   		boolean finishing,int configChanges, boolean getNonConfigInstance) {
           ActivityClientRecord r = mActivities.get(token);
           Class<? extends Activity> activityClass = null;
           if (localLOGV) Slog.v(TAG, "Performing finish of " + r);
           if (r != null) {
               activityClass = r.activity.getClass();
               r.activity.mConfigChangeFlags |= configChanges;
               if (finishing) {
                   r.activity.mFinished = true;
               }
   
               performPauseActivityIfNeeded(r, "destroy");
   
               if (!r.stopped) {
                   try {
                       r.activity.performStop(r.mPreserveWindow);
                   } catch (SuperNotCalledException e) {
                       throw e;
                   } catch (Exception e) {
                       if (!mInstrumentation.onException(r.activity, e)) {
                           throw new RuntimeException(
                                   "Unable to stop activity "
                                   + safeToComponentShortString(r.intent)
                                   + ": " + e.toString(), e);
                       }
                   }
                   r.stopped = true;
                   EventLog.writeEvent(LOG_AM_ON_STOP_CALLED, UserHandle.myUserId(),
                           r.activity.getComponentName().getClassName(), "destroy");
               }
               if (getNonConfigInstance) {
                   try {
                       r.lastNonConfigurationInstances
                               = r.activity.retainNonConfigurationInstances();
                   } catch (Exception e) {
                       if (!mInstrumentation.onException(r.activity, e)) {
                           throw new RuntimeException(
                                   "Unable to retain activity "
                                   + r.intent.getComponent().toShortString()
                                   + ": " + e.toString(), e);
                       }
                   }
               }
               try {
                   r.activity.mCalled = false;
                   mInstrumentation.callActivityOnDestroy(r.activity);
                   if (!r.activity.mCalled) {
                       throw new SuperNotCalledException(
                           "Activity " + safeToComponentShortString(r.intent) +
                           " did not call through to super.onDestroy()");
                   }
                   if (r.window != null) {
                       r.window.closeAllPanels();
                   }
               } catch (SuperNotCalledException e) {
                   throw e;
               } catch (Exception e) {
                   if (!mInstrumentation.onException(r.activity, e)) {
                       throw new RuntimeException(
                      		"Unable to destroy activity " + 
                           safeToComponentShortString(r.intent) + ": " + e.toString(), e);
                   }
               }
           }
           mActivities.remove(token);
           StrictMode.decrementExpectedActivityCount(activityClass);
           return r;
       }
   ```

   这里面可以看到首先执行了 `performPauseActivityIfNeeded(r, "destroy")`，当然这个方法里面会判断，如果 `ActivityClientRecord` 对象中 `paused` 标志为 `true`，就不会继续执行了；然后判断 `ActivityClientRecord` 对象的 `stop` 标志，不为 `true` 的情况下调用 `Activity.performStop()` 将会走 `onStop()` 生命周期；最后，使用 `Instrumentation.callActivityOnDestroy()` -> `Activity.performDestroy()` -> `Activity.onDestroy()`。