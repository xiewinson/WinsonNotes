# PopupWindow 源码分析

PopupWindow 的作用是用来显示一个悬浮窗口，虽然名字里有 Window，但它并不是继承于 Window。与之相关的类还有 ListPopupWindow、PopupMenu 等，这些东西的实质还是显示一个 PopupWindow，因此这篇文章分析一下其源码实现。

要显示 PopupWindow 的方法只有两个， `showAtLocation()` 和 `showAsDropDown()`，前者相当于是绝对位置，后者相当于是相对于 target 下方的相对位置（这个说法不太准确，因为如果 PopupWindow 超出父窗口的可显示边缘的话，会默认调整位置让其可以全部显示，不过 `setClippingEnabled(false)` 这个方法可以防止这个做法）。现在看看显示 PopupWindow 的 show 方法的源码：

```java
	public void showAtLocation(View parent, int gravity, int x, int y) {
        showAtLocation(parent.getWindowToken(), gravity, x, y);
    }

	@hide
	public void showAtLocation(IBinder token, int gravity, int x, int y) {
        if (isShowing() || mContentView == null) {
            return;
        }

        TransitionManager.endTransitions(mDecorView);

        detachFromAnchor();

        mIsShowing = true;
        mIsDropdown = false;
        mGravity = gravity;

        final WindowManager.LayoutParams p = createPopupLayoutParams(token);
        preparePopup(p);

        p.x = x;
        p.y = y;

        invokePopup(p);
    }

	public void showAsDropDown(View anchor, int xoff, int yoff, int gravity) {
        if (isShowing() || mContentView == null) {
            return;
        }

        TransitionManager.endTransitions(mDecorView);

        attachToAnchor(anchor, xoff, yoff, gravity);

        mIsShowing = true;
        mIsDropdown = true;

        final WindowManager.LayoutParams p = createPopupLayoutParams(anchor.getWindowToken());
        preparePopup(p);

        final boolean aboveAnchor = findDropDownPosition(anchor, p, xoff, yoff,
                p.width, p.height, gravity);
        updateAboveAnchor(aboveAnchor);
        p.accessibilityIdOfAnchor = (anchor != null) ? anchor.getAccessibilityViewId() : -1;

        invokePopup(p);
    }
	
```

PopupWindow 提供了一系列的方法，例如 `setInputMethodMode()`、`setAnimationStyle()`、`setTouchable()` 等，根据经验这些一看就是设置 Window  相关属性的，果然有个方法去创建 WindowManager.LayoutParams:

```java
	private WindowManager.LayoutParams createPopupLayoutParams(IBinder token) {
        final WindowManager.LayoutParams p = new WindowManager.LayoutParams();

        // These gravity settings put the view at the top left corner of the
        // screen. The view is then positioned to the appropriate location by
        // setting the x and y offsets to match the anchor's bottom-left
        // corner.
        p.gravity = computeGravity();
        p.flags = computeFlags(p.flags);
        p.type = mWindowLayoutType;
        p.token = token;
        p.softInputMode = mSoftInputMode;
        p.windowAnimations = computeAnimationResource();

        if (mBackground != null) {
            p.format = mBackground.getOpacity();
        } else {
            p.format = PixelFormat.TRANSLUCENT;
        }

        if (mHeightMode < 0) {
            p.height = mLastHeight = mHeightMode;
        } else {
            p.height = mLastHeight = mHeight;
        }

        if (mWidthMode < 0) {
            p.width = mLastWidth = mWidthMode;
        } else {
            p.width = mLastWidth = mWidth;
        }

        p.privateFlags = PRIVATE_FLAG_WILL_NOT_REPLACE_ON_RELAUNCH
                | PRIVATE_FLAG_LAYOUT_CHILD_WINDOW_IN_PARENT_FRAME;

        // Used for debugging.
        p.setTitle("PopupWindow:" + Integer.toHexString(hashCode()));

        return p;
    }
```

这是个 private 方法，也只有这个入口去设置 LayoutParams，所以我们是没有办法去修改除了 PopupWindow 提供了方法来修改的参数了。注意，这里的 token 是怎么获取的！两个方法中的 token 是需要传入一个 View 使用 `getWindowToken()` 来获取的，以上上上篇文章中的所学知道，这个方法所获取到的就是该 View 所在窗口所拥有的 ViewRootImpl 对象中的 IWindow 对象，并且那篇文章提过，系统类型的窗口的 token 很特殊不讨论，应用窗口的 token 必须是 Activity 的 AppToken，使用 IWindow 作为 token 的是子窗口，这里面这么使用，那么就只能是子窗口了，别无他法，PopupWindow 只能是个子窗口了。那我们看看它的 type，这里代码中可以看到取值自 mWindowLayoutType，看看是如何被赋值的：

