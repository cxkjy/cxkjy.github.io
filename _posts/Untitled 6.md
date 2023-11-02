# Class Transformation

## ClassReader

ClassWriter用于生成字节码文件，而ClassReader用于读取字节码文件

```java
/* 
A parser to make a ClassVisitor visit a ClassFile structure, as defined in the Java Virtual Machine Specification (JVMS). This class parses the ClassFile content and calls the appropriate visit methods of a given ClassVisitor for each field, method and bytecode instruction encountered.
*/
public class ClassReader {
    // A byte array containing the JVMS ClassFile structure to be parsed.
    final byte[] classFileBuffer;
    // The offset in bytes, in classFileBuffer, of each cp_info entry of the ClassFile's constant_pool array, plus one
    private final int[] cpInfoOffsets;
    // The offset in bytes of the ClassFile's access_flags field.
    public final int header;

    public ClassReader(final byte[] classFile) {
        this(classFile, 0, classFile.length);
    }

    public ClassReader(final String className) throws IOException {
        this(
            readStream(
                ClassLoader.getSystemResourceAsStream(className.replace('.', '/') + ".class"), true));
    }
}
```

- classFileBuffer: 读取到的字节码数据
- cpInfoOffsets: 字节码中的常量池的位置
- header: 字节码的访问标识符位置

读取class的信息

```java
public class payload {
    public static void main(String[] args) throws IOException {
        ClassReader cr=new ClassReader("sample.HelloWorld");

        System.out.println("access: " + cr.getAccess());
        System.out.println("className: " + cr.getClassName());
        System.out.println("superName: " + cr.getSuperName());
        System.out.println("interfaces: " + Arrays.toString(cr.getInterfaces()));
    }
}
access: 33
className: sample/HelloWorld
superName: java/lang/Exception
interfaces: [java/io/Serializable, java/lang/Cloneable]
```

ClassReader提供一个accept方法来让ClassVisitor访问字节码文件

```java
/*
Makes the given visitor visit the JVMS ClassFile structure passed to the constructor of this ClassReader.
*/
public void accept(final ClassVisitor classVisitor, final int parsingOptions)
```

第二个参数`parsingPotions`可选值有以下5个，会对`ClassVisitor`的visit行为造成不通的影响

1. 0：生成所有ASM代码
2. ClassReader.SKIP_CODE: 忽略代码信息，如`visitXxxInsn`调用
3. ClassReader.SKIP_DEBUG: 忽略调试信息，如visitParameter、visitLineNumber、visitLocalVariable
4. ClassReader.SKIP_FRAMES: 忽略frame信息，如visitFrame
5. ClassReader.EXPAND_FRAMES: 对frame信息进行扩展

使用`ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES`能得到功能完整，复杂度低的字节码文件

修改字节码文件的流程如下：

![image-20231102193318874](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231102193318874.png)

ClassReader是旧字节码的入口，ClassWriter 是新字节码的出口，中间可以有多个 ClassVisitor 来修改字节码，

```java
ClassReader cr = new ClassReader("classfile");
ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
ClassVisitor cv = new ClassVisitor(Opcodes.ASM9, cw) {
    // TODO
};
cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
cw.toByteArray();
```

ClassVisitor是抽象类，将上面的cv替换为我们自定义的`ClassVisitor`子类即可

## Best Practice

重写visit方法

```java
import org.objectweb.asm.ClassVisitor;

public class InterfaceVisitor extends ClassVisitor {
    public InterfaceVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, new String[]{"java/lang/Serializable"});
    }
}
```

修改了visit的interfaces参数，同样其他参数也可以修改，如修改Java版本Version，类名name，父类superName

### Modify Class Field

重写visitField方法

#### Remove Field

```java
public class FieldDelVisitor extends ClassVisitor {
    private final String fieldName;
    private final String fieldDesc;

    public FieldDelVisitor(int api, ClassVisitor classVisitor, String fieldName, String fieldDesc) {
        super(api, classVisitor);
        this.fieldName = fieldName;
        this.fieldDesc = fieldDesc;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        if (name.equals(fieldName) && descriptor.equals(fieldDesc)) {
            return null;
        }
        return super.visitField(access, name, descriptor, signature, value);
    }
}
```

正确情况下 ClassVisitor#visitField 会返回 一个`FieldVisitor`对象，最后会调用其fv.visitEnd,返回null就断掉了

#### Add Field

同样，想要添加新字段，只需再调用一次`ClassVisitor#visitField`, 再调用 `fv.visitEnd`

但需要在哪里进行字段插入的操作（🤔好熟悉上篇文章通过调用方式写过）

