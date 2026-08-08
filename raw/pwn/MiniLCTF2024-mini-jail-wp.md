# miniLCTF 2024 mini-jail Writeup

## 题目简述

服务把一行不超过 120 字符的输入交给保存下来的原始 `eval`，但把 `eval`、`process`、`File` 和正常 `console.log` 从常用全局名中移除。输入还会经过 `/flag|write|read|fs|proc/ig` 黑名单。目标是在 Node.js 执行边界中绕过源码级过滤，从环境变量取得 flag。

本题的决定性障碍是 JavaScript jail 逃逸及受限输出，不是 Web 请求漏洞，因此归入 Pwn。

## 解题过程

程序没有创建 `vm` 隔离上下文，只是修改了几个全局变量；保存下来的原始 `eval` 仍会在完整 Node.js 进程中执行输入。ES 动态导入也没有被禁用，所以可以导入内置文件系统模块。

黑名单检查的是输入源码，不会先解释 JavaScript 转义。把敏感单词的一部分写成十六进制转义即可让正则看不到这些连续字符，而运行时字符串仍会恢复原值：

```text
fs       -> \x66s
readFile -> \x72eadFile
proc     -> \x70roc
write    -> \x77rite
```

原 `console.log` 被替换为只报告字符串长度的包装器，直接打印环境变量没有用。但 Node 服务的标准输出就是连接到客户端的文件描述符 1，可以调用 `fs.write(1, data, callback)` 绕过包装器。

最终 payload 正是仓库随附的 `solve.js`：

```javascript
import('\x66s').then(f=>{f["\x72eadFile"]('/\x70roc/self/environ','utf8',(_,d)=>{f["\x77rite"](1,d,()=>{})})})
```

它的长度小于 120，源码中不含任一黑名单单词。执行顺序为：

1. 动态导入 `fs`；
2. 读取 `/proc/self/environ`；
3. 将环境块直接写入 fd 1；
4. 在以 `\0` 分隔的输出中查找 `FLAG=`。

也可以导入 `process` 并访问 `env`，或枚举 `global` 找回超长名字保存的原始 `console.log`；但前述 payload 更短，且不依赖全局属性枚举顺序。

## 方法总结

黑名单只匹配源代码字面量，却允许字符串转义和动态模块加载，因而不是安全边界；替换 `console.log` 也没有隔离底层标准输出。JavaScript jail 应使用真正的进程/权限隔离和严格能力白名单，而不能仅删除全局名字或正则过滤关键词。
