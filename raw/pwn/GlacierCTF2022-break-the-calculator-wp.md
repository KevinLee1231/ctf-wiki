# GlacierCTF2022 - Break the Calculator

## 题目简述

Node.js 计算器去掉空白后，只允许不含英文字母的表达式，再把输入拼进 `Function('return ' + parsedFormula)()` 执行。目标是在字符过滤下恢复 JavaScript 代码执行并读取 `/app/flag.txt`。

## 解题过程

过滤条件仅排除了字母：

```javascript
if (parsedFormula.match(/^[^a-zA-Z\s]*$/)) {
    const result = Function("return " + parsedFormula)();
}
```

JavaScript 的隐式类型转换允许只用 `[]()!+` 构造布尔值、数字和字符串。例如 `![]` 是 `false`，`!![]` 是 `true`，`[]+{}` 等表达式可产生带字母的内建字符串，再按索引拼出属性名。继续沿数组方法的 `constructor` 取得 `Function` 构造器，就能执行任意生成出的源码，这就是 JSFuck 的核心。

需要编码的第二阶段源码为：

```javascript
console.log(
  process.mainModule.require("fs").readFileSync("/app/flag.txt", "utf-8")
)
```

将它转换为只含 `[]()!+` 的完整表达式并提交。仓库中的官方 payload 长约 20 KiB，所有字符都通过正则；`Function` 执行后由 Node 的 `process.mainModule.require` 取得 `fs`，读取并打印：

```text
glacierctf{JaVa$cR!pT_!S_a_Gr3At_Es0t3r!c_LaNgUaG3}
```

## 方法总结

字符白名单不是 JavaScript 沙箱。动态求值、隐式转换、原型链和构造器共同提供了从少量标点恢复任意源码的路径。若业务只需要算术表达式，应解析为受限 AST 并只解释明确允许的节点，不能把过滤后的字符串交给 `Function` 或 `eval`。
