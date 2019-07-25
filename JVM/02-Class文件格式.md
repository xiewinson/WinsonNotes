# 02-Class 文件格式

### 结构

```java
ClassFile {
  u4 magic;
  u2 minor_version;
  u2 major_version;
  u2 constant_pool_count;
  cp_info constant_pool[constant_pool_count - 1];
  u2 access_flags;
  u2 this_class;
  u2 super_class;
  u2 interfaces_count;
  u2 interfaces[interfaces_count];
  u2 fields_count;
  field_info fields[fields_count];
  u2 method_count;
  method_info methods[method_count];
  u2 attributes_count;
  attribute_info attributes[attributes_count];
}
```

### 描述

* magic 魔数，用来确定这个文件是否为一个能被虚拟机接受的 class 文件，值为 0xCAFEBABE  (咖啡宝贝)

* minor_version 和 major_version，Class 文件的次版本号和主版本号

* constant_pool_count 常量池数量，为实际数量 +1，常量池的索引也是从 1 开始

* constant pool 常量池，最底层的常量池数据类型为 UTF8 字符串, Integer, Float, Long, Double
  * CONSTANT_Utf8_info 存储 UTF8 字符串
  * CONSTANT_Integer_info int 字面量
  * CONSTANT_Float_info float 字面量
  * CONSTANT_Double_info double 字面量
  * CONSTANT_Long_info long 字面量
  * CONSTANT_String_info 实际上存储的数据是指向的 UTF8 常量中的数据
  * CONSTANT_Class_info 类或接口的符号引用
  * CONSTANT_Fieldref_info 字段的符号引用
  * CONSTANT_Methodref_info 类中方法的符号引用
  * CONSTANT_InterfaceMethodref_info 接口中方法的符号引用
  * CONSTANT_NameAndType_info 字段或方法的部分符号引用，一个字段和方法如果有被使用会出现
  * CONSTANT_MethodHandle_info 表示方法句柄
  * CONSTANT_MethodType_info 表示方法类型
  * CONSTANT_InvokeDynamic_info 表示一个动态方法的调用点
  
* access Flag 类或接口的访问标记
  * ACC_PUBLIC 标识 public
  * ACC_FINAL 标识 final，只有类可以设置
  * ACC_SUPER 标识运行使用 invokespecial 字节码指令的新语意，JDK 1.0.2 后编译的类这个标志都为真
  * ACC_INTERFACE 标识接口
  * ACC_ABSTRACT 标识为 abstract 类型，接口和抽象类为真
  * ACC_SYNTHETIC 标识这个类并非由用户代码产生
  * ACC_ANNOTATION 标识注解
  * ACC_ENUM 标识枚举
  
* this_class 表示当前类所定义的类或接口，指向 CONSTANT_Class_info 中的值

* super_class 表示当前类或接口的父类，指向 CONSTANT_Class_info 中的值，若是 Object 类，则该处为 0；若是接口，该处指向的是 Object

* interfaces_count 接口数量

* interfaces 每个值必须是 CONSTANT_Class_info 中的索引值，可以 0，顺序和源码中声明的顺序一致

* fields_count 当前 class 文件 fields 表的成员个数