```java
private int mWindowLayoutType = WindowManager.LayoutParams.TYPE_APPLICATION_PANEL;
public void setWindowLayoutType(int layoutType) {
	mWindowLayoutType = layoutType;
}
```

默认就赋值的是 TYPE_APPLICATION_PANEL，这个 type 不行啊，它的大小会受制于父窗口，例如在 Dialog 中显示一个 PopupWindow 的话，它的大小就会受制于 Dialog，超出 Dialog 的部分就被截断了，效果很差劲！！还好它提供了一个 `setWindowLayoutType()` 来修改，然而这个方法是 API23 添加的，这就很坑了，之前的版本想改个 type 就得使用反射的方式来做，改成 TYPE_APPLICATION_ATTACHED_DIALOG 这个 type 的话，大小就不会再受制于父窗口了，但用反射的方式总感觉不太好，还好 v4 包里提供了一个兼容类 PopupWindowCompat，看看其源码：

```java
	static final PopupWindowImpl IMPL;
    static {
        final int version = android.os.Build.VERSION.SDK_INT;
        if (version >= 23) {
            IMPL = new Api23PopupWindowImpl();
        } else if (version >= 21) {
            IMPL = new Api21PopupWindowImpl();
        } else if (version >= 19) {
            IMPL = new KitKatPopupWindowImpl();
        } else {
            IMPL = new BasePopupWindowImpl();
        }
    }
	
	public static void setWindowLayoutType(PopupWindow popupWindow, int layoutType) {
        IMPL.setWindowLayoutType(popupWindow, layoutType);
    }

	static class BasePopupWindowImpl implements PopupWindowImpl {
    	@Override
        public void setWindowLayoutType(PopupWindow popupWindow, int layoutType) {
            if (!sSetWindowLayoutTypeMethodAttempted) {
                try {
                    sSetWindowLayoutTypeMethod = PopupWindow.class.getDeclaredMethod(
                            "setWindowLayoutType", int.class);
                    sSetWindowLayoutTypeMethod.setAccessible(true);
                } catch (Exception e) {
                    // Reflection method fetch failed. Oh well.
                }
                sSetWindowLayoutTypeMethodAttempted = true;
            }

            if (sSetWindowLayoutTypeMethod != null) {
                try {
                    sSetWindowLayoutTypeMethod.invoke(popupWindow, layoutType);
                } catch (Exception e) {
                    // Reflection call failed. Oh well.
                }
            }
        }

        @Override
        public int getWindowLayoutType(PopupWindow popupWindow) {
            if (!sGetWindowLayoutTypeMethodAttempted) {
                try {
                    sGetWindowLayoutTypeMethod = PopupWindow.class.getDeclaredMethod(
                            "getWindowLayoutType");
                    sGetWindowLayoutTypeMethod.setAccessible(true);
                } catch (Exception e) {
                    // Reflection method fetch failed. Oh well.
                }
                sGetWindowLayoutTypeMethodAttempted = true;
            }

            if (sGetWindowLayoutTypeMethod != null) {
                try {
                    return (Integer) sGetWindowLayoutTypeMethod.invoke(popupWindow);
                } catch (Exception e) {
                    // Reflection call failed. Oh well.
                }
            }
            return 0;
        }
    }

```

好的，在 API23 以上，直接使用 get/set 方法去获取/修改 type，而 API23 以下则提供了反射的方法去做到，终归是做到了。关于 token 和 type 的问题和修改弄清楚了，下面继续看它是怎么秀的。调整 PopupWindow 的位置的代码我们就不多说了，下面有个关键的方法是 `preparePopup()`：

```java
	private void preparePopup(WindowManager.LayoutParams p) {
        if (mContentView == null || mContext == null || mWindowManager == null) {
            throw new IllegalStateException("You must specify a valid content view by "
                    + "calling setContentView() before attempting to show the popup.");
        }

        // The old decor view may be transitioning out. Make sure it finishes
        // and cleans up before we try to create another one.
        if (mDecorView != null) {
            mDecorView.cancelTransitions();
        }

        // When a background is available, we embed the content view within
        // another view that owns the background drawable.
        if (mBackground != null) {
            mBackgroundView = createBackgroundView(mContentView);
            mBackgroundView.setBackground(mBackground);
        } else {
            mBackgroundView = mContentView;
        }

        mDecorView = createDecorView(mBackgroundView);

        // The background owner should be elevated so that it casts a shadow.
        mBackgroundView.setElevation(mElevation);

        // We may wrap that in another view, so we'll need to manually specify
        // the surface insets.
        p.setSurfaceInsets(mBackgroundView, true /*manual*/, true /*preservePrevious*/);

        mPopupViewInitialLayoutDirectionInherited =
                (mContentView.getRawLayoutDirection() == View.LAYOUT_DIRECTION_INHERIT);
    }

```

