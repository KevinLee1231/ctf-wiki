# Al's Advanced ATMs

## 题目简述

网站首页只有一个始终返回 “Invalid credentials” 的 React 登录表单。认证并未发送到后端，真正的数据表和隐藏路由都被打包进前端 JavaScript，因此应检查构建产物而不是继续猜口令。

## 解题过程

在 `static/js/main.*.chunk.js` 中搜索 `Super Secret Database`，可以定位到 React 路由：

```text
/eonline
```

该组件列出多组用户名和 Base64 密码。`umd-csec` 对应的值为：

```text
VU1EQ1RGLXtvcDNuX2RAdEBfYjY0X2JAZH0=
```

解码：

```python
import base64

value = b"VU1EQ1RGLXtvcDNuX2RAdEBfYjY0X2JAZH0="
print(base64.b64decode(value).decode())
```

得到：

```text
UMDCTF-{op3n_d@t@_b64_b@d}
```

## 方法总结

纯前端应用无法保守秘密。即使路由没有出现在页面导航中，组件、路径和数据仍会随 JavaScript bundle 下载到浏览器；Base64 也只是表示编码，不构成访问控制或加密。
