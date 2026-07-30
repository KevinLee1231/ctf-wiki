# SU_HappyAST

## 题目简述

附件包含经过重度 AST 混淆的 `HappyAST` 和一个导出 `BinGbing()` 的辅助文件 `reversethefl4g.js`。程序会把 `BinGbing()` 返回的字符串加密并输出十六进制密文，但题目要求解密另一段给定密文：

```text
38e207c3d0e0d6342f4af654205b6710e4e8306ce7eb032116927fae56f722a484fe6ca66c159e4212ab65ef99d4c1b0
```

表面上看，主要困难是找出加密算法；实际上程序还在 AST 中插入了大量有意义但不参与主加密流程的节点。完整解法需要同时完成两件事：

1. 去除控制流与表达式层面的 JavaScript 混淆，恢复被改造的 AES-CBC 加密逻辑；
2. 从插入的 `Bing?Bing` 相关 AST 节点中恢复 flag 前缀。

虽然末尾存在 SM3 和 AES 运算，但这些算法的参数在混淆代码中并不直接可见，决定性障碍是程序行为还原，因此本题归入 Reverse。

## 解题过程

先查看未混淆的辅助文件，可以得到一组已知明文：

```javascript
function BinGbing() {
    return "thisistest";
}

module.exports = { BinGbing };
```

执行原程序时，这个字符串会被加密为：

```text
68d2b55f15cc6d41ca060ce3d9fc6ca1
```

这组明密文对可以用来验证后续还原出的算法是否正确。若解密器不能把该密文还原成 `thisistest`，就不能直接拿它处理题目密文。

题目使用的是在常见 `jsobf` 思路上继续修改的混淆器。除了字符串数组、成员访问和控制流改写外，它还向更深层的语法树位置插入无用的 `Literal`、`UnaryExpression` 和 `Identifier` 节点。适合使用 Babel AST、restringer 一类工具逐轮化简：

1. 计算可以静态求值的常量表达式；
2. 将计算属性访问还原为普通属性访问；
3. 展开字符串数组和包装函数；
4. 删除结果未被使用且没有副作用的表达式；
5. 反复生成代码并重新解析，直到代码规模不再明显缩小。

对插入节点不能一概删除。题目把 flag 前缀藏在形如 `Bing?Bing` 的节点中，先提取再清理更稳妥。下面给出一个用于定位候选节点的 Babel 遍历框架：

```javascript
const fs = require("fs");
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const source = fs.readFileSync("HappyAST", "utf8");
const ast = parser.parse(source);
const candidates = new Set();

traverse(ast, {
    StringLiteral(path) {
        if (/^Bing.Bing$/.test(path.node.value)) {
            candidates.add(path.node.value);
        }
    },
    Identifier(path) {
        if (/^Bing.Bing$/.test(path.node.name)) {
            candidates.add(path.node.name);
        }
    }
});

console.log([...candidates].sort());
```

不同节点类型还可能把字符编码在运算结果、属性名或标识符变体里，所以实际处理时需要同时记录节点类型、父节点和静态求值结果，不能只保留字符串字面量。官方解法对这些候选节点求值并去重后得到八个字符：

```text
S U C T F { H i
```

由标准 flag 头可确定这一部分的顺序为：

```text
SUCTF{Hi
```

继续从去混淆结果的末尾向上追踪数据流，可以看到程序先计算：

```javascript
SM3("bingbing").slice(0, 16)
```

结果是：

```text
50aca6ed2feffa0c
```

该 16 字符字符串同时作为 AES 的 key 和 IV。原代码中虽然短暂出现过字符串 `0123456789abcdef`，但随后会被上述派生值覆盖，不能把前一个赋值误当成最终 IV。

AES 部分来自 JavaScript 实现 `aes-js`，但题目调整了密钥扩展使用的 Rcon 值顺序。因此不能直接调用标准 AES-CBC 库；必须从去混淆后的程序中保留其密钥扩展、分组解密和 CBC 异或逻辑，再把入口改成解密。正确流程是：

```javascript
const key = "50aca6ed2feffa0c";
const iv = "50aca6ed2feffa0c";

// modifiedAesCbcDecrypt 必须保留题目中的 Rcon 顺序，
// 输入和输出分别按原程序的十六进制、字节串约定转换。
const test = modifiedAesCbcDecrypt(
    "68d2b55f15cc6d41ca060ce3d9fc6ca1",
    key,
    iv
);
console.log(test); // thisistest

const suffix = modifiedAesCbcDecrypt(
    "38e207c3d0e0d6342f4af654205b6710e4e8306ce7eb032116927fae56f722a484fe6ca66c159e4212ab65ef99d4c1b0",
    key,
    iv
);
console.log(suffix);
```

通过已知明密文对后，题目密文解出：

```text
_H4PpY_AsT_TTTTTTBing_BinG_Bing}
```

与 AST 节点中恢复的前缀拼接，得到最终 flag：

```text
SUCTF{Hi_H4PpY_AsT_TTTTTTBing_BinG_Bing}
```

## 方法总结

本题把有效信息拆成了两层：主程序尾部的改造 AES 用于恢复 flag 后半段，散布在 AST 中的特殊节点用于恢复前缀。只做常规去混淆并追踪加密函数，会缺少 `SUCTF{Hi`；只搜索 `Bing?Bing`，又无法得到剩余内容。

分析这类 AST 混淆程序时，应先保留可疑节点的类型、位置和求值结果，再执行无副作用代码删除。识别出熟悉的密码实现后，也不能仅凭算法名称替换成现成库，必须对照常量表和密钥扩展逻辑。本题最可靠的校验点是附件自带的 `thisistest` 明密文对：它可以同时发现 Rcon 顺序、字符编码、填充和 CBC 参数处理是否还原错误。
