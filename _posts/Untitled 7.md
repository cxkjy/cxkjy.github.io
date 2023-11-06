# RASP命令执行

```java
终于见到了RASP的身影🐅🐆🐈🐕
```

### 原理

```java
Rasp的大概原理就是，利用Java Agent插桩技术，在JVM加载特定字节码前进行hook，或者重新加载某个类的字节码，对字节码进行修改，在敏感函数执行前添加安全检测的逻辑。
```

因此重点就放在了类和函数的hook点。

在实际应用中，还得考虑如下元素：

1. RASP对源程序性能的影响
2. 插桩后源程序是否还能正常稳定运行
3. RASP依赖与原项目依赖的冲突

### 本篇尝试实现一个简易的RASP来防御命令执行

这个不陌生，命令执行底层通过native C代码实现的

```java
java.lang.Runtime#exec
​ -> java.lang.ProcessBuilder#start
​ -> java.lang.ProcessImpl#start
​ -> ProcessImpl#
​ -> native create
```

![image-20231103193449299](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103193449299.png)

为屏蔽操作系统差异，我们这里hook瞄在`ProcessImpl#start处。`也可以通过更底层的方法绕过。

### preamin

Java程序启动时就加载了`java.lang.ProcessImpl`类，通过premain来hook这个类。

(agent main也行|retransformClasses重新加载这个类)

直接给了三个类

```java
package com.demo.rasp.agent;

import com.demo.rasp.transformer.RaspTransformer;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;

public class RaspAgent {
    public static void premain(String agentArgs, Instrumentation inst){
        System.out.println("premain start");
        ClassFileTransformer transformer=new RaspTransformer();
        inst.addTransformer(transformer,true);
    }
}
//实现了一个premain的方法
```

```java
package com.demo.rasp.transformer;

import com.demo.rasp.adpator.ProcessImplAdaptor;
import static org.objectweb.asm.Opcodes.*;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class RaspTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (className.equals("java/lang/ProcessImpl")) {
            ClassReader cr = new ClassReader(classfileBuffer);
            ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
            ClassVisitor cv = new ProcessImplAdaptor(ASM9, cw);
            cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.EXPAND_FRAMES);
            return cw.toByteArray();
        }
        return classfileBuffer;
    }
}//重写了ClassFileTransformer#transform
//如果类名等于ProcessImpl，读取它的字节码，初始化classwriter，然后classvisitor遍历，最后输出字节码
```

实现了一个ClassFileTransformer的子类。addTransformer方法配置之后，后续的类加载都会被Transformer拦截，在`transform`方法中对字节码进行修改后再返回。判断当前类名是否为`java/lang/ProcessImpl`(注意这里已经是字节码层面的了，所以类名格式是`Internal Name`)。修改字节码的步骤在前面ASM已经介绍过了。



### transformation

根据`ProcessImpl#start`的方法名和方法描述符来hook（ProcessImpl就只有一个start方法，没有重载方法，也可以不判断方法描述符）

```java
static Process start(String cmdarray[],
                     java.util.Map<String,String> environment,
                     String dir,
                     ProcessBuilder.Redirect[] redirects,
                     boolean redirectErrorStream)
```



```java
package com.demo.rasp.adpator;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.commons.AdviceAdapter;
import static org.objectweb.asm.Opcodes.*;

public class ProcessImplAdaptor extends ClassVisitor {
    public ProcessImplAdaptor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
        System.out.println("init ProcessImplAdaptor");
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        if (name.equals("start") &&
                descriptor.equals("([Ljava/lang/String;Ljava/util/Map;Ljava/lang/String;[Ljava/lang/ProcessBuilder$Redirect;Z)Ljava/lang/Process;")) {
            System.out.println("hooked");
            MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
            return new AdviceAdapter(ASM9, mv, access, name, descriptor) {
                //重写方法目的是，onMethodEnter能直接判断return之前加入
                @Override
                protected void onMethodEnter() {
                    mv.visitVarInsn(ALOAD, 0);
                    super.visitMethodInsn(INVOKESTATIC, "com/demo/rasp/protection/ProcessImplThrow", "protect", "([Ljava/lang/String;)V", false);//判断完是start后，就执行方法   protect方法  参数字符串  返回值V（void）
                }
            };
        }
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }
}
```

```java
static Process start(String cmdarray[],
                     java.util.Map<String,String> environment,
                     String dir,
                     ProcessBuilder.Redirect[] redirects,
                     boolean redirectErrorStream)
```

