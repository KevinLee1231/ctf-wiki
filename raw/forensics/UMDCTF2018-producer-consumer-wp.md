# UMDCTF 2018 - Producer Consumer

## 题目简述

附件包含 `OBJECTS.DATA`、`INDEX.BTR` 和三份 `MAPPING*.MAP`，它们不是普通数据库，而是离线 Windows WMI Repository。需要从持久化的 WMI 对象中恢复攻击者创建的事件过滤器、消费者及其绑定关系。

## 解题过程

可以使用 [WMI_Forensics](https://github.com/davidpany/WMI_Forensics) 解析仓库。该工具会读取 `OBJECTS.DATA` 中的对象，并借助索引和映射文件恢复命名空间、类实例及对象之间的引用；这些关键能力已经足以离线检查永久 WMI 订阅。

在解析结果中搜索 `__FilterToConsumerBinding`、`__EventFilter` 和 `CommandLineEventConsumer`，可以找到同名的异常对象：

```text
fioadjfoiq23-fioadjfoiq23
```

对应过滤器监视注册表 Run 项中的可疑值：

```text
avicoamvklamdfcxz
```

与它绑定的 `CommandLineEventConsumer` 启动隐藏 PowerShell，并携带一段 Base64 编码命令。将 `-EncodedCommand` 参数按 UTF-16LE 解码后，命令正文中出现：

```text
UMDCTF-{consumers_and_chill}
```

该字符串的 SHA-256 与仓库 `README.md` 中的摘要一致。

## 方法总结

永久 WMI 事件订阅由“过滤器、消费者、绑定”三部分构成，只找到其中一个对象不足以说明完整持久化链。离线仓库分析时应先恢复对象关系，再解码消费者命令；外部工具只是解析载体，关键对象名、触发条件和最终命令都应记录进 WP。
