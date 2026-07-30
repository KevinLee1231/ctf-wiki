# L3akCTF 2025 Useless VM Writeup

## 题目简述

题目只给出一个约 5 MB 的 `chal.js`。脚本先用 JSFuck 构造 JavaScript 源码，再叠加自定义虚拟机、RC4 和大量字节码数据。直接阅读完整文件既低效，也很难区分真正载荷与解释器噪声。

flag 最终以多段 Base58 文本藏在即将执行的源码注释中。决定性障碍是截获动态生成的 JavaScript，而不是完整反编译虚拟机，因此本文按 Reverse 归档。

## 解题过程

### 选择动态截获点

JSFuck 虽然只使用 `[]()!+` 等字符表示程序，但最终仍要把还原出的源码交给 JavaScript 引擎执行。题目路径会通过 `Function.prototype.apply` 调用动态构造的函数，因此可以在载荷执行前替换该方法，检查它收到的第一个参数：

```javascript
const originalApply = Function.prototype.apply;

Function.prototype.apply = function (thisArg, args) {
  inspectSource(args?.[0]);
  return originalApply.call(this, thisArg, args);
};
```

这个挂钩点位于 JSFuck 和自定义 VM 的输出端。无论前面堆叠多少编码层，只要最终仍依赖标准 JavaScript 引擎编译字符串，就能直接取得解码后的源码。

### 提取注释片段

观察截获的字符串，可以看到多段采用固定格式的注释：

```text
/*"第一段"*/
...
/*"第二段"*/
```

按源码出现顺序提取引号内文本并拼接：

```javascript
function inspectSource(source) {
  if (typeof source !== "string" || !source.includes("/*")) return;

  const parts = [...source.matchAll(/\/\*"(.*?)"\*\//g)];
  if (parts.length === 0) return;

  const encoded = parts.map(match => match[1]).join("");
  console.log(base58Decode(encoded).toString("utf8"));
}
```

片段不能分别解码。Base58 是把整段字节串视作一个大整数，拆开后每段的高位边界和前导零语义都会变化。

### Base58 解码

题目使用 Bitcoin Base58 字母表，其中排除了容易混淆的 `0`、`O`、`I` 和 `l`：

```javascript
function base58Decode(text) {
  const alphabet =
    "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";

  let value = 0n;
  for (const ch of text) {
    const digit = alphabet.indexOf(ch);
    if (digit < 0) throw new Error(`bad Base58 character: ${ch}`);
    value = value * 58n + BigInt(digit);
  }

  const bytes = [];
  while (value > 0n) {
    bytes.unshift(Number(value & 0xffn));
    value >>= 8n;
  }

  for (const ch of text) {
    if (ch !== "1") break;
    bytes.unshift(0);
  }
  return Buffer.from(bytes);
}
```

把挂钩代码放在原始挑战脚本之前运行，程序在进入后续聊天提示前就会打印：

```text
L3AK{jsfuck_is_easily_recoverable_it_doesn't_matter_how_much_you_layer_on_it}
```

之后出现的交互式 chatbot 与 flag 恢复无关；得到并验证上述字符串后即可结束进程。

## 方法总结

面对多层解释器和字符串混淆，最有效的切入点通常是最终的语义边界，而不是最外层文本。`eval`、`Function`、`Function.prototype.apply`、动态模块加载等位置必须接收已经还原的代码或数据，挂钩这些 API 可以跨过大量无关混淆。

动态截获后仍要理解载荷格式。本题把 Base58 串拆进多条注释，必须按源码顺序拼接后一次性解码，并正确处理字母表与前导 `1` 所代表的零字节。这样既解释了 flag 的来源，也避免把“直接运行官方脚本”当作不可复现的黑盒步骤。
