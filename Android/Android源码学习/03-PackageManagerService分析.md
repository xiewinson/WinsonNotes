# 03-PackageManagerService分析

### 主要构成
`PackageManagerService` 负责 Android 中应用程序的安装、卸载、信息查询等工作，后面简称之为 PMS。Android 中的这种东西都是使用 `Binder` 通信的，PMS 也不例外，所以它的代码分为客户端和服务端两部分：

1. 客户端：在使用 `Context#getPackageManager` 取得的对象其实是 `ApplicationPackageManager`，这个类被标注了 `@hide` 并继承于 `PackageManager` 这个抽象类，`ApplicationPackageManager` 实际上才是真正实现了去和服务端通信的工作。
2. 服务端：`PackageManagerService` 继承于 `IPackageManager.Stub`，`IPackageManager` 是一个 `aidl` 接口类，定义了服务端要实现的方法，`IPackageManager.Stub` 是一个编译生成的继承于 `Binder` 的 抽象类，封装了 `Binder` 通信的具体细节，而 `PackageManagerService` 继承 `Stub` 对这些抽象方法进行了实现让客户端得以获取结果。
### 客户端的启动流程
首先看 `Context#getPackageManager` 的源码，按照之前分析 `Context` 的文章中所知，我们应该去 `ContextImpl` 中才会找到真正实现：
```Java
public PackageManager getPackageManager() {
    if (mPackageManager != null) {
        return mPackageManager;
    }
    IPackageManager pm = ActivityThread.getPackageManager();
    if (pm != null) {
        // Doesn't matter if we make more than one instance.
        return (mPackageManager = new ApplicationPackageManager(this, pm));
    }
    return null;
}
```
这时候看 `ActivityThread#getPackageManager` 做了什么
```Java
public static IPackageManager getPackageManager() {
    if (sPackageManager != null) {
        //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
        return sPackageManager;
    }
    IBinder b = ServiceManager.getService("package");
    //Slog.v("PackageManager", "default service binder = " + b);
    sPackageManager = IPackageManager.Stub.asInterface(b);
    //Slog.v("PackageManager", "default service = " + sPackageManager);
    return sPackageManager;
}
```
根据以上两段代码，可以看到这里从 `ServiceManager` 里已加入的 `package` 服务取得用来跟服务端通信的 Binder Proxy 对象 `IPackageManager`，然后传递给 `ApplicationPackageManager` 的构造函数，`ApplicationPackageManager` 里的具体方法的代码就不贴了，就是利用这个 `IPackageManager` 进行进程间通信去获取想要的结果
### 服务端的启动流程
上篇文章介绍了 `SystemServer` 的作用，其中就有去启动一些系统服务例如本文的主角 PMS。我们去 `SystemServer` 中找到 PMS 相关的代码
```Java
private void startBootstrapServices() {
    // ...
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    // ...
}      
```
可以看到这里去调用了它的 `main` 方法
```Java
public static PackageManagerService main(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();

    PackageManagerService m = new PackageManagerService(context, installer, factoryTest, onlyCore);
    m.enableSystemUserPackages();
    ServiceManager.addService("package", m);
    final PackageManagerNative pmn = m.new PackageManagerNative();
    ServiceManager.addService("package_native", pmn);
    return m;
}
```
这里面创建了 `PackageManagerService` 对象，然后利用 `Binder` 通信将它添加到了 `ServiceManager` 中。此时再进入 `PackageManagerService` 的构造函数中，这个代码太长就不贴了。这里首先进行一些准备工作，扫描和解析系统的一些 `XML` 文件并将其信息保存在特定的数据结构中，为下一步骤的工作提供重要的参考信息。之后的工作就是扫描系统中的 APK 了，先后从 `/system/frameworks`、`/system/app`、`/vendor/app` 三个系统 package 目录和 `/data/app`、`/data/app-private` 两个非系统 package 目录下扫描 APP，然后解析 APK 中的 `AndroidManifest.xml` 文件，解析的结果在 `PackageParser.Package` 下的 `activities`、`services`、`permissions` 等几个 `ArrayList` 中。但这里要注意，以 `activities` 为例，虽然保存在 `ArrayList<Activity>` 中，但这个 `Activity` 并非我们通常所说的 `Activity`，它继承于 `Component<ActivityIntentInfo>`，`ActivityIntentInfo` 的顶层基类是 `IntentFilter`。再然后，又将 package 的内容保存在 `PackageManagerService#mActivities` 等响应的组件中，以 `mActivities` 为例，它是一个 `ActivityIntentResolver` 对象，它其中又有一个 ` mActivities` 对象，是 `ArrayMap<ComponentName, PackageParser.Activity>`。`ActivityIntentInfo` 继承于 `IntentResolver` 类，看看 `addActivity` 这个重点方法：
```Java
public final void addActivity(PackageParser.Activity a, String type) {
        mActivities.put(a.getComponentName(), a);
        if (DEBUG_SHOW_INFO)
            Log.v(
                TAG, "  " + type + " " +
                (a.info.nonLocalizedLabel != null ? a.info.nonLocalizedLabel : a.info.name) + ":");
        if (DEBUG_SHOW_INFO)
            Log.v(TAG, "    Class=" + a.info.name);
        final int NI = a.intents.size();
        for (int j=0; j<NI; j++) {
            PackageParser.ActivityIntentInfo intent = a.intents.get(j);
            if ("activity".equals(type)) {
                final PackageSetting ps = mSettings.getDisabledSystemPkgLPr(intent.activity.info.packageName);
                final List<PackageParser.Activity> systemActivities =
                        ps != null && ps.pkg != null ? ps.pkg.activities : null;
                adjustPriority(systemActivities, intent);
            }
            if (DEBUG_SHOW_INFO) {
                Log.v(TAG, "    IntentFilter:");
                    intent.dump(new LogPrinter(Log.VERBOSE, TAG), "      ");
            }
            if (!intent.debugCheck()) {
                Log.w(TAG, "==> For Activity " + a.info.name);
            }
            addFilter(intent);
        }
    }
```
这个方法首先添加 `activity` 到 `ArrayMap<ComponentName, PackageParser.Activity> mActivities` 中，然后调用了父类 `IntentResolver` 的 `addFilter` 方法，这里是为了加快匹配的速度所以存储更多维度的匹配信息，用表格说明 `IntentResolver` 中用来保存这些维度的匹配信息的作用：

