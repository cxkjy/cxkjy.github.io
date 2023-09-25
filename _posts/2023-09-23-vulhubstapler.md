---
layout: post
title: 第一篇博客
categories: [blog ]
tags: [Java,]
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

![image-20230923201032069](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923201032069.png)

扫描端口

```java
nmap -sS -sV -O -p- 192.168.100.45
nmap -p- -sC --min-rate 5000 192.168.100.45
```

![image-20230923202427879](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923202427879.png)

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

![image-20230923203548030](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923203548030.png)

直接ftp登陆，没密码

![image-20230923204159722](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923204159722.png)

发现里面有一个note文件

![image-20230923204732413](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923204732413.png)

在ftp上get note，就会下载到本地

![image-20230923205739889](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923205739889.png)

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

![image-20230923211659726](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923211659726.png)

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

![image-20230923214920751](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923214920751.png)

![image-20230923215306513](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923215306513.png)

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

![image-20230923220509162](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923220509162.png)

利用smbclient工具连进SMB服务

`smbclient -N //192.168.189.130/tmp` // -N：空口令登录

![image-20230923220800325](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923220800325.png)

同样方式进入/kathy

发现了一个txt文档`I'm making sure to backup anything important for Initech, Kathy`

![image-20230923221112055](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230923221112055.png)

### 端口666未知服务

对于未知的服务一般用nc来探测，然后发现是一个压缩包里面还有图片

![image-20230924132752053](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924132752053.png)

将数据下载到本地解压，得到一张图片但是目前感觉用处不大

```java
nc 192.168.100.45 666 > 666 && unzip 666
```

![image-20230924133219366](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924133219366.png)

### 端口3306：MySQL服务

nc搞了一下发现只能看出操作系统ubuntu，其他的并没信息

![image-20230924133537882](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924133537882.png)

### 端口12380：HTTP服务

直接访问终于看到页面了

![image-20230924134100917](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924134100917.png)

这里看wp说回想起之前SSH连进去后发现的https目录，尝试用https访问，终于发现靶机的web页面

也就是这里之前ssh连接发现的https

![image-20230924134349654](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924134349654.png)

从ssh中看出https的目录结构

![image-20230924135212609](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924135212609.png)

![image-20230924135433391](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924135433391.png)

![image-20230924135557968](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924135557968.png)

这里获得了mysql数据库的账号密码，直接mysql -u root -p登陆进去，密码可以看到 是经过加密的

![image-20230924141702753](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924141702753.png)

`用hash-identifiter看这是什么加密`

![image-20230924141907278](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924141907278.png)

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

![image-20230924142537533](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924142537533.png)

发现John还是一个admin权限

![image-20230924142634192](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230924142634192.png)







### 补充

#### nikto工具扫出https