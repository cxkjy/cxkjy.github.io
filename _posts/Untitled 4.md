## Unsafe的黑魔法

`前言`

```
为什么要学习这个类，因为在高版本的jdk中反射就被禁用了，所以需要unsafe这个类
```

构造方法是私有的，所以我们并不能直接 new实例化，需要通过getUnsafe这个对象得到 

![image-20231020174408831](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231020174408831.png)

这里举个例子

```java
public class Main2 {

    public static void main(String[] args) throws Exception {
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        Unsafe unsafe = (Unsafe) theUnsafe.get(null);
        //这里必须预先实例化Person,否则它的静态字段不会加载
        Person person = new Person();
        Class<?> personClass = person.getClass();
        Field name = personClass.getField("NAME");
        //注意，上面的Field实例是通过Class获取的，但是下面的获取静态属性的值没有依赖到Class
        System.out.println(unsafe.getObject(unsafe.staticFieldBase(name), unsafe.staticFieldOffset(name)));
    }
}
@Data
public class Person {

    public static String NAME = "doge";
    public String age;
}
这样就可以获得 doge的值
```

```java
staticFieldBase#
public native Object staticFieldBase(Field f);
返回给定的静态属性的位置，配合staticFieldOffset方法使用。实际上，这个方法返回值就是静态属性所在的Class对象的一个内存快照。注释中说到，此方法返回的Object有可能为null，它只是一个'cookie'而不是真实的对象，不要直接使用的它的实例中的获取属性和设置属性的方法，它的作用只是方便调用上面提到的像getInt(Object,long)等等的任意方法。
    
staticFieldOffset#
public native long staticFieldOffset(Field f);
返回给定的静态属性在它的类的存储分配中的位置(偏移地址)。不要在这个偏移量上执行任何类型的算术运算，它只是一个被传递给不安全的堆内存访问器的cookie。注意：这个方法仅仅针对静态属性，使用在非静态属性上会抛异常。下面源码中的方法注释估计有误，staticFieldOffset和objectFieldOffset的注释估计是对调了，为什么会出现这个问题无法考究
    
getObject#
public native Object getObject(Object o, long offset);
通过给定的Java变量获取引用值。这里实际上是获取一个Java对象o中，获取偏移地址为offset的属性的值，此方法可以突破修饰符的抑制，也就是无视private、protected和default修饰符。类似的方法有getInt、getDouble等等。
```

```java
var unsafe = getunsafe();
var group = java.lang.Thread.currentThread().getThreadGroup();
var f = group.getClass().getDeclaredField("threads");
var threads = unsafe.getObject(group, unsafe.objectFieldOffset(f));
```

