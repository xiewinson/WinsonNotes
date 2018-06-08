# 在 Android 上使用 Java 8 的新特性
> 在 Android Studio 3.0 以后，可以在 Android 开发中好好使用 Java 8 的部分语言功能了。`javac` 工具将 `.java` 文件编译成 `.class` 文件之后，一个名为 `desugar` 的工具对 `javac` 输出执行字节码转换，再使用 `dex` 工作编译为 `.dex` 文件，从而实现新语言功能

### 使用步骤
* 更新 Android Plugin for Gradle 到 3.0 及以上版本
* 在使用 Java 8 的 module 的 `build.gradle` 中添加配置

    ```java
    android {
        // ...
        compileOptions {
            sourceCompatibility JavaVersion.VERSION_1_8
            targetCompatibility JavaVersion.VERSION_1_8
        }
    }
    ```

* 不要使用已经废弃的 Jack 工具链 
### Java 8 语法的兼容

#### 需要 API24 才能使用的功能（目前不太推荐使用，毕竟版本要到 7.0）:
* java.lang.annotation.Repeatable
* AnnotatedElement.getAnnotationsByType(Class)
* java.util.stream
* java.lang.FunctionalInterface
* java.lang.reflect.Method.isDefault()
* java.util.function
#### 任意版本均能使用的功能
* Lambda 表达式
* 函数引用
* 类型注解
* 默认和静态接口函数
* 重复注解

#### Lambda 表达式

> 实际开发中很多时候有传递一个代码块的需求，例如各种监听器、回调函数的情况，但 Java 中一直以来没有类似 C# 中委托这种东西，所以往往要实现这个需要就只有通过匿名内部类的方式了，难受！！！Lambda 就可以一定程度上弥补这个痛，但是一直以来 Android 对 Java8 的支持不太好，之前有用 [Retrolambda](https://github.com/orfjackal/retrolambda) 这个东西，但是需要配置一系列后来因为懒就直接放弃了。Android Studio 3.0 之后终于可以畅快地使用 Java8 的部分功能了，不过很多功能例如 Stream 必须要 API24 及以上才可以使用，但是 Lambda 标注了适用于任何版本，所以终于可以开开心心用 Lambda 表达式啦

## 基本语法
参数、箭头 -> 和一个表达式
```Java
    (String str, int aInt) -> {
        // ...
    }
```
* 参数：类型可以省略，在只有一个参数的情况下 `()` 也可以省略
* 表达式：表达式只有一行的情况下可以省略 `{}`，这种情况下也不需要使用 `return `，会推导出来

#### 函数式接口
* 只包含一个抽象方法的接口，就可通过 Lambda 表达式来创建该接口的对象，这就叫函数式接口
* Lambda 表达式可以想象成一个函数，而不是对象，并且它可以转化为一个函数式接口，并且转换成函数式接口看起来是在 Java 中使用 Lambda 的唯一作用，因为 Java 中到目前为止并没有函数类型，在 Kotlin 中你可以用 `        var funcName: (String) -> Int = { name ->  Log.d("winson", name)  }
  ` 这种形式声明一个函数类型对象
## 方法引用
* 虽然 Java 现在没有函数类型，但是依然有办法可以做到传递方法，这就是方法引用
* 对象::实例方法
* 类::静态方法
* 类::实例方法，这个相比于前两个要特殊一点，这是表示第一个参数会用来调用方法，第二个参数是这个方法的参数

    ```Java
    // 一般操作
    Arrays.sort(strs, (o1, o2) -> o1.compareToIgnoreCase(o2));
    
    // 方法引用
    Arrays.sort(strs, String::compareToIgnoreCase);
        
    ```

* 构造器::new