# ReactNative 源码学习-04-通信

> JNI 作为 C++ 与 Java 的桥梁，JavaScriptCore 作为 C++ 与 JavaScript 的桥梁，而 C++ 最终连接了 Java 与 JavaScript。

前面的文章有提到 `JavaScript Module 注册表` 和 `Native Module 注册表` 两个重要的概念：
1. `JavaScript Module注册表` 
    * `JavaScriptModule`：这是一个接口，JS Module都会继承此接口，它表示在 JS 层会有一个相同名字的 JS 文件，该 JS 文件实现了该接口定义的方法。
    * `JavaScriptModuleRegistry`：JS Module 注册表，内部维护了一个 HashMap<Class<? extends JavaScriptModule>, JavaScriptModule> mModuleInstances，
      JavaScriptModuleRegistry 利用动态代理生成接口 JavaScriptModule 对应的代理类，再通过 C++ 传递到 JS 层，从而调用 JS 层的方法。
2. `Java Module 注册表`
    * `NativeModule`: 是一个接口，实现了该接口则可以被 JS 层调用，我们在为 JS 层提供 Java API 时通常会继承 `BaseJavaModule/ReactContextBaseJavaModule`，这两个类就
      实现了 `NativeModule` 接口。
    * `ModuleHolder`：NativeModule的一个Holder类，可以实现NativeModule的懒加载。
    * `NativeModuleRegistry`: Java Module 注册表，内部持有 `Map<Class<? extends NativeModule>, ModuleHolder> mModules`，`NativeModuleRegistry` 可以遍历
      并返回 Java Module 供调用者使用。

* 使用 JS 调用 Java

   `NativeModuleRegistry` 中的 module 表在 `initializeBridge` 这个前文提到过的 native 方法传递给了 C++ 层，那么之后 JS 想要调用 Java 的方法，就可以通过这个来进行了。
* 使用 Java 调用 JS

    `JavaScriptModuleRegistry` 中从 module 表去找到相应的 module，如果没有就用动态代理的方式使用 `CatalystInstance` 用 JNI 到 C++ 中去寻找 module