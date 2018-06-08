# 00-Context 源码分析

### 概念

`Context` 类是 Android 开发中最常见的，首先我们来对它的概念进行一个总结： 
> `Context` 是应用程序环境信息的接口，表示上下文，通过它可以获取到应用程序的资源和类，也可以进行应用程序的操作，比如启动 Activity、发送广播、接收 Intent 等等。

### 数量
那么一个老生常谈的问题，`Context` 的数量有多少个呢？`Context` 的总数 = `Activity` 数量 +  `Service` 数量 + 1 (`ApplicationContext`)。

### 继承关系
在源码中 `Context` 是一个 `abstract` 类，`ContextImpl` 实现了 `Context` 的 `abstract` 方法，它的继承关系是这样的：
* `Context-->ContextWrapper-->Application`
* `Context-->ContextWrapper-->Service`
* `Context-->ContextWrapper-->ContextThemeWrapper-->Activity`

从代码中可以看出来，`ContextWrapper/ContextThemeWrapper` 中需要传入一个 `Context` 实例给 `mBase`，然后在使用 `ContextWrapper/ContextThemeWrapper` 的方法时实际调用的是 `mBase` 的同名方法。从这里可以看出来，这可以算是一个装饰器模式，`ContextImpl` 相当于是组件具体实现类
， `ContextWrapper/ContextThemeWrapper` 是装饰器。

### Application Context 创建过程
这篇文章中不分析 `ActivityManagerService` 相关的操作，留待后面分析 `Activity` 启动流程时再分析。在打开 `Activity` 时会调用到 `ActivityThread#performLaunchActivity` 这个方法，可以看到里面有这样的代码:
```Java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    // ...
}
```
这里的 `r.packageInfo` 是一个 `LoadedApk` 对象，贴出它的 `makeApplication` 的部分代码：
```Java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {

    if (mApplication != null) {
            return mApplication;
    }
    // ...
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    appContext.setOuterContext(app);
    // ...
}
```
`makeApplication` 方法中会先判断 `mApplication` 是否为空，否则直接返回已经初始化的 `mApplication`。通过静态方法 `ContextImpl.createAppContext` 创建一个 `ContextImpl` 对象，然后使用 `mActivityThread#mInstrumentation.newApplication` 方法会创建一个 `Application` 对象，代码如下：
```Java
static public Application newApplication(Class<?> clazz, Context context) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    Application app = (Application)clazz.newInstance();
    app.attach(context);
    return app;
}
```
从这代码看出来这里调用了 `Application#attach(Context)` 方法：
```Java
final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}
```
将 `ContextImpl` 传给 `Application`，并使用 `attachBaseContext` 方法将 刚才创建的 `ContextImpl` 对象通过赋值给 `ContextWrapper` 的 `mBase` 变量，前面已经讲过这个 `mBase` 其实就是装饰器中的组件具体实现。然后回到上文 `makeApplication` 方法，那里使用了 `ContextImpl#setOuterContext(app)`，将刚创建的 `Application` 对象又传给了 `ContextImpl` 中赋值给 `mOuterContext` 变量。

### Application Context 获取过程
获取 `Application Context` 通常可以使用 `Context#getApplication` 或者 `Context#getApplicationContext` 方法。前者是去获取 `Activity/Service` 中的 `mApplication` 变量，这个变量是在 `Activity/Service` 创建时 `attach` 这一步赋值的，后续的文章再分析这个。现在来看看 `Context#getApplicationContext` 方法，这个方法会走到 `ContextImpl#getApplicationContext` 中：
```Java
public Context getApplicationContext() {
    return (mPackageInfo != null) 
    ? mPackageInfo.getApplication()
    : mMainThread.getApplication();
}
```
如果 `mPackageInfo` 这个 `LoadedApk` 对象为空，则调用 `mMainThread#getApplication` 获取 `ActivityThread` 对象中的 `mInitialApplication` 变量，否则就调用 `mPackageInfo#getApplication` 获取 `LoadedApk` 对象中的 `Application`，从源码中可以看到，这个 `Application` 对象是 `LoadedApk#makeApplication` 这一步时候赋值给 `LoadedApk` 中的变量的。

### Activity Context 创建过程
这里避不开要谈 `ActivityManagerService` 相关，但这篇文章只略谈相关的。在 `Activity` 的创建过程中，`ActivityManagerService` 会通过 `IPC` 的方式调用到 `ActivityThread` 的 `scheduleLaunchActivity` 方法，而这个方法因为是在 `Binder` 线程中执行，所以要去主线程需要用可以处理主线程的 `Handler` 发送一个 `LAUNCH_ACTIVITY` 事件。`Handler` 接受到此事件后，会调用 `handleLaunchActivity` 方法来进行处理，然后此方法中会使用 `performLaunchActivity` 来创建 `Activity`，贴出 `performLaunchActivity` 的关键代码：
```Java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    Activity activity = null;
    java.lang.ClassLoader cl = appContext.getClassLoader();
    activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    // ...
    appContext.setOuterContext(activity);
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback);
    // ...
}

private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
    final int displayId;
    try {
        displayId = ActivityManager.getService().getActivityDisplayId(r.token);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }

    ContextImpl appContext = ContextImpl.createActivityContext(
            this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);

    final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
    // For debugging purposes, if the activity's package name contains the value of
    // the "debug.use-second-display" system property as a substring, then show
    // its content on a secondary display if there is one.
    String pkgName = SystemProperties.get("debug.second-display.pkg");
    if (pkgName != null && !pkgName.isEmpty()
            && r.packageInfo.mPackageName.contains(pkgName)) {
        for (int id : dm.getDisplayIds()) {
            if (id != Display.DEFAULT_DISPLAY) {
                Display display = m.getCompatibleDisplay(id, appContext.getResources());
                appContext = (ContextImpl) appContext.createDisplayContext(display);
                break;
            }
        }
    }
    return appContext;
}
```
从上面的代码可以看出，其实和 `Application Cotnext` 的创建过程有一些类似。使用 `ContextImpl.createActivityContext` 来创建了 `ContextImpl` 实例，之后再 `Activity#attach` 方法中类似上文在 `Application` 中那样做的，把这个 `ContextImpl` 对象赋值给了 `ContextWrapper` 中的 `mBase` 变量。

### Service Context 创建过程
与 `Activity Context` 的创建过程差不多，但发生在 `ActivityThread#handleCreateService` 中，在创建 `ContextImpl` 这个步骤使用的 `ContextImpl.createAppContext` 这个方法。