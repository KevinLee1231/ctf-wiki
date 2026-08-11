# the other minimal php

## 题目简述

服务端 PHP 只有一行：

```php
<?php eval(~htmlspecialchars($_GET[0]));
```

它对参数先做 HTML 转义，再按字节取反并交给 `eval`。这是将可控输入二次解释为 PHP 代码的直接 RCE，决定性障碍是 PHP 解释语义，归入 Web。

## 解题过程

PHP 的按位取反满足 `~(~P)=P`。因此只要让 URL 解码后的参数为目标 PHP 片段 `P` 的逐字节补码，服务端执行 `~htmlspecialchars($_GET[0])` 后就会恢复 `P` 并 `eval`。有效载荷中的非打印字节必须百分号编码，不能直接把原始字节放进 URL。

官方生成器先用不依赖常规字母的表达式构造函数名 `system` 与参数 `cat /flag`，再将整段 PHP 源码逐字节 XOR `0xff`，输出每个字节的 `%xx` 形式。请求流程可以抽象为：

```text
percent-encoded complement -> URL decode -> htmlspecialchars -> bitwise NOT -> eval
```

服务器容器将 flag 复制到 `/flag`，所以恢复后的 PHP 调用读取该文件并把内容写回响应。题目 flag 为：

```
DUCTF{crouching tiger hidden php}
```

## 方法总结

不能把 `htmlspecialchars` 当作代码执行防护：它面向 HTML 输出上下文，既不能阻止 PHP 代码被 `eval`，也无法消除随后可逆的逐字节变换。审计此类极简代码时，要沿数据流逐步还原最终解释器看到的字节，而不是只看过滤函数的名字。根本修复是彻底取消 `eval`，用固定操作和白名单数据结构表达需求。
