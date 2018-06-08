# Context 的装饰器模式

在 `Context` 的设计上，就是一个典型的装饰器模式。

1. `Context` 是一个 abstract 类，里面定义了如 `getResources()`、`getAssets()`、`startActivity(Intent intent)` 等 abstract 方法
2. `ContextImpl` 继承 `Context` 并实现了它的方法，这相当于是组件具体实现类
3. `Activity` 继承于 `ContextThemeWrapper`，后者又继承于 `ContextWrapper`，这就是装饰器。`ContextWrapper` 的传入一个 `Context` 实例，然后所有方法都是使用这个实例进行操作。从 Android 源码中可以看到，`ActivityThread` 这个类有一个方法 `performLaunchActivity`，其中调用了 `Acitivty#attach` 方法需要传入一个 `Context` 对象，而这个 `Context` 就是由 `ContextImpl.createActivityContext` 创建的一个 `ContextImpl` 实例。然后 `Acitivty#attach` 中调用了继承自 `ContextWrapper` 的方法 `ContextWrapper#attachBaseContext`，将 `ContextImpl` 实例传递给 `ContextWrapper` 使用