```java
重写了onMethodEnter方法。onMethodEnter方法在方法开始执行时被调用，在方法的入口位置插入指定的字节码指令。在这里，代码使用mv.visitVarInsn指令加载第一个方法参数（索引为0）到栈顶，然后通过调用INVOKESTATIC指令调用名为"com/demo/rasp/protection/ProcessImplThrow"的类的"protect"方法，传入加载的方法参数作为参数。


([Ljava/lang/String;：这是第一个参数的类型。[表示一个数组，Ljava/lang/String;表示java.lang.String类型。所以[Ljava/lang/String;表示一个String类型的数组。

Ljava/util/Map;：这是第二个参数的类型。Ljava/util/Map;表示java.util.Map类型。

Ljava/lang/String;：这是第三个参数的类型，一个java.lang.String。

[Ljava/lang/ProcessBuilder$Redirect;：这是第四个参数的类型。同样，[表示一个数组，Ljava/lang/ProcessBuilder$Redirect;表示java.lang.ProcessBuilder.Redirect类型的数组。

Z：这是第五个参数的类型，一个boolean。

)Ljava/lang/Process;：这是返回类型，一个java.lang.Process。

所以，整个方法的参数类型是String[]、Map、String、ProcessBuilder.Redirect[]和boolean，返回类型是Process
```

`ProcessImpl#start`是静态方法，局部变量表里第一个(0号索引)存的为方法的第一个参数，即待执行的命令。

`aload_0`将其入栈，到这就获取到执行的命令。为方便处理，这里调用了自己写的一个类的方法。

```java
package com.demo.rasp.protection;

import java.util.Arrays;

public class ProcessImplThrow {
    public static void protect(String[] cmd) {
        System.out.println("Evil Command: " + Arrays.toString(cmd));
        throw new RuntimeException("protected by rasp :)");
    }
}
```

执行的这个方法，方便后面打印输出

直接maven package,打jar包

 ![image-20231103210010857](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210010857.png)

然后创建一个新的项目，通过刚才的jar包运行

```java
import java.io.IOException;

public class Test {
    public static void main(String[] args) throws IOException {
        Runtime.getRuntime().exec("calc");
    }
}
```

![image-20231103210146945](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210146945.png)

![image-20231103210133526](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103210133526.png)

百度说我少了mf文件，😱😱😱cn，服啦怪不得看那个premain贼眼熟

```java
PreMainDemo  启动时加载    premain这个文件
```

##### **agent.mf**(建立在resources/META-INF下面)

```java
Manifest-Version: 1.0
Premain-Class: com.demo.rasp.agent.RaspAgent
```

![image-20231103212214672](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103212214672.png)

服啦还是不对，你的结果我的结果好像不一样？这意味着什么，意味着要改很久很久了🤢🤢🤢

```java
文章是用的maven-assembly-plugin 插件进行的打包mvn clean package
想到的全试了，可还是报这个错误不太理解
```

```java
Manifest-Version: 1.0
Premain-Class: com.demo.rasp.agent.RaspAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

![image-20231103214917974](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103214917974.png)

通过看这个报错信息，可以发现只加载了有premain的类，别的类没加载上

## 只能先搁浅一下，去探究探机RASP的原理

![image-20231103220817696](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103220817696.png)

![image-20231103221204529](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231103221204529.png)

这两幅图片其实就可以看出RASP的作用，相当于用RASP做了一层隔离修改了底层的字节码，让你的攻击没用

绕过方法的话：就是前面的那个还没成功的例子，再通过底层绕一下。

防御手段：继续底层修改字节码🤑🤑🤑

```java
这里问了问学长，那RASP直接把底层命令执行的字节码全换了不就行了嘛
🤬:但是有些实现了公司业务功能咋办
有道理，那咋办，RASP不能用了？
🙄：污点分析看数据源，比如来自jndi或者反序列化就hook了
soga涨知识了
```



## 终终于成功了👀👀👀

![image-20231105121349414](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105121349414.png)

```java
 <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Premain-Class>com.demo.rasp.agent.RaspAgent</Premain-Class>
     <!--<Agent-Class>com.demo.agent.MyAgent</Agent-Class>-->
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

```java
Premain-Class：包含premain方法的类，需要配置为类的全路径
Agent-Class：包含agentmain方法的类，需要配置为类的全路径
Can-Redefine-Classes：为true时表示能够重新定义Class
Can-Retransform-Classes：为true时表示能够重新转换Class，实现字节码替换
Can-Set-Native-Method-Prefix：为true时表示能够设置native方法的前缀
```

直接点生命周期的package进行打包即可

![image-20231105195637953](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105195637953.png)

