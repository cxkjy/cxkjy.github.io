## livebos

#### PublicRequestFilter

上面没用，直接从这获得session()开始看

#### ![image-20240712155928211](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712155928211.png)

最后在session中检测PortalToken，必定没null呀。。

![image-20240712160707714](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712160707714.png)


？？？玩不了一点，换下一个
![image-20240712160949066](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712160949066.png)

#### PrivateRequestFilter

这咋玩？？？，不对，🤢这好像不是request.getSession()

![image-20240712161632283](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712161632283.png)

跟进去是一个接口

![image-20240712162056478](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712162056478.png)



![image-20240712162342310](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712162342310.png)

继续进入PortalToken.class
![image-20240712162425241](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712162425241.png)




```java
 private String decrypt(String enc) {
        try {
            Key key = this.getKey();
            return this.decrypt(enc, key);
        } catch (Exception var3) {
            logger.warn(var3, var3);
            return null;
        }
    }
```



```java
    private Key getKey() throws NoSuchAlgorithmException, UnsupportedEncodingException {
        if (this.key == null) {   //这里大概率是为null的
            synchronized(this) {
                if (this.key == null) {
                    if (SystemConfig.INSTANCE.isSessionCluster()) {//这里固定返回false
                        try {
                            this.key = (Key)ClusterStorageTaskManager.getInstance().get("global", "portal.token.key");
                        } catch (Exception var5) {
                            logger.warn(var5, var5);
                        }
                    }

                    if (this.key == null) {
                        String password = TokenUtil.generateToken(this);  //这里是生成的时间戳+sessionid
                        KeyGenerator generator = KeyGenerator.getInstance("DES");
                        generator.init(new SecureRandom(password.getBytes("UTF-8")));
                        this.key = generator.generateKey();  //随机生成的key
                    }
                }
            }
        }

        if (SystemConfig.INSTANCE.isSessionCluster() && System.currentTimeMillis() - this.lastUpdate > 3600000L) {
            try {
                ClusterStorageTaskManager.getInstance().set("global", "portal.token.key", this.key);
                //判断赋值key，没啥用感觉
            } catch (Exception var4) {
                logger.warn(var4, var4);
            }

            this.lastUpdate = System.currentTimeMillis();
        }

        return this.key;
    }
```

看起来这里是生成token，时间戳、sessionid一些因素

![](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712163718527.png)

![image-20240712164144812](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712164144812.png)

然后返回回来，对字符串进行加密

![image-20240712164534427](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712164534427.png)

就是对输入的`PortalToken`进行了一个解密的 
![image-20240712164858269](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240712164858269.png)

#### 总结：

```java
最后发现
PublicRequestFilter、PrivateRequestFilter、AdminRequestFilter 都是需要session的，所以第一个if判断就过不去后面就不用看了。
```

![image-20240715115157848](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240715115157848.png)

这里其实看J2EE框架这种很蒙，只会看一些.xml的配置，然后就是一个文件夹里面全是jsp，不知道从何开始看。。

## struct2鉴权

上面的只是spring的鉴权web.xml，但是问了问👴，说看struct2的配置呀
struct2的象征，文件默认后缀为*.action，在structs.xml或者web.xml中配置
![image-20240716172947753](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716172947753.png)

### web.xml

#### LogonFilter

```java
 LogonFilter     *.do  *.ui   
FormOperate、CalendarServlet、StartWorkflow、ShowWorkflow 、WorkOperate、UIProcessor、OperateProcessor、GeneralObj、NotificationServlet  这些servlet都是走的这个过滤器
```

进来之后else直接doFilter，这么快嘛我丢。。。从请求中获取呃呃呃，那肯定没东西
![image-20240716173745112](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716173745112.png)

`uri=getRequestURI` 、`qs=getQueryString()`，这两个基本都是bug..

![image-20240716174521346](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716174521346.png)

然后啥session，获取LogonUser无所谓，因为没有做判断return不用管，直接看 `isIgnoreUri`里面
![image-20240716174954911](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716174954911.png)

或的条件，只需要结尾是`.css.jsp`即可，比如 `/cxk;.css.jsp?aaa` 搭配tomcat特性，看后端代码处理即可

![image-20240716175126493](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716175126493.png)

下面还有一个`doFilter` 看是否能走到这里，

![image-20240716175414993](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716175414993.png)

上面是一些日志的处理不用关注， `loginUser!=null`，我们肯定为null，直接看else if我们希望返回false不然就直接return了

