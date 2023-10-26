

```HTML
layout: post
title: JavaAgent学习
categories: [blog ]
tags: [Java,]
description: 探测
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: azeril.com

```

```
DASCTF月赛劝退要学RASP，所以需要先学JAVAAGent
```

### JavaAgent简介

参考：https://www.anquanke.com/post/id/239200

```java
Java Agent直译叫做java代理，它本质是一个jar包，只不过这个jar包不能独立运行，需要依附到我们的目标JVM进程中
```

Agent是在Java虚拟机启动之时加载的，这个加载处于虚拟机初始化的早期，在这个时间点上：

1. 所有的java类都未被初始化
2. 所有的对象实例都未被创建
3. 因而，没有任何Java代码被执行

Javaagent是java命令的一个参数。参数javaagent可以用于指定一个jar包，并且对该java包有2个要求；

1. agent的这个jar包的MANIFEST.MF文件必须指定Premain-Class项
2. Premain-Class指定的那个类必须实现premain()方法

premain方法，从字面上理解，就是运行在main函数之前的类。当java虚拟机启动时，在执行main函数之前，JVM会先执行`-javaagent`所指定jar包内Premain-Class这个类的premain方法。

## Agent分为二种加载方式 

### `第一种是启动时加载Agent，即运行在主程序之前`

前边提到`premain()`函数，它是在main函数之前运行的，也就是启动时加载的Agent，函数声明如下

拥有`Instrumentation inst`参数的方法优先级更高：

```java
public static void agentmain(String agentArgs, Instrumentation inst) {
    ...
}

public static void agentmain(String agentArgs) {
    ...
}

public static void premain(String agentArgs, Instrumentation inst) {
    ...
}

public static void premain(String agentArgs) {
    ...
}
```

第一个参数`String agentArgs`就是Java agent的参数

第二个参数`Instrumentation inst`相当重要，会在之后的进阶内容中提到。

#### `这里举一个例子`

PreMainDemo

```java
import java.lang.instrument.Instrumentation;

public class PreMainDemo {
    public static void premain(String agentArgs, Instrumentation inst){
        System.out.println(agentArgs);
        for(int i=0 ; i<5 ;i++){
            System.out.println("premain is loading.....");
        }
    }
}
```

接着打包，先创建 mainfest(注：在前边说到过文件中一定要有Premain-Class属性，其次最后要有空行)

##### **agent.mf**(建立在resources/META-INF下面)

```java
Manifest-Version: 1.0
Premain-Class: Agent.PreMainDemo   //包名.类名
```

##### **agent.jar**

将msf文件和PreMainDemo达成一个jar包

```java
jar cvfm agent.jar agent.mf Agent\PreMainDemo.class
```

##### 然后需要调一下配置（换成VM选项、换成jar包）

![image-20231025171114848](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025171114848.png)

![image-20231025171850391](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025171850391.png)

然后写一个主程序测试类

Hello.java

```java
package cxk;

public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello,Sentiment!");
    }
}
```

前面需要加一个-javaagent

![image-20231025172326845](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025172326845.png)

 发现premain是优先于我们的程序执行的，这个null是因为我们没传参数

![image-20231025172405012](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025172405012.png)

如果是cmd命令行执行

```java
java -javaagent:agent.jar=Sentiment -jar hello.jar  就会显示Sentiment
```

这种有个比较明显的弊端：若目标服务器已启动，则无法预先加载premain，所以就需要另一种加载模式了

## 第二种、启动后加载Agent

在前面说到agent中用到的两种加载方式，第二种就是 `agentmain`，它是启动后加载的

```java
在Java SE 6的Instrumentation 当中，提供了一个新的代理操作方法：agentmain，可以在main函数开始运行之后再运行
```

函数声明如下：

```java
public static void agentmain (String agentArgs, Instrumentation inst)
public static void agentmain (String agentArgs)
```

官方为了实现启动后加载，提供了`Attach API`。Attach API只有二个主要的类，都在`com.sun.tools.attach`包里面。这里有二个比较重要的类，分别是`VirtualMachine和VirtualMachineDescriptor`。

默认是不加载这个tools这个包的，我们需要手动导入，jdk中是自带这个jar包的

![image-20231025211556061](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025211556061.png)

![image-20231025211850587](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025211850587.png)

### VirtualMachine（简单分析）

可以来实现获取系统信息，内存dump、线程dump、类信息统计（例如JVM加载的类）。

