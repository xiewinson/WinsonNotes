# ReactNative 源码学习-02-启动流程

首先总结一下自己手动新建一个支持 React Native 的 Android 项目的过程，引入 Library 这一部分就略过了。
1. `Application#onCreate` 中需要调用 `SoLoader.init(this, false)`，加载 C++ 库
2. `Application` 需要实现 `ReactApplication` 接口，必须这样做，因为源码里有很多地方 `getApplication()` 然后强转为 `ReactApplication`。然后实现方法 `getReactNativeHost`，提供一个 `ReactNativeHost` 对象
3. `ReactNativeHost` 的工作就是创建一个 `ReactInstanceManager`，在这里能对其进行各种的配置，例如是否支持 dev 模式、加入自定义 package 等等
4. 新建 `Activity` 继承与 `ReactActivity`，复写 `getMainComponentName` 方法返回正确的 ComponentName，好了，之后就靠这个 `Activity` 来展示了

下面再进入源码的分析：

`ReactActivity` 里有一个名为 `ReactActivityDelegate` 的类，名字说明了一切，这个委托类里实现了 `ReactActivity` 要做的操作。
```Java
public class ReactActivityDelegate {

  protected void onCreate(Bundle savedInstanceState) {
    boolean needsOverlayPermission = false;
    //开发模式判断以及权限检查
    if (getReactNativeHost().getUseDeveloperSupport() && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
      // Get permission to show redbox in dev builds.
      if (!Settings.canDrawOverlays(getContext())) {
        needsOverlayPermission = true;
        Intent serviceIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:" + getContext().getPackageName()));
        FLog.w(ReactConstants.TAG, REDBOX_PERMISSION_MESSAGE);
        Toast.makeText(getContext(), REDBOX_PERMISSION_MESSAGE, Toast.LENGTH_LONG).show();
        ((Activity) getContext()).startActivityForResult(serviceIntent, REQUEST_OVERLAY_PERMISSION_CODE);
      }
    }

    //mMainComponentName就是上面ReactActivity.getMainComponentName()返回的组件名
    if (mMainComponentName != null && !needsOverlayPermission) {
        //载入app页面
      loadApp(mMainComponentName);
    }
    mDoubleTapReloadRecognizer = new DoubleTapReloadRecognizer();
  }

  protected void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    //创建ReactRootView作为根视图,它本质上是一个FrameLayout
    mReactRootView = createRootView();
    //启动RN应用
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      appKey,
      getLaunchOptions());
    //Activity的setContentView()方法  
    getPlainActivity().setContentView(mReactRootView);
  }
}
```
从上面的代码可以看出，`ReactActivityDelegate#onCreate` 主要做了三件事:
1.  创建一个本质上是 `FrameLayout` 的 `ReactRootView` 作为容器
2.  使用 `ReactRootView#startReactApplication` 执行应用启动流程
3.  将 `ReactRootView` 交给 `Activity` 设置为 `ContentView`

