## ByteCode

在字节码中，所有变量和方法都是以符号引用的形式保存在class文件的常量池中。字节码被类加载器加载后，class文件中的常量池会被加载到方法区的运行时常量池，动态链接会将运行时常量池中的符号引用转化为调用方法的直接引用。

```java
package com.demo.asm;

public class Test {
    private int num = 1;
    public static int NUM = 100;

    public int func(int a, int b) {
        return add(a, b);
    }

    public int add(int a, int b) {
        return a + b + num;
    }

    public int sub(int a, int b) {
        return a - b - NUM;
    }
}
```

javap -c Test.class

javap是java Class文件分解器，可以用于反编译，也可以用于查看字节码

-c 输出类中的所有方法以及字节码信息

```java
public com.demo.asm.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: iconst_1
       6: putfield      #2                  // Field num:I
       9: return
```

- 左边的数字表示每个指令的偏移量，保存在PC程序计数器中
- 中间为JVM指令的助记符
- 右边#1、#2 表示操作数