```java
public abstract class VirtualMachine {
    // 获得当前所有的JVM列表
    public static List<VirtualMachineDescriptor> list() { ... }

    // 根据pid连接到JVM
    public static VirtualMachine attach(String id) { ... }

    // 断开连接
    public abstract void detach() {}

    // 加载agent，agentmain方法靠的就是这个方法
    public void loadAgent(String agent) { ... }

}
```

1. `list`: 获取所有JVM列表
2. `Attach`: 允许我们通过给attach方法传入一个JVM的pid（进程id），远程连接到JVM上
3. `loadAgent`: 向jvm注册一个代理程序agent,在该agent的代理程序中会得到一个Instrumentation实例，该实例可以在class加载前改变class的字节码，也可以在class加载后重新加载。在调用Instrumentation实例的方法时，这些方法会使用ClassFileTransformer接口中提供的方法进行处理。

VirtualMachineDescriptor 就不做探究了，其实就是个描述虚拟机的容器类，配合 VirtualMachine 使用的。

#### `接下来继续举一个例子`

```java
public class AgentMainDemo {
    public static void agentmain(String agentArgs, Instrumentation inst) {
        System.out.println("agentmain start.........");
    }
}
```

agent.mf

```java
Manifest-Version: 1.0
Agent-Class: Agent.AgentMainDemo
```

打包

```java
jar cvfm agent.jar agent.mf Agent\AgentMainDemo.class
```

Hello.java

```java
public class Hello {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("Hello.main() in test project start!!");
        Thread.sleep(300000000);
        System.out.println("Hello.main() in test project  end!!");
    }
}
```

运行Hello.java后，会sleep等待状态来模拟正常服务，此时查看java服务发现进程是6568

![image-20231025213249247](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025213249247.png)

接着就用attach绑定pid进程，并通过**loadAgent**绑定对应的agent.jar来调用

AttchDemo

```java
public class AttachDemo {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        VirtualMachine attach= com.sun.tools.attach.VirtualMachine.attach("6568");
        attach.loadAgent("C:\\Users\\c'x'k\\Desktop\\cc\\javaagent1\\target\\classes\\META-INF\\agent.jar");
        attach.detach();
    }
}
```

`很神奇这里会先执行main函数，然后加载我们的jar包中的agentmain`

![image-20231025214152159](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025214152159.png)

#### `其实到这里脑子中的疑问愈发激烈`

```java
比如这里我们还需要运行另一个main，把jar包放入那个进程中
如何获得进程号？    //解决
能远程进程号加载嘛？
```

### 动态修改字节码

agentmain中有一个形参Instrumentation，通过它能和目标JVM进行交互，结合javassist修改数据，达到真正Agent的效果。

```java
public class AgentMainDemo {
    public static void agentmain(String agentArgs, Instrumentation inst){
        System.out.println("agentmain start.......");
    }
}
```

#### Instrumentation是一个接口类

```java
public interface Instrumentation {

    // 增加一个 Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。addTransformer方法配置之后，后续的类加载都会被Transformer拦截。对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。
    void addTransformer(ClassFileTransformer transformer);

    // 删除一个类转换器
    boolean removeTransformer(ClassFileTransformer transformer);

    // 在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

    // 判断目标类是否能够修改。
    boolean isModifiableClass(Class<?> theClass);

    // 获取目标已经加载的类。
    @SuppressWarnings("rawtypes")
    Class[] getAllLoadedClasses();

    ......
}
大致理解为，增加addTransformer转换器，然后需要retransformClasses 重新定义class
```

##### `getAllLoadedClasses`

获取所有已经加载的类

```java
public class AgentMainDemo {
    public static void agentmain(String agentArgs, Instrumentation inst) {//遍历获取得到
        Class[] classes = inst.getAllLoadedClasses();
        for(Class aClass:classes){
            String result="class===>"+aClass.getName();
            System.out.println(result);
        }
    }
}
```

`isModifiableClasses`

判断该类是否可以修改

```java
public class AgentMainDemo {
    public static void agentmain(String agentArgs, Instrumentation inst) {
        Class[] classes = inst.getAllLoadedClasses();
        for (Class aClass : classes) {
            String result = "class ==> " + aClass.getName()+ aClass.getName() + "\n\t" + "Modifiable ==> " + (inst.isModifiableClass(aClass) ? "true" : "false") + "\n";
            if (result.contains("true")){
                System.out.println(result);
            }
        }
    }
}
```

#### `ClassFileTransformer`

Instrumentation中还有二个比较重要的类，但是这两个类都有一个共同类型的形参`ClassFileTransformer`,