所以接下来应该看看 `ReactRootView#startReactApplication` 里面是怎么做的
```Java
public class ReactRootView extends SizeMonitoringFrameLayout implements RootView {

  /**
   * Schedule rendering of the react component rendered by the JS application from the given JS
   * module (@{param moduleName}) using provided {@param reactInstanceManager} to attach to the
   * JS context of that manager. Extra parameter {@param launchOptions} can be used to pass initial
   * properties for the react component.
   */
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
    UiThreadUtil.assertOnUiThread();

    // TODO(6788889): Use POJO instead of bundle here, apparently we can't just use WritableMap
    // here as it may be deallocated in native after passing via JNI bridge, but we want to reuse
    // it in the case of re-creating the catalyst instance
    Assertions.assertCondition(
        mReactInstanceManager == null,
        "This root view has already been attached to a catalyst instance manager");

    mReactInstanceManager = reactInstanceManager;
    mJSModuleName = moduleName;
    mLaunchOptions = launchOptions;

    //创建RN应用上下文
    if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
      mReactInstanceManager.createReactContextInBackground();
    }

    // We need to wait for the initial onMeasure, if this view has not yet been measured, we set which
    // will make this view startReactApplication itself to instance manager once onMeasure is called.ReactInstanceManager
    if (mWasMeasured) {
      attachToReactInstanceManager();
    }
  }

}
```
从这里面可以看到这里面调用了 `ReactInstanceManager#createReactContextInBackground`，从名字看起来是要在后台去创建 `ReactContext`，这个方法里面去调用了 `recreateReactContextInBackgroundInner()`
```Java
private void recreateReactContextInBackgroundInner() {
    //
    if (mUseDeveloperSupport
        && mJSMainModulePath != null
        && !Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
      final DeveloperSettings devSettings = mDevSupportManager.getDevSettings();

      // 如果 JS debugging 开启，从 dev server 拉取 bundle
      if (mDevSupportManager.hasUpToDateJSBundleInCache() &&
          !devSettings.isRemoteJSDebugEnabled()) {
        // If there is a up-to-date bundle downloaded from server,
        // with remote JS debugging disabled, always use that.
        onJSBundleLoadedFromServer();
      } else if (mBundleLoader == null) {
        mDevSupportManager.handleReloadJS();
      } else {
        mDevSupportManager.isPackagerRunning(
            new PackagerStatusCallback() {
              @Override
              public void onPackagerStatusFetched(final boolean packagerIsRunning) {
                UiThreadUtil.runOnUiThread(
                    new Runnable() {
                      @Override
                      public void run() {
                        if (packagerIsRunning) {
                          mDevSupportManager.handleReloadJS();
                        } else {
                          // If dev server is down, disable the remote JS debugging.
                          devSettings.setRemoteJSDebugEnabled(false);
                          recreateReactContextInBackgroundFromBundleLoader();
                        }
                      }
                    });
              }
            });
      }
      return;
    }

    recreateReactContextInBackgroundFromBundleLoader();
  }
  
private void recreateReactContextInBackgroundFromBundleLoader() {
    Log.d(
      ReactConstants.TAG,
      "ReactInstanceManager.recreateReactContextInBackgroundFromBundleLoader()");
    PrinterHolder.getPrinter()
        .logMessage(ReactDebugOverlayTags.RN_CORE, "RNCore: load from BundleLoader");
    recreateReactContextInBackground(mJavaScriptExecutorFactory, mBundleLoader);
  }
  
  private void recreateReactContextInBackground(
    JavaScriptExecutorFactory jsExecutorFactory,
    JSBundleLoader jsBundleLoader) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.recreateReactContextInBackground()");
    UiThreadUtil.assertOnUiThread();

    final ReactContextInitParams initParams = new ReactContextInitParams(
      jsExecutorFactory,
      jsBundleLoader);
    if (mCreateReactContextThread == null) {
      runCreateReactContextOnNewThread(initParams);
    } else {
      mPendingReactContextInitParams = initParams;
    }
  }
```
从上述的源码可以看到，这一连串的操作下来有点杂，因为如果是 dev 的情况下本地又没缓存，还要先去 dev server 下载 JS bundle 包，不过不管怎么样都会走到 `recreateReactContextInBackground` 这一步。关于这一步里面的操作，我看原作者的分析文章中，这一段是使用的 `ReactContextInitAsyncTask` 来操作的，但我基于 `0.55.3` 版本的源码，这一段是直接使用一个 `Thread` 进行操作的。贴出 `0.55.3` 版本的代码
```Java
 private void runCreateReactContextOnNewThread(final ReactContextInitParams initParams) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.runCreateReactContextOnNewThread()");
    UiThreadUtil.assertOnUiThread();
    synchronized (mReactContextLock) {
      if (mCurrentReactContext != null) {
        tearDownReactContext(mCurrentReactContext);
        mCurrentReactContext = null;
      }
    }

    mCreateReactContextThread =
        new Thread(
            new Runnable() {
              @Override
              public void run() {
                ReactMarker.logMarker(REACT_CONTEXT_THREAD_END);
                synchronized (ReactInstanceManager.this.mHasStartedDestroying) {
                  while (ReactInstanceManager.this.mHasStartedDestroying) {
                    try {
                      ReactInstanceManager.this.mHasStartedDestroying.wait();
                    } catch (InterruptedException e) {
                      continue;
                    }
                  }
                }
                // As destroy() may have run and set this to false, ensure that it is true before we create
                mHasStartedCreatingInitialContext = true;

                try {
                  Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
                  final ReactApplicationContext reactApplicationContext =
                      createReactContext(
                          initParams.getJsExecutorFactory().create(),
                          initParams.getJsBundleLoader());

                  mCreateReactContextThread = null;
                  ReactMarker.logMarker(PRE_SETUP_REACT_CONTEXT_START);
                  final Runnable maybeRecreateReactContextRunnable =
                      new Runnable() {
                        @Override
                        public void run() {
                          if (mPendingReactContextInitParams != null) {
                            runCreateReactContextOnNewThread(mPendingReactContextInitParams);
                            mPendingReactContextInitParams = null;
                          }
                        }
                      };
                  Runnable setupReactContextRunnable =
                      new Runnable() {
                        @Override
                        public void run() {
                          try {
                            setupReactContext(reactApplicationContext);
                          } catch (Exception e) {
                            mDevSupportManager.handleException(e);
                          }
                        }
                      };

                  reactApplicationContext.runOnNativeModulesQueueThread(setupReactContextRunnable);
                  UiThreadUtil.runOnUiThread(maybeRecreateReactContextRunnable);
                } catch (Exception e) {
                  mDevSupportManager.handleException(e);
                }
              }
            });
    ReactMarker.logMarker(REACT_CONTEXT_THREAD_START);
    mCreateReactContextThread.start();
  }
```
这里前面有一段等待 `mHasStartedDestroying` 为 `false` 之后才进行操作的操作，那么 `mHasStartedDestroying` 啥时候会改变值呢？这个是在调用 `destroy()` 方法的时候。
```Java
 public void destroy() {
    UiThreadUtil.assertOnUiThread();
    PrinterHolder.getPrinter().logMessage(ReactDebugOverlayTags.RN_CORE, "RNCore: Destroy");

    mHasStartedDestroying = true;

    if (mUseDeveloperSupport) {
      mDevSupportManager.setDevSupportEnabled(false);
      mDevSupportManager.stopInspector();
    }

    moveToBeforeCreateLifecycleState();

    if (mCreateReactContextThread != null) {
      mCreateReactContextThread = null;
    }

    mMemoryPressureRouter.destroy(mApplicationContext);

    synchronized (mReactContextLock) {
      if (mCurrentReactContext != null) {
        mCurrentReactContext.destroy();
        mCurrentReactContext = null;
      }
    }
    mHasStartedCreatingInitialContext = false;
    mCurrentActivity = null;

    ResourceDrawableIdHelper.getInstance().clear();
    mHasStartedDestroying = false;
    synchronized (mHasStartedDestroying) {
      mHasStartedDestroying.notifyAll();
    }
  }
```
就是在 `destroy()` 这个步骤的时候，开始会将 `mHasStartedDestroying` 置为 `true`，表示正在进行摧毁，摧毁完毕再置为 `false`。好的说回刚才，看得到这里调用了 `ReactInstanceManager#createReactContext` 去创建 `ReactContenxt`，可以看到，这里使用了 `initParam` 中的两个参数，代码很简单不贴这块了。第一个参数是 `JsExecutorFactory`，在前面提到的用来构建 `ReactInstanceManager` 的 `ReactNativeHost` 中可以指定这个东西，若不指定，默认使用的是 `JSCJavaScriptExecutorFactory`；第二个参数是 `JSBundleLoader`，这个东西也可以在 `ReactNativeHost` 中配置，主要是用来指定从哪里加载 JS Bundle，例如指定目录的文件、`Assets` 中等进行加载。现在看 `ReactInstanceManager#createReactContext` 里的代码
```Java
private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.createReactContext()");
    ReactMarker.logMarker(CREATE_REACT_CONTEXT_START);
    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);

    if (mUseDeveloperSupport) {
      reactContext.setNativeModuleCallExceptionHandler(mDevSupportManager);
    }

    NativeModuleRegistry nativeModuleRegistry = processPackages(reactContext, mPackages, false);

    NativeModuleCallExceptionHandler exceptionHandler = mNativeModuleCallExceptionHandler != null
      ? mNativeModuleCallExceptionHandler
      : mDevSupportManager;
    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
      .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
      .setJSExecutor(jsExecutor)
      .setRegistry(nativeModuleRegistry)
      .setJSBundleLoader(jsBundleLoader)
      .setNativeModuleCallExceptionHandler(exceptionHandler);

    ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_START);
    // CREATE_CATALYST_INSTANCE_END is in JSCExecutor.cpp
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createCatalystInstance");
    final CatalystInstance catalystInstance;
    try {
      catalystInstance = catalystInstanceBuilder.build();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_END);
    }
    if (mJSIModulesProvider != null) {
      catalystInstance.addJSIModules(mJSIModulesProvider
        .getJSIModules(reactContext, catalystInstance.getJavaScriptContextHolder()));
    }

    if (mBridgeIdleDebugListener != null) {
      catalystInstance.addBridgeIdleDebugListener(mBridgeIdleDebugListener);
    }
    if (Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
      catalystInstance.setGlobalVariable("__RCTProfileIsProfiling", "true");
    }
    ReactMarker.logMarker(ReactMarkerConstants.PRE_RUN_JS_BUNDLE_START);
    catalystInstance.runJSBundle();
    reactContext.initializeWithInstance(catalystInstance);

    return reactContext;
  }
```
可以看到，这里创建了 `CatalystInstance`，上篇文章提过
这是 Java 层、C++ 层、JS 层通信总管理类，总管 Java 层、JS 层核心 Module 映射表与回调，三端通信的入口与桥梁。然后 `CatalystInstance#runJSBundle` 里调用 `JSBundleLoader` 加载 JS，这会走到 JNI 方法中去做，具体实现本次未分析。最后代码中使用 `ReactContext#initializeWithInstance(CatalystInstance catalystInstance)` 关联了 `ReactContext` 和 `CatalystInstance`。这时候回到 `runCreateReactContextOnNewThread` 方法，在进行了创建 `ReactContext` 之后，在 `NativeModulesQueueThread` 这个线程上执行了 `setupReactContext(reactApplicationContext)`，代码如下
```Java
 private void setupReactContext(final ReactApplicationContext reactContext) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.setupReactContext()");
    ReactMarker.logMarker(PRE_SETUP_REACT_CONTEXT_END);
    ReactMarker.logMarker(SETUP_REACT_CONTEXT_START);
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "setupReactContext");
    synchronized (mReactContextLock) {
      mCurrentReactContext = Assertions.assertNotNull(reactContext);
    }
    CatalystInstance catalystInstance =
      Assertions.assertNotNull(reactContext.getCatalystInstance());

    // 进行初始化 NativeModules
    catalystInstance.initialize();
    mDevSupportManager.onNewReactContextCreated(reactContext);
    mMemoryPressureRouter.addMemoryPressureListener(catalystInstance);
    // 复位生命周期
    moveReactContextToCurrentLifecycleState();

    ReactMarker.logMarker(ATTACH_MEASURED_ROOT_VIEWS_START);
    synchronized (mAttachedRootViews) {
      for (ReactRootView rootView : mAttachedRootViews) {
        attachRootViewToInstance(rootView, catalystInstance);
      }
    }
    ReactMarker.logMarker(ATTACH_MEASURED_ROOT_VIEWS_END);

    ReactInstanceEventListener[] listeners =
      new ReactInstanceEventListener[mReactInstanceEventListeners.size()];
    final ReactInstanceEventListener[] finalListeners =
        mReactInstanceEventListeners.toArray(listeners);

    UiThreadUtil.runOnUiThread(
        new Runnable() {
          @Override
          public void run() {
            for (ReactInstanceEventListener listener : finalListeners) {
              listener.onReactContextInitialized(reactContext);
            }
          }
        });
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    ReactMarker.logMarker(SETUP_REACT_CONTEXT_END);
    reactContext.runOnJSQueueThread(
        new Runnable() {
          @Override
          public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          }
        });
    reactContext.runOnNativeModulesQueueThread(
        new Runnable() {
          @Override
          public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          }
        });
  }

  private void attachRootViewToInstance(
      final ReactRootView rootView,
      CatalystInstance catalystInstance) {
    Log.d(ReactConstants.TAG, "ReactInstanceManager.attachRootViewToInstance()");
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachRootViewToInstance");
    UIManager uiManagerModule = rootView.isFabric() ? catalystInstance.getJSIModule(UIManager.class) : catalystInstance.getNativeModule(UIManagerModule.class);
    final int rootTag = uiManagerModule.addRootView(rootView);
    rootView.setRootViewTag(rootTag);
    rootView.invokeJSEntryPoint();
    Systrace.beginAsyncSection(
      TRACE_TAG_REACT_JAVA_BRIDGE,
      "pre_rootView.onAttachedToReactInstance",
      rootTag);
    UiThreadUtil.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        Systrace.endAsyncSection(
          TRACE_TAG_REACT_JAVA_BRIDGE,
          "pre_rootView.onAttachedToReactInstance",
          rootTag);
        rootView.onAttachedToReactInstance();
      }
    });
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }
```
从上面的代码大致能看出来，这个步骤里面，最终会将 ` `UIManager`.addRootView(rootView)`。按照原文摘抄的 渲染流程：
1. React Native 将代码由 JSX 转化为 JS 组件，启动过程中利用 instantiateReactComponent 将 ReactElement 转化为复合组件 ReactCompositeComponent 与元组件 ReactNativeBaseComponent，利用  ReactReconciler 对他们进行渲染。
2. UIManager.js 利用 C++  层的 Instance.cpp 将 UI 信息传递给 UIManagerModule.java，并利用 UIManagerModule.java 构建 UI。
3. UIManagerModule.java 接收到 UI 信息后，将 UI 的操作封装成对应的 UIOperation，放在队列中等待执行。各种 UI 的操作，例如创建、销毁、更新等便在队列里完成，UI 最终得以渲染在屏幕上。

