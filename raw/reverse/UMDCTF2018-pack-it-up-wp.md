# UMDCTF 2018 - Pack It Up

## 题目简述

附件 `packed.js` 使用典型 JavaScript Packer 风格的 `eval(function(...){...})` 包装真实代码。题目的决定性步骤是静态解包脚本，而不是访问某个在线站点。

## 解题过程

外层函数使用词表替换编号标记，最终把还原出的字符串交给 `eval`。分析时无需真的执行未知脚本，可以把最外层的 `eval` 改为输出参数，或按词表手工替换。混淆数据中可见：

```text
alert|UMDCTF|p|cK_1t_uP_B0iz
```

解包后的有效语句是：

```javascript
alert("UMDCTF-{p@cK_1t_uP_B0iz}")
```

因此 flag 为：

```text
UMDCTF-{p@cK_1t_uP_B0iz}
```

README 中保存的是 flag 加换行符后的 SHA-256：

```text
b353091cb0371f3db8a9b51bb597777afe72131ea11140342179ba950388e55d
```

## 方法总结

处理未知 JavaScript 时，应阻断 `eval` 并观察它准备执行的字符串，避免直接运行潜在副作用代码。Packer 的变量名和控制流虽然冗长，真正载荷通常仍由词表和短模板组合而成。
