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

## 先看个老洞（LiveBOS ShowImage.do 任意文件读取漏洞)

```plain
/feed/ShowImage.do;.js.jsp?type=&imgName=../../../../../../../../../../../../../../../etc/passwd
```

取出最后一个点后面的为后缀名
然后type的值需要是小写字母

后缀白名单

```java
jpeg 、jpg 、png、。。。。图片音频那些，然后对应的加前缀  video/  或者 image/
```



![image-20240715174728154](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20240715174728154.png)