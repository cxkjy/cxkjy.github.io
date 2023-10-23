## Java数据流分析

#### 本地数据流

本地数据流库位于模块DataFlow中，它定义了表示数据可以通过的任何元素的类节点。节点分为表达式节点（ExprNode)和参数节点（ParameterNode)。可以使用成员谓词asExpr 和 ASAMETER在数据流节点和表达式/参数之间进行映射：

```java
class Node{
    Expr asExpr() {....}
    Parameter asParameter() {....}
}
```

例如，可以在零个或多个本地步骤中找到从参数源到表达式接收器的流：

此查询查找传递给新FileReader()的文件名

### 感觉是这种 class FileReader()  ，获取它的有参构造 

```java
import java 
from Constructor fileReader,Call call
where 
    fileReader.getDeclaringType().hasQualifiedName("java.io","FileReader") and call.getCallee()=fileReader
    select call.getArgument(0)
    
 定义了一个java.io.FileReader的构造器，并且查询了第一个参数
```

### 那如果我们使用本地数据流来查找流入参数的所有表达式：

(这里我觉得应该是初始化这个类，调用构造器)

````java
import java
import semmle.code.java.dataflow.DataFlow	
 
from Constructor fileReader,Call call, Expr src
where
   fileReader.getDeclaringType().hasQualifiedName("java.io","FileReader") and call.getCallee()=fileReader and 
    DataFlow::localFlow(DataFlow::exprNode(src),
    DataFlow::exproNode(call.getArgument(0)))
        
     select src
````



练习1：编写一个查询，查找用于创建java.net.URL，使用本地数据流。

```java
import java
import seemle.code.java.dataflow.DataFlow

from Constructor neturl,Call call, Expr src
where 
    neturl.getDeclaringType().hasQualifieldName("java.io","FileReader") and call.getCallee()=neturl and 
    DataFlow::localFlow(DataFlow::exprNode(src),
    DataFlow::exproNode(call.getArument(0)))
        
        select src
```

#### 调用不包含此格式的字符串函数

```java
import java
import semmle.code.java.dataflow.DataFlow
import semmle.code.java.StringFormat

from StringFormatMethod format,MethodAccess call, Expr formatString
where
    call.getMethod()=format and call.getArgument(format.getFormatStringIndex())=formatString and 
    not exists(DataFlow::Node source,DataFlow::Node sink | DataFlow::localFlow(source,sink) and source.adExpr() instanceof StringLiteral  and sink.asExpr()=formatString)
   
    select call,"Argument to String format method isn't hard-coded"
```





## 案例实现

 ```java
 依旧是拿micro旧项目来改造，但是第一次加了 psvm执行方法，查询就报错了
 读了N篇文章，硬是没有一篇文章是自己创建的数据库，都是 git clone真不理解，自己把psvm去掉发现就可以了 
 codeql database create X:\codeqldatabase\old1023 --language="java" --command="mvn clean install --file pom.xml" --source-root=X:\codeqldatabase\micro_service_seclab --overwrite
 ```

`很好理解，就是调用实例化FileReader对象的第一个参数`

```java
from Constructor fileReader, Call call
where 
  fileReader.getDeclaringType().hasQualifiedName("java.io", "FileReader")  and
    //构造器 获取类型，如果它的权限类名为  java.io.FileReader
  call.getCallee()=fileReader
    //并且 被调用对象的名字是这个 fileReader
select call.getArgument(0)    //获取参数
```

![image-20231022214514327](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231022214514327.png)

### 这一种和上面的区别,这个会有一个指向的路径

这个查询是从哪传参传过去的

![image-20231022224111088](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231022224111088.png)

从污染源 -->new FileReader，上面那个只获得了传递的参数

```java
import java
import semmle.code.java.dataflow.DataFlow	
 
from Constructor fileReader,Call call, Expr src
where
   fileReader.getDeclaringType().hasQualifiedName("java.io","FileReader") and call.getCallee()=fileReader and 
    DataFlow::localFlow(DataFlow::exprNode(src),
    DataFlow::exproNode(call.getArgument(0)))
        
     select src
```

### `查询出来的是到java.net.URL构造参数，的传参可控的`

```java
import java
import semmle.code.java.dataflow.DataFlow
from Constructor fileReader, Call call, Parameter p
where
 fileReader.getDeclaringType().hasQualifiedName("java.net",
"URL") and
 call.getCallee() = fileReader and
 DataFlow::localFlow(DataFlow::parameterNode(p),
DataFlow::exprNode(call.getArgument(0)))
select p
```

![image-20231022223805848](X:\github\cxkjy.github.io\cxkjy.github.io\img\final\image-20231022223805848.png)





