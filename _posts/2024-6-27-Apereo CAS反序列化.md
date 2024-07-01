---
layout: post
title: Apereo CAS反序列化
categories: [blog ]
tags: [Java,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: shzq
---

 

## 1、漏洞描述

Apereo Cas一般是用来做身份认证的，所以有一定的攻击面，漏洞的成因是因为key的默认硬编码，导致可以通过反序列化配合Gadget使用。

## 2、环境搭建

```java
x:\资料\蓝凌ekp

```



这里[下载地址](https://mvnrepository.com/artifact/org.jasig.cas/cas-server-webapp)，直接选择对应的版本，选择war进行下载，然后导入到tomcat中，运行即可。

直接下载war包，tomcat启动即可
如果想要调试的话，catalina.bat中加上监听端口然后启动
![image-20240628125945743](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628125945743.png)

然后用idea远程Debug链接即可
![image-20240628130014481](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628130014481.png)

![image-20240628130023274](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628130023274.png)

## Apereo CAS 4.1.X ~ 4.1.6

当前测试版本4.15  ，大概流程就是对execution这个参数经过解密方式然后进行了readObject()，然后通过cmd进行回显。
这里和之前跟过的JSF框架中的viewstate反序列化很相似。

![image-20240628113517215](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628113517215.png)

![image-20240628113604031](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628113604031.png)

经过一系列转发，会来到`FlowHandlerAdapter#handle`方法中

![image-20240628114404010](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628114404010.png)

进入`DefaultFlowUrlHandler#getFlowExecutionKey()`中
![image-20240628114842379](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628114842379.png)

然后进入resumeExecution()中，参数为（参数的值，上下文）
![image-20240628115038169](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628115038169.png)

然后转换为字节流

![image-20240628135822485](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628135822485.png)

然后对解密的数据进行了反序列化（看不看解密流程意义不大，因为加密和解密在`AbstractCipherBean`中encrypt和decrypt）
![image-20240628141832031](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628141832031.png)

### 调用链子

![image-20240628141900845](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240628141900845.png)

2000*3=6000+1400+2000=9400 
2850*    3  =8400   1000
