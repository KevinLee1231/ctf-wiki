# TSGCTF2020 Slick Logger WP

## 题目简述

应用把 Slack 导出数据按频道目录保存，公开频道 ID 以 `C` 开头，含 flag 的私有频道以 `G` 开头。搜索接口要求 `channel` 和 `q` 看起来像带双引号的字符串，再用 `strconv.Unquote` 解析：

```go
dir, _ := strconv.Unquote(channelID[0])
query, _ := strconv.Unquote(queries[0])

if strings.HasPrefix(dir, "G") { /* 拒绝私有频道 */ }
re, _ := regexp.Compile("(?i)" + query)
```

代码忽略 `Unquote` 和正则编译错误。传入外观满足验证、实际却不是合法 Go 引号字符串的 `channel="a"a"` 时，`Unquote` 返回空字符串；目录前缀检查被绕过，而文件遍历前缀变成 `/var/lib/data/`，于是搜索覆盖公开和私有频道。

## 解题过程

搜索范围已经包含私有数据，题目预期接着使用 Blind Regex Injection。运行在 Apache CGI 下的 Go 程序有 1 秒限制：

```apache
ScriptAliasMatch /api/.* /usr/local/apache2/cgi-bin/index.cgi
CGIDScriptTimeout 1s
```

Go 的 regexp 使用 RE2 风格的非回溯引擎，经典灾难性回溯表达式不起作用。不过其运行量仍与自动机活跃状态数有关。单个重复上限为 1000，但可以串联大量 `(.?){1000}`，制造极多并行状态：

```text
^<待验证前缀>(.?){1000}(.?){1000}...(重复约200次)...z$
```

末尾的 `z$` 确保整个表达式最终失败。若消息不满足待验证前缀，RE2 很早就淘汰该路径，CGI 正常返回 200；若前缀匹配包含 flag 的私有消息，引擎必须在长串可选重复中维护大量状态，超过 1 秒后 Apache 返回 504。因此状态码构成前缀 oracle。

请求参数需要保留外层引号，`channel` 刻意使用无效字符串触发空目录，`q` 则是可成功 `Unquote` 的完整正则：

```javascript
const heavy = '(.?){1000}'.repeat(200);
const params = {
  channel: '"a"a"',
  q: `"^${prefix}${candidate}${heavy}z$"`,
};
```

从已知消息前缀 `Reminder: The flag is ` 开始，依次尝试格式允许的字符。收到 504 就接受当前字符并进入下一位，否则继续枚举；正则中的 `{`、`}` 需按 Go 字符串和正则两层语法转义。最终得到：

```text
TSGCTF{Y0URETH3W1NNNER202OH}
```

## 方法总结

利用链包含两个独立缺陷：忽略 `strconv.Unquote` 错误把非法频道参数降级为空字符串，导致目录前缀覆盖全部数据；可控正则再利用 CGI 超时形成盲注 oracle。RE2 能避免指数级回溯，并不意味着攻击者无法构造昂贵表达式，重复的可选状态仍可放大线性常数。修复时必须处理所有解析错误、对频道 ID 做严格白名单和真实路径边界检查，并对用户正则设置复杂度、长度和执行预算，不能用网关超时充当唯一保护。
