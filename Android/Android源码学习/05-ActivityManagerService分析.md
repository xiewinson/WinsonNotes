# 05-ActivityManagerService分析

### 结构

`ActivityManagerService` 是 Android 最核心的服务，负责四大组件的启动、切换、调度以及应用程序的管理、调度，后面简称 AMS。与 `PackageManagerService` 一样，AMS 也是需要 `Binder` 进行客户端和服务端的通信，大部分博客和书籍是基于 API26 及之前讲解的，但是在 API26 版本，AMS 的代码结构出现了一些变化，例如 `ActivityManagerNative` 这个曾经很重要的类已经弃用了，本文基于 API27 进行分析。
1. 客户端：使用 `ActivityManager` 类，内部有个 `getService()` 方法取得 `IActivityManager`，使用它与服务端进行通信
2. 服务端：`ActivityManagerService` 继承于 `IActivityManager.Stub`，`IActivityManager` 是 `aidl` 接口，`Stub` 继承于 `Binder` 类中封装了 `Binder` 通信细节

### 客户端使用
在上篇文章中已经讲了系统服务 Manager 的获取方式，我们平时要使用 `ActivityManager` 时，一般通过这样的代码取得：
```Java
ActivityManager activityManager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
// 或者
ActivityManager activityManager1 =  getSystemService(ActivityManager.class);
```
这时候看看 `ActivityManager` 里的方法是怎么跟 `ActivityManagerService` 通信的，随便找一个方法，例如：
```Java
public boolean clearApplicationUserData() {
    return clearApplicationUserData(mContext.getPackageName(), null);
}

@RequiresPermission(anyOf={Manifest.permission.CLEAR_APP_USER_DATA, Manifest.permission.ACCESS_INSTANT_APPS})
public boolean clearApplicationUserData(String packageName, IPackageDataObserver observer) {
    try {
        return getService().clearApplicationUserData(packageName, observer, UserHandle.myUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
可以看到这里有个 `getService()` 方法去取得 `IActivityManager` 这个 `Binder` 对象：
```Java
/**
* @hide
*/
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}
```
可以看到这里取得 `IActivityManager` 的实例了，这是一个 `public static` 的方法，那我们可以直接 `ActivityManager.getService()` 来直接跟 `ActivityManagerService` 通信吗？答案是不行的，因为方法上面标注了 `@hide`，所以我们没法直接使用它，但是反射是可以获取到的，虽然目前还行，但是 Android P 开始限制反射使用系统隐藏 API 了，所以还是放弃这个念头吧。`ActivityManagerService` 是一个非常核心的服务，直接让所有人使用它的所有方法不太安全，而且考虑到代码结构等等因素，封装一层 `ActivityManager` 只允许开发者调用部分方法也是极好的。这时候看看 `IActivityManagerSingleton.get()` 是怎么做的
```Java
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}

class ActivityManager {
    // ...
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
            }
        };
    // ...
}
```
这里搞了一个支持泛型的单例模式，实现了懒加载，加载的时候就是从 `ServiceManager` 中去获取 `Binder` 对象。前面的文章已经讲过了，`ServiceManager#getService` 获取到 `Binder` 对象，后面在讲 `Activity` 的启动流程时再细讲 `ServiceManager#getService` 的源码分析。以上的分析是我们自己要和 `ActivityManagerService` 通信时的流程分析。在 `ActivityThread` 中，当然也有大量和 `ActivityManagerService` 通信的例子，看看它是怎么获取 `Binder` 对象的：
```Java
ActivityManager.getService().activityResumed(token);
```
每个地方的使用就这么的直接，不让我们直接用 `IActivityManager`，但它自己是直接使用的。
### 服务端的初始化
通常在 `SystemServer` 中启动一个服务，直接调用 `SystemServiceManager#startService` 方法就行了，例如启动 `PowerManagerService`：
```Java
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
```
进入这个 `startService` 的源码看看：
```Java
public SystemService startService(String className) {
    final Class<SystemService> serviceClass;
    try {
        serviceClass = (Class<SystemService>)Class.forName(className);
    } catch (ClassNotFoundException ex) {
        Slog.i(TAG, "Starting " + className);
        throw new RuntimeException("Failed to create service " + className
                + ": service class not found, usually indicates that the caller should "
                + "have called PackageManager.hasSystemFeature() to check whether the "
                + "feature is available on this device before trying to start the "
                + "services that implement it", ex);
    }
    return startService(serviceClass);
}

public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.clas);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }

        startService(service);
        return service;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
}

public void startService(@NonNull final SystemService service) {
    // Register it.
    mServices.add(service);
    // Start it.
    long time = SystemClock.elapsedRealtime();
    try {
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException("Failed to start service " + service.getClass().getName()
                + ": onStart threw an exception", ex);
    }
    warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
}
```
这几个方法很长，但其实加起来就是反射获取类然后创建一个 `SystemService` 的子类的对象，然后存在一个名为 `mServices` 的 `ArrayList` 之中，再然后就调用这个对象的 `onStart` 方法，上文说的 `PowerManagerService` 就是 `SystemService` 的子类，看 `SystemService` 的注释描述：这是一个给运行在系统进程的 `service` 的基类，去实现需要的生命周期回调方法。其实大部分的服务都是这样的步骤去启动，但是，AMS 并不是这个样子的。前面的文章讲过，`startBootstrapService` 这个方法里用来启动的是那些相互交织影响的服务，AMS 就是这样的，在 `startBootstrapService` 中有这样的代码：
```Java
mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
mActivityManagerService.setInstaller(installer);
```
最上面的代码没有直接启动 AMS，而是启动的其中的 `Lifecycle`，因为前面的分析已经讲了，AMS 继承的是 `IActivityManager.Stub` 而非 `SystemService`，所以只能这样 “曲线救国”。
```Java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start();
    }

    @Override
    public void onCleanupUser(int userId) {
        mService.mBatteryStatsService.onCleanupUser(userId);
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```
可以看到 `Lifecycle` 的作用其实就是在里面去创建 AMS 然后调用各个生命周期回调方法。之后的一些工作是初始化一些变量，通过名称可以推测大致作用，目前暂时不管，等后面再遇到的时候再结合实际场景进行进一步了解。回到 `startBootstrapService` 方法中，然后看到这行 `mActivityManagerService.setSystemProcess()`：
```Java
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this));
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this));
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        synchronized (this) {
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true;
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);
                updateOomAdjLocked();
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
```
这里主要做了两件事，一是注册 AMS、meminfo、gfxinfo 等到 `ServiceManager` 中；二是根据 PMS 查询所返回的 `ApplicationInfo` 初始化 Android 运行环境，并创建一个代表 `system_server` 进程的 `ProcessRecord`，从此，`system_server` 进程也并入了 AMS 的管理。

之后在 `SystemServer` 中调用 `installSystemProviders` 方法，用于启动 `SettingsProvider`，这个的作用是提供给别的 Service 查询配置信息，这是一个 `ContentProvider`。最后调用了 `systemReady` 方法，表示完成了系统就绪的必要工作，然后调用 `startHomeActivityLocked` 方法启动 Home Activity，再借 `WindowManagerService` 之手通过 `SurfaceFlinger` 去停止开机动画，至此 AMS 的启动流程完成。
