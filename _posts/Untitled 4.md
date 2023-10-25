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

#### 启动后加载Agent
