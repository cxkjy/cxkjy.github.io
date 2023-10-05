---
layout: post
title: 第二届华为杯
categories: [blog ]
tags: [Ctf,]
description: "测验"
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: cxkjy.github.io

---





## 前言





## Xstream绕过姿势

会默认转化16进制



`因为在XmlFriendlyNameCoder#decodeName方法中会对16进制进行解析`

![image-20231004201053269](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231004201053269.png)

```
<org.s_.0070ringframework.aop.support.AbstractBeanFactoryPointcutAdvisor>

就是先把_之前的保存下来，然后截取_.之后的四位也就是 007016进制转化再拼接上
```

![image-20231004201215274](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231004201215274.png)

![image-20231004201237821](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231004201237821.png)

#### 但是本地实验了一下直接改为16进制并没有成功，他也会进入到decodeName里面

然后在xml->Object上面我又加了一条语句，decodeName这次成功了

推测是因为decodeName在实现过程转化的后面，调试一下

![image-20231004205600147](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231004205600147.png)

![image-20231004205632395](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231004205632395.png)

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