```java
// 添加 Transformer
void addTransformer(ClassFileTransformer transformer);
// 触发 Transformer
boolean removeTransformer(ClassFileTransformer transformer);
```

也是一个接口，但只有一个方法

```java
public interface ClassFileTransformer {
    default byte[]
    transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer) {
        ....
    }
}
//简单分析一下，像是默认字节方法，参数五个  （加载器，类名，剩下的都看不懂。。。）
```

看其他文章说，

classBeingRedefined为我们 要修改的类，它的值受retransformClasses函数传入的值影响，即：

```java
inst.retransformClasses(Hello);
```

当`retransformClasses`中的值是Hello类时，那此时的`classBeingRedefined`对应的类也就是Hello，根据调用栈也不难看出(**这个后续会用到**)

#### 这里暂时没看懂先打个问好？？？？



#### Javassist

在之前写java链子的时候用到过一点，大概就是直接创建一个class字节码文件。

```java
如果程序运行在JBoss 或者 Tomcat等Web服务器上，ClassPool可能无法找到用户的类，因为Web服务器使用多个类加载器作为系统加载器。在这种情况下，ClassPool必须添加额外的类搜索路径，使用insertClassPath()函数
```

```jaca
cp.insertClassPath(new ClassClassPath(<Class>));
```

insertClassPath ccp=new ClassClassPath(classBeeingRedefined);

这样就可以避免无法加载类的情况

#### `动态修改字节码例子`

发现确实改了代码，好神奇，左边是项目的结构

简单分析一下！

![image-20231025232055956](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231025232055956.png)

`AgentMainDemo.java`

```java
public class AgentMainDemo {
    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        Class[] classes=inst.getAllLoadedClasses();//获取所有已经加载的类。
        for(Class aClass:classes){//达到一个遍历数组
            if(aClass.getName().equals(TransformerDemo.editClassName)){//
                //添加Transformer
                inst.addTransformer(new TransformerDemo(),true);
                //触发Transformer
                inst.retransformClasses(aClass);
```

`AttachDemo.java`

```java
public class AttchDemo {
    public static void main(String[] args) throws AgentLoadException, IOException, AgentInitializationException, AttachNotSupportedException, IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        VirtualMachine attach = VirtualMachine.attach("35588");  // 命令行找到这个jvm的进程号
        attach.loadAgent("C:\\Users\\c'x'k\\Desktop\\cc\\javaagent1\\target\\classes\\META-INF\\DDD.jar");
        //加载agent的一个命令
        attach.detach();
    }
}
```

`Hello.java`

```java
public class Hello {//一个Hello方法，就是修改这个类的方法体
    public void Hello(){
        System.out.println("This is Sentiment !");
    }
}
```

`HelloWorld.java`

```java
public class HelloWorld {
    public static void main(String[] args) throws InterruptedException {
        Hello h1=new Hello();
        h1.Hello();
        Thread.sleep(150000); //延时效果就是为了，重新加载agent做铺垫
        Hello h2=new Hello();
        h2.Hello();
    }
}
```

`TransformerDemo.java`

```java
package lastone;

import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class TransformerDemo implements ClassFileTransformer {

    public static final String editClassName = "lastone.Hello";//设置包.类名（准备修改的类）
    public static final String editMethod = "Hello";//(准备修改类的方法)

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {
            ClassPool cp = ClassPool.getDefault();//javassist语句
            if (classBeingRedefined != null) {//这里是从参数拿来的
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);
                cp.insertClassPath(ccp);
            }
            CtClass ctc = cp.get(editClassName);//这边都是反射的东西
            CtMethod method = ctc.getDeclaredMethod(editMethod);
            //将Hello中的函数体改成System.out.println("Has been modified");
            String source = "{System.out.println(\"Has been modified\");}";
            method.setBody(source);
            byte[] bytes = ctc.toBytecode();
            ctc.detach();
            return bytes;
        } catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
}
```

JYcxk.mf

**`注意`**：如果需要修改已经被JVM加载过的类的字节码，那么还需要设置在 MANIFEST.MF 中添加 Can-Retransform-Classes: true 或 Can-Redefine-Classes: true，其次别忘了空格

而第一种方法是不用的，因为还没加载

```java
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Agent-Class: lastone.AgentMainDemo

```

```java
jar cvfm DDD.jar JYcxk.mf ..\lastone\AgentMainDemo.class
```

