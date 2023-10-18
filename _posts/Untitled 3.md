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

感觉是这种 class FileReader()  

```java
import java 
from Constructor fileReader,Call call
where 
    fileReader.getDeclaringType().hasQualifiedName("java.io","FileReader") and call.getCallee()=fileReader
    select call.getArgument(0)
    
 定义了一个java.io.FileReader的构造器，并且查询了第一个参数
```

那如果我们使用本地数据流来查找流入参数的所有表达式：

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



