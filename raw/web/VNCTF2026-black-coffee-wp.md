# black coffee

## 题目简述

题目是 JDK17 / Spring Boot 环境下的 Java 原生反序列化利用。服务端存在可控反序列化入口，选手可以提交序列化对象触发服务端 `readObject`，但类名前缀黑名单禁用了 `javax.swing` 和 `java.security`，导致常见的 `EventListenerList`、`TextAndMnemonicHashMap` 和 `SignedObject` 链路都不可直接使用。

关键场景不是寻找入口，而是在 JDK17 模块限制和黑名单下重组 gadget：`TemplatesImpl` 仍可作为字节码执行 sink，Spring/Jackson 环境中的 `POJONode` 能触发 getter，`HashMap + XString` 组合可以从反序列化流程触发 `POJONode.toString()`。最终 payload 需要序列化后做 Base64 和 URL 编码提交。

## 解题过程

先不看题目，下面这些信息是 java 反序列化初学者应该提前了解的：

1. sink点：JDK17下也可以使用 TemplatesImpl 加载字节码

2. springboot 环境下用 POJONode 去调用 getter

3. JDK17 下可用的 readObject -> toString gadget 有：EventListenerList, TextAndMnemonicHashMap,

HashMap+XString 组合。

4. java.security.SignedObject 可以打二次反序列化。

接着回到这题，题目禁用了 javax.swing 和 java.security 开头的类，即 EventListenerList 和
TextAndMnemonicHashMap 和 SignedObject都不能用了。HashMap+XString 组合还可以用。
所以思路就是：

1. 用 TemplatesImpl 加载字节码

2. 用 POJONode 调用 TemplatesImpl 的 getter

3. 用 HashMap+XString 触发 POJONode 的 toString

完整 exp 如下：

```
package cn.org.unk;
import com.fasterxml.jackson.databind.node.POJONode;
import javax.swing.event.EventListenerList;
import java.net.URLEncoder;
import java.util.Base64;
import java.util.HashMap;
import java.util.Hashtable;
public class Main {
    public static void main(String[] args) throws Exception{
// Util17.file2ByteArray("target/classes/FilterShell.class")
        Object calc = Gadget17.getTemplatesImpl("calc");
        POJONode pojoNode = Gadget17.getPOJONode(calc);
        HashMap hashMapXString = Gadget17.get_HashMap_XString(pojoNode);
        byte[] serialize = Util17.serialize(hashMapXString);
        System.out.println(URLEncoder.encode(Base64.getEncoder().encodeToString(serialize)));
}
}
```

用到的所有代码都可以在这里找到：
[https://github.com/un1novvn/Java-unser-utils-17/blob/master/src/main/java/cn/org/unk/Gadget17.java](https://github.com/un1novvn/Java-unser-utils-17/blob/master/src/main/java/cn/org/unk/Gadget17.java)
初学者着重看这份代码的以下几个部分，因为这些关键点每个都细讲的话篇幅很长，这里就留给读者自行学习
了：

1. TemplatesImpl 为什么要套一层 JDKDynamicProxy

2. HashMap 构造的时候，yy zZ 这两个键有什么特殊作用吗

3. 仓库里面有个 Util17，他的静态代码块里有个 bypassModule，尝试把这个静态代码块注释掉，看看会发生什

么

4. getPOJONode 里面有个 removeMethod 的操作，作用是什么？注释掉看看会发生什么？

## 方法总结

JDK17 下做 Java 反序列化时，重点不是死记某条 gadget，而是先看过滤掉了哪些包和类，再检查剩余的触发面。`TemplatesImpl` 仍可作为字节码执行 sink，Spring/Jackson 场景下 `POJONode` 可以把 getter 调用接入链路；当 `SignedObject` 二次反序列化被禁时，`HashMap + XString` 是一个可替代的 `readObject -> toString` 触发器。