看一下报错，NoClassDefFoundError,hook到方法了，就是修改字节码时出错了，找不到我们自定义的`ProcessImplThrow`类

这里通过源码可以看到我们输出了hooked，在后面修改字节码调用ProcessImplThrow的时候出的错误

![image-20231105201803143](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105201803143.png)

```java
 protected void onMethodEnter() {
                    mv.visitVarInsn(ALOAD, 0);
                    super.visitMethodInsn(INVOKESTATIC, "com/demo/rasp/protection/ProcessImplThrow", "protect", "([Ljava/lang/String;)V", false);
                }
            };
```

看✌文章说是，NoClassDefFoundError是字节码找到了，但加载时出错了。那就看一下加载器

```java
        System.out.println(Class.forName("com.demo.rasp.protection.ProcessImplThrow").getClassLoader());//写入hook中

```

![image-20231105204535576](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105204535576.png)

可以发现是同一个类加载器-————系统类加载器（也成为应用程序类加载器）

注意到我们的`ProcessImplThrow#protect`是由`ProcessImpl#start`去调用的，这个类是由BootStrap去加载的。根据双亲委派模型，类加载时会先交给其parent去加载，若parent加载不了，再由自己加载。BootStrapClassLaoder已经是最顶上的类加载器了，其搜索范围是<JAVA_HONE>\lib，这是找不到我们的classpath下的类。因此我们需要把这个rasp jar包的位置添加到BootStrapClassLoader的搜索路径中。`Instrumentation`刚好提供了一个方法`appendToBootstrapClassLoaderSearch`来实现这点。

```java
简单的阐述一下： ProcessImpl#start 是启动类加载器加载的（为什么？因为它是在AVA_HOME的lib目录,双亲委派加载，一直会让父加载器加载，而Bootstrap加载器能调用），但是却找不到这个ProcessImplThrow#protect,因为它没在lib目录中，所以我们采取一种方法让RASPjar包添加进目录即可
这时候你可能提问 能识别jar包嘛？一张图解释🙈🙈🙈
```

![image-20231105213631611](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105213631611.png)

### 所以i我们现在就是想如何把这个RASP jar包添加进lib目录中

实际上OpenRasp用的就是这个来解决。

```java
// premain
addJarToBootStrap(inst);
// =====================
public static void addJarToBootStrap(Instrumentation inst) {
    URL localUrl = RaspAgent.class.getProtectionDomain().getCodeSource().getLocation();
    try {
        String path = URLDecoder.decode(
            localUrl.getFile().replace("+", "%2B"), "UTF-8");
        System.out.println(path);
        //前面就是保护域啥的为了获取到jar包
        //C:/Users/c'x'k/Desktop/cc/LASTRASP/target/LASTRASP-1.0-SNAPSHOT-jar-with-dependencies.jar
        inst.appendToBootstrapClassLoaderSearch(new JarFile(path));//这里是把jar包添加进 lib目录中
        
        //但是添加了我的lib目录没看到，可能只是一个虚拟添加那种
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

（这里添加到`BootstrapClassLoader`搜索范围的类貌似都需要是公开类，`addJarToBootStrap`要放在premain的开头，否则会有一些奇怪的错误）	

![image-20231105214736605](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105214736605.png)

当然也可以不调用自定义的类，直接给`ProcessImpl`加个方法，这样就不存在类加载的问题了。但是需要手搓ASM

（这个方法本质上和上面的没有区别，就是麻烦点）





## Bypass	

上面的hook点在`ProcessImpl#start`,我们可以通过更底层的函数来绕过。

以windows为例，直接调用`ProcessImpl`的native方法`create`

利用`sun.misc.Unsafe#allocateInstance`去实例化`ProcessImpl`

![image-20231106091936729](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106091936729.png)

```java
Process process = Runtime.getRuntime().exec("whoami");

BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
```

之前调用`Runtime#exec`会返回一个`Process`对象，而`ProcessImpl`是`Process`的实现类

`getInputStream`返回`Process`对象的`stdout_stream`标准输出流，我们获取命令执行的结果大概是这样子的

`先简单看一下getInputStream方法干了什么（就返回了一个输出流对象,说明运算啥的不是在这完成的，那肯定就是构造方法）

![image-20231106095511021](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106095511021.png)

果然在构造方法实现了，一些stdHandles的赋值，但是create是native方法，所以我们不能获得完整的ProcessImpl对象，就需要自己把这个逻辑写出来

(如果方法是用native关键字声明的，也就是原生方法（Native methods），则不能通过反射直接获取和执行这些方法。)

![image-20231106100249416](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106100249416.png)

```java
import sun.misc.JavaIOFileDescriptorAccess;
import sun.misc.Unsafe;

