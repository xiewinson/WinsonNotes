# ReactNative 源码学习-03-线程

先说结论，React Native 中有 3 个主要线程:
1. UI 线程，这是 Android 中的主线程，用来绘制 UI 和监听用户操作
2. Native 线程，执行 C++ 代码，负责 Java 与 C++ 的通信
3. JS 线程，解释执行 JS

下面说说这三个线程的创建时机(主线程就不用创建了)。在上篇文章分析启动流程时，有一个重要过程是创建 `ReactContext`，在 `ReactApplicationContext#createReactContext` 中创建了 `CatalystInstance` 这个 Java、C++、JS 通信总管理类
```Java
CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
    .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
    .setJSExecutor(jsExecutor)
    .setRegistry(nativeModuleRegistry)
    .setJSBundleLoader(jsBundleLoader)
    .setNativeModuleCallExceptionHandler(exceptionHandler);
```
这篇文章的主角就是在 `ReactQueueConfigurationSpec.createDefault()` 中诞生的，看它的代码
``` Java
public static ReactQueueConfigurationSpec createDefault() {
    MessageQueueThreadSpec spec = Build.VERSION.SDK_INT < 21 ?
        MessageQueueThreadSpec.newBackgroundThreadSpec("native_modules", LEGACY_STACK_SIZE_BYTES) :
        MessageQueueThreadSpec.newBackgroundThreadSpec("native_modules");
    return builder()
        .setJSQueueThreadSpec(MessageQueueThreadSpec.newBackgroundThreadSpec("js"))
        .setNativeModulesQueueThreadSpec(spec)
        .build();
  }
```
其实这里没啥好说的，ReactQueueConfigurationSpec 就像它的名字 Spec 一样，就是一个说明书，在上面代码里搞了两个线程的说明，类型都为 `NEW_BACKGROUND`，名字分别为 `js` 和 `native_modules`。现在去 `CatalystInstanceImpl` 里面是怎么具体处理的这个说明书。在它的构造器里有这样一行
```Java
mReactQueueConfiguration = ReactQueueConfigurationImpl.create(
        reactQueueConfigurationSpec,
        new NativeExceptionHandler());
```
既然是这样，就该看看这个 `create` 方法了
```Java
 public static ReactQueueConfigurationImpl create(
      ReactQueueConfigurationSpec spec,
      QueueThreadExceptionHandler exceptionHandler) {
    Map<MessageQueueThreadSpec, MessageQueueThreadImpl> specsToThreads = MapBuilder.newHashMap();

    MessageQueueThreadSpec uiThreadSpec = MessageQueueThreadSpec.mainThreadSpec();
    MessageQueueThreadImpl uiThread =
      MessageQueueThreadImpl.create(uiThreadSpec, exceptionHandler);
    specsToThreads.put(uiThreadSpec, uiThread);

    MessageQueueThreadImpl jsThread = specsToThreads.get(spec.getJSQueueThreadSpec());
    if (jsThread == null) {
      jsThread = MessageQueueThreadImpl.create(spec.getJSQueueThreadSpec(), exceptionHandler);
    }

    MessageQueueThreadImpl nativeModulesThread =
        specsToThreads.get(spec.getNativeModulesQueueThreadSpec());
    if (nativeModulesThread == null) {
      nativeModulesThread =
          MessageQueueThreadImpl.create(spec.getNativeModulesQueueThreadSpec(), exceptionHandler);
    }

    return new ReactQueueConfigurationImpl(
      uiThread,
      nativeModulesThread,
      jsThread);
  }
```
从以上代码可以看出，这创建了三个 `MessageQueueThreadImpl`，分别为 uiThread、nativeModulesThread、jsThread。那我们看看 `MessageQueueThreadImpl#create` 怎么做的吧
```Java
 private MessageQueueThreadImpl(
      String name,
      Looper looper,
      QueueThreadExceptionHandler exceptionHandler) {
    mName = name;
    mLooper = looper;
    mHandler = new MessageQueueThreadHandler(looper, exceptionHandler);
    mAssertionErrorMessage = "Expected to be called from the '" + getName() + "' thread!";
  }

public static MessageQueueThreadImpl create(
      MessageQueueThreadSpec spec,
      QueueThreadExceptionHandler exceptionHandler) {
    switch (spec.getThreadType()) {
      case MAIN_UI:
        return createForMainThread(spec.getName(), exceptionHandler);
      case NEW_BACKGROUND:
        return startNewBackgroundThread(spec.getName(), spec.getStackSize(), exceptionHandler);
      default:
        throw new RuntimeException("Unknown thread type: " + spec.getThreadType());
    }
  }

  /**
   * @return a MessageQueueThreadImpl corresponding to Android's main UI thread.
   */
  private static MessageQueueThreadImpl createForMainThread(
      String name,
      QueueThreadExceptionHandler exceptionHandler) {
    Looper mainLooper = Looper.getMainLooper();
    final MessageQueueThreadImpl mqt =
        new MessageQueueThreadImpl(name, mainLooper, exceptionHandler);

    if (UiThreadUtil.isOnUiThread()) {
      Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
    } else {
      UiThreadUtil.runOnUiThread(
          new Runnable() {
            @Override
            public void run() {
              Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
            }
          });
    }
    return mqt;
  }

  /**
   * Creates and starts a new MessageQueueThreadImpl encapsulating a new Thread with a new Looper
   * running on it. Give it a name for easier debugging and optionally a suggested stack size.
   * When this method exits, the new MessageQueueThreadImpl is ready to receive events.
   */
  private static MessageQueueThreadImpl startNewBackgroundThread(
      final String name,
      long stackSize,
      QueueThreadExceptionHandler exceptionHandler) {
    final SimpleSettableFuture<Looper> looperFuture = new SimpleSettableFuture<>();
    Thread bgThread = new Thread(null,
        new Runnable() {
          @Override
          public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
            Looper.prepare();

            looperFuture.set(Looper.myLooper());
            Looper.loop();
          }
        }, "mqt_" + name, stackSize);
    bgThread.start();

    Looper myLooper = looperFuture.getOrThrow();
    return new MessageQueueThreadImpl(name, myLooper, exceptionHandler);
  }
```
这一部分代码可以说是非常清晰明了啦，如果刚才说明里写的 MAIN_UI，那就直接取得 `Main` 线程的 `Looper`，如果说明里是 NEW_BACKGROUND，那就新开一个 Thread，并且使用 `Looper.prepare()` 和 `Looper.loop()` 创建并开启该线程的消息队列循环，然后新建 `MessageQueueThreadImpl` 对象，并在构造器里创建 `Handler`。然后可以看到 `MessageQueueThreadImpl` 的重要方法，添加东西去队列
```Java
public void runOnQueue(Runnable runnable) {
    if (mIsFinished) {
      FLog.w(
          ReactConstants.TAG,
          "Tried to enqueue runnable on already finished thread: '" + getName() +
              "... dropping Runnable.");
    }
    mHandler.post(runnable);
  }

  public <T> Future<T> callOnQueue(final Callable<T> callable) {
    final SimpleSettableFuture<T> future = new SimpleSettableFuture<>();
    runOnQueue(
        new Runnable() {
          @Override
          public void run() {
            try {
              future.set(callable.call());
            } catch (Exception e) {
              future.setException(e);
            }
          }
        });
    return future;
  }
```
恩，没什么魔法，就是把 `Runnable/Callable` 交给 `Handler#post(Runnable)` 来执行。`Handler#post(Runnable)` 只支持 `Runnable`，所以执行 `Callback` 其实也是封在了 `Runnable` 中。回到 `CatalystInstanceImpl` 的构造器中，创建完线程后，有这一个初始化
```Java
initializeBridge(
      new BridgeCallback(this),
      jsExecutor,
      mReactQueueConfiguration.getJSQueueThread(),
      mNativeModulesQueueThread,
      mNativeModuleRegistry.getJavaModules(this),
      mNativeModuleRegistry.getCxxModules());
      
private native void initializeBridge(
      ReactCallback callback,
      JavaScriptExecutor jsExecutor,
      MessageQueueThread jsQueue,
      MessageQueueThread moduleQueue,
      Collection<JavaModuleWrapper> javaModules,
      Collection<ModuleHolder> cxxModules);

```
看得出来，这是个 natvie 方法，把执行 JS 的线程和执行 C++ 的线程传到了 C++ 层去了。