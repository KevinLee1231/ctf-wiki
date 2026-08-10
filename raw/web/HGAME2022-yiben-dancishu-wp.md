# 一本单词书

## 题目简述

题目由两段漏洞组成：登录处使用 PHP 7.4 的弱类型比较，登录后又把用户提交的键值对编码为自定义 Session 格式。编码函数没有保护键名中的分隔符 `|`，攻击者可以借此把任意序列化对象注入解码流程，并触发 `Evil::__wakeup()` 读取 `/flag`。

## 解题过程

源码给出了用户名 `adm1n`，密码比较使用 `==` 而不是严格比较。在 PHP 7.4 的旧式数字字符串规则下，可以提交以 `1080` 开头、但类型或尾部字符不同的值，例如：

```text
username=adm1n
password=1080a
```

通过登录后，继续审计 Session 的编码与解码逻辑。核心行为可以简化为：

```php
function encode($data): string {
    $result = '';
    foreach ($data as $key => $value) {
        $result .= $key . '|' . serialize($value);
    }
    return $result;
}

function decode(string $data): array {
    // 读取“键名|序列化值”，并对 | 后面的内容调用 unserialize()
}
```

正常数据 `{"a":"1","b":"s"}` 会被编码为：

```text
a|s:1:"1";b|s:1:"s";
```

问题在于 `encode()` 原样拼接键名。只要把 `|` 和序列化对象放进键名，`decode()` 就会把攻击者提供的对象当作值传给 `unserialize()`。题目已经提供 `Evil` 类；令其 `file` 属性指向 `/flag`，可使用官方题解中的 JSON：

```json
{"a|O:4:\"Evil\":2:{s:4:\"file\";s:5:\"/flag\";s:4:\"flag\";N;};b":2}
```

编码后的关键片段如下：

```text
a|O:4:"Evil":2:{s:4:"file";s:5:"/flag";s:4:"flag";N;};b|i:2;
```

解码器先把 `a` 识别为键，再从 `O:4:"Evil"...` 开始反序列化，于是实例化 `Evil` 并调用 `__wakeup()`。该方法读取 `file` 指向的文件。虽然源码在检测到 `hgame` 后曾把属性设为 `hacker!`，但缺少 `return`，随后仍会用文件原文覆盖该属性，因此响应最终泄露 flag。

这套 `encode()`/`decode()` 逻辑源自 imi 框架的 Session 编码思路；此题已经直接给出可利用类，所以决定性步骤是分隔符注入，而不是寻找完整 POP 链。公开参赛题解还给出了以 `|O:4:"Evil"...` 开头的等价构造，可作为输入格式不同时的调整参考：[HGAME 2022 Week2 writeup](https://pankas.top/2022/02/03/week2-pankas/)。

## 方法总结

自定义序列化协议必须对键名、分隔符和长度进行无歧义编码；把未转义的用户输入直接拼进 `键|serialize(值)` 会破坏协议边界。PHP 中还应使用 `===`、`hash_equals()` 等符合语义的严格比较，并避免对不可信数据调用 `unserialize()`。即使暂时没有危险类，代码或依赖更新也可能引入可利用的魔术方法。
