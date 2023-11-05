# RASP命令执行

```java
终于见到了RASP的身影🐅🐆🐈🐕
```

### 原理

```java
Rasp的大概原理就是，利用Java Agent插桩技术，在JVM加载特定字节码前进行hook，或者重新加载某个类的字节码，对字节码进行修改，在敏感函数执行前添加安全检测的逻辑。
```

因此重点就放在了类和函数的hook点。

在实际应用中，还得考虑如下元素：

1. RASP对源程序性能的影响
2. 插桩后源程序是否还能正常稳定运行
3. RASP依赖与原项目依赖的冲突

### 本篇尝试实现一个简易的RASP来防御命令执行

这个不陌生，命令执行底层通过native C代码实现的

```java
java.lang.Runtime#exec
​ -> java.lang.ProcessBuilder#start
​ -> java.lang.ProcessImpl#start
​ -> ProcessImpl#
​ -> native create
```

![image-20231103193449299](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103193449299.png)

为屏蔽操作系统差异，我们这里hook瞄在`ProcessImpl#start处。`也可以通过更底层的方法绕过。

### preamin

Java程序启动时就加载了`java.lang.ProcessImpl`类，通过premain来hook这个类。

(agent main也行|retransformClasses重新加载这个类)

直接给了三个类

```java
package com.demo.rasp.agent;

import com.demo.rasp.transformer.RaspTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class RaspAgent {
    public static void premain(String agentArgs, Instrumentation inst){
        System.out.println("premain start");
        ClassFileTransformer transformer=new RaspTransformer();
        inst.addTransformer(transformer,true);
    }
}
//实现了一个premain的方法
```

```java
package com.demo.rasp.transformer;

import com.demo.rasp.adpator.ProcessImplAdaptor;
import static org.objectweb.asm.Opcodes.*;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class RaspTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (className.equals("java/lang/ProcessImpl")) {
            ClassReader cr = new ClassReader(classfileBuffer);
            ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
            ClassVisitor cv = new ProcessImplAdaptor(ASM9, cw);
            cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.EXPAND_FRAMES);
            return cw.toByteArray();
        }
        return classfileBuffer;
    }
}//重写了ClassFileTransformer#transform
//如果类名等于ProcessImpl，读取它的字节码，初始化classwriter，然后classvisitor遍历，最后输出字节码
```

实现了一个ClassFileTransformer的子类。addTransformer方法配置之后，后续的类加载都会被Transformer拦截，在`transform`方法中对字节码进行修改后再返回。判断当前类名是否为`java/lang/ProcessImpl`(注意这里已经是字节码层面的了，所以类名格式是`Internal Name`)。修改字节码的步骤在前面ASM已经介绍过了。



### transformation

根据`ProcessImpl#start`的方法名和方法描述符来hook（ProcessImpl就只有一个start方法，没有重载方法，也可以不判断方法描述符）

```java
static Process start(String cmdarray[],
                     java.util.Map<String,String> environment,
                     String dir,
                     ProcessBuilder.Redirect[] redirects,
                     boolean redirectErrorStream)
```



```java
package com.demo.rasp.adpator;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.commons.AdviceAdapter;
import static org.objectweb.asm.Opcodes.*;

public class ProcessImplAdaptor extends ClassVisitor {
    public ProcessImplAdaptor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
        System.out.println("init ProcessImplAdaptor");
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        if (name.equals("start") &&
                descriptor.equals("([Ljava/lang/String;Ljava/util/Map;Ljava/lang/String;[Ljava/lang/ProcessBuilder$Redirect;Z)Ljava/lang/Process;")) {
            System.out.println("hooked");
            MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
            return new AdviceAdapter(ASM9, mv, access, name, descriptor) {
                //重写方法目的是，onMethodEnter能直接判断return之前加入
                @Override
                protected void onMethodEnter() {
                    mv.visitVarInsn(ALOAD, 0);
                    super.visitMethodInsn(INVOKESTATIC, "com/demo/rasp/protection/ProcessImplThrow", "protect", "([Ljava/lang/String;)V", false);//判断完是start后，就执行方法   protect方法  参数字符串  返回值V（void）
                }
            };
        }
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }
}
```

```java
static Process start(String cmdarray[],
                     java.util.Map<String,String> environment,
                     String dir,
                     ProcessBuilder.Redirect[] redirects,
                     boolean redirectErrorStream)
```

```java
重写了onMethodEnter方法。onMethodEnter方法在方法开始执行时被调用，在方法的入口位置插入指定的字节码指令。在这里，代码使用mv.visitVarInsn指令加载第一个方法参数（索引为0）到栈顶，然后通过调用INVOKESTATIC指令调用名为"com/demo/rasp/protection/ProcessImplThrow"的类的"protect"方法，传入加载的方法参数作为参数。


([Ljava/lang/String;：这是第一个参数的类型。[表示一个数组，Ljava/lang/String;表示java.lang.String类型。所以[Ljava/lang/String;表示一个String类型的数组。

Ljava/util/Map;：这是第二个参数的类型。Ljava/util/Map;表示java.util.Map类型。

Ljava/lang/String;：这是第三个参数的类型，一个java.lang.String。

[Ljava/lang/ProcessBuilder$Redirect;：这是第四个参数的类型。同样，[表示一个数组，Ljava/lang/ProcessBuilder$Redirect;表示java.lang.ProcessBuilder.Redirect类型的数组。

Z：这是第五个参数的类型，一个boolean。

)Ljava/lang/Process;：这是返回类型，一个java.lang.Process。

所以，整个方法的参数类型是String[]、Map、String、ProcessBuilder.Redirect[]和boolean，返回类型是Process
```