`UIManagerModule` 的方法中被 `@ReactMethod` 注解就表示可以在 JS 中被调用，从源码中可以看到，它委托了 `UIImplementation` 来具体实现功能。下面看看 `UIImplementation#createView` 的代码：
```Java
public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    ReactShadowNode cssNode = createShadowNode(className);
    ReactShadowNode rootNode = mShadowNodeRegistry.getNode(rootViewTag);
    cssNode.setReactTag(tag);
    cssNode.setViewClassName(className);
    cssNode.setRootNode(rootNode);
    cssNode.setThemedContext(rootNode.getThemedContext());

    mShadowNodeRegistry.addNode(cssNode);

    ReactStylesDiffMap styles = null;
    if (props != null) {
      styles = new ReactStylesDiffMap(props);
      cssNode.updateProperties(styles);
    }

    handleCreateView(cssNode, rootViewTag, styles);
  }
```
我们从代码中可以追踪到
`ReactShadowNode` 看起来是用来描述 DOM 树的节点，将 JS 传来的 UI 信息包装起来，然后调用方法 `handleCreateView`，这到了 `UIViewOperationQueue`  中，不但是 `createView`，其他的 `removeView`、`updateView` 等等操作都会走到这一步，这差不多就能猜到：在 `UIViewOperationQueue` 有个类似 Handler 机制的队列，不断从中取得 UIOperation 出来进行执行，而也有 UIOperation 陆续插入到队列中。

回到 `UIImplementation#createView` 中，`mViewManagers#get(className)` 是通过 `className` 取得之前在 `ReactPackage` 中注册过的 `ViewManager`。`ViewManager` 的概念就很熟悉了，在自定义封装 Android 控件的时候就需要用到它，继承 `ViewManager` 的类必须实现 `createViewInstance` 方法，在里面就会去创建一个真实的 `View` 了。