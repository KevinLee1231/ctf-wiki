# GreyCTF Survey

## 题目简述

投票接口要求 `vote` 是 JavaScript number 且位于 $(-1,1)$，却又对它调用 `parseInt` 后累加。对于足够小的浮点数，JavaScript 转字符串会使用科学计数法；`parseInt` 只解析指数符号前的整数部分，于是小于 1 的输入可变成大整数。

## 解题过程

后端逻辑为：

```javascript
if (typeof vote === "number" && vote < 1 && vote > -1) {
    score += parseInt(vote);
    if (score > 1) return flag;
}
```

提交 JSON 数字 `0.00000005`。它通过数值范围检查，但在 `parseInt` 内部先被转换为字符串：

```javascript
String(0.00000005)  // "5e-8"
parseInt(0.00000005) // 5
```

`parseInt` 从开头读到非数字字符 `e` 为止，因此结果是 5，而不是 0。初始分数为 `-0.42069`，一次请求后变成 $4.57931>1$：

```json
{"vote": 0.00000005}
```

接口随即返回：

```text
grey{50m371m35_4_l177l3_6035_4_l0n6_w4y}
```

## 方法总结

`parseInt` 是字符串解析函数，不适合把已经是 number 的值“转成整数”。数值代码应先明确需要截断、向下取整还是四舍五入，并使用 `Math.trunc`、`Math.floor` 或 `Math.round`。任何隐式的 number-to-string 转换都要考虑科学计数法、`NaN` 和无穷值。
