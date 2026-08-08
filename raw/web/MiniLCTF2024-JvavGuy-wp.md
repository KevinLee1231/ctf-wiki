# miniLCTF 2024 Jvav Guy Writeup

## 题目简述

目标是 RuoYi/Spring Boot 应用。Spring Actuator 的 heapdump 端点未受保护，堆转储中保存了 Apache Shiro rememberMe 密钥。取得密钥后可以构造 AES-GCM 加密的 Shiro 反序列化 Cookie，借助应用已有依赖执行 Java 代码并读取 flag。

## 解题过程

### 从 heapdump 提取 Shiro 密钥

直接访问：

```text
GET /actuator/heapdump
```

下载的 HPROF 很大，不适合用普通字符串浏览。用 heapdump 分析器搜索 `shirokey`、`cipherKey`、`rememberMe`，最终从存活对象中恢复 Base64 密钥：

```text
3ScQ/9pIhkTTSylCwe86qg==
```

这个值解码后是 16 字节 AES key，符合 Shiro rememberMe 加密配置。heapdump 不只是“诊断文件”，它包含运行时对象、配置、凭据和密钥。

### 构造 rememberMe 反序列化载荷

对目标检测可知其 rememberMe 使用 AES-GCM。选择应用 classpath 中可用的 Spring/Jackson 链，把最终危险对象设置为 `TemplatesImpl`，其字节码在反序列化过程中加载。工具化复现参数应至少明确：

```text
target URL: http://challenge.example/
key:        3ScQ/9pIhkTTSylCwe86qg==
cipher:     AES-GCM
gadget:     Spring native JacksonWithXString
```

环境不保证出网，因此比反弹 shell 更稳定的方式是注入 Tomcat 命令回显或内存马。命令执行后读取：

```text
cat /ruoyi/flag.txt
```

仓库镜像中的 flag 为：

```text
miniLCTF{w0w_Y0u_Are_a_re@lly_good_SpringActu@t0r_H4cker!}
```

RuoYi 4.7.8 的既有 JNDI 漏洞也可能形成另一条链，但它依赖具体 JDK 的远程类加载或本地 gadget 条件；本题预期且证据最完整的路径是 Actuator heapdump 泄密后打 Shiro 反序列化。

## 方法总结

暴露 heapdump 等同于暴露整个进程内存中的秘密。利用链必须同时满足密钥、加密模式和可用 gadget 三个版本条件，不能只拿到一个 Base64 字符串就宣称可利用。对不出网 Java 环境，应优先采用命令回显或内存驻留载荷。
