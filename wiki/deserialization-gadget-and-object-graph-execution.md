---
type: technique
tags: [web, deserialization, gadget-chain, object-graph, rce]
skills: [ctf-web]
raw:
  - ../raw/web/php-java-python-deserialization.md
  - ../raw/web/sqli-upload-deser-and-command-rce.md
updated: 2026-07-27
---

# Deserialization Gadget and Object-Graph Execution

## 适用场景

攻击者可控的 Java/PHP/Python/.NET/Ruby 序列化数据被应用反序列化，类路径、magic method、回调或对象属性形成可到达文件、模板、网络或命令 sink 的 gadget chain。

## 识别信号

- Cookie、参数、缓存、上传或消息队列中出现语言特有序列化格式。
- 错误暴露类名、`readObject/__wakeup/__destruct/reduce` 等回调。
- 依赖中存在已知 gadget，或源码对象图可控。

## 最小证据

- 确认真实反序列化入口、语言/库版本和可加载类集合。
- 构造无副作用对象证明回调会触发。
- 从入口到 sink 的每个属性类型和调用条件都可满足。

## 解法骨架

1. 解析格式并还原服务端对象类型与字段。
2. 枚举依赖和源码中的 magic method、反射、模板、文件和进程 sink。
3. 自后向前构造对象图，先用可观察低风险 sink 验证。
4. 处理签名/压缩/Base64 包装后，按真实请求编码提交。

## 关键变体

- Java gadget chain：依赖版本与 classpath 决定可用链。
- PHP object injection：property visibility 和析构顺序关键。
- Python pickle/.NET formatter：构造阶段即可调用任意 callable/type。

## 常见陷阱

- 只识别到格式，却未证明服务端执行反序列化。
- 使用与目标依赖版本不匹配的现成 gadget。
- 忽略序列化数据外层 MAC、压缩或 URL 编码。

## 关联技巧

- [php-java-python-deserialization.md](php-java-python-deserialization.md)
- [sqli-upload-deser-and-command-rce.md](sqli-upload-deser-and-command-rce.md)
- [server-side-expression-and-command-context-escape.md](server-side-expression-and-command-context-escape.md)

## 原始资料

- [php-java-python-deserialization.md](../raw/web/php-java-python-deserialization.md)
- [sqli-upload-deser-and-command-rce.md](../raw/web/sqli-upload-deser-and-command-rce.md)
