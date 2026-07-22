# 静态网页

## 题目简述

首页看似静态，但右下角 Live2D 模型换装时会请求后端 API。API 的 JSON 响应泄漏下一阶段路由；最终 PHP 代码再组合 `is_numeric()`、PHP 7.3 弱比较和数组参数，要求构造一个既“非数字”又能与 `0` 弱相等的键。

## 解题过程

查看页面源代码可见提示“她是怎么换衣服的”，点击 Live2D 的换装功能后，在开发者工具 Network 面板中能观察到 `/api/get/` 请求。源码中的 API 会在模型 JSON 里额外加入：

```json
{"flag":"Please turn to final1l1l_challenge.php"}
```

访问 `/final1l1l_challenge.php` 后得到核心判断：

```php
$a = $_GET['a'];
$b = $_POST['b'];
if (!is_numeric($a) && !is_numeric($b)) {
    if ($a == 0 && md5($a) == $b[$a]) {
        echo $flag;
    }
}
```

Dockerfile 使用 PHP 7.3。取 `$a = "0a"` 时，`is_numeric("0a")` 为假，但 PHP 7.3 的宽松比较会把以数字开头的字符串转换后与 `0` 比较，因此 `"0a" == 0` 成立。`$b` 必须是数组：数组本身不是数字，且可令键 `0a` 对应 `md5("0a")`。

```text
GET /final1l1l_challenge.php?a=0a
Content-Type: application/x-www-form-urlencoded

b[0a]=e99bb33727d338314912e86fbdec87af
```

注意，这里不是常见的 `0e...` MD5 哈希碰撞。`md5($a)` 没有参与与另一个哈希的弱比较；真正的要求只是让数组元素 `$b['0a']` 精确等于 `md5('0a')`。

## 方法总结

“静态页面”仍可能通过资源 API 与后端交互，Network 面板比只看首页 HTML 更可靠。做 PHP 类型题时应逐个标注变量类型：本题 `$a` 是特殊字符串，`$b` 是数组，`$b[$a]` 才是哈希字符串。还要锁定运行版本；PHP 8 对非数字字符串与数字的比较规则已变化，不能把 PHP 7.3 的结论无条件迁移。
