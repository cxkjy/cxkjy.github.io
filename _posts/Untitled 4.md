## 前言

```java
看RASP字节码指令，脑子都快炸了，赶紧跑路看看别的==今年西南国赛的一道java
```

前置知识（题目就给了这两个提示）：

- Hessian 原生JDK利用
- Kryo反序列化

之前调试Hessian的链子但是现在咳咳基本全忘了（bushi）

先自己尝试做一下看看能做到哪一步骤

### 首先看有价值的信息，第三方依赖有什么

```java
SpringBoot 2.3.0
kryo  4.0.2
Build-Jdk: 1.8.0_351
```

主要有2个有价值的类；

MessageController.java

 ```java
 @RequestMapping({"/"})
     @ResponseBody
     public Object message(String message) throws Exception {//这里的message是我们可控的
         byte[] decodemsg;
         if (message == null) {
             decodemsg = Base64.getDecoder().decode("ASsBAQIDAWnkAQBqYXZhLnV0aWwuVVVJxAHLyYj656nh3Rj89bSK7ufJrcoDAXRpbWVzdGFt8AnMwumxjGIBAWNvbS5zZWEuVXNl8gEBMbABc2VhY2xvdWTz");
         } else {
             try {
                 decodemsg = Base64.getDecoder().decode(message);
             } catch (Exception e) {
                 decodemsg = Base64.getDecoder().decode("ASsBAQIDAWnkAQBqYXZhLnV0aWwuVVVJxAGBw5uOyvHs1sGsg/nqhOyP9pIDAXRpbWVzdGFt8AnmifmxjGIBAWNvbS5zZWEuVXNl8gEBMbABZXJyb/I=");
             }
         }
         return new CodecMessageConverter(new MessageCodec()).toMessage(decodemsg, null).getPayload();
     }
 }
 //可以这道题不像常规的那种直接进行readObject，而是调用了一些方法
 ```

到这里发现这些调用的方法的类比如toMessage、getPayload()这些貌似都不是出题人写的。先看一下

CodecMessageConverter.java

```java
package org.springframework.integration.codec;

public class CodecMessageConverter extends IntegrationObjectSupport implements MessageConverter {
    private final Codec codec;
    private final Class<?> messageClass = GenericMessage.class;

    public CodecMessageConverter(Codec codec) {
        this.codec = codec;
    }

    @Override // org.springframework.messaging.converter.MessageConverter
    public Object fromMessage(Message<?> message, Class<?> targetClass) {
        try {
            return this.codec.encode(message);
        } catch (IOException e) {
            throw new MessagingException(message, "Failed to encode Message", e);
        }
    }

    @Override // org.springframework.messaging.converter.MessageConverter
    public Message<?> toMessage(Object payload, MessageHeaders headers) {
        //payload传递的是自己可控的   headers是null
        Assert.isInstanceOf(byte[].class, payload);
        try {
            Message<?> decoded = (Message) this.codec.decode((byte[]) payload, this.messageClass);
            if (headers == null) {
                return decoded;
            }
            AbstractIntegrationMessageBuilder<?> builder = getMessageBuilderFactory().fromMessage(decoded);
            builder.copyHeaders(headers);
            return builder.build();
        } catch (IOException e) {
            throw new MessagingException("Failed to decode", e);
        }
    }
}
```

看的脑瓜子疼，既然是MessageCodec也就是kryo里面的，所以还是先学kryo反序列化叭

![image-20231026200712942](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026200712942.png)



## Kryo序列化学习

```java
Kryo 是一个快速序列化/反序列化工具，依赖于字节码生成机制（底层使用了 ASM 库)，因此在序列化速度上有一定的优势，但正因如此，其使用也只能限制在基于 JVM 的语言上（Scala、Kotlin）
其他类似的序列化工具：原生JDK、Hessian、FTS
```

首先引入依赖：

```java
<dependency>
  <groupId>com.esotericsoftware</groupId>
  <artifactId>kryo</artifactId>
  <version>5.2.0</version>
</dependency>
```

```java
package com.test;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;

public class Cxk {
    public static void main(String[] args) throws FileNotFoundException {
        Kryo kryo=new Kryo();
        kryo.register(User.class);
        User u=new User();

        Output output=new Output(new FileOutputStream("file.bin"));
        kryo.writeClassAndObject(output,u);
        output.close();

        Input input=new Input(new FileInputStream("file.bin"));
        User o=(User) kryo.readClassAndObject(input);
        input.close();

        System.out.println(o.getName());
    }
}
```

这里仔细调试一下，序列化和反序列化的过程

1. 从5.0.0版本后，kryo整体进行了较大的重构，其中一个重大的改造是将`com.esotericsoftware.kryo.Kryo`类的`registrationRequired`属性默认设置为true。相当于开启了白名单，只有注册过的类才能被序列化和反序列化。

如果没有注册程序会报错的（如果是基础类型在白名单中就不需要了）

```java
kryo.register(User.class);
```

![image-20231026215219496](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026215219496.png)





#### 首先在序列化之前先注册

kryo.register(User.class)

![image-20231026204101378](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026204101378.png)

关键的地步在这里

这里hash啥的判断目的，就是看你注册的这个类是否在这个白名单中，如果在就不需要注册了直接使用，如果不在则需要注册否则不能进行序列化

![image-20231026204442022](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026204442022.png)

#### 开始序列化

beginObject没啥用，判断多线程的大概是省略即可

![image-20231026205125159](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026205125159.png)

序列化是从这里开始写入数据的

![image-20231026214552368](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026214552368.png)

fields就是我们序列化User中的字段

![image-20231026214638745](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026214638745.png)

然后里面包含了一个Unsafe通过

```java
在JVM中，对实例的Field进行了有规律的存储，通过一个偏移量可以从内存中找到相应的Field值
unsafe实现了在内存层面，通过成员字段偏移量offset来获取对象的属性值
接着获取成员的序列化器，步骤跟上面的一样（getRegistration(type).getSerializer()）
```

![image-20231026214723110](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231026214723110.png)

没啥东西其实，但是Kryo有三种序列化的方式

`类未知且对象可能为null`

```java
kryo.writeClassAndObject(output, object);
Object object = kryo.readClassAndObject(input);
```

`类已知且对象为null`

```java
kryo.writeObjectOrNull(output, object);
SomeClass object = kryo.readObjectOrNull(input, SomeClass.class);
```

`类已知且对象不为null`

```java
kryo.writeObject(output, object);
SomeClass object = kryo.readObject(input, SomeClass.class);
```

这些方法首先都是找到合适的序列化器（serializer），再进行序列化或反序列化，序列化器会递归地调用这些方法。