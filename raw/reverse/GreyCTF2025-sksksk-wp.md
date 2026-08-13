# sksksk

## 题目简述

题目给出一份没有输入、也没有任何输出语句的 JavaScript。程序只定义了两个基础组合子：

```javascript
const s = x => y => z => x(z)(y(z));
const k = x => y => x;
```

其余 `a` 到 `l` 以及最终的 `flag` 都由 `S`、`K` 的函数应用组合而成。核心任务是识别其中的 Church 编码，并在不手工化简整条巨大组合子表达式的情况下恢复字符序列。

## 解题过程

Church 数字 $n$ 的行为是把函数 $f$ 应用 $n$ 次：

$$
n\ f\ x=f^n(x).
$$

因此只要拿到疑似 Church 数字 `value`，执行

```javascript
value(x => x + 1)(0)
```

就能把它转回普通整数。观察最终表达式可见，`l(...)` 在每个嵌套层级接收一个由前述组合子构造的数值；它正好是每个字符的稳定边界。与其完全求出 `l` 和整条列表的闭式表达式，不如在构造 `flag` 之前包装 `l`：记录参数对应的 Church 数字，然后仍调用原函数，保证后续组合保持原语义。

下面的 Node.js 脚本会在内存中插入这一观察点，不改动原附件：

```javascript
const fs = require("fs");

let source = fs.readFileSync("sksksk.js", "utf8");
source = source
  .replace("const l =", "let l =")
  .replace(
    "const flag =",
    `const originalL = l;
const codes = [];
l = value => {
  codes.push(value(x => x + 1)(0));
  return originalL(value);
};
const flag =`,
  );

eval(source + `
console.log(codes);
console.log(String.fromCharCode(...codes));
`);
```

运行后得到 32 个字符码：

```text
103 114 101 121 123 115 107 115
107 95 55 117 114 49 110 103
95 99 48 109 112 49 51 55
51 63 95 115 107 115 107 125
```

按 ASCII 转换即为：

```text
grey{sksk_7ur1ng_c0mp1373?_sksk}
```

## 方法总结

SK 组合子具有完备表达能力，但这不意味着解题时必须把整个表达式逐项归约。先寻找重复出现的语义边界，再用符合 Church 定义的探针观察参数，能在保留原程序执行路径的同时直接恢复数据。这里的关键不是函数名 `l`，而是它在最终嵌套结构中恰好每个字符调用一次；若换一份样本，应先验证调用次数与编码形态，不能机械套用函数名。