| 变量                 | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| mSchemeToFilter      | 保存 uri 和 scheme 相关的 IntentFilter 信息                  |
| mActionToFilter      | 保存仅设置Action条件的IntentFilter信息                       |
| mTypedActionToFilter | 用于保存既设置了Action又设置了Data的MIME类型的IntentFilter信息 |
| mFilters             | 用于保存所有IntentFilter信息                                 |
| mWildTypeToFilter    | 保存设置了Data类型类似“image/*”的IntentFilter，但是设置MIME类型类似“Image/jpeg”的不算在此类 |
| mTypeToFilter        | 除了包含mWildTypeToFilter外，还包含那些指明了Data类型为确定参数的IntentFilter信息，例如“image/*”和”image/jpeg“等都包含在mTypeToFilter中 |
| mBaseTypeToFilter    | 包含MIME中Base 类型的IntentFilter信息，但不包括Sub type为“*”的IntentFilter |
最后在 `SystemServer` 中调用 `mPackageManagerService.systemReady()` 通知其他服务就绪设置相关信息
### 查询组件

仍然以查询 `activity` 为例，基本上在客户端查询 `activity` 最后都会走到服务端 PMS 类的 `queryIntentActivitiesInternal` 类，源码太多太杂，基本上的流程就是：
1. 如果 `Intent` 指明了 `Component`，则直接查询该 `Component` 对应的 `ActivityInfo`
2. 如果Intent指明了 `Package` 名，则根据 `Package` 名找到该 `Package`，然后再从该 `Package` 包含的 `Activities` 中进行匹配查询。
3. 如果上面条件都不满足，则需要在全系统范围内进行匹配查询

这部分的代码都不复杂，复杂的是各种数据结构，绕来绕去，知道个流程先。