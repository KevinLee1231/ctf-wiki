# Signin

## 题目简述

题目是 PHP `include` 伪协议利用。服务端把用户可控参数传入 `include`，目标是在长度和关键字过滤下构造可执行的包含内容。过滤规则禁用了 `/`、`convert`、`base`、`text`、`plain`，并限制输入长度不超过 20，因此常规 `php://filter` 或带完整 MIME 的 `data://text/plain` 写法都会被拦。

根据 RFC2397，`data:` 协议在 `text/plain` 场景下可以省略 MIME，因此可以用极短的 `data:,<?=...` 形式塞入 PHP 短标签代码，再通过反引号执行参数里的命令。斜杠也可通过二次编码绕过。

## 解题过程

ban掉了/ 、convert 、base 、text 、plain ，同时限制长度不能大于20

题目给了include，可以用伪协议，但是斜杠被ban掉了，一般的伪协议都没法用
根据RFC2397，data协议中如果是text/plain是可以省略的，因此可以用data:,.....
同时有长度限制，用短标签和反引号写马即可

```
data:,<?=`$_GET[1]`;
```

也可以直接用二次编码绕过

```
data:,<?=`nl %252F*`;
```

## 方法总结

短长度 PHP include 题不要只盯 `php://filter`。当 `data` 协议未禁且 MIME 关键字被过滤时，可以利用 `data:,payload` 的省略写法压缩长度；命令中的 `/` 等字符再用 URL 编码层数处理。判断绕过是否成立时，要同时算过滤前后长度、协议解析和 PHP 代码实际执行语义。
