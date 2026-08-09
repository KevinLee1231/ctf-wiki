# Conditions

## 题目简述

用户名原始长度必须小于 40，但转成大写后的长度必须大于 50。ASCII 字符无法满足，Unicode 中存在大写映射会扩展为多个字符的码点。

## 解题过程

后端按如下顺序判断：

```python
if len(username) >= 40:
    return "Username is too long!"
elif len(username.upper()) <= 50:
    return "Username is too short!"
else:
    return flag
```

德语小写 `ß` 的大写映射为两个字符 `SS`。提交 30 个 `ß` 时，原始长度是 30，而大写后长度是 60，两个条件同时通过：

```python
payload = "ß" * 30
assert len(payload) == 30
assert len(payload.upper()) == 60
```

服务返回：

```text
n00bz{1mp0551bl3_c0nd1t10n5_m0r3_l1k3_p0551bl3_c0nd1t10ns}
```

## 方法总结

Unicode 大小写转换不是一对一字节替换，转换前后长度可能变化。长度、安全和规范化检查必须定义统一的字符串表示，并在规范化后执行一致验证。