一个类有几个字段就会执行几次 `visitField`，若要在这里插入还需设置一个全局标志位来判断新字段是否插入，否则后面的`visitField`会造成重复插入

`visitEnd`最后调用一次，是个不错的选择，就在这插入新字段。（上篇也是它）

```java
public class FieldAddVisitor extends ClassVisitor {
    private final String fieldName;
    private final String fieldDesc;
    private final int fieldAccess;
    private final Object fieldValue;

    public FieldAddVisitor(int api, ClassVisitor classVisitor, String fieldName, String fieldDesc, int fieldAccess, Object fieldValue) {
        super(api, classVisitor);
        this.fieldName = fieldName;
        this.fieldDesc = fieldDesc;
        this.fieldAccess = fieldAccess;
        this.fieldValue = fieldValue;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        return super.visitField(access, name, descriptor, signature, value);
    }

    @Override
    public void visitEnd() {
        FieldVisitor fv = super.visitField(fieldAccess, fieldName, fieldDesc, null, fieldValue);
        fv.visitEnd();
        super.visitEnd();
    }
}
```

### Modify Class Method

重写 MethodDelVisitor方法

##### Remove Method

和上面删除字段的思路一样。 

(这里一开始蒙了，没搞懂意思，其实就是它指定了一个方法，如果比较相同，则会返回null，就不会调用visitMethod进而触发不了那个方法)

```java
public class MethodDelVisitor extends ClassVisitor {
    private final String methodName;
    private final String methodDesc;

    public MethodDelVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc) {
        super(api, classVisitor);
        this.methodName = methodName;
        this.methodDesc = methodDesc;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        if (name.equals(methodName) && descriptor.equals(methodDesc)) {
            return null;
        }
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }
}
```

```java
int api, ClassVisitor classVisitor, String methodName, String methodDesc, int methodAccess
ASM9,                          cw,                 "mul",         "(II)I",      ACC_PUBLIC
    
```



#### Add Method

```java
public abstract class MethodAddVisitor extends ClassVisitor {
    private final String methodName;
    private final String methodDesc;
    private final int methodAccess;

    public MethodAddVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc, int methodAccess) {
        super(api, classVisitor);
        this.methodName = methodName;
        this.methodDesc = methodDesc;
        this.methodAccess = methodAccess;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }

    @Override
    public void visitEnd() {
        MethodVisitor mv = super.visitMethod(methodAccess, methodName, methodDesc, null, null);
        //这条语句是添加了那个方法
        generateMethodBody(mv);//这个是添加方法体
        super.visitEnd();
    }

    protected abstract void generateMethodBody(MethodVisitor mv);
}
```

```java
ClassVisitor cv = new MethodAddVisitor(ASM9, cw, "mul", "(II)I", ACC_PUBLIC) {
    @Override
    protected void generateMethodBody(MethodVisitor mv) {
        mv.visitVarInsn(ILOAD, 1);
        mv.visitVarInsn(ILOAD, 2);
        mv.visitInsn(IMUL);
        mv.visitInsn(IRETURN);
        mv.visitMaxs(0, 0);
        mv.visitEnd();
    }
};
```

### Update Method

**enter & exit**

如何在方法进入和退出时添加一些逻辑呢 ？

回想`MethodVisitor`的调用顺序

visitCode(方法体开始)->visitXxxIns(方法体)  ->visitMaxs-->visitEnd

方法进入时的逻辑可以在visitCode处添加，此时尚未进入方法体。但注意调用visitMaxs时已经退出方法体了，可能执行了return或throw异常，两种情况都是通过 visitInsn(opcode)实现的，所以方法退出的逻辑可以在`visitInsn`处添加。

```java
public class MethodInOutVisitor extends ClassVisitor {
    private final String methodName;
    private final String methodDesc;

    public MethodInOutVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDesc) {
        super(api, classVisitor);
        this.methodName = methodName;
        this.methodDesc = methodDesc;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
        if (name.equals(methodName) && descriptor.equals(methodDesc)) {
            mv = new MethodExitAdapter(api, mv);
        }
        return mv;
    }

    private static class MethodExitAdapter extends MethodVisitor {
        public MethodExitAdapter(int api, MethodVisitor methodVisitor) {
            super(api, methodVisitor);
        }

        @Override
        public void visitCode() {
            super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            super.visitLdcInsn("Method Enter...");
            super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

            super.visitCode();
        }

        @Override
        public void visitInsn(int opcode) {
            if(opcode == ATHROW || (opcode >= IRETURN && opcode <= RETURN)) {
                super.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                super.visitLdcInsn("Method Exit...");
                super.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            }

            super.visitInsn(opcode);
        }
    }
}
```