* fields 字段表，每个成员都是一个 field_info 结构，用于表示类/接口某个字段的完整描述，描述当前类/接口的字段，不包括从父类/接口继承而来的
  * access_flags 字段的访问权限合基本属性
    * ACC_PUBLIC 声明 public
    * ACC_PRIVATE 声明 private
    * ACC_PROTECTED 声明 protected
    * ACC_STATIC 声明 static
    * ACC_FINAL 声明 final 
    * ACC_VOLATILE 声明 volatile，被标识的字段不能缓存
    * ACC_TRANSIENT 声明 transient，被标识的字段不会为持久化对象管理器写入或读取
    * ACC_SYNTHETIC 表示字段由编译器产生，没有写在源代码中
    * ACC_ENUM 表示字段为某个枚举类型的成员
    
  * name_index 字段名，非限定名，指向 CONSTANT_Utf8_info
  
  * descriptor_index 字段描述符，指向 CONSTANT_Utf8_info
  
    * B 基本类型 byte  
    * C 基本类型 char
    * D 基本类型 double
    * F 基本类型 float
    * I 基本类型 int
    * J 基本类型 long
    * S 基本类型 short
	  * Z 基本类型 boolean 
  	* V 表示 void
    * L 表示 reference，对象，如 Ljava/lang/Object
    * [ 表示 reference，一个一维数组
  
  * attribute_count 和 attributes 相关属性
  
* method_count 当前 class 文件 method 表的成员个数

* methods 方法表，每个成员都是一个 method_info 结构，用于表示类/接口某个方法的完整描述，可以表示所有方法，包括实例方法、类方法、实例初始化方法、类/接口初始化方法，不包括父类/接口继承而来的

  * access_flags
  * ACC_PUBLIC 声明 public
    * ACC_PRIVATE 声明 private
    * ACC_PROTECTED 声明 protected
    * ACC_STATIC 声明 static
    * ACC_FINAL 声明 final 
    * ACC_SYNCHRONIZED 声明 synchronized
    * ACC_BRIDGE 声明为 bridge 方法，编译器产生
    * ACC_ABSTRACT 声明 abstract
    * ACC_STRICT 声明 strictfp，使用 FP-strict 浮点模式
    * ACC_SYNTHETIC 表示字段由编译器产生，没有写在源代码中
    * ACC_NATIVE 声明为 native 方法
    * ACC_VARAGS 声明方法带有变长参数
  * name_index 字段名，非限定名，指向 CONSTANT_Utf8_info，有一些特殊方法，例如 \<init\> 和 \<clinit\>
  * descriptor_index 方法描述符，指向 CONSTANT_Utf8_info。按照先参数列表、后返回值的顺序描述，参数列表按顺序放入 () 之内，返回值跟在最后面。例如 void inc() 的描述符为 ()void，int(String str) 为 (Ljava/lang/String)I

* attributes_count 当前 class 文件属性表的成员个数

* attributes 属性表，每个成员都是一个 attribute_info 结构

  * Code 用于方法表，编译后的字节码指令，native 或 abstract 修饰的方法不能有此属性，其他情况下 method_info 中只允许有一个 Code 属性。

    ```java
    Code_attributes {
      u2 attribute_name_index;
      u4 attrbute_length;
      u2 max_stack;
      u2 max_locals;
      u4 code_length;
      u1 code[code_length];
      u2 exception_table_length;
      {
        u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
      } exception_table[exception_table_length];
      u2 attributes_count;
      attribute_info attributes[attributes_count]
    }
    ```

    * attribute_name_index 指向 CONSTANT_Utf8_info，恒为 "Code"
    * attribute_length 长度，不包含前 6 个字节
    * max_stack 操作数栈存在的最大深度，long 和 double 会占两个单位的栈深度，其他为一个
    * max_locals 局部变量表所需的存储空间，单位为 Slot，小或等于 32 位的数据类型为一个 Slot，long 和 double 是两个 Slot。
    * code_length code[] 的字节数
    * 实际代码字节码内容
    * exception_table_length 表示 exception_table 表的成员个数
    * exception_table 异常表，从 start_pc 到 end_pc 偏移量为止的这段代码中，如果如果了 catch_type 指定的异常，代码就跳转到 handler_pc 的位置执行。若 catch_type 为 0，那么在所有异常抛出时都调用这个异常处理器，可用于实现 finally

  * ConstantValue，用于字段表，final 关键字定义的常量值

  * Deprecated，用于类、方法表、字段表，声明 deprecated

  * Exceptions，用于方法表，方法抛出的异常

  * EnclosingMethod，用于类文件，当一个类为局部类或匿名类才有这个属性

  * InnerClasses，用于类文件，内部类列表

  * LineNumberTable，用于 Code 属性，Java 源码的行号与字节码指令的对应关系

  * LocalVariableTable，用于 Code 属性，方法的局部变量和参数的描述

  * LocalVariableTypeTable，用于类，使用特征签名代替描述符，为了引入泛型语法后能描述泛型参数化类型

  * StackMapTable，用于 Code 属性，用于新的类型检查验证器 (Type Checker) 检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配

  * Signature，用于类、方法表、字段表，用于支持泛型下的方法签名，由于 Java 的泛型采用擦除法实现，为避免类型擦除后签名混乱，需要这个属性记录泛型中的相关信息

  * SourceFile，用于类文件，记录源文件名称

  * SourceDebugExtension，用于类文件，存储额外调试信息

  * Synthetic，用于类、方法表，字段表，表示方法或字段为编译器自动生成

  * RuntimeVisibleAnnotations，用于类、方法表、字段表，支持动态注解，指明哪些注解是运行时可见

  * RuntimeInInVisibleAnnotations，用于类、方法表、字段表，支持动态注解，指明哪些注解是运行时不可见

  * RuntimeVisibleParameter，用于方法表，类似 RuntimeVisibleAnnotations

  * RuntimeInVisibleParameter，用于方法表，类似 RuntimeInInVisibleAnnotations

  * AnnotationDefault，用于方法表，记录注解类元素的默认值

  * BootstrapMethods，用于类文件，保存 invokedynamic 指令引用的引导方法限定符


