# Android 进程优先级

Android 系统在内存低的情况下会杀死一些进程，系统根据重要程度来决定进程的生死，这个优先级按照分为 4 个等级，由上至下为：

1. __前台进程（Foreground Process）__ 
   * 正在展示的位于顶端的 `Activity`，即 `onResume()` 函数被调用了
   * 正在执行 `BroadcastReceiver.onReceive()` 的 `BroadcastReceiver`
   * 正在执行生命周期方法例如 `Service.onCreate()/onStart()/onDestroy` 的 `Service`
2. __可见进程（Visible Process）__
   * 一个 `Activity` 可见但不是处于前台，即调用了 `onPause()` 方法。例如，一个被 Dialog Style 的 `Activity` 遮挡的 `Activity` 就处于这种状态
   * 前台 `Service`，即调用了 `Service.startForeground()` 的 `Service`
   * 正在承载一个特定功能的系统使用的并且用户察觉得到的服务，例如动态壁纸、输入法等
3. __服务进程（Service Process）__
   * 通过 `startService()` 启动的服务，对于用户不可见但往往在执行一些对用户来说有用的工作，例如上传、下载等。长时间运行的 `Service`，例如超过 30 分钟 或者更久的，系统会降低其重要性以利于杀死，避免内存泄漏或其他问题造成内存的消耗
4. __缓存进程（Cached Process）__
   * 这类进程不是必要的，所以系统内存不足时会优先被杀，非紧急的情况下只会杀从最老最旧的缓存进程杀起。

除了上面所说的情况，进程的优先级会因为其所依赖的其他进程的优先级而提高，例如 A 进程 使用 `Context.BIND_AUTO_CREATE` 这个 flag 来绑定了一个位于 B 进程的 `Service` 或者使用了 B 进程中的 `ContentProvider`，那么 B 进程的优先级至少会和 A 进程处于一级