PopupWindow 有提供一个 `setBackgroundDrawable()` 方法用来设定一个背景，在 `preparePopup()` 中判断这个是否为空，如果不为空的话，创建一个 PopupBackgroundView，背景设为这个 drawable，包裹你传入的 contentView， PopupBackgroundView 继承自 FrameLayout。然后把这个背景 View 传入 `createDecorView()` 方法中，如果没有 View，则直接传入 contentView：

```java
	 private PopupDecorView createDecorView(View contentView) {
        final ViewGroup.LayoutParams layoutParams = mContentView.getLayoutParams();
        final int height;
        if (layoutParams != null && layoutParams.height == WRAP_CONTENT) {
            height = WRAP_CONTENT;
        } else {
            height = MATCH_PARENT;
        }

        final PopupDecorView decorView = new PopupDecorView(mContext);
        decorView.addView(contentView, MATCH_PARENT, height);
        decorView.setClipChildren(false);
        decorView.setClipToPadding(false);

        return decorView;
    }
```

这里又创建了一个 PopupDecorView，将刚才的 View 使用 `addView()` 加载进去，这个 PopupDecorView 跟之前 PhoneWindow 中的 DecorView 不是一个东西，但这个也是继承自 FrameLayout，在里面处理了点击返回键和点击超出它的 View 显示区域的地方调用 `dismiss()` 关闭 PopupWindow:

```java
		public boolean dispatchKeyEvent(KeyEvent event) {
            if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
                if (getKeyDispatcherState() == null) {
                    return super.dispatchKeyEvent(event);
                }

                if (event.getAction() == KeyEvent.ACTION_DOWN && event.getRepeatCount() == 0) {
                    final KeyEvent.DispatcherState state = getKeyDispatcherState();
                    if (state != null) {
                        state.startTracking(event, this);
                    }
                    return true;
                } else if (event.getAction() == KeyEvent.ACTION_UP) {
                    final KeyEvent.DispatcherState state = getKeyDispatcherState();
                    if (state != null && state.isTracking(event) && !event.isCanceled()) {
                        dismiss();
                        return true;
                    }
                }
                return super.dispatchKeyEvent(event);
            } else {
                return super.dispatchKeyEvent(event);
            }
        }
		
		public boolean onTouchEvent(MotionEvent event) {
            final int x = (int) event.getX();
            final int y = (int) event.getY();

            if ((event.getAction() == MotionEvent.ACTION_DOWN)
                    && ((x < 0) || (x >= getWidth()) || (y < 0) || (y >= getHeight()))) {
                dismiss();
                return true;
            } else if (event.getAction() == MotionEvent.ACTION_OUTSIDE) {
                dismiss();
                return true;
            } else {
                return super.onTouchEvent(event);
            }
        }
```

注意一定要调用 `setFocusable(true)`，因为需要获得焦点了，PopupWindow 才能截获键盘事件，这样点击返回键才会关闭它自己，而默认的情况下会被设置为 false。所以 `preparePopup()` 这一步骤就是去搞定了 PopupDecorView，最后一步所做的是 `invokePopup()`：

```java
	private void invokePopup(WindowManager.LayoutParams p) {
        if (mContext != null) {
            p.packageName = mContext.getPackageName();
        }

        final PopupDecorView decorView = mDecorView;
        decorView.setFitsSystemWindows(mLayoutInsetDecor);

        setLayoutDirectionFromAnchor();

        mWindowManager.addView(decorView, p);

        if (mEnterTransition != null) {
            decorView.requestEnterTransition(mEnterTransition);
        }
    }
```

这一步所做的就是调用 `WindowManager.addView()` 去添加窗口了。最后，至于 `dismiss()` 方法，就是开启一个 Transition 做消失动画， 完成后调用 `dismissImmediate()` 方法，在其中使用了 `WindowManager.removeView()` 去移除窗口。

至此，就分析完了 Android 中常见的几种窗口，下一篇文章将重点分析 ViewRootImpl 的部分机制。