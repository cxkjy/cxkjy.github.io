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















