# sniffy

## 题目简述

首页把 flag 放进 PHP session：`$_SESSION['flag'] = FLAG`，并把可控 `theme` 同样写入该 session。`audio.php` 又允许通过路径遍历读取文件，只要 `mime_content_type` 判断为 `audio/*`。因此可以利用可控 session 文件的内容伪装 MIME 类型，再把 session 文件当作音频读取。

## 解题过程

先固定自己的会话标识，例如令 `PHPSESSID=abcd`，访问首页后 PHP 会写入对应的 `/tmp/sess_abcd`。该文件包含序列化的 `flag` 与 `theme` 字段。

接着多次访问首页并给 `theme` 传入长的重复字符串。官方求解器尝试零到三个前导 `a`，后接大量 `M.K.`：

```text
a...a + "M.K." 重复 300 次
```

不同前导长度会改变 session 序列化内容的字节对齐；其中一种排列会被 libmagic 误判为音频，而不是普通文本。随后请求：

```text
/audio.php?f=../../../../tmp/sess_abcd
```

`file_exists` 对拼接后的路径成立，`mime_content_type` 返回 `audio/*` 时即可通过检查，`readfile` 将整个 session 文件回显。读取其中 `flag|...` 对应的值，得到：

```text
DUCTF{koo-koo-koo-koo-koo-ka-ka-ka-ka-kaw-kaw-kaw!!}
```

## 方法总结

下载接口不能把用户参数直接拼进文件路径；应将逻辑文件名映射到预定义对象，并在规范化后确认其仍位于允许目录。MIME 检查只适合作为响应类型提示，不能当作访问控制；攻击者经常能通过文件内容、扩展名或解析器差异影响 MIME 推断结果。