import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class ByPass {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("sun.misc.Unsafe");
        Field field = clazz.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        Unsafe unsafe = (Unsafe) field.get(null);
        Class<?> processImpl = Class.forName("java.lang.ProcessImpl");
        Process process = (Process) unsafe.allocateInstance(processImpl);
        Method create = processImpl.getDeclaredMethod("create", String.class, String.class, String.class, long[].class, boolean.class);
        create.setAccessible(true);
        long[] stdHandles = new long[]{-1L, -1L, -1L};
        create.invoke(process, "whoami", null, null, stdHandles, false);

        JavaIOFileDescriptorAccess fdAccess
            = sun.misc.SharedSecrets.getJavaIOFileDescriptorAccess();
        FileDescriptor stdout_fd = new FileDescriptor();
        fdAccess.setHandle(stdout_fd, stdHandles[1]);
        InputStream inputStream = new BufferedInputStream(
            new FileInputStream(stdout_fd));

        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }
    }
}
```

![image-20231106101504422](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106101504422.png)

## Hook Native

那就把hook点改成native方法不就行了。

但native方法不在java层面，不存在方法体，如何用ASM去修改呢？

`Instrumentation`提供了一个方法`setNativeMethodPrefix`（设置 原生方法前缀）

```java
此方法通过允许使用应用于名称的前缀重试来修改本机方法解析的失败处理。与 ClassFileTransformer 一起使用时，它支持检测本机方法。
由于本机方法不能直接检测（它们没有字节码），因此必须使用可以检测的非本机方法包装它们。例如，如果我们有：
    
native boolean foo(int x);


我们可以转换类文件（在类的初始定义期间使用 ClassFileTransformer），以便它变成：
boolean foo(int x) {
... record entry to foo ...
return wrapped_foo(x);
}
native boolean wrapped_foo(int x);	


其中 foo 成为附加前缀“wrapped_”的实际本机方法的包装器。
包装器将允许在本机方法调用中收集数据，但现在问题变成了将包装方法与本机实现链接起来。也就是说，方法wrapped_foo需要解析为 foo 的原生实现，可能是：
Java_somePackage_someClass_foo(JNIEnv* env, jint x)
此函数允许指定前缀并发生正确的解析。具体而言，当标准解析失败时，将考虑前缀重试解析。
有两种解决方式，一种是使用 JNI 函数 RegisterNatives 进行显式解决，另一种是正常的自动解决。对于 RegisterNatives，JVM 将尝试以下关联：
method(foo) -> nativeImplementation(foo)
如果此操作失败，将重试解析，并在方法名称前面加上指定的前缀，从而产生正确的解决方法：
method(wrapped_foo) -> nativeImplementation(foo)
    
    
为了自动解析，JVM 将尝试：
method(wrapped_foo) -> nativeImplementation(wrapped_foo)
如果此操作失败，将重试解析，并从实现名称中删除指定的前缀，从而产生正确的解决方法：
method(wrapped_foo) -> nativeImplementation(foo)

```

给原本的native方法加上一个前缀，再套一层方法来调用添加前缀的native方法。

这时候需要重新建立java方法和native方法的映射关系。

以openJDK为例，`ProcessImpl#create`和其C实现对应如下：

![image-20231106114313290](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106114313290.png)

https://github.com/openjdk/jdk/blob/master/src/java.base/windows/native/libjava/ProcessImpl_md.c

![image-20231106114323357](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231106114323357.png)

native方法的名称格式为`Java_PackageName_ClassName_MethodName`，这个规则称为标准解析(`standard resolution`)

如果给jvm增加一个ClassFileTransformer并设置native prefix，jvm将进行自动解析(`normal automatic resolution`)

```java
// premain
if (inst.isNativeMethodPrefixSupported()) {
    // 添加native方法前缀解析
    inst.setNativeMethodPrefix(transformer, NATIVE_PREFIX);
} else {
    throw new UnsupportedOperationException("Native Method Prefix UnSupported");
}
```

```
setNativeMethodPrefix`要在`inst.addTransformer`之后调用，否则会抛出异常`transformer not registered in setNativeMethodPrefix
```

要开启native prefix，还得在`MANIFEST.MF`中设置`Can-Set-Native-Method-Prefix: true`

```
<Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
```

换`javassist`，虽然灵活性没有ASM高，但在这个案例中使用够够的了。

修改上面的`RaspTransformer`