`ProcessImpl#start`是静态方法，局部变量表里第一个(0号索引)存的为方法的第一个参数，即待执行的命令。

`aload_0`将其入栈，到这就获取到执行的命令。为方便处理，这里调用了自己写的一个类的方法。

```java
package com.demo.rasp.protection;

import java.util.Arrays;

public class ProcessImplThrow {
    public static void protect(String[] cmd) {
        System.out.println("Evil Command: " + Arrays.toString(cmd));
        throw new RuntimeException("protected by rasp :)");
    }
}
```

执行的这个方法，方便后面打印输出

直接maven package,打jar包

 ![image-20231103210010857](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210010857.png)

然后创建一个新的项目，通过刚才的jar包运行

```java
import java.io.IOException;

public class Test {
    public static void main(String[] args) throws IOException {
        Runtime.getRuntime().exec("calc");
    }
}
```

![image-20231103210146945](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210146945.png)

![image-20231103210133526](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210133526.png)

百度说我少了mf文件，😱😱😱cn，服啦怪不得看那个premain贼眼熟

```java
PreMainDemo  启动时加载    premain这个文件
```

##### **agent.mf**(建立在resources/META-INF下面)

```java
Manifest-Version: 1.0
Premain-Class: com.demo.rasp.agent.RaspAgent
```

![image-20231103212214672](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103212214672.png)

服啦还是不对，你的结果我的结果好像不一样？这意味着什么，意味着要改很久很久了🤢🤢🤢

```java
文章是用的maven-assembly-plugin 插件进行的打包mvn clean package
想到的全试了，可还是报这个错误不太理解
```

```java
Manifest-Version: 1.0
Premain-Class: com.demo.rasp.agent.RaspAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

![image-20231103214917974](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103214917974.png)

通过看这个报错信息，可以发现只加载了有premain的类，别的类没加载上

## 只能先搁浅一下，去探究探机RASP的原理

![image-20231103220817696](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103220817696.png)

![image-20231103221204529](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103221204529.png)

这两幅图片其实就可以看出RASP的作用，相当于用RASP做了一层隔离修改了底层的字节码，让你的攻击没用

绕过方法的话：就是前面的那个还没成功的例子，再通过底层绕一下。

防御手段：继续底层修改字节码🤑🤑🤑

```java
这里问了问学长，那RASP直接把底层命令执行的字节码全换了不就行了嘛
🤬:但是有些实现了公司业务功能咋办
有道理，那咋办，RASP不能用了？
🙄：污点分析看数据源，比如来自jndi或者反序列化就hook了
soga涨知识了
```



## 终终于成功了👀👀👀

![image-20231105121349414](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105121349414.png)

```java
 <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Premain-Class>com.demo.rasp.agent.RaspAgent</Premain-Class>
     <!--<Agent-Class>com.demo.agent.MyAgent</Agent-Class>-->
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

```java
Premain-Class：包含premain方法的类，需要配置为类的全路径
Agent-Class：包含agentmain方法的类，需要配置为类的全路径
Can-Redefine-Classes：为true时表示能够重新定义Class
Can-Retransform-Classes：为true时表示能够重新转换Class，实现字节码替换
Can-Set-Native-Method-Prefix：为true时表示能够设置native方法的前缀
```

直接点生命周期的package进行打包即可















## java通过JNI调用DLL文件

#### JNI简介

```java
JNI是Java Natice Interface的缩写，它提供了若干的API实现了Java和其他语言的通信（主要是C&C++)。允许Java代码和其他语言写的代码进行交互。JNI是JDK提供的一个native编程接口。JNI允许Java程序调用其他语言编写的程序库或者代码库，比如C/C++。Java在内存管理和性能上有一定的局限，通过JNI我们就可以利用Native程序来克服这些限制
```

具体示例：

##### 1、写一个Java调用c的加减乘除的dll文件

```java
public class HelloWorld {
private native void print();
static
{
System.loadLibrary("Hello");
}
public static void main(String[] args) {
new HelloWorld().print();
}
}
```

##### **2、对你写好的这个java进行编译成class文件**

javac HelloWorld.java

##### **3、使用javah命令生成c所需要的头文件**

**注意：没有.class 而且 如果有包名的话 记得要把包名也写上**.       javah  com.li.dll.JToD11

##### **4、使用vs2017创建dll动态链接库**

但是我的vs不太会用，直接用DEVc++来实现了

文件->新建项目->

![image-20231105115231521](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105115231521.png)

把里面的dll.h换成 刚才我们javah生成的内容

#include "jni.h" 这里记得换成""而不是<>

然后把c文件的内容换成

```java
#include "jni.h"  //这是环境变量java jdk中的
#include<stdio.h> 
#include "dll.h"  //就是刚才我们的h
JNIEXPORT void JNICALL
Java_HelloWorld_print(JNIEnv *env, jobject obj)
{
printf("Hello World!");
return;
}
```

直接编译，修改报错，需要把jdk中的两个文件，jni.h和jni_md.h拖进文件中

![image-20231105115632945](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105115632945.png)

就会生成一个新的dll文件

（这里遇到了个坑，System.loadLibrary("Hello");如果用这个函数会去环境变量jdk的bin目录找Hello.dll，如果是load("绝对路径")）

终于成功了

![image-20231105114308571](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105114308571.png)

```java
所以我们之前看的java中的native方法，就会有一个动态链接库.dll文件去实现它。
```

