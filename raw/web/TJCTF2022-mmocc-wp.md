# TJCTF2022 mmocc

## 题目简述

Node.js 服务把未匹配路由当作静态文件路径，并试图递归移除所有 `../`。清理函数复用了带全局标志 `g` 的正则表达式；`RegExp.test` 会保存 `lastIndex`，下一轮递归从旧偏移继续搜索，可能漏掉清理后新形成的穿越片段。

## 解题过程

清理逻辑的核心为：

```javascript
const regex = /\.\.\//g;
const clean = (p) => {
    const replaced = p.replace('../', '');
    if (regex.test(p)) return clean(replaced);
    return replaced;
};
```

提交路径 `/..../././flag.txt` 时，第一次 `replace` 得到 `/..././flag.txt`，全局正则匹配后把 `lastIndex` 留在 6。第二轮替换得到 `/../flag.txt`，但 `test` 从旧偏移 6 开始，漏掉开头的 `../` 并直接返回。随后：

```javascript
path.join(__dirname, 'static', '/../flag.txt')
```

规范化到应用目录中的 `flag.txt`。客户端还必须禁止自动折叠路径，例如：

```bash
curl --path-as-is 'http://target/..../././flag.txt'
```

响应为 `tjctf{h0w_h1gh_c4n_w3_g3t}`。

## 方法总结

路径清理不能依赖字符串替换，更不应在递归中复用有状态的全局正则。攻击重点是观察每轮替换后是否会生成新的危险片段，以及客户端、代理和服务端各自何时做路径规范化。安全做法是解析并规范化最终绝对路径，再验证它仍位于允许的静态根目录内。
