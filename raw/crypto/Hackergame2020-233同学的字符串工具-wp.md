# Hackergame2020 233 同学的字符串工具 WP

## 题目简述

服务先用 ASCII 正则拒绝字面上的 `flag`，随后分别执行 Unicode 大写转换或 UTF-7 解码；若转换结果是 `FLAG`/`flag` 就返回两段 flag。漏洞来自“转换前校验、转换后使用”以及 Unicode 表示不唯一。

## 解题过程

### 字符串大写工具

检查顺序是：

```python
if re.match(r"[fF][lL][aA][gG]", s):
    reject()
elif s.upper() == "FLAG":
    give_flag()
```

Unicode 连字 `ﬂ`（U+FB02）在大写转换时展开成两个字符 `FL`。输入：

```text
ﬂag
```

原始字符串不匹配四个 ASCII 字母的正则，但 `s.upper()` 恰好等于 `FLAG`，得到：

```text
flag{badunic0debadbad}
```

### UTF-7 转换工具

第二问先在原始输入上做同一正则检查，再调用 `s.decode('utf-7')`。UTF-7 的 `+...-` 段使用改造 Base64 表示 UTF-16BE 数据。字符 `f` 的码位为 `0x0066`，按 6 bit 分组后得到 Base64 字符 `AGY`，因此：

```text
+AGY-lag
```

不会被原始 ASCII 正则拦截，却会解码成 `flag`，返回：

```text
flag{please_visit_www.utf8everywhere.org}
```

## 方法总结

安全校验必须作用于与后续业务逻辑相同的规范化表示。大小写折叠、兼容分解、Unicode 正规化和字符编码都可能产生多对一映射。应先按唯一规范解码与正规化，再在规范结果上做完整匹配，并避免继续支持浏览器环境中早已淘汰的 UTF-7。
