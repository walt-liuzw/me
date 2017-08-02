# 探秘JVM内部结构，第二部分

[吴晟](https://github.com/wu-sheng)，
[sky-walking APM](https://github.com/wu-sheng/sky-walking)

[回到第一部分...](JVMInternals.md)

### Class File Structure
一个编译后的文件由以下部分构成：
```class
ClassFile {
    u4			magic;
    u2			minor_version;
    u2			major_version;
    u2			constant_pool_count;
    cp_info		contant_pool[constant_pool_count – 1];
    u2			access_flags;
    u2			this_class;
    u2			super_class;
    u2			interfaces_count;
    u2			interfaces[interfaces_count];
    u2			fields_count;
    field_info		fields[fields_count];
    u2			methods_count;
    method_info		methods[methods_count];
    u2			attributes_count;
    attribute_info	attributes[attributes_count];
}
```

- magic, minor_version, major_version：JDK规范制定的类文件版本，以及对应的编译器JDK版本
- constant_pool：类似符号表，但存储更多的信息。查看“Run Time Constant Pool”章节
- access_flags：class的修饰符列表
- this_class：指向constant_pool中完整类名的索引。如：`org/jamesdbloom/foo/Bar`
- super_class：指向constant_pool中父类完整类名的索引。如：`java/lang/Object`
- interfaces：指向存储在constant_pool中，该类实现的所有接口的完整名称的索引集合。
- fields：指向存储在constant_pool中，该类中所有属性的完成描述的索引集合。
- methods：指向存储在constant_pool中，该类中所有方法签名的索引集合，如果方法不是抽象或本地方法，则方法体也存储在对应的constant_pool中。
- attributes：指向存储在constant_pool中，该类的所有RetentionPolicy.CLASS和RetentionPolicy.RUNTIME级别的标注信息。

可以通过`javap`命令，查看一个编译完成的类的字节码信息。

如果你编译下面这样的一个简单java类：
```java
package org.jvminternals;

public class SimpleClass {

    public void sayHello() {
        System.out.println("Hello");
    }

}
```

然后，你运行`javap -v -p -s -sysinfo -constants classes/org/jvminternals/SimpleClass.class`,你会得到以下输出：
```
public class org.jvminternals.SimpleClass
  SourceFile: "SimpleClass.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #18.#19        //  java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #20            //  "Hello"
   #4 = Methodref          #21.#22        //  java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            //  org/jvminternals/SimpleClass
   #6 = Class              #24            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lorg/jvminternals/SimpleClass;
  #14 = Utf8               sayHello
  #15 = Utf8               SourceFile
  #16 = Utf8               SimpleClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = Class              #25            //  java/lang/System
  #19 = NameAndType        #26:#27        //  out:Ljava/io/PrintStream;
  #20 = Utf8               Hello
  #21 = Class              #28            //  java/io/PrintStream
  #22 = NameAndType        #29:#30        //  println:(Ljava/lang/String;)V
  #23 = Utf8               org/jvminternals/SimpleClass
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
  #30 = Utf8               (Ljava/lang/String;)V
{
  public org.jvminternals.SimpleClass();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
        0: aload_0
        1: invokespecial #1    // Method java/lang/Object."<init>":()V
        4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      5      0    this   Lorg/jvminternals/SimpleClass;

  public void sayHello();
    Signature: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
        0: getstatic      #2    // Field java/lang/System.out:Ljava/io/PrintStream;
        3: ldc            #3    // String "Hello"
        5: invokevirtual  #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
          0      9      0    this   Lorg/jvminternals/SimpleClass;
}
```

这个class文件包含三个主要主要部分，constant pool常量池，构造函数，sayHello方法。

- Constant Pool - 和符号表提供类似信息，详情查看“constant_pool”章节。
- Methods - 每一个方法区包含四个部分：
  - 签名和访问控制符
  - 字节码
  - 行号表 - 用于调试器标识字节码对应的源代码行。例如，sayHello方法的字节码方法0，对应源代码第6行；字节码方法8，对应源代码第7行。
  - 局部变量表 - 列出当前帧的所有局部变量，例子中的方法都只有this这个变量。
  
  下面列出了class文件中所有的字节码操作符
  - **aload_0**：这个操作符是`aload_<n>`操作符组中的一个，这一系列操作都是将一个对象引用压入操作栈中。`<n>`代表当前局部变量数组的索引值，注意，只能取0,1,2,3四个值。对于非对象引用类型，还有一些类似的操作符，`iload_<n>`, `lload_<n>`, `fload_<n>` 和 `dload_<n>`，其中i代表操作int，l代表操作l，f代表操作float，d代表操作double。iload,lload,float,dload操作支持局部变量的索引大于3。这些操作符，都可以通过单次操作，通过指定的局部变量索引，加载变量值。
  - **ldc**：这个操作符将constant pool中的常量值，压入操作栈
  - **getstatic**：将constant pool中，静态列表中的值，压入操作栈
  - **invokespecial**和**invokevirtual**：`invokedynamic`, `invokeinterface`, `invokespecial`, `invokestatic`, `invokevirtual`操作符组中的一个。`invokevirutal`是执行一个Object对象的方法，`invokespecial`是执行实例的初始化函数、私有函数、父类的函数。
  - **return**：`ireturn`, `lreturn`, `freturn`, `dreturn`, `areturn` 和 `return`操作符组中的一个。每一个操作符对应一种返回值类型，i代表int，l代表long，f代表float，d代表double，a代表对象引用，`return`操作则表示返回**void**.

作为一个典型的操作栈，局部变量、操作栈和运行时constant pool的交互操作如下。

构造函数用两个操作，将`this`压到操作栈中，然后父类的构造函数执行，在过程中使用`this`，将其弹出操作栈。
<img src="http://blog.jamesdbloom.com/images_2013_11_17_17_56/bytecode_explanation_SimpleClass.png"/>

**sayHello()**方法的执行更复杂一些，它需要使用constant pool，来解决逻辑引用指针到实际引用指针之间的映射关系（如Dynamic Linking章节所述）。第一个操作符`getstatic`是将System类的out静态方法，压入到操作栈中。接下去，`ldc`操作将"Hello"字符串压入到操作栈中。最后一个操作是`invokevirtual`执行System.out的println方法，将"Hello"作为一个参数弹出操作栈，并为这个方法(println方法)创建一个新的帧。
<img src="http://blog.jamesdbloom.com/images_2013_11_17_17_56/bytecode_explanation_sayHello.png"/>

### Classloader
JVM使用bootstrap classloader来加载启动类。这个类在`public static void main(String[])`方法执行前被链接和初始化。这个方法将按需依次驱动加载、链接、其他相关类和架构的初始化。

**Loading**加载是一个通过类或者接口的名字寻找class文件，并读取到二进制数组中的过程。然后确认加载的内容包含正确的版本号。任何一个类或者接口的父类也会被加载。一旦这个过程完成，类或接口的二进制加载过程就完成了。

**Linking**链接是对类和接口进行验证和类型准备，以及他们的直接父类和父级接口。链接操作有三步构成：verifying 验证，preparing 准备和resolving 解析（可选）。

    - **Verifying**验证时一个确认类和接口的结构正确，符合Java和JVM的语义规范。如，检查一下内容
      1. 正确的符号表
      1. final的方法和类没有被复写和继承
      1. 方法访问控制符符合要求
      1. 方法的参数数量和类型正确
      1. 字节码对栈的操作符合要求
      1. 变量在读取前已经被初始化
      1. 变量值设置正确
  
   


___
[返回吴晟的首页](https://wu-sheng.github.io/me/)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[英文版](http://blog.jamesdbloom.com/JVMInternals.html)