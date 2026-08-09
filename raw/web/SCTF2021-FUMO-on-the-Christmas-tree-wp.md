# FUMO on the Christmas tree

## 题目简述

题目为每个来源地址动态生成一份约 16 万行的 PHP 类集合，并在入口执行：

```php
include "class.php";
@unserialize($_GET["data"]);
```

生成器先构造一条长度为 20 的可达主链，再不断挂接更短的分支，形成近似二叉树；死分支会对输入执行不可逆变换，部分叶子还会重新指向主链形成环。唯一可信的入口是某个类的 `__destruct`，有效终点会判断输入是否以 `/fumo` 开头并调用 `readfile(strtolower($input))`。因此主障碍不是手工寻找普通 POP 链，而是自动恢复对象调用图、筛出没有被不可逆变换破坏的路径，并构造对应的序列化对象图。

## 解题过程

先下载当前实例公开的 `class.code`。它与正在执行的类定义对应，实例刷新后类名和字段名会变化，所以分析和利用必须针对同一份代码完成。

每个生成类只有一个公共方法。将方法归一化后，边只有三种形式：

```php
// 普通调用
$this->nextObject->nextMethod($value);

// 进入 __invoke
call_user_func($this->nextObject, ['fixedKey' => $value]);

// 经 __call 转发
if (is_callable([$this->nextObject, $method]))
    call_user_func([$this->nextObject, $method], ...$value);
```

对普通方法建立“方法名到类名”的索引；对 `__invoke` 记录其 `base64_decode` 后使用的固定键；对 `__call` 则从 `extract([$name => 'nextMethod'])` 和后续动态调用中恢复真实下一方法。这样便能把随机名称还原成有向图的边。起点由 `__destruct` 唯一确定，终点由包含 `/fumo` 和 `readfile` 的方法确定。

同时给每条边标注其数据变换。以下操作会丢失原值，应把所在分支视为死链：

```text
md5、sha1、crypt、把输入替换成随机常量
```

以下操作可逆，可以在找到路径后反向补偿：

```text
str_rot13  <-> str_rot13
strrev     <-> strrev
base64_decode 的逆操作是 base64_encode
base64_encode 的逆操作是 base64_decode
```

`ucfirst` 需要结合相邻操作判断：它本身不完全可逆，而且位于 `base64_decode` 前时可能破坏编码首字符。为了避免把边界情况误判为活链，可以直接运行经过插桩的副本，而不只做正则静态推断。具体做法是：

1. 将 `readfile` 替换为记录 `debug_backtrace()` 的函数，保存到达终点的类序列。
2. 将可逆变换替换为记录当前类和对应逆函数的 hook。
3. 为每个类增加一次性访问标记，遇到回边时立即返回，防止分析副本死循环。
4. 根据调用栈从叶到根创建对象，把每个对象的下一跳属性设为前一个对象；无关属性用普通占位对象填充。
5. 从目标值 `/fumo` 逆序执行记录到的逆函数，得到入口查询参数的正确值。

对象链的构造逻辑可以概括为：

```php
$last = new stdClass();
foreach ($pathFromSinkToRoot as $node) {
    $className = $node['class'];
    $current = new $className();
    foreach ($node['fields'] as $field) {
        $current->$field = $last;
    }
    $last = $current;
}

$payload = urlencode(serialize($last));
```

最终向当前实例的 `index.php` 同时传入序列化数据和生成器随机选定的输入参数名。反序列化结束时根对象的 `__destruct` 启动链，输入沿着恢复出的活链到达读取函数，页面返回：

```text
SCTF{U_aRe_th3_T_m@stEr_0f_FUMO}
```

## 方法总结

这道题考察的是面向大规模随机代码的自动化 POP 链分析。可靠方法不是逐行阅读，而是先提取稳定语法模板，再分别恢复普通调用、`__invoke` 和 `__call` 三类边，最后把数据变换作为边属性参与可达性判断。

静态图搜索负责缩小候选范围，运行插桩负责处理环、动态调用与变换顺序。只有同时满足“从唯一析构入口可达终点”和“沿途变换可逆”两项条件，路径才是真正可利用的活链。生成结果与来源地址绑定且会刷新，因此还要保证分析文件、序列化类名和发送目标属于同一实例。