`visitMethod`中判断当前方法名和方法描述符是否为目标方法，是则返回自定义的`MethodVisitor`。`visitInsn`判断当前`opcode`是否为throw或return

### **AdviceAdapter**

ASM提供了一个抽象类来实现在方法进入前后添加逻辑，它有两个方法`onMethodEnter`和`onMethodExit`

```java
/*
Generates the "before" advice for the visited method. The default implementation of this method does nothing. Subclasses can use or change all the local variables, but should not change state of the stack. This method is called at the beginning of the method or after super class constructor has been called (in constructors).
*/
protected void onMethodEnter() {}

/*
Generates the "after" advice for the visited method. The default implementation of this method does nothing. Subclasses can use or change all the local variables, but should not change state of the stack. This method is called at the end of the method, just before return and athrow instructions. The top element on the stack contains the return value or the exception instance. For example:
  public void onMethodExit(final int opcode) {
    if (opcode == RETURN) {
      visitInsn(ACONST_NULL);
    } else if (opcode == ARETURN || opcode == ATHROW) {
      dup();
    } else {
      if (opcode == LRETURN || opcode == DRETURN) {
        dup2();
      } else {
        dup();
      }
      box(Type.getReturnType(this.methodDesc));
    }
    visitIntInsn(SIPUSH, opcode);
    visitMethodInsn(INVOKESTATIC, owner, "onExit", "(Ljava/lang/Object;I)V");
  }
 
  // An actual call back method.
  public static void onExit(final Object exitValue, final int opcode) {
    ...
  }
*/
protected void onMethodExit(final int opcode) {}
```



看完之后的我🤢依托答辩，服啦看蒙蔽了。

## 个人理解总结

看了这几篇文章本质上就是找到方法、字段的调用顺序，然后根据顺序添加代码

`MethodVisitor`

```java
visitAnnotationDefault?
(visitAnnotation |visitParameterAnnotation |visitAttribute )* ( visitCode
(visitTryCatchBlock |visitLabel |visitFrame |visitXxxInsn | visitLocalVariable |visitLineNumber )*
visitMaxs )? visitEnd
```

1. 方法开始：我们选择复写 visitCode 方法；
2. RETURN 之前：我们选择复写 visitXxxInsn，再其内部判断当前指令是否是 RETURN；

`ClassVisitor`

```java
visit visitSource? visitOuterClass? ( visitAnnotation |
   visitAttribute )*
   ( visitInnerClass | visitField | visitMethod )*
   visitEnd
```

字段直接复写 visitEnd

#### 让我们对下面这个例子，仔细分析，虽然也分析过

![image-20231102214300463](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231102214300463.png)

![image-20231102214606395](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231102214606395.png)

```java
package com.imooc.blogdemo.blog03;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

import static org.objectweb.asm.Opcodes.*;

public class AddTimerClassVisitor extends ClassVisitor {//写了一个类继承ClassVisitor
    private String mOwner;
    public AddTimerClassVisitor(int api, ClassVisitor classVisitor) {//调用父类，固定格式
        super(api, classVisitor);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
        mOwner = name;
    }//也没啥，固定，赋个值

    @Override
    //这里是添加方法的重点
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {

        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);

        if (methodVisitor != null && !name.equals("<init>")) {//如果方法不是构造方法
            MethodVisitor newMethodVisitor = new MethodVisitor(api, methodVisitor) {//这里相当于一个重写
                @Override
                public void visitCode() {
                    mv.visitCode();//相当于开启字节码

                    mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");//先让这个timer变量入栈
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                            "currentTimeMillis", "()J");//调用这个 currentTimesMillis方法
                    mv.visitInsn(LSUB);//从操作数栈中弹出long类型的值   减法
                    mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");//重新给字段赋值

                }

                @Override
                public void visitInsn(int opcode) {
//这里为啥这么写，因为这条语句在return上，首先判断是否是 return  throw 也就是最后一句
                    if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {
                        
                        mv.visitFieldInsn(GETSTATIC, mOwner, "timer", "J");
                        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System",
                                "currentTimeMillis", "()J");
                        mv.visitInsn(LADD);//从操作数栈中弹出long类型的值   加法
                        mv.visitFieldInsn(PUTSTATIC, mOwner, "timer", "J");
                    }
                    mv.visitInsn(opcode);

                }
            };
            return newMethodVisitor;
        }

        return methodVisitor;
    }

    @Override
    public void visitEnd() {

        FieldVisitor fv = cv.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_STATIC, "timer",
                "J", null, null);
        if (fv != null) {
            fv.visitEnd();
        }
        cv.visitEnd();
    }
}

```



