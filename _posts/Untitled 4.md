### JavaAgent简介

参考：https://www.anquanke.com/post/id/239200

```java
Java Agent直译叫做java代理，它本质是一个jar包，只不过这个jar包不能独立运行，需要依附到我们的目标JVM进程中
```

Agent是在Java虚拟机启动之时加载的，这个加载处于虚拟机初始化的早期，在这个时间点上：

1. 所有的java类都未被初始化
2. 所有的对象实例都未被创建
3. 因而，没有任何Java代码被执行

Javaagent是java命令的一个参数。参数javaagent可以用于指定一个jar包，并且对该java包有2个要求；

1. agent的这个jar包的MANIFEST.MF文件必须指定Premain-Class项
2. Premain-Class指定的那个类必须实现premain()方法
