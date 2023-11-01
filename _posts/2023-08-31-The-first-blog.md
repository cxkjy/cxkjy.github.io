---
layout: post
title: ASM内容（实践主要）
categories: [blog ]
tags: [Java,]
description: "测试"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 
---







## 前言

```java
就想学个RASP，前置内容太多了呜呜呜，本来想复现题发现也需要RASP躲不过躲不过，先学习ASM前置知识，本篇文章先实践代码不然直接看文章根本看不懂，下一篇文章详细介绍。
```

![image-20231101192710222](..\img\final\image-20231101192710222.png)

## 首先需要010配置一个新的模板

[sweetscape.com/010editor/repository/files/DEX.bt](https://www.sweetscape.com/010editor/repository/files/DEX.bt)

直接复制进去就可以，然后直接F5

最后成功的是这个，bt模板

[sweetscape.com/010editor/repository/files/CLASSAdv.bt](https://www.sweetscape.com/010editor/repository/files/CLASSAdv.bt)

```java
 <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm</artifactId>
            <version>9.1</version>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-util</artifactId>
            <version>9.1</version>
        </dependency>
        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-commons</artifactId>
            <version>9.1</version>
        </dependency>
```

#### 先举一个例子：

```java
public class Hello{
	
	public static void main(String[] args){
		System.out.println(Hello.class.getSuperclass().getName());
	}
	
}
```

![image-20231031185746949](..\img\final\image-20231031185746949.png)

可以看到我们class文件在一系列的常量池之后，会包含访问修饰符，当前类名，以及父类名。

父类名对应的值为7，7代表了常量池中的第7个元素，我们找到第7个常量：

![image-20231031190016169](..\img\final\image-20231031190016169.png)

 `u1 tag=7`代表这个是Class类型的常量，常量值的索引为25

我们再往下看第25个常量：

![image-20231031190118279](..\img\final\image-20231031190118279.png)

16个字符，ascii翻译成字符为，java/lang/object

历经这么多流程我们终于找到了对应父类的 16 进制代码的代码和编码了。

我们现在把 Hello 的继承类换成`java/lang/Number`。

那么只需要把 Object对应的 16 进制代码换成 Number 就可以了。

![image-20231031191012320](..\img\final\image-20231031191012320.png)

然后重新生成一下

```java
javap Hello.class	
```

感觉有些神奇，试试改个不存在的（也是可以的）amazing

![image-20231031191204225](..\img\final\image-20231031191204225.png)

可以看到只要我们能够找到指定区域，去修改这个区域的二进制代码，就可以修改实质的文件。

但事实上并没有这么简单，如果内容非常多，发生字符串常量池复用的时候，我们就不能这么随意的修改某个常量池的内容了。

二来刚好`java/lang/Number`与`java/lang/Object`长度完全一致，否则我们还要做非常多的对齐工作。

所以，修改 class 文件也不是那么容易的事情。

别担心，我们有 ASM。

### 尝试分析Class文件

我们可以先学习下怎么读取class文件内部的各个部分。

比如我想在编译期间通过编译的*.class的文件，获取其内部的所有方法名称，字段名称。

#### Tree Api

看一下ClassNode.java

发现有字段、方法的列表（看看后面能不能得到直接遍历）

![image-20231031195257510](..\img\final\image-20231031195257510.png)

编写一个User：

```java
public class User {
    private String name;
    private int age;
    public String getName() {
        return name;
    }
    public int getAge() {
        return age;
    }
}
```

然后我们希望获取 User.class 中包含的所有方法以及字段：

```java
package com.test.JYcxk;
import jdk.internal.org.objectweb.asm.ClassReader;
import jdk.internal.org.objectweb.asm.Opcodes;
import jdk.internal.org.objectweb.asm.tree.ClassNode;
import jdk.internal.org.objectweb.asm.tree.FieldNode;
import jdk.internal.org.objectweb.asm.tree.MethodNode;

import java.io.FileInputStream;
import java.util.List;

public class TreeApiTest {

    public static void main(String[] args) throws Exception {
      

        Class clazz = User.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);//拿到User的路径
        System.out.println(clazzFilePath);


        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));
        ClassNode classNode = new ClassNode(Opcodes.ASM5);//构造一个ClassNode对象
        classReader.accept( classNode, 0);//对class进行遍历，并把相关信息记录道ClassNode对象中

        List<MethodNode> methods = classNode.methods;// 所有方法
        List<FieldNode> fields = classNode.fields;//所有字段

        System.out.println("methods:");
        for (MethodNode methodNode : methods) {
            System.out.println(methodNode.name + ", " + methodNode.desc);
        }

        System.out.println("fields:");
        for (FieldNode fieldNode : fields) {
            System.out.println(fieldNode.name + ", " + fieldNode.desc);
        }
    }
}
```

##### Utils.java

```java
package com.test.JYcxk;

import java.io.File;

public class Utils {
    public static String getClassFilePath(Class clazz) {
        // file:/Users/zhy/hongyang/repo/BlogDemo/app/build/intermediates/javac/debug/classes/
        String buildDir = clazz.getProtectionDomain().getCodeSource().getLocation().getFile();
        String fileName = clazz.getSimpleName() + ".class";
        File file = new File(buildDir + clazz.getPackage().getName().replaceAll("[.]", "/") + "/", fileName);
        return file.getAbsolutePath();
    }
}
```

看下我们代码的流程；

1. 首先我们拿到class文件的路径
2. 然后交给ClassReader
3. 再构造一个ClassNode对象
4. 调用 ClassReader.accept()方法完成对class遍历，并把相关信息记录到 ClassNode对象中；

这个时候，我们就能通过ClassNode去拿我们所想要的信息了，看一下输出：

```java
methods:
<init>, ()V
getName, ()Ljava/lang/String;
getAge, ()I
fields:
name, Ljava/lang/String;
age, I
```

上述的 API，称为 Tree Api，即我们分析完成 class 文件，把信息存储到 ClassNode，然后通过 ClassNode 再读取即可，有点类似 xml 文件解析时，把整个 xml 文件读取到内存中的方式

(但是上面只是进行了查询的功能，没有提到修改)

#### 	Core Api

但是其实在很多代码的实现中都是基于事件驱动的，顾名思义也就是执行到哪就输出到哪

```java
package com.test.JYcxk;

import jdk.internal.org.objectweb.asm.Opcodes;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;

import java.io.FileInputStream;

public class VisitApiTest {
    public static void main(String[] args) throws Exception {
        Class clazz = User.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));
        
        
        ClassVisitor classVisitor = new ClassVisitor(Opcodes.ASM5) {
            @Override
            public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
                System.out.println("visit field:" + name + " , desc = " + descriptor);
                return super.visitField(access, name, descriptor, signature, value);
            }

            @Override
            public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
                System.out.println("visit method:" + name + " , desc = " + descriptor);
                return super.visitMethod(access, name, descriptor, signature, exceptions);
            }
        };
        classReader.accept(classVisitor, 0);
    }
}
```

```java
visit field:name , desc = Ljava/lang/String;
visit field:age , desc = I
visit method:<init> , desc = ()V
visit method:getName , desc = ()Ljava/lang/String;
visit method:getAge , desc = ()I
```

我们再次梳理下步骤：

1. 首先我们拿到class文件的路径
2. 然后交给ClassReader;
3. 再构造一个ClassVistor对象
4. 将ClassVistor对象传入ClassReader.accept()方法来接收对class文件解析时的“节点”回调信息

Tree Api 是解析完成后，将所有信息保存到一个具体的对象，我们去读取。

一个是参与到解析的流程，监听解析节点的回调，输出结构。

```java
ClassNode是ClassVisitor的子类
public class ClassNode extends ClassVisitor
```

### 简单修改下字节码

```java
 public class C {
      public void m() throws Exception {
         Thread.sleep(100);
      }
}
```

修改为：

```java
public class C {
    public static long timer;

    public void m() throws Exception {
        timer -= System.currentTimeMillis();
        Thread.sleep(100);
        timer += System.currentTimeMillis();
    }
}
```

我们前面学习了，如何读取、遍历解析一个class文件，但还没有尝试过如果回写一个class文件，即进行修改并覆盖原来的文件。

`如果是单纯的覆盖，javassist技术不就能直接实现嘛？有啥区别？这里先打个问好`

#### 初识ClassWriter

先自己简单的过一下

遍历的时候拿到了信息，读取了直接修改然后写入write不就可以了

```java
public class ClassWriter extends ClassVisitor
```

并且也是基础ClassVisitor，不就和上面的大致一样，适配器那种

```java
package com.test.JYcxk;

import jdk.internal.org.objectweb.asm.ClassReader;
import jdk.internal.org.objectweb.asm.ClassWriter;

import java.io.FileInputStream;
import java.io.FileOutputStream;

public class ClassWriterTest {

    public static void main(String[] args) throws Exception {
        Class clazz = C.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);//获得class路径
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));//通过classreader加载

        ClassWriter classWriter = new ClassWriter(0);
        classReader.accept(classWriter, 0);//进行一个遍历

        // 写入文件
        byte[] bytes = classWriter.toByteArray();//生成字节流
        FileOutputStream fos = new FileOutputStream("C:\\Users\\c'x'k\\Desktop\\cc\\Kryo\\target\\classes\\com\\test\\JYcxk\\copyed.class");
        fos.write(bytes);
        fos.flush();
        fos.close();

    }
}
```

#### 开始尝试添加字段 

ClassReader.accept方法，只能接受一个ClassVistor对象，因为我们必须要修改字节码，所以ClassWriter肯定是要传入的。

那么我们怎么传入另一个ClassVisitor对象去修改字节码呢？

我们看一个ClassVisitor 的构造方法：

```java
public ClassVisitor(final int api,final ClassVistor classVisitor)
```

我们完全可以给ClassVisitor传入一个实际对象，自己作为代理对象。

首先尝试添加一个字段，用 visit API 来实现，先编写一个ClassVisitor的子类:

```java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }   
}
```

##### `找个合适的位置插入一个字段：`

其实就是复写哪个方法了，那肯定需要ClassVisitor大概会执行哪些方法，还有其中的执行顺序：

（否则如果执行多次，不就添加了多个重复的字段）

```java
visit visitSource? visitOuterClass? ( visitAnnotation |
   visitAttribute )*
   ( visitInnerClass | visitField | visitMethod )*
   visitEnd
```

可以看到ClassVistor再遍历一个类的时候，相关调用顺序如上，？代表这个方法可能不会调用，*标识可能会调用0次或者多次。 

那么这个合适的位置，首先要选择一个一定会调用的地方法，其次最好只执行一次

选择了方法，那么如何才能插入一个field呢？

```java
其实我们的class最终是由ClassWriter 去生成的，它会通过 visitField 去收集相关信息，也就是说，你调用一次 ClassWriter.visitField方法，他就会以为真有这个 field，然后记录下来。
```

```java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public void visitEnd() {

        FieldVisitor fv = cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "timer",
                "J", null, null);
        //这里就相当于假装访问字段，public static J类型的 timer
        if (fv != null) {
            fv.visitEnd();
        }
        cv.visitEnd();
    }
}
```

我们给原类添加了一个 timer 字段，访问修饰符是 public static，并且其类型是 J 也就是 long 类型。

我们把代码组装到一起

```java
package com.test.lastpoc;

import com.test.JYcxk.Utils;
import jdk.internal.org.objectweb.asm.Opcodes;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassWriter;

import java.io.FileInputStream;
import java.io.FileOutputStream;


public class ClassWriterTest {

    public static void main(String[] args) throws Exception {
        Class clazz = C.class;
        String clazzFilePath = Utils.getClassFilePath(clazz);
        ClassReader classReader = new ClassReader(new FileInputStream(clazzFilePath));

        ClassWriter classWriter = new ClassWriter(0);

        AddTimerClassVisitor addTimerClassVisitor = new AddTimerClassVisitor(Opcodes.ASM5, classWriter);
        classReader.accept(addTimerClassVisitor, 0);

        // 写入文件
        byte[] bytes = classWriter.toByteArray();
        FileOutputStream fos = new FileOutputStream("C:\\Users\\c'x'k\\Desktop\\cc\\Kryo\\target\\classes\\com\\test\\lastpoc\\vvv.class");
        fos.write(bytes);
        fos.flush();
        fos.close();


    }
}
```

有些许神奇，产生了一个 public static long timer;

![image-20231101134311376](..\img\final\image-20231101134311376.png)

### 修改方法

通过上文的学习，我们之前对于方法的遍历，会执行 ClassVisitor的 visitMethod 方法，修改方法肯定是离不开这个方法了，所以我们详细的看下这个方法：

```java
# ClassVisitor
public MethodVisitor visitMethod(
      final int access,
      final String name,
      final String descriptor,
      final String signature,
      final String[] exceptions) {
    if (cv != null) {
      return cv.visitMethod(access, name, descriptor, signature, exceptions);
    }
    return null;
  }
```

```java
其实自己想一下无非也就是和字段一样，看重写哪个方法更加适合，然后就是调用visitMethod了，但是方法对比字段是有内容的，很显然这个visitMethod只是声明了方法的相关信息
```

不过可以看到，这个方法的返回值并不是null，而是一个MethodVisitor，所以我们ClassReader遍历class文件的思路肯定是：先声明相关信息，然后返回一个MethodVisitor，它拿到这个MethodVisitor,再通过MethodVisitor开始遍历这个方法内部的所有信息。

所以。。。我们需要自定义一个MethodVisitor完成代码的插入

```java
public class AddTimerClassVisitor extends ClassVisitor {

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {

        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);
        //这个是本来有的

        MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {
           
        };
        
        return newMethodVisitor;
    }
...
```

问题又来了？通过举一反三的思想，我们应该能够猜到 MethodVisitor 跟 ClassVisitor 设计应该是类似的，里面一堆 visitXXX 方法，我们这次修改字节码是在方法前后分别注入代码，那么到底该选择复写哪些方法呢？

这就要求我们知道 MethodVisitor 中各种 visitXXX 方法的执行顺序了：

```java
visitAnnotationDefault?
(visitAnnotation |visitParameterAnnotation |visitAttribute )* ( visitCode
(visitTryCatchBlock |visitLabel |visitFrame |visitXxxInsn | visitLocalVariable |visitLineNumber )*
visitMaxs )? visitEnd
```

首先是遍历一些注解、参数相关信息；从visitCode开始遍历一整个方法。

我们的注入是：

1. 方法开始：我们选择复写visitCode方法
2. RETURN之前：我们选择复写 visitXxxInsn，再其内部判断当前指令是否是 RETURN；

修改之前，我们要看分别看一下修改前与修改后对应的方法字节码：

```
public void m() throws Exception {
       Thread.sleep(100);
      }
```

对应字节码：

```java
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc2_w        #2                  // long 100l
         3: invokestatic  #4                  // Method java/lang/Thread.sleep:(J)V
         6: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/imooc/blogdemo/blog03/C;
    Exceptions:
      throws java.lang.Exception
}

public void m() throws Exception {
        timer -= System.currentTimeMillis();
        Thread.sleep(100);
        timer += System.currentTimeMillis();
    }
```

对应字节码：

![image-20231101174056335](..\img\final\image-20231101174056335.png)

其实就是写出红框对应的字节码内容

先看我们方法最前面添加的指令：

```java
   timer -= System.currentTimeMillis();

public void visitCode(){
    mv.visitCode();
    
    mv.visitFieldInsn(GETSTATIC,mOwner,"timer","J");//long类型
    mv.visitMethodInsn(INVOKESTATIC,"java/lang/System","currentTimeMillis","()J");
    mv.visitInsn(LSUB);//调用 "timer - System. System.currentTimeMillis"，结果压栈
    mv.visitFieldInsn(PUTSTATIC,mOwner,"timer","J");//将 3 得到的值，再次赋值给 timer 字段；
}
```

和框起来的字节码对比：

```java
0: getstatic     #2                  // Field timer:J
3: invokestatic  #3                  // Method java/lang/System.currentTimeMillis:()J
6: lsub
7: putstatic     #2                  // Field timer:J
```

同样的，我们把方法RETURN前的代码也写了：

```java
@Override
public void visitInsn(int opcode) {

    if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
        mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                "currentTimeMillis", "()J");
        mv.visitInsn(LADD);
        mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");
    }
    mv.visitInsn(opcode);
}
```

有一点不同的是，对于 RETURN 这个指令，我们判断了多个，因为我们并不知道当前方法的返回值情况，如果确定方法没有返回值，那么只要判断 RETURN 即可。

好了，我们贴下完整的代码：

```java
package com.imooc.blogdemo.blog03;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

import static org.objectweb.asm.Opcodes.*;

public class AddTimerClassVisitor extends ClassVisitor {


    private String mOwner;

    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
        mOwner = name;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {

        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);

        if (methodVisitor != null && !name.equals("<init>")) {
            MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {
                @Override
                public void visitCode() {
                    mv.visitCode();

                    mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                            "currentTimeMillis", "()J");
                    mv.visitInsn(LSUB);
                    mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");

                }

                @Override
                public void visitInsn(int opcode) {

                    if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
                        mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
                        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                                "currentTimeMillis", "()J");
                        mv.visitInsn(LADD);
                        mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");
                    }
                    mv.visitInsn(opcode);

                }
            };
            return newMethodVisitor;
        }

        return methodVisitor;
    }

    @Override
    public void visitEnd() {

        FieldVisitor fv = cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "timer",
                "J", null, null);
        if (fv != null) {
            fv.visitEnd();
        }
        cv.visitEnd();
    }
}
```

![image-20231101181351901](..\img\final\image-20231101181351901.png)

使用命令 javap -verbose Test查看字节码 是可以的

但是  java com.test.lastpoc.C 就报错，栈溢出？

![image-20231101181917217](..\img\final\image-20231101181917217.png)

![image-20231101182132628](..\img\final\image-20231101182132628.png)

![image-20231101182324692](..\img\final\image-20231101182324692.png)

看原因:`Exceeded max stack  size`

超出最大堆栈大小

因为我们忽略了一个细节

```java
// 前
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1

// 后
public void m() throws java.lang.Exception;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=1, args_size=1
```

stack的值发生了变化

为什么 getstatic timer，压栈是 2 不是 1 吗？因为 long 类型占两个位置。

##### 第一种方法就是直接提前写出(多没事，主要是少)

```java
@Override
public void visitMaxs(int maxStack, int maxLocals) {
    mv.visitMaxs( maxStack + 4, maxLocals);
}
```

##### 但如果要适合

```java
我们构建ClassWriter的时候
    ClassWriter classWriter=new ClassWriter(0);
    注意构造方法传入了一个 0，实际上接受的是一个 flag，其实有个 flag 是：
         ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
它会自动帮我们重新计算 stackSize。
```

```java
可以看出来，如果你想通过 ASM修改class文件，最起码你得：

字节码指令要非常清楚；
了解操作数栈；
```

### 这篇文章就简单了解一下，下篇文章详细介绍

```java
回答一下上面的问题和javassist的区别， 
javassist是直接生成一个class文件
而ASM可以修改指定文件的字节码，并且依赖的第三方库也不相同
```



文章参考：https://www.wanandroid.com/blog/show/2937

特别详细