```java
现在步骤大概了解了：
首先定义一个需要修改的Hello类里面有hello方法
然后AgentMainDemo类中有agentmain方法用于修改类转换器，包含TransformerDemo
TransformerDemo implements ClassFileTransformer,里面代码是反射修改hello方法的内容
```

### 这里梳理一下关系先

![image-20231026101328874](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026101328874.png)

## 大概就是以上的内容（这里解决几个问题）

1. 获取JVM进程id需要jps，能否通过代码实现呢？
2. 默认的java是不会加载tool.jar的，那如何用代码实现？

根据这两个问题又找到了一篇文章：

```java
import com.sun.tools.attach.*;
import java.io.IOException;
import java.util.List;
public class AttachMain {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        //获取当前系统中所有运行中的虚拟机
        System.out.println("running JVM start ");
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            //如果虚拟机的名称为xxx则该虚拟机为目标虚拟机，获取该虚拟机的pid并attach
            //然后加载 agent.jar 发送给该虚拟机
            System.out.println(vmd.displayName());
            if (vmd.displayName().endsWith("com.test.Demo")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("path\\JavaAgent01-1.0-SNAPSHOT.jar");
                virtualMachine.detach();
            }
        }
    }
}
```

![image-20231026104758300](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026104758300.png)

二者id不同时因为，我重启了一下HelloWorld这个类导致的（无伤大雅）

![image-20231026105305829](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026105305829.png)

这样就解决了第一个问题，VirtualMachine是 tools.jar包中的类 

对于第二个问题，我们要做的就是获得当前jdk的路径，然后一步步取出tool.jar，加载它

```java
 File toolsPath = new java.io.File(System.getProperty("java.home").replace("jre", "lib") + java.io.File.separator + "tools.jar");
        System.out.println(System.getProperty("java.home"));
        System.out.println(File.separator);
//输出内容
C:\Program Files\Java\jdk1.8.0_65\jre
\
```

发现其实挺简单的，但是自己想不到。。。(用的基础的反射实现)

```java
package lastone;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import java.io.File;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.List;

public class test {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        File toolsPath = new java.io.File(System.getProperty("java.home").replace("jre", "lib") + java.io.File.separator + "tools.jar");
        System.out.println(System.getProperty("java.home"));
        System.out.println(File.separator);
        System.out.println(toolsPath.toURI());
        URL url = toolsPath.toURI().toURL();

        URLClassLoader classLoader = new URLClassLoader(new URL[]{url});//加载tools.jar包
        Class<?> MyVirtualMachine = classLoader.loadClass("com.sun.tools.attach.VirtualMachine");
        //加载jar包中的VirtualMachine类
        Class<?> MyVirtualMachineDescriptor = classLoader.loadClass("com.sun.tools.attach.VirtualMachineDescriptor");
//
        Method listMethod = MyVirtualMachine.getDeclaredMethod("list");//获取list这个方法
        List<Object> list = (List<Object>) listMethod.invoke(MyVirtualMachine);//invoke执行这个方法

        for (Object o : list) {
            Method displayName = MyVirtualMachineDescriptor.getDeclaredMethod("displayName");//获取名字 包.类名 在虚拟进程中存在的
            String name = (String) displayName.invoke(o);//执行这个方法
            System.out.println(name);//打印输出

            if (name.contains("com.test.Demo")) {//如果包含我们 要修改的那个进程
                Method getId = MyVirtualMachineDescriptor.getDeclaredMethod("id");//获得id方法
                Method attach = MyVirtualMachine.getDeclaredMethod("attach", String.class);
                String id = (String) getId.invoke(o);//这里会得到id进程号
                Object vm = attach.invoke(o, id);//执行attach进程方法


                Method loadAgent = MyVirtualMachine.getDeclaredMethod("loadAgent", String.class);
                //加载进程 loadAgent方法
                loadAgent.invoke(vm, "D:\\Code\\Java\\JavaSec\\JavaAgent01\\target\\JavaAgent01-1.0-SNAPSHOT.jar");
                //参数是string类型
                Method detach = MyVirtualMachine.getDeclaredMethod("detach");
                detach.invoke(vm);
            }
        }
    }
}
```

## 最后总结

```java
PreMainDemo  启动时加载
agentmain    启动后加载
目前，已经能用代码完成加载tool.jar包，以及里面的类，进程等等，但是如何实现呢，比如java反序列化的题目都会给一个readobject入口这种，了解到有agent内存马，继续学习，这里先打一个问号，就是如果有一个这种网站怎么给他打进去替换？
```

