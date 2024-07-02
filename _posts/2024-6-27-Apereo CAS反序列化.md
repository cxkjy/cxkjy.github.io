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

通过路由404报错查看版本：`https://cas.example.com/cas/iwana404;)`

- 

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

### 分析漏洞成因

![image-20240701103728773](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701103728773.png)

这里调用了HandlerAdapter(处理器适配器)，也就是spring-webflow-2.4.1.RELEASE.jar中的`FlowExecutorImpl#resumeExecution`
![image-20240701103832210](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701103832210.png)

进而调用 spring-webflow-client-repo-1.0.0.jar的 `getFlowExecution`方法

![image-20240701105525879](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701105525879.png)

### 出现的小问题

在本地自己编写exp测试的时候发现，只有加上了这个uuid_的才会反序列化成功，疑惑了很久跟一下流程：

![image-20240701134928729](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701134928729.png)

这里会调用`resumeExecution方法`flowExecutionKey的值，就是传入的恶意序列化
![image-20240701135035907](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701135035907.png)

继续调用 `parseFlowExecutionKey`方法
![image-20240701135301878](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701135301878.png)

继续调用`parse`方法
![image-20240701135340908](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701135340908.png)

取出uuid的值，然后把uuid后面的值进行base64解密
![image-20240701135746361](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701135746361.png)

然后后面就是，取出后半段的data，然后进行解密操作。
![image-20240701135919676](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701135919676.png)

加密的主要方式就是 `EncryptedTranscoder`这个类
![image-20240701141131581](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701141131581.png)

重要的是这个keyStore硬编码密钥
加解密相关的配置会先去配置文件中获取，没有配置密钥信息的会使用jar包默认的密钥信息（默认keystore文件位于spring-webflow-client-repo-1.0.0.jar包当中）。

![image-20240701141155493](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701141155493.png)

![image-20240701141307086](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701141307086.png)

#### 然后可以发现看的这个工具是直接生成了一个带回显的内存马，自己尝试一下：

`我自己的想法就是，找到一个上下文context对象，然后把里面的request和response对象提取出来即可`

![image-20240701145935025](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701145935025.png)

这里也是参考其它师傅的文章

![image-20240701150826277](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240701150826277.png)

## Apereo CAS 4.1.7 ～ 4.2.X

![image-20240702133412049](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240702133412049.png)

高版本的依赖去掉了commmons-collections4.但是加了
C3p0和CB的依赖

高版本的额话就是密钥可以从cas.properties中设置
![](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240702135813062.png)



![image-20240702135918870](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240702135918870.png)

c

直接打出现的报错

![image-20240702154007872](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240702154007872.png)



通过设置cas.properties中的字段

```java
warn.cookie.secure=0
webflow.encryption.key=K81v0XNPR2tZKuJA
webflow.signing.key=WlNYnlAlHRtPPPS6rygh5y_-7H1UTzAtJHVpzFoWyogANdoxd99LdjmLEuDKzPeo5Q5IB40zWcteAkDglHy2ZA
```

打算进行二开

cas_exploit-1.0-SNAPSHOT-all.jar
