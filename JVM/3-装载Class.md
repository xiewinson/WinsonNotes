# 3-装载 Class

### 类装载条件 — 主动使用

* 创建类的实例，使用 new，或者反射、克隆、反序列化
* 调用类的静态方法，即使用 invokestatic
* 使用类/接口的静态字段 (final 常量除外)，如  getstatic 或 putstatic
* 使用 java.lang.reflect 中的方法反射类的方法
* 初始化子类时会先初始化父类
* 含入口 main() 方法的类

### 被动引用的情况下不会装载

* 引用字段时，只有直接定义该字段的类才会初始化:

  ```java
  public class Parent {
    static {
      System.out.println("Parent init;");
    }
    public static int v = 100;
  }
  
  public class Child extends Parent {
    static {
      System.out.println("Child init;");
    }
  }
  
  public class Test {
    public static void main(String[] args){
      System.out.println(Child.v);
    }
  }
  
  // 执行结果：
  // Parent init
  // 100 
  // 这种情况下虽然 Child 没有初始化，但已被系统加载，只是没有进入初始化流程
  ```

* 数组定义引用类不会触发初始化

  ```java
  public class A {
    static {
      System.out.println("Child init;");
    }
  }
  
  public class Test {
    public static void main(String[] args){
      A[] array = new A[3];
    }
  }
  ```

  

* 使用某个类中定义的常量不会引起装载

  ```java
  public class A {
    public static final String STR = "HAHAHAHHAHA";
    static {
      System.out.println("Child init;");
    }
  }
  
  public class Test {
    public static void main(String[] args){
      System.out.println(A.STR);
    }
  }
  
  // 执行结果：
  // HAHAHAHHAHA
  ```

### 装载过程

* 加载

  * 通过类全名获取类的二进制数据流

  * 解析该二进制数据流为方法区中的数据结构

  * 在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口，Class 虽是对象但未存放在堆中而是在方法区
* 链接

  * 验证

    * 格式检查：魔数检查、版本检查、长度检查

    * 语义检查：是否继承 final 类、是否有父类、抽象方法是否实现

    * 字节码验证：利用 StackMapTable 检查，跳转指令是否指向正确位置、操作数类型是否合理

    * 符号引用验证：符号引用的直接引用是否存在
	  
  * 准备：创建类/接口的静态字段 (注意：没有实例变量) 并设置初始值。被 static final 修饰的基本数据或 String 常量会含有 ConstantValue 属性，直接存放于常量值，准备阶段会被赋值
  
  * 解析：将类、接口、字段、方法的符号引用转化为直接引用
* 初始化：如果有静态变量或有静态代码块，将会生成 \<clinit\> 并且在这个阶段进行执行和赋值操作，在执行子类的 \<clinit\> 前会确保父类的 \<clinit\> 已经执行过

### 类加载器的双亲委派模型

1. 启动类加载器 (Bootstrap ClassLoader)：负责加载存放在 \<JAVA_HOME\>/lib 目录中的的类
2. 扩展类加载器 (Extension ClassLoader)：负责加载存放在 \<JAVA_HOME\>/lib/ext 目录中的的类
3. 应用类加载器 (Application ClassLoader)：负责加载用户类路径上所指定的类库，一般是默认的类加载器
4. 自定义类加载器

如果一个类加载器收到了类加载的请求，首先不会自己去尝试加载这个类，而是把请求委派给父类加载器去完成，并且每层都如此，所以所有的加载请求最终都会传递到顶层的启动类加载器，只有父加载器无法完成加载请求时，子加载器才尝试自己加载。

* 优点：由不同类加载器加载的类属于不同的类型，不同互相转化和兼容，双亲委派模型使得 Java 类随着它的类加载器天生有了优先级的层次概念，一些通用的类比如 Object 由于被上层类加载器所加载，使得所有地方所用的 Object 都是同一份

* 弊端：顶层类加载器无法访问到底层类加载器的类

### 类加载器主要方法

- public Class<?> loadClass(String name) throws ClassNotFoundException

  给定一个类名加载一个类并返回代表这个类的 Class 实例，首先会 findLoadedClass 查看是否加载了这个类，如果没有会去请求双亲进行加载，如果双亲是空，则使用启动类加载器去加载，如果为空则调用自己的 findClass 去加载类

- protected final Class<?> defineClass(byte[] b, int off, int len)

  根据给定的字节码流定义一个类

- protected Class<?> findClass(String name) throws ClassNotFoundException

  会在 loadClass() 时被调用，用于双亲类加载器都找不到的时候进行加载

- protected final Class<?> findLoadedClass(String name)

  寻找已加载的类