# SpringShiro

## 题目简述

附件是 Spring Boot 2.7.8 与 Apache Shiro 1.13.0 组成的应用。所有路由都要求登录，但反编译 `MyRealm.class` 可以直接得到账号 `admin`、密码 `123456`。应用同时启用了 Spring Boot Actuator 的全部 Web 端点，登录后能够下载 `/actuator/heapdump`。

堆快照中包含运行时的 `CookieRememberMeManager` 和 rememberMe 加密密钥。提取密钥后，可构造 Shiro rememberMe 反序列化 Cookie，在目标类路径中选择可用 gadget 获得代码执行。容器里的 `/flag` 权限为 `400`，但附件提供了 SUID root 程序 `/readflag`，因此 RCE 后执行它即可输出真实 flag。

## 解题过程

### 登录并下载 JVM 堆快照

反编译认证逻辑可以看到硬编码凭据：

```java
if (username.equals("admin") && password.equals("123456")) {
    return new SimpleAuthenticationInfo(username, password, getName());
}
throw new IncorrectCredentialsException("username or password is incorrect");
```

`application.properties` 又暴露了全部 Actuator Web 端点：

```properties
management.endpoints.web.exposure.include=*
```

先登录并保存会话 Cookie，再下载堆快照：

```bash
curl -c cookies.txt -b cookies.txt -L \
  -d "username=admin&password=123456" \
  http://target/login

curl -b cookies.txt \
  http://target/actuator/heapdump \
  -o heapdump
```

### 从 heapdump 提取 Shiro key

[JDumpSpider](https://github.com/whwlsfb/JDumpSpider) 是堆快照敏感信息提取工具，能够识别 `CookieRememberMeManager` 并输出 Shiro key。下载其 Release 中的完整 JAR 后执行：

```bash
java -jar JDumpSpider-1.0-SNAPSHOT-full.jar heapdump
```

在结果的 `ShiroKey` 或 `CookieRememberMeManager` 项中记录 Base64 形式的运行时密钥。这里不能直接套用 Shiro 旧版本的公开默认 key：题目需要的是本次进程实例实际持有、并被 heapdump 泄露的密钥。

### 构造 rememberMe 反序列化并读取 flag

[ShiroAttack2](https://github.com/SummerSec/ShiroAttack2) 可以使用给定 key 加密序列化 payload，并自动探测目标类路径中可用的 gadget 与回显方式。启动工具后按以下顺序操作：

1. 将目标地址设为题目根 URL，确认响应中存在 Shiro 的 `rememberMe` Cookie 行为。
2. 把 JDumpSpider 提取的 Base64 key 填入密钥栏并验证。
3. 依次测试可用 gadget；成功后选择命令回显模式，不必先建立反弹 shell。
4. 执行 `/readflag`，读取命令输出。

漏洞的本质是：服务端会解密并反序列化客户端可控的 rememberMe Cookie；一旦其对称密钥泄露，攻击者就能生成能通过解密校验的恶意序列化对象。具体 gadget 取决于目标 JAR 中实际存在的依赖，因此应以工具探测结果为准。

附件 Dockerfile 对权限边界的设置如下：

```dockerfile
RUN chmod 400 /flag && \
    chmod 4755 /readflag
USER app
```

`/readflag` 以 SUID root 身份读取 `/flag`，其输出就是平台下发的真实 flag。源码中的 `0xGame{test}` 只是本地构建占位值，不能作为比赛答案。

## 方法总结

本题的完整链是“硬编码弱口令登录 → Actuator heapdump 泄露运行时 Shiro key → rememberMe 反序列化 RCE → SUID 程序读取 flag”。外部工具分别承担堆对象检索和 payload 生成，但正文中的输入、产物和后续用途必须明确；仅写“用工具打一把”无法解释漏洞为何成立，也容易遗漏 RCE 后的权限边界。