```java
package com.demo.rasp.transformer;

import com.demo.rasp.hook.ProcessImplHook;
import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class RaspTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        if (className.equals("java/lang/ProcessImpl")) {
            try {
                return ProcessImplHook.transform();
            } catch (Exception e) {
                e.printStackTrace();
                throw new RuntimeException(e);
            }
        }
        return classfileBuffer;
    }
}
```

创建一个`ProcessImplHook`类，用于读取目标类并修改类字节码后返回字节数组

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
        CtMethod create = clazz.getDeclaredMethod("create");//得到create或者方法
        CtMethod wrapped = CtNewMethod.copy(create, clazz, null);//复制给了clazz
        wrapped.setName("RASP_create");
        clazz.addMethod(wrapped);

        create.setModifiers(create.getModifiers() & ~AccessFlag.NATIVE);//java类中访问标志取反
        create.setBody("{if($1.equals(\"calc\")) throw new RuntimeException(\"protected by RASP :)\");return RASP_create($1,$2,$3,$4,$5);}");
        clazz.detach();
        return clazz.toBytecode();
    }
}
```

`CtNewMethod.copy`将`create`方法复制一份，并加上前缀`RASP_`，后面jvm将自动解析这个native方法

接下来在原来的方法的基础上去掉`NATIVE`的访问修饰符，设置方法体，`$1`表示第一个参数，以此类推。

判断第一个参数（即`cmdstr`）是否为恶意命令，是的话则抛出异常，否则调用加上前缀`RASP_`的`native`方法，直接return，参数原封不动传入。

（简单阐述，复制了create方法为RASP_create，然后把本来的create的native删掉了，那么create其实就没用了，接着对RASP_create传的参数进行了判断）

## Native Bypass

可以看到上面hook本地方法本质上就是给native方法换了个名，再套上原来壳，如果我们知道native方法的前缀，理论上应该是能绕过的。

```java
这里的意思其实就是，如果我们知道了 RASP_create不就直接调用这个了嘛，相当于这个是最最底层的了
```

## 彻底总结

```java
上面我们实现一个Demo，防御和绕过RASP
防御，我们尽可能的去修改恶意代码的字节码，比如那个ProcessImpl我们就是判断start方法，如果符合就添加一个hook方法然后最后throw抛出异常提前退出就不会真正执行到start方法体中。
    
最后还涉及到了RASP setNativeMethodPrefix这个方法的解析方式，手动加一个前缀然后复制create类（相当于一个套娃，多了层保护罩）除非你能猜到我的前缀名称，然后通过判断参数来看是否是危险的（calc）
```



## java通过JNI调用DLL文件

#### JNI简介

```java
JNI是Java Natice Interface的缩写，它提供了若干的API实现了Java和其他语言的通信（主要是C&C++)。允许Java代码和其他语言写的代码进行交互。JNI是JDK提供的一个native编程接口。JNI允许Java程序调用其他语言编写的程序库或者代码库，比如C/C++。Java在内存管理和性能上有一定的局限，通过JNI我们就可以利用Native程序来克服这些限制
```

具体示例：

##### 1、写一个Java调用c的加减乘除的dll文件

```java
public class HelloWorld {
private native void print();
static
{
System.loadLibrary("Hello");
}
public static void main(String[] args) {
new HelloWorld().print();
}
}
```

##### **2、对你写好的这个java进行编译成class文件**

javac HelloWorld.java

##### **3、使用javah命令生成c所需要的头文件**

**注意：没有.class 而且 如果有包名的话 记得要把包名也写上**.       javah  com.li.dll.JToD11

##### **4、使用vs2017创建dll动态链接库**

但是我的vs不太会用，直接用DEVc++来实现了

文件->新建项目->

![image-20231105115231521](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105115231521.png)

把里面的dll.h换成 刚才我们javah生成的内容

#include "jni.h" 这里记得换成""而不是<>

然后把c文件的内容换成

```java
#include "jni.h"  //这是环境变量java jdk中的
#include<stdio.h> 
#include "dll.h"  //就是刚才我们的h
JNIEXPORT void JNICALL
Java_HelloWorld_print(JNIEnv *env, jobject obj)
{
printf("Hello World!");
return;
}
```

直接编译，修改报错，需要把jdk中的两个文件，jni.h和jni_md.h拖进文件中

![image-20231105115632945](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105115632945.png)

就会生成一个新的dll文件

（这里遇到了个坑，System.loadLibrary("Hello");如果用这个函数会去环境变量jdk的bin目录找Hello.dll，如果是load("绝对路径")）

终于成功了

![image-20231105114308571](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231105114308571.png)

```java
所以我们之前看的java中的native方法，就会有一个动态链接库.dll文件去实现它。
```

