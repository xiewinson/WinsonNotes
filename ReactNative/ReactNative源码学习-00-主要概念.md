# ReactNative 源码学习-01-主要概念

- **ReactContext**

继承自 ContextWrapper，这个东西是一个装饰器模式的装饰器角色，装饰器的分析在 [Context的装饰器模式](https://km.sankuai.com/page/34065214)。 Context 的概念不必多讲，是 android 中的上下文，提供一个完整的运行环境，可以取得各种“工具”。 ReactContext 在 Context 的基础上扩充了一些方法，例如 getCurrentActivity 等等。ReactApplicationContext 继承了 ReactContext，使用中也到处是它。它的源码很简单，只是在传给 ReactContext 的 Context 使用了 Application 的 Context

- **Java层 ReactInstanceManager**

ReactNative 应用总的管理类，创建 ReactContext、CatalystInstance 等类，解析 ReactPackage 生成映射表，并且配合 ReactRootView 管理 View 的创建与生命周期等功能。

- **Java/C++层 CatalystInstance**

ReactNative 应用 Java 层、C++ 层、JS 层通信总管理类，总管 Java 层、JS 层核心 Module 映射表与回调，三端通信的入口与桥梁。

- **C++层 NativeToJsBridge/JsToNativeBridge**

NativeToJsBridge 是 Java 调用 JS 的桥梁，用来调用 JS Module，回调 Java。JsToNativeBridge 是 JS 调用 Java 的桥梁，用来调用 Java Module。

- **C++层 JSCExecutor**

掌管 Webkit 的 JavaScriptCore，JS 与 C++ 的转换桥接都在这里中转处理。

- **JS层 MessageQueue**

用来处理 JS 的调用队列、调用 Java 或者 JS Module 的方法、处理回调、管理 JS Module 等。

- **JavaScriptModule/NativeModule**

两者均由 CatalystInstance 管理。 JavaScriptModule：JS 暴露给 Java 调用的 API 集合，例如：AppRegistry、DeviceEventEmitter 等。业务放可以通过继承 JavaScriptModule 接口类似自定义接口模块，声明 与 JS 相对应的方法即可。NativeModule/UIManagerModule：NativeModule 是 Java 暴露给 JS 调用的 API 集合，例如：ToastModule、DialogModule等，UIManagerModule 也是供 JS 调用的 API 集 合，它用来创建 View。业务放可以通过实现 NativeModule 来自定义模块，通过 getName() 将模块名暴露给JS层，通过 @ReactMethod 注解将 API 暴露给 JS 层。

- **JavascriptModuleRegistry/NativeModuleRegistry**

JavascriptModuleRegistry 是 JS Module 映射表，NativeModuleRegistry 是 Java Module 映射表