![image-20240716175811840](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716175811840.png)


![image-20240716180122321](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716180122321.png)

![image-20240716180315354](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716180315354.png)

我们直接不传`isClientRefresh`参数即可，那么会返回false
![image-20240716180352492](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716180352492.png)

进入到这里面，可控的也就只有 `isIgnoreUri `这个点，也就是上面的那个 ，不然进入if就是return
![image-20240716180721134](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716180721134.png)

这里也可以执行到doFilter，`只不过和上面那个用的漏洞点是一个没太大意义`

![image-20240716180859166](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240716180859166.png)

#### CompressionFilter（header无gzip直接过）

userCompressionFilter 是从system.properties中取得`compress.enabled`的值为true
然后进入if，就是看 hearders头中是否有gzip，如果有就退出，这里我们弄成没有。

![image-20240717113039749](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717113039749.png)

到了`cookieHttpOnly`这个if语句，但是在system.properties中没定义`system.cookie.httponly`的值，默认为false

然后进入if(compress)这个判断，compress一开始赋值为**false**，直接进入else语句

![image-20240717114238450](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717114238450.png)

就这😼😼😼，拿下

#### WebServiceFilter

```java
String uri = hpReq.getRequestURI();
String qs = hpReq.getQueryString();  //🤔我？？？
```

上面是日志操作直接略过就行，直接看这个`isRESTFul`
这里希望返回false，不然就直接return了

![image-20240717114814940](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717114814940.png)

进入逻辑就是不能进入/service/LBREST这个路径，这里//不知道能否绕过
![image-20240717115738818](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717115738818.png)

后面的就是一些版本没啥用，也直接没拦截直接下一层过滤器了
![image-20240717120126033](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717120126033.png)

#### structs-feed-config.xml

`里面相当于路由配置的功能`

![image-20240717124032652](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717124032652.png)

#### FeedFilter

在web.xml中被注释调了

![image-20240717124225416](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717124225416.png)

没登录，就现实错误界面，绕不过去

![image-20240717125427136](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717125427136.png)

### 然后看<servlet-mapping\>

#### WorkflowsServlet

### 遇到了一个问题

```java
就是看servlet简单的过一下，看了几个发现都是数据处理的逻辑没啥用。这时候只有两种思路
1、从servlet名字入手，比如叫uploadservlet
2、看历史漏洞入手
```

### FileUpload

这里从 `system.properties`中获得`session.id.parameter`但是没这个值，默认就是`JESSIONID`

![image-20240717141623653](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717141623653.png)

最后会走到这里`error.FileUpload.sessionInvalid`

### SimpleUploaderServlet（任意文件上传  位置不可控）

![image-20240717142916870](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717142916870.png)

```java
private static final String[] supportTypes = new String[]{"Image", "Flash"};
也就是 Type的值需要是上面二种
```

![image-20240717142944010](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717142944010.png)

继续向下走

`enabled`的值初始化定义了为true,

![image-20240717143511287](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717143511287.png)
![image-20240717143421712](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717143421712.png)

首先将multipart中的请求存入map中，然后提取中名字为NewFile的赋值给 FileItem uplFile
这里就是提取出 文件名 +后缀名，去除冗余

![image-20240717145911130](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717145911130.png)

这里的relativePath 是这里getId（）的值
![image-20240717150244723](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717150244723.png)
后面的就是写入文件的操作

这里我感觉是可能出东西的，就是这个写入的位置不可控。。。

发现是loginFilter是真的可以绕过

![image-20240717150937036](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717150937036.png)

## 先看个老洞（LiveBOS ShowImage.do 任意文件读取漏洞)

```plain
/feed/ShowImage.do;.js.jsp?type=&imgName=../../../../../../../../../../../../../../../etc/passwd
```

取出最后一个点后面的为后缀名,imgName,本来是判断后缀是什么类型的
然后type的值需要是小写字母

后缀白名单

```java
jpeg 、jpg 、png、。。。。图片音频那些，然后对应的加前缀  video/  或者 image/
```

![image-20240715174728154](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240715174728154.png)

```java
new FSFile("/feed/upload/" + type + "/", imgName);
//这里type必须是小写字母，imgName可控，直接../../../../../etc/passwd即可，所以导致漏洞
```

这里分析一下路由
```java
/feed/ShowImage.do;.js.jsp
这个do不太明白，虽然知道.do是struct2特有的，但为啥需要+.do呢，如果不添加则访问不到
```

![image-20240717154118203](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240717154118203.